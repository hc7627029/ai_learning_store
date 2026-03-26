---
name: te-data-upload
description: 通过 RESTful API 向 TE（数数科技/ThinkingEngine）系统批量上报 JSONL 格式的用户属性（user_set）和事件（track）数据。当用户提到"上报数据到TE"、"TE数据上传"、"ThinkingEngine上报"、"数数科技数据上报"、"sync_server"、"user_set上报"、"track事件上报"、"数据入库"等关键词时触发。即使用户只说"把数据传到TE"或"帮我导入TE"也应触发。
---

# TE 数据上报 Skill

通过 TE RESTful API（`/sync_server` 接口）批量上报 JSONL 数据。涵盖完整的上报流程、数据格式要求和错误排查方法。

## 上报前：收集必要信息

开始前需要确认以下信息，缺一不可：

| 信息 | 示例 | 获取方式 |
|------|------|----------|
| 接收地址 | `http://xxxx/sync_server` | 用户提供或从 TE 后台获取 |
| APP_ID | `xxxxx` | TE 后台「项目管理」页面 |
| JSONL 文件路径 | `/path/to/data.jsonl` | 用户提供 |

如果用户未提供，主动询问。

## 接口规范（核心，务必严格遵守）

TE 的 `/sync_server` 接口对请求格式有严格要求，格式错误时接口可能返回成功（code:0）但数据实际无法入库，排查成本很高。

### 正确格式

```
POST http://<host>:<port>/sync_server
Header:
  Content-Type: application/json
  appid: <APP_ID>
Body:
  [{record1}, {record2}, ...]
```

三个关键点：
1. **appid 放 Header**，不是 body — 放 body 里会返回 `code: -2 Invalid AppID`
2. **Body 是 JSON 数组**，不是 `{"data": [...]}`、不是 `{"appid":"...", "data": [...]}` — 包裹对象会返回 `code: -1`
3. **URL 直接用 `/sync_server`**，不要拼接 `/user_set` 或 `/track` — 数据类型由每条记录的 `#type` 字段决定

### 响应码

| code | 含义 | 排查方向 |
|------|------|----------|
| 0 | 接收成功 | 注意：不代表一定入库，仍需检查数据格式 |
| -1 | 数据格式错误 | 检查 body 是否为 JSON 数组 |
| -2 | Invalid AppID | appid 是否在 Header 中、值是否正确 |

## JSONL 数据格式

### 常见文件结构

JSONL 文件每行是一个 JSON 对象，通常有两种格式：

**格式 A：带 appid 包裹（需提取 data 字段）**
```json
{"appid": "xxx", "data": {"#account_id": "u001", "#type": "user_set", "#time": "2024-01-22 05:40:50.567", ...}}
```
上报时提取 `data` 字段：`json.loads(line)["data"]`

**格式 B：裸数据（直接使用）**
```json
{"#account_id": "u001", "#type": "user_set", "#time": "2024-01-22 05:40:50.567", "properties": {...}}
```

读取文件时先看前几行判断格式，再决定是否需要提取 `data` 字段。

### 关键字段

| 字段 | 必须 | 说明 |
|------|------|------|
| `#account_id` | 是 | 用户账号 ID |
| `#type` | 是 | `user_set`（用户属性）或 `track`（事件） |
| `#time` | 是 | **格式必须为 `YYYY-MM-DD HH:MM:SS.mmm`（毫秒精度）**，格式不合法会导致接口返回成功但数据无法入库 |
| `#distinct_id` | 建议 | 设备/访客 ID |
| `#uuid` | 建议 | 去重 ID，防止重复入库 |
| `#event_name` | track 必须 | 事件名称，仅 track 类型需要 |
| `properties` | 是 | 属性键值对 |

`#time` 是最容易踩坑的字段 — 缺失或格式不对都会导致"接口成功但不入库"的静默失败。

## 批量上报策略

数据量可能从几百到几十万条不等，需要分层控制：

