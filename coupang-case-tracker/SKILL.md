---
name: coupang-case-tracker
description: >
  Coupang Case Tracker workflow — compare uploaded ticket list (xlsx) against
  the Canvas Case Tracker, add new cases, sync Closed/Resolved status, and
  update Next Action from Zoom Mail. Triggers: '케이스 트래커 업데이트',
  'ticket list compare canvas', '활성 케이스 Next Action 업데이트',
  'coupang-case-tracker', '티켓 리스트 비교'.
---

# Coupang Case Tracker

Manage the Coupang Case Tracker Canvas by comparing uploaded ServiceNow ticket
lists, syncing statuses, and updating Next Actions from Zoom Mail.

## Configuration

| Parameter | Default | Description |
|-----------|---------|-------------|
| Canvas File ID | `AN01gwcuTEuskheHWuO1Ug` | Zoom Canvas document ID |
| Table ID | `90e88703f6e64444a7393057d87db310` | Data table ID within the Canvas |
| Mail search period | 2 weeks | How far back to search Zoom Mail |

## Workflow Overview

This skill performs three main tasks, executed in order:

1. **Case List Comparison** — compare uploaded xlsx against Canvas, add new cases
2. **Status Sync** — update Canvas status to Closed for cases that are Closed/Resolved in the list
3. **Next Action Update** — search Zoom Mail for active cases, analyze content, update Canvas

Each task can be run independently or all together.

## Task 1: Case List Comparison & New Record Creation

### Steps

1. Read the uploaded xlsx file using `openpyxl`. Extract all rows where State is
   NOT Closed and NOT Resolved (these are active tickets).
2. Query the Canvas data table using `canvas_data_table_query_rows` with
   `zoom_docs` connector. Fetch all rows with fields: Case Number, Status.
3. Compare by Case Number. Identify tickets in the xlsx that are NOT in Canvas.
4. For each new ticket, map xlsx columns to Canvas columns:

   | xlsx Column | Canvas Column | Type |
   |-------------|---------------|------|
   | Number | Case Number | text |
   | Short description | Description | text (primary) |
   | Priority | Priority | single_select: map "1 - Critical"→Critical, "2 - High"→High, "3 - Normal"→Medium, "4 - Low"→Low |
   | State | Status | single_select: Awaiting Info, Open, etc. |
   | Assigned to | Assigned | single_select |
   | Contact | Requester | single_select |
   | Opened | Case opened | date (ISO 8601 UTC) |
   | — | Sub-Status | single_select: default "Pending Customer" |
   | — | Last Update | date: set to today |
   | Resolution notes | Comment | text (if available) |

5. Create rows using `canvas_data_table_create_rows` (max 10 per call).
6. Present summary of added cases to user.

## Task 2: Status Sync (Canvas ↔ List)

### Steps

1. For each case in Canvas that has Status = Open or Awaiting Info, check if the
   same Case Number in the xlsx has State = Closed or Resolved.
2. If yes, update the Canvas row's Status to "Closed" using
   `canvas_data_table_update_rows`.
3. Also update Last Update to today's date.
4. Present summary of status changes to user.

## Task 3: Next Action Update from Zoom Mail

### Steps

1. Query Canvas for all active cases (Status = Open or Awaiting Info).
2. For each case, search Zoom Mail using the case Description as search query.

### CRITICAL: Email Search Keyword Extraction

Long or mixed-language Descriptions fail when used directly as search queries.
Apply these rules:

- Strip leading prefixes in parentheses: `(Premier)`, `(Problem Report Sent)`, etc.
- Strip content in brackets: `[New Tenant]`, `[신규 테넌트]`, etc.
- For mixed Korean-English descriptions with `- ` separator, use the Korean
  portion if it's longer than 10 characters.
- Remove special characters (parentheses, brackets, hyphens) and collapse whitespace.
- Keep the query under 80 characters.

Example:
- Input: `(Premier) [신규 테넌트] 관리자 콘솔에서 등록한 가상 배경 이미지가 이슈- [New Tenant] Issue with Virtual Background Images`
- Cleaned: `관리자 콘솔에서 등록한 가상 배경 이미지가 이슈`

3. Use `search_zmail` with `after` = 2 weeks ago, `before` = tomorrow, `limit` = 5.
4. Filter results: only keep emails whose subject contains the case Description
   (partial match, case-insensitive, ignoring Re:/RE:/FW:/[Request Closed] prefixes).
5. For matched emails, use `call_llm` to analyze and generate a concise Next Action:
   - Prompt: summarize current status and next step in 1-2 sentences
   - Format: "현재 상태 요약. Next: 다음 액션"
   - Language: Korean if Description is Korean, English otherwise
   - Max ~100 characters
6. Update Canvas using `canvas_data_table_update_rows`:
   - `Next Action`: the generated summary
   - `Last Update`: the timestamp of the most recent email (converted to date)

## Canvas Data Table API Reference

All Canvas operations use the `zoom_docs` connector:

```python
# Search tools
await search_connector_tool(connector_name="zoom_docs", query="...", top_k=5)

# Get schema
await get_connector_tool_schema(connector_name="zoom_docs", tool_name="...")

# Execute
await execute_connector_tool(connector_name="zoom_docs", tool_name="...", arguments={...})
```

Key tools:
- `canvas_data_table_list_tables` — get table_id
- `canvas_data_table_list_columns` — get column schema and valid select options
- `canvas_data_table_query_rows` — read rows with filters (max 50/page)
- `canvas_data_table_create_rows` — add new rows (max 10/call)
- `canvas_data_table_update_rows` — update existing rows (max 10/call)

## Zoom Mail Search Reference

```python
threads = await search_zmail(
    query="<cleaned keywords>",
    after="YYYY/MM/DD",
    before="YYYY/MM/DD",
    limit=5,
)
```

Post-process with:
```python
import sys, os
sys.path.insert(0, os.environ.get("SYNORA_ROOT", "/home/tmp") + "/skills/system/search")
from scripts.zmail_utils import parse_threads
parsed = parse_threads(threads)
```

## Error Handling

- If email search returns no results for a case, skip that case's Next Action update.
- If Canvas update fails, retry once. On second failure, report to user.
- If xlsx file is missing or unreadable, ask user to re-upload.
