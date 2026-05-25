# Hướng Dẫn Sử Dụng — Security Research Workspace v2 + Gemini CLI

---

## 1. Cài đặt ban đầu (chạy 1 lần)

**Windows:**
```powershell
powershell -ExecutionPolicy Bypass -File setup_v2.ps1
```

**Linux / macOS:**
```bash
chmod +x setup_v2.sh && ./setup_v2.sh
```

Script tự tạo toàn bộ cấu trúc sau (Mặc định `D:\Security\` trên Windows hoặc `~/Security/` trên Linux/Mac):

```
D:\Security\          (Windows)
~/Security/           (Linux/Mac)
├── _global\
│   ├── GEMINI.md                    ← Config + methodology + session rules
│   ├── lessons.md                   ← Ghi lại kinh nghiệm giữa các session
│   ├── pentest_state_template.json  ← Template brain state cho mỗi engagement
│   ├── .gemini\
│   │   └── commands\                ← 11 slash commands (TOML)
│   │       ├── new-engagement.toml
│   │       ├── session-start.toml   ← [MỚI] Auto-load context
│   │       ├── session-end.toml     ← [MỚI] Save state + history
│   │       ├── update-state.toml    ← [MỚI] Quick discovery save
│   │       ├── scope-check.toml
│   │       ├── new-finding.toml
│   │       ├── recon.toml
│   │       ├── burp-analyze.toml
│   │       ├── draft-report.toml
│   │       ├── bb-report.toml
│   │       └── gen-office-report.toml
│   ├── templates\
│   ├── wordlists\
│   └── scripts\
│       ├── scope_guard.py           ← Tool Gateway: validate target vs scope
│       ├── result_aggregator.py     ← Normalize + dedup tool output
│       └── excel_extractor.py       ← Đọc file Excel xuất báo cáo
│
├── engagements\
│   └── 2026-05-ACME-webapp\
│       ├── GEMINI.md                ← Scope + rules riêng
│       ├── scope.md                 ← In/out-of-scope chi tiết
│       ├── session_notes.md         ← [MỚI] Đang làm gì hiện tại
│       ├── pentest_state.json       ← [MỚI] Brain: toàn bộ state
│       ├── history.md               ← [MỚI] Lịch sử tất cả sessions
│       ├── checklist-owasp.md
│       ├── .gemini\commands\
│       ├── recon\
│       │   ├── client-provided\
│       │   ├── passive\
│       │   └── active\
│       ├── findings\
│       │   └── 001-sqli-login\
│       ├── finding_summary\
│       ├── evidence\
│       ├── burp\
│       └── report\
│
└── bugbounty\
    ├── GEMINI.md
    ├── hackerone\
    └── bugcrowd\
```

---

## 2. Prerequisites

| Tool | Install | Ghi chú |
|------|---------|---------|
| **Node.js 20+** | https://nodejs.org/ | Runtime cho Gemini CLI |
| **Gemini CLI** | `npm install -g @google/gemini-cli` | AI engine |
| **Google API Key** | https://aistudio.google.com/ | Free: 1000 req/day |
| **Python 3.8+** | https://python.org/ | Chạy scripts phụ trợ |
| **OpenPyXL** | `pip install openpyxl` | Thư viện Python đọc Excel báo cáo |
| **Burp Suite Pro** | https://portswigger.net/ | Proxy + MCP server |

**Setup API key:**
```bash
# Linux/Mac
export GEMINI_API_KEY=your_key_here

# Windows
set GEMINI_API_KEY=your_key_here

# Hoặc dùng Google account (không cần key)
gemini    # Lần đầu → browser auth tự động
```

---

## 3. Sau khi cài xong — Chỉnh sửa cần thiết

| File | Sửa gì |
|------|---------|
| `bugbounty\GEMINI.md` | Điền username HackerOne / Bugcrowd / Intigriti |
| `~/.gemini/settings.json` | Cấu hình Burp Suite đã được merge tự động. Cần chỉnh nếu đổi port. |
| `_global\wordlists\` | Bỏ wordlist hay dùng vào đây (SecLists, custom) |

---

## 4. Nguyên tắc cốt lõi của v2

**Thông tin chỉ cung cấp 1 lần duy nhất** — tại `/new-engagement`.

Sau đó mọi session Gemini tự biết tất cả:

| Thông tin | Cung cấp lần nào |
|-----------|-----------------|
| Scope, target URL | 1 lần — `/new-engagement` |
| Credentials (username/password) | 1 lần — `/new-engagement` |
| Architecture hints | 1 lần — `/new-engagement` |
| Tech stack chi tiết | **0 lần** — Gemini tự discover qua `/recon` |
| Auth mechanism | **0 lần** — Gemini tự discover qua `/burp-analyze` |
| Endpoint map | **0 lần** — Gemini tự build + lưu |
| "Đang làm gì?" | **0 lần** — `/session-start` tự biết |
| OWASP progress | **0 lần** — tracked tự động |

**Mọi session:** chỉ cần `/session-start` → Gemini in ra toàn bộ context, gợi ý tiếp theo.

---

## 5. Bắt đầu engagement mới

```bash
cd ~/Security/_global    # hoặc D:\Security\_global
gemini
```

Gõ:
```
/new-engagement
```

Gemini hỏi lần lượt (1 câu, đợi trả lời):
1. Client name
2. Engagement type (webapp / api / mobile / network / cloud)
3. Target URLs / IPs
4. Out-of-scope items
5. Testing window + timezone
6. Test accounts (role + username + password)
7. API keys từ client (nếu có)
8. Emergency contact
9. Architecture hints từ client (optional)
10. Report language
11. Report deadline
12. Có sẵn recon data không?

Sau đó Gemini tự tạo:
- Folder structure đầy đủ
- `pentest_state.json` pre-filled với tất cả info trên
- `session_notes.md` + `history.md` khởi tạo
- `scope.md` + templates

---

## 6. Workflow hàng ngày — Mỗi session

### Bắt đầu session
```bash
cd ~/Security/engagements/2026-05-ACME-webapp
gemini
```

Gõ ngay:
```
/session-start
```

Gemini tự đọc và in ra:
```
Engagement: ACME-webapp | Phase: 3 | Last active: 2026-05-23
Target: https://api.acme.com | Tech: Node.js/Express, PostgreSQL (guessed)
Findings: C=1 H=2 M=3 L=0 I=5
OWASP: INFO=done CONF=partial ATHN=done ATHZ=partial INPV=todo...
Last session: Tested /api/v2/users IDOR — found ATHZ-001 (Critical)

Was working on: Testing /api/v2/admin/export for privilege escalation
Next planned: Complete PoC for ATHZ-001, test DELETE endpoints

Continue from here, or something specific?
```

**Không cần `@pentest_state.json`, không cần nhắc scope, không cần nhắc credentials.**

### Trong lúc test
```
# Gemini tự save sau mỗi command lớn (/recon, /burp-analyze, /new-finding)

# Save nhanh một discovery:
/update-state "tech: Redis cache detected via X-Cache header"
/update-state "auth: JWT HS256, secret appears weak"
/update-state "endpoint: DELETE /api/v1/users/{id} no auth check"
/update-state "owasp: WSTG-ATHN done"

# Khi tìm được vuln:
/new-finding

# Hoặc nói tự nhiên:
> Tôi vừa confirm IDOR ở /api/v2/users/{id}, tạo finding và kéo request từ Burp
```

### Kết thúc session
```
/session-end
```

Gemini tự:
- Update `pentest_state.json` với mọi discoveries
- Overwrite `session_notes.md` cho session sau
- Append session summary vào `history.md`
- Báo: "Next session: /session-start → resumes at: [specific task]"

---

## 7. Các slash commands (11 total)

| Command | Mô tả | Khi nào dùng |
|---------|--------|-------------|
| `/new-engagement` | Tạo project pentest mới | Nhận engagement mới — **chỉ 1 lần** |
| `/session-start` | **[MỚI]** Auto-load toàn bộ context | **ĐẦU MỖI SESSION** |
| `/session-end` | **[MỚI]** Lưu state + history | **CUỐI MỖI SESSION** |
| `/update-state <discovery>` | **[MỚI]** Save nhanh 1 discovery | Trong lúc test |
| `/scope-check <target>` | Validate target vs scope.md | Trước mọi active scan |
| `/new-finding` | Tạo finding + auto-update state | Phát hiện vuln |
| `/recon` | Structured recon, skip phần đã có, auto-save | Phase 1 |
| `/burp-analyze` | Phân tích Burp proxy, auto-save endpoints + auth | Sau khi browse target |
| `/draft-report` | Tổng hợp findings → báo cáo | Kết thúc engagement |
| `/bb-report` | Draft bug bounty report | Submit lên HackerOne/Bugcrowd |
| `/gen-office-report` | Tạo security report từ Excel | Báo cáo chính thức cho khách |

> **Lưu ý:** Commands lưu trong `.gemini/commands/*.toml`.
> Sau khi sửa, gõ `/commands reload` — không cần restart Gemini CLI.

---

## 8. Ba files quản lý state

### `pentest_state.json` — Single source of truth
Lưu **tất cả** thông tin về engagement, không bao giờ phải nhập lại:

```json
{
  "meta": {
    "engagement": "ACME-webapp",
    "current_phase": "Phase 3: Exploitation",
    "current_session_id": "session-005"
  },
  "target": {
    "primary_url": "https://api.acme.com",
    "tech_stack": {
      "backend": "Node.js Express 4.18",
      "database": "PostgreSQL (from error messages)",
      "waf": "Cloudflare (detected via 403 pattern)"
    },
    "auth": {
      "type": "JWT Bearer",
      "algorithm": "HS256",
      "login_endpoint": "POST /api/v1/auth/login",
      "token_expiry": "15min"
    }
  },
  "credentials": {
    "test_accounts": [
      {"role": "user", "username": "user@acme.com", "password": "Pass123", "user_id": "42"},
      {"role": "admin", "username": "admin@acme.com", "password": "Admin456", "user_id": "1"}
    ]
  },
  "endpoints": {
    "discovered": [{"url": "/api/v1/users", "method": "GET", "auth_required": true}],
    "tested": ["/api/v1/auth/login"],
    "interesting": [{"url": "/api/v2/admin/export", "reason": "possible privesc"}]
  },
  "owasp_checklist": {
    "WSTG-INFO": "done",
    "WSTG-ATHN": "done",
    "WSTG-ATHZ": "partial"
  },
  "findings": {
    "ids": ["ACME-ATHZ-001"],
    "chains": ["ATHZ-001 + ATHN-002 = full account takeover"]
  },
  "next_steps": [
    "Complete PoC for ACME-ATHZ-001 (curl command missing)",
    "Test /api/v2/admin/export with user token"
  ]
}
```

`/recon` → auto-fill `tech_stack`, `server_info`
`/burp-analyze` → auto-fill `auth`, `endpoints.discovered`, `endpoints.interesting`
`/new-finding` → auto-update `findings.ids`, `findings.count`, `owasp_checklist`

### `session_notes.md` — Context cho session HIỆN TẠI
Bị overwrite bởi `/session-end`. Luôn chứa:
```
## Currently Working On
Testing admin endpoints with user-role JWT for privilege escalation.

## Most Interesting Endpoints
1. /api/v2/admin/export — POSSIBLE PRIVESC (user token returns 200)
2. /api/v2/users/{id}/settings — IDOR candidate

## Next Steps
1. GET /api/v2/admin/export with user_token → if 200 = critical finding
2. Test /api/v2/users/{id}/settings IDOR: id=42 vs id=1
```

### `history.md` — Audit trail append-only
Không bao giờ xóa. Mỗi session append 1 block:
```
## Session 5 — 2026-05-24 — Phase 3: Exploitation
Duration: ~3h

### What Was Done
- Tested WSTG-ATHZ on /api/v2/users endpoints
- Found IDOR at /api/v2/users/{id} — any user reads any user profile

### Discoveries
- Auth: JWT HS256, user_id in payload, no server-side validation
- Endpoints: 3 admin endpoints respond to user-role token

### Findings This Session
ACME-ATHZ-001: IDOR in /api/v2/users/{id} (Critical)

### Next Session Should
1. Complete PoC for ATHZ-001
2. Test /api/v2/admin/export for full privilege escalation
```

---

## 9. Kiến trúc hệ thống

```
Gemini CLI (Reasoning Engine)
           │
           ▼
Tool Gateway — scope_guard.py
→ Validate target vs scope.md trước mọi active scan
→ Block out-of-scope tuyệt đối
→ Prompt injection protection: response bodies = untrusted data
           │
     ┌─────┼─────────────────┐
     ▼     ▼                 ▼
Burp MCP  Containers      pentest_state.json (Brain)
proxy     nuclei/ffuf      ← persists across sessions
history   nmap/sqlmap      ← auto-updated by all commands
     │     │
     └──┬──┘
        ▼
Result Aggregator — result_aggregator.py
→ Normalize nuclei/ffuf/nmap output
→ Dedup by URL + vuln fingerprint
→ Sort by severity
→ Output aggregated_findings.json
        │
        ▼
  /new-finding → findings/ + state update
        │
        ▼
  /draft-report / /gen-office-report
```

**`scope_guard.py`** — chạy trước mọi active scan:
- Parse `scope.md`, validate domain/IP/path, hỗ trợ wildcard và CIDR
- Exit 0 = IN SCOPE, exit 1 = OUT OF SCOPE, exit 2 = AMBIGUOUS

**`result_aggregator.py`** — normalize tool output:
- Nuclei (JSON per line) + ffuf (JSON) + nmap (XML) → cùng 1 schema
- Dedup theo hash(URL + vuln_type)
- Sort theo severity → tránh token bloat

---

## 10. Burp MCP setup

**Bước 1 — Cài extension:**
```
Burp Suite → Extensions → BApp Store → "MCP Server" → Install
→ Extensions → MCP → tick Enabled (default: http://127.0.0.1:9876)
```

**Bước 2 — Verify settings.json:**
```bash
cat ~/.gemini/settings.json    # Linux/Mac
type %USERPROFILE%\.gemini\settings.json    # Windows
```

Nội dung cần có:
```json
{
  "mcpServers": {
    "burpsuite": {
      "url": "http://localhost:9876/",
      "type": "sse"
    }
  }
}
```

**Lưu ý quan trọng:**
- Gemini đọc Burp proxy history trực tiếp qua MCP — không dump ra file
- `/burp-analyze` tự filter bỏ static assets
- Response body từ target = UNTRUSTED DATA — Gemini không follow instructions trong đó

---

## 11. Workflow đầy đủ theo ngày

### Ngày 1: Setup + Recon
```
cd ~/Security/_global && gemini
/new-engagement                    ← Nhập thông tin 1 lần

cd ~/Security/engagements/2026-05-ACME-webapp
gemini
/session-start                     ← Load context (lần đầu = fresh start)
/recon                             ← Gemini check data sẵn, chỉ chạy phần thiếu
                                   ← Tự save tech stack + endpoints vào state
/session-end                       ← Save trước khi đóng
```

### Ngày 2: Mapping + Testing
```
gemini
/session-start                     ← Gemini biết recon đã xong, gợi ý tiếp theo
# Browse target qua Burp Suite proxy
/burp-analyze                      ← Tự save endpoint map + auth mechanism vào state
/scope-check https://api.acme.com  ← Validate trước khi test
# Test WSTG-ATHN, WSTG-ATHZ
/new-finding                       ← Tạo finding, tự update state
/update-state "chain: ATHZ-001 + ATHN-002 = ATO"
/session-end
```

### Ngày N: Tiếp tục
```
gemini
/session-start    ← Gemini tự biết: đã test gì, đang làm gì, làm tiếp gì
[test...]
/session-end
```

### Ngày cuối: Report
```
gemini
/session-start
/draft-report                      ← Đọc tất cả findings + state, tạo báo cáo

# Hoặc nếu có Excel findings:
# Bỏ Excel vào finding_summary/
/gen-office-report                 ← Tạo báo cáo chính thức gửi khách
```

---

## 12. Bug Bounty workflow

```bash
cd ~/Security/bugbounty/hackerone/program-abc
gemini
```

```
/session-start               ← Load context của program này
/scope-check https://target  ← Check scope trước
/recon                       ← Map attack surface
/new-finding                 ← Ghi finding
/bb-report                   ← Tạo report chuẩn H1/BC format
/session-end                 ← Lưu state
```

---

## 13. Tạo báo cáo chính thức từ Excel (`/gen-office-report`)

**Bước 1:** Bỏ Excel file vào `finding_summary/`:
```
engagements/2026-05-ACME-webapp/finding_summary/
└── ACME_Security_Summary.xlsx
```

**Bước 2:**
```bash
cd ~/Security/engagements/2026-05-ACME-webapp
gemini
/session-start         ← Load context (pre-fill client info từ state)
/gen-office-report     ← Gemini tự pre-fill từ pentest_state.json
```

Gemini sẽ:
1. Scan `finding_summary/` → tìm Excel → hỏi confirm
2. Pre-fill từ `pentest_state.json`: client name, URL, dates — chỉ hỏi gì còn thiếu
3. Đọc Excel sheet "Summary Finding"
4. Extract hình PoC từ các sheet → lưu vào `report/evidence/`
5. Auto-map OWASP category, auto-generate recommendations (nếu cột trống)
6. Tạo `report/{CLIENT}-{APP}-Security-Report.md`

**Excel format yêu cầu:**
- Sheet name: `Summary Finding`
- Columns: `#`, `Date`, `Finding Code`, `Finding Name`, `Description`, `CVSS Score`, `Impact`, `Status`, `Position`, `POC`, `CVSS Vector`, `Recommendation`
- Cột `POC`: hyperlink đến sheet riêng chứa screenshot PoC

---

## 14. Mẹo nâng cao

**Chạy nhiều session song song:**
```bash
# Terminal 1: testing
cd engagement-folder && gemini

# Terminal 2: viết report
cd engagement-folder && gemini
```
Cả 2 session đọc chung `pentest_state.json` — sau khi một session update, session kia thấy ngay khi đọc file.

**Thêm MCP server khác:**
```json
// ~/.gemini/settings.json
{
  "mcpServers": {
    "burpsuite": { "url": "http://localhost:9876/", "type": "sse" },
    "your-tool": { "command": "node", "args": ["path/to/mcp.js"] }
  }
}
```

**Inject file cụ thể vào context:**
```
@recon/active/nuclei.json    ← Inject raw output để Gemini phân tích
@recon/client-provided/openapi.yaml    ← Inject API spec
```

**Reload commands sau khi sửa TOML:**
```
/commands reload    ← Không cần restart
/commands list      ← Xem tất cả đang load
```

**Import data recon có sẵn:**
```
# Bỏ file vào đúng folder trước khi chạy /recon:
recon/client-provided/   ← Tài liệu từ khách (API docs, diagram, endpoint list)
recon/passive/           ← Passive recon đã chạy
recon/active/            ← Active recon đã chạy (nmap, ffuf, nuclei output)
```
Khi `/recon` chạy, Gemini đọc Recon Status trong GEMINI.md, in `[SKIP]`/`[TODO]`, chỉ chạy phần còn thiếu.

**Dùng `!{...}` trong TOML để inject shell output:**
```toml
# Trong custom command, thêm context tự động:
prompt = """
!{cat scope.md}
!{python3 ~/Security/_global/scripts/scope_guard.py auto scope.md}
[... rest of prompt ...]
"""
```

---

## 15. So sánh v1 vs v2

| Tính năng | v1 (setup.ps1) | v2 (setup_v2.ps1) |
|-----------|----------------|-------------------|
| Commands | 9 | 11 |
| Cung cấp thông tin | Mỗi session | **1 lần duy nhất** |
| Session context | Manual `@file` inject | **Auto qua /session-start** |
| Tech stack tracking | Không | **Tự discover + lưu** |
| Auth mechanism | Không | **Tự discover + lưu** |
| Session history | lessons.md (manual) | **history.md (auto append)** |
| Session handoff | Không | **session_notes.md (auto)** |
| OWASP progress | Không track | **owasp_checklist trong state** |
| Attack chains | Không | **findings.chains trong state** |
| brain-sync | Manual | **/session-end (tự động hơn)** |