- **BATCH_SIZE = 100**：每次 HTTP 请求发送的记录数
- **ROUND_SIZE = 10000**：每轮上报的记录上限（用户可能要求每轮不超过一定量）
- **轮次间隔 2 秒**：避免瞬时压力过大
- **失败重试 3 次**：每次间隔 2 秒

上报脚本模板见下方，可直接执行。
```python
#!/usr/bin/env python3
"""
TE 数据上报脚本（通用）
支持 user_set 和 track 类型，自动识别 JSONL 文件格式。

用法:
    python upload_te_data.py --url http://<receiver>/sync_server --app-id <APP_ID> --file <path.jsonl>

参数:
    --url         TE 数据接收地址（必填）
    --app-id      项目 APP_ID（必填）
    --file        JSONL 数据文件路径（必填）
    --batch-size  每次 HTTP 请求条数（默认 100）
    --round-size  每轮上报上限（默认 10000，0 表示不限）
    --dry-run     仅验证格式不实际上报
"""

import json
import sys
import time
import argparse
import requests

RETRY_COUNT = 3
RETRY_DELAY = 2
ROUND_DELAY = 2


def load_records(file_path: str) -> list[dict]:
    """读取 JSONL 文件，自动判断是否需要提取 data 字段。"""
    records = []
    has_wrapper = None

    with open(file_path, encoding="utf-8") as f:
        for lineno, line in enumerate(f, 1):
            line = line.strip()
            if not line:
                continue
            try:
                obj = json.loads(line)
            except json.JSONDecodeError as e:
                print(f"[WARN] 第 {lineno} 行 JSON 解析失败，已跳过: {e}")
                continue

            # 首行判断格式
            if has_wrapper is None:
                has_wrapper = "data" in obj and "appid" in obj
                fmt = "带 appid 包裹（提取 data 字段）" if has_wrapper else "裸数据（直接使用）"
                print(f"文件格式: {fmt}")

            records.append(obj["data"] if has_wrapper else obj)

    return records


def upload_batch(session, url, headers, batch, batch_num, round_num):
    """上报一批数据，带重试。"""
    for attempt in range(1, RETRY_COUNT + 1):
        try:
            resp = session.post(url, data=json.dumps(batch), headers=headers, timeout=30)
            code = resp.json().get("code")
            if code == 0:
                return len(batch), None
            msg = f"HTTP {resp.status_code}, 响应={resp.text[:200]}"
        except requests.RequestException as e:
            msg = str(e)

        if attempt < RETRY_COUNT:
            time.sleep(RETRY_DELAY)

    return 0, f"第{round_num}轮 批次{batch_num}: {msg}"


def main():
    parser = argparse.ArgumentParser(description="TE 数据上报脚本")
    parser.add_argument("--url", required=True, help="TE 数据接收地址")
    parser.add_argument("--app-id", required=True, help="项目 APP_ID")
    parser.add_argument("--file", required=True, help="JSONL 数据文件路径")
    parser.add_argument("--batch-size", type=int, default=100, help="每次 HTTP 请求条数")
    parser.add_argument("--round-size", type=int, default=10000, help="每轮上报上限，0=不限")
    parser.add_argument("--dry-run", action="store_true", help="仅验证格式不实际上报")
    args = parser.parse_args()

    records = load_records(args.file)
    total = len(records)
    if total == 0:
        print("没有读取到有效记录")
        sys.exit(1)

    # 统计数据类型
    types = {}
    for r in records:
        t = r.get("#type", "unknown")
        types[t] = types.get(t, 0) + 1
    type_summary = ", ".join(f"{t}: {c}条" for t, c in types.items())

    print(f"共读取 {total} 条记录 ({type_summary})")
    print(f"接收地址: {args.url}")
    print(f"APP_ID  : {args.app_id}")
    print(f"批次大小: {args.batch_size}, 每轮上限: {args.round_size or '不限'}")

    if args.dry_run:
        print(f"\n[Dry-run] 数据预览（前 2 条）:")
        for r in records[:2]:
            print(json.dumps(r, ensure_ascii=False, indent=2))
        print("\n[Dry-run] 模式，不实际上报。")
        return

    headers = {"Content-Type": "application/json", "appid": args.app_id}
    session = requests.Session()

    round_size = args.round_size if args.round_size > 0 else total
    total_success = 0
    all_failures = []
    round_num = 1

    for start in range(0, total, round_size):
        end = min(start + round_size, total)
        chunk = records[start:end]
        print(f"\n=== 第 {round_num} 轮: 第 {start+1}~{end} 条 ===")

        round_success = 0
        for i in range(0, len(chunk), args.batch_size):
            batch = chunk[i:i + args.batch_size]
            batch_num = i // args.batch_size + 1
            ok, err = upload_batch(session, args.url, headers, batch, batch_num, round_num)
            round_success += ok
            if err:
                all_failures.append(err)
                print(f"  [FAIL] {err}")

        total_success += round_success
        print(f"第 {round_num} 轮完成: 成功 {round_success}/{len(chunk)} 条")
        round_num += 1

        if end < total:
            print(f"间隔 {ROUND_DELAY} 秒后继续...")
            time.sleep(ROUND_DELAY)

    print(f"\n{'='*50}")
    print(f"上报完成: 成功 {total_success}/{total} 条")
    if all_failures:
        print(f"失败 {len(all_failures)} 个批次:")
        for f in all_failures:
            print(f"  - {f}")
        sys.exit(1)
    else:
        print("全部上报成功！")


if __name__ == "__main__":
    main()
```

