# Coupang Case Tracker

Zoom AI Companion Custom Skill for managing Coupang support cases.

This skill automates the workflow of comparing ServiceNow (SNOW) ticket exports with a Zoom Canvas Case Tracker, syncing statuses, and updating Next Actions from Zoom Mail — all within the Zoom AI Companion platform.

## What It Does

The skill performs **three tasks** in sequence:

### Task 1: Case List Comparison & New Record Creation
- Reads an uploaded ServiceNow ticket export (`.xlsx`)
- Compares against the existing Zoom Canvas Case Tracker
- Identifies new active tickets (non-Closed/Resolved) not yet in Canvas
- Automatically creates new rows with mapped fields

### Task 2: Status Sync
- Finds cases that are `Open` or `Awaiting Info` in Canvas but `Closed` or `Resolved` in the SNOW list
- Updates Canvas status to `Closed` for matched cases

### Task 3: Next Action Update from Zoom Mail
- Queries Zoom Mail for emails matching each active case's description
- Uses AI (LLM) to analyze email threads and generate concise status summaries
- Updates the `Next Action` and `Last Update` fields in Canvas

## Key Features

- **Bilingual support**: Handles both Korean and English headers, descriptions, and state values
- **Smart email search**: Strips special characters, brackets, and prefixes from case descriptions to improve Zoom Mail search accuracy for long mixed-language subjects ([details](coupang-case-tracker/references/keyword-extraction.md))
- **Automatic field mapping**: Maps ServiceNow fields to Canvas columns with priority and state translation

## Repo Structure

```
coupang-case-tracker/
├── README.md                          ← You are here
├── .gitignore                         ← Git ignore rules
└── coupang-case-tracker/              ← Skill package
    ├── SKILL.md                       ← Main skill definition & workflow
    └── references/
        └── keyword-extraction.md      ← Email search keyword extraction logic
```

## Skill Configuration

| Parameter | Default | Description |
|-----------|---------|-------------|
| Canvas File ID | `AN01gwcuTEuskheHWuO1Ug` | Zoom Canvas document containing the Case Tracker |
| Table ID | `90e88703f6e64444a7393057d87db310` | Data table ID within the Canvas |
| Mail search period | 2 weeks | How far back to search Zoom Mail |

## Canvas Column Schema

| Column | Type | Description |
|--------|------|-------------|
| Case Number | text | e.g., TS1234567 |
| Description | text (primary) | Ticket short description |
| Priority | single_select | Critical / High / Medium / Low |
| Status | single_select | Open / Awaiting Info / Closed / Resolved |
| Sub-Status | single_select | Pending Customer / TSE / Engineering / TAM |
| Case opened | date | When the case was created |
| Last Update | date | Last email date or update date |
| Next Action | text | AI-generated status summary and next step |
| Assigned | single_select | TSE name |
| Requester | single_select | Customer contact |
| JIRA | url | JIRA ticket reference |
| Comment | text | Additional notes |

## Trigger Phrases

- `케이스 트래커 업데이트` / `케이스 업데이트 해줘`
- `티켓 리스트 비교`
- `활성 케이스 Next Action 업데이트`
- `coupang-case-tracker`

## Prerequisites

This skill runs inside **Zoom AI Companion** and requires:
- Zoom Canvas (Zoom Docs) access
- Zoom Mail access
- A Zoom AI Companion project with this skill bound

## How to Use

1. **With a SNOW ticket file**: Upload a ServiceNow export `.xlsx` and say "케이스 업데이트 해줘" — all 3 tasks run
2. **Without a file**: Say "Next Action 업데이트 해줘" — only Task 3 runs (mail-based update)

## License

Internal use — Zoom Video Communications