## 上报流程

### Step 1: 读取并预览数据

```python
# 先看前几行，判断格式
with open(file_path) as f:
    first_line = json.loads(f.readline())

# 判断是否需要提取 data 字段
has_wrapper = "data" in first_line and "appid" in first_line
```

### Step 2: Dry-run 验证

上报前先用 1-3 条数据测试接口连通性和格式正确性，确认返回 `code: 0`。

### Step 3: 分批上报

按 BATCH_SIZE 和 ROUND_SIZE 分批发送，打印每批结果，汇总成功/失败数。

### Step 4: 确认入库

提醒用户去 TE 后台确认：
- 用户属性 → 「用户分析 → 用户属性」
- 事件数据 → 「事件分析」
- 注意筛选时间范围要覆盖数据的 `#time`

## 错误排查清单

### 接口返回成功（code:0）但后台看不到数据

这是最常见也最难排查的情况，按以下顺序逐一检查：

1. **`#time` 格式不合法** — 必须精确到毫秒 `YYYY-MM-DD HH:MM:SS.mmm`，缺毫秒、用 ISO 格式、用时间戳都可能导致静默丢弃
2. **后台开启了「拦截上报」** — 在 TE 数据接收设置中检查
3. **查看位置不对** — user_set 在「用户属性」页面，track 在「事件分析」页面
4. **后台筛选的时间范围没覆盖数据** — 数据的 `#time` 可能在非默认范围内
5. **入库延迟** — 私有化部署可能有 5~30 分钟延迟

### 接口返回错误

| 现象 | 原因 | 修复 |
|------|------|------|
| `code: -2, Invalid AppID` | appid 放在了 body 里 | 移到 HTTP Header |
| `code: -1` | body 格式错误 | 确保 body 是 `[{...}]` 数组，不是 `{"data":[...]}` |
| 连接超时 | 网络不通或地址错误 | 检查接收地址和端口 |

## 注意事项

- **幂等性**：user_set 是幂等的，重复上报只会覆盖属性值，不会产生重复记录
- **track 去重**：如果数据包含 `#uuid`，TE 会自动去重，重复上报不会产生重复事件
- **删除数据**：TE RESTful API 不支持删除已入库数据，需在 TE 后台操作或联系管理员
- **依赖**：需要 Python `requests` 库，缺失时 `pip install requests`

