# UltraFuzz — AI-Powered Enterprise Fuzzing Skill for Claude

> **Internal Security Testing · No External API Key Required · Beats ffufai & ffuf_claude_skill**

UltraFuzz is a Claude skill that transforms Claude itself into a complete, autonomous web fuzzing engine. It reads your target folder, profiles each host, generates AI-crafted wordlists, executes `ffuf`, triages results against known vulnerability patterns, and delivers a structured security report — all without requiring any OpenAI or Anthropic API key.

---

## Table of Contents

1. [Why UltraFuzz Beats the Competition](#1-why-ultrafuzz-beats-the-competition)
2. [Architecture Overview](#2-architecture-overview)
3. [Prerequisites](#3-prerequisites)
4. [Installation](#4-installation)
5. [Target Folder Structure — What Files to Prepare](#5-target-folder-structure--what-files-to-prepare)
6. [File Format Reference](#6-file-format-reference)
7. [How to Run a Scan](#7-how-to-run-a-scan)
8. [What Happens Automatically](#8-what-happens-automatically)
9. [Vulnerability Classes Detected](#9-vulnerability-classes-detected)
10. [Output & Reports](#10-output--reports)
11. [Safety & Authorization Policy](#11-safety--authorization-policy)
12. [Troubleshooting](#12-troubleshooting)

---

## 1. Why UltraFuzz Beats the Competition

Having reviewed the full source code of both reference repositories, the comparison below is grounded in actual implementation — not marketing.

### ffufai (`jthack/ffufai`) — What It Actually Does

- A **268-line Python CLI wrapper** around `ffuf`
- Accepts **one URL at a time** via `-u` flag
- Makes a single AI call to suggest file extensions (e.g. `.php`, `.bak`) or generate a basic wordlist
- **Requires** `OPENAI_API_KEY` or `ANTHROPIC_API_KEY` set as an environment variable
- No bulk target handling, no triage, no vulnerability classification, no reporting

### ffuf_claude_skill (`jthack/ffuf_claude_skill`) — What It Actually Does

- A **prompt-only skill** that tells Claude to run `ffuf` commands when asked
- No scripts, no wordlist intelligence, no automation
- Claude decides flags manually per request
- No result analysis, no follow-up scan logic, no structured output

### UltraFuzz — What It Actually Does

| Capability | ffufai | ffuf_claude_skill | **UltraFuzz** |
|---|:---:|:---:|:---:|
| External API key required | ✗ Required | ✗ Required | **✓ None — Claude IS the AI** |
| Bulk folder of targets | ✗ One URL only | ✗ Manual | **✓ Auto-batch entire folder** |
| Tech stack detection | ✗ | ✗ | **✓ Header-based host profiling** |
| 404 baseline calibration | ✗ | ✗ | **✓ Auto-derives `-fs` per target** |
| AI wordlist generation | Partial (1 call) | ✗ | **✓ 5-phase multi-category strategy** |
| Vulnerability triage | ✗ | ✗ | **✓ Pattern-matched vuln classes** |
| CRITICAL/HIGH/MEDIUM/LOW severity | ✗ | ✗ | **✓ Full severity matrix** |
| Follow-up scan automation | ✗ | ✗ | **✓ Auto-queued from findings** |
| Structured markdown report | ✗ | ✗ | **✓ Executive summary + remediation** |
| Parameter fuzzing support | ✗ | ✗ | **✓ URL, GET param, POST body** |
| POST body fuzzing | ✗ | ✗ | **✓ JSON payload fuzzing** |
| Iterative refinement | ✗ | ✗ | **✓ Re-fuzzes based on discoveries** |
| Audit log / scan metadata | ✗ | ✗ | **✓ Timestamped JSON + markdown** |

---

## 2. Architecture Overview

```
ultrafuzz-skill/
├── SKILL.md                        ← Claude's instruction set & onboarding flow
├── scripts/
│   ├── setup.py                    ← Folder parser, URL extractor, host profiler
│   ├── triage.py                   ← ffuf JSON result analyser & vuln classifier
│   └── report.py                   ← Structured markdown report generator
└── references/
    ├── wordlist_strategy.md        ← 5-phase AI wordlist generation rules
    ├── ffuf_execution.md           ← Smart ffuf flag selection per target type
    └── vuln_patterns.md            ← Vulnerability pattern catalog with remediation
```

**Execution Pipeline:**

```
Folder Input
    │
    ▼
setup.py          ← Parses all .txt files, extracts URLs, probes hosts
    │
    ▼
Claude AI         ← Reads headers, detects stack, classifies targets
    │
    ▼
wordlist_strategy ← Generates custom wordlists per target (5 phases)
    │
    ▼
ffuf_execution    ← Constructs smart ffuf command with correct flags
    │
    ▼
ffuf              ← Executes scan, outputs JSON results
    │
    ▼
triage.py         ← Classifies findings by severity + vulnerability type
    │
    ▼
report.py         ← Generates markdown security assessment report
```

---

## 3. Prerequisites

### Required

- **Claude Code** (Claude Desktop with skills support)
- **ffuf** installed and accessible in `$PATH`
- **Python 3.8+**
- **curl** (standard on Linux/macOS)

### Installing ffuf

**Linux (Go install):**
```bash
go install github.com/ffuf/ffuf/v2@latest
```

**Linux (apt):**
```bash
sudo apt install ffuf
```

**macOS:**
```bash
brew install ffuf
```

**Verify:**
```bash
ffuf -V
# Expected: ffuf v2.x.x
```

---

## 4. Installation

```bash
# 1. Download and extract the skill
unzip ultrafuzz-skill.skill -d ultrafuzz-skill/

# 2. Create Claude skills directory if it doesn't exist
mkdir -p ~/.claude/skills

# 3. Install the skill
cp -r ultrafuzz-skill/ ~/.claude/skills/

# 4. Verify structure
ls ~/.claude/skills/ultrafuzz-skill/
# Should show: SKILL.md  scripts/  references/

# 5. Create working directories
mkdir -p /tmp/ultrafuzz_wordlists /tmp/ultrafuzz_results
```

Claude Code will automatically detect and load the skill on next session start.

---

## 5. Target Folder Structure — What Files to Prepare

This is the most important section. UltraFuzz reads an entire folder of `.txt` files you prepare in advance. The folder can contain any combination of the file types below.

### Recommended Folder Layout

```
/your/targets/folder/
│
├── fuzz_urls.txt           ← URLs with FUZZ keyword already placed
├── plain_urls.txt          ← Plain URLs (FUZZ appended to path automatically)
├── api_endpoints.txt       ← REST API base URLs for endpoint discovery
├── admin_targets.txt       ← Admin panel URLs to enumerate
├── params.txt              ← Parameter names for GET/POST fuzzing
├── internal_hosts.txt      ← Internal hostnames for path discovery
└── custom_wordlist.txt     ← Any custom words specific to your environment
```

All `.txt` files in the folder (including subdirectories) are auto-detected. File naming does not matter — UltraFuzz detects content type automatically.

---

## 6. File Format Reference

### Type A — FUZZ URLs (`fuzz_urls.txt`)

These are the highest-priority files. Each line contains a full URL with the literal keyword `FUZZ` exactly where you want fuzzing to occur.

**Format:**
```
https://internal.corp.com/FUZZ
https://api.corp.com/v1/FUZZ
https://portal.corp.com/admin/FUZZ
https://app.corp.com/api/users/FUZZ/settings
https://internal.corp.com/search?query=FUZZ
https://api.corp.com/v2/actions?method=FUZZ
```

**Rules:**
- One URL per line
- `FUZZ` is case-sensitive — must be uppercase
- `FUZZ` can appear in path, query string, or header values
- Lines starting with `#` are treated as comments and skipped
- Blank lines are ignored

**Example — Path fuzzing:**
```
# Production internal API
https://api.internal.corp.com/v1/FUZZ
https://api.internal.corp.com/v2/FUZZ

# Admin panel
https://manage.corp.com/admin/FUZZ
```

**Example — Parameter name discovery:**
```
# Discover hidden GET parameters
https://app.corp.com/search?FUZZ=test
https://app.corp.com/api/filter?FUZZ=1
```

**Example — Parameter value fuzzing (IDOR, injection):**
```
# Fuzz user ID values
https://app.corp.com/api/users/FUZZ/profile
https://app.corp.com/api/orders/FUZZ
```

---

### Type B — Plain URLs (`plain_urls.txt`)

URLs without `FUZZ`. UltraFuzz automatically appends `/FUZZ` to the path for directory/file discovery.

**Format:**
```
https://internal.corp.com
https://api.corp.com/v1
https://portal.corp.com/admin
https://app.corp.com/api/users
http://192.168.1.50:8080
http://192.168.1.50:8443/management
```

**Rules:**
- One URL per line
- Trailing slashes are handled automatically
- Both HTTP and HTTPS are supported
- IP addresses with custom ports are fully supported
- UltraFuzz will also try to detect and profile the host automatically

**What UltraFuzz does with these:**
Each plain URL becomes `<url>/FUZZ` and is queued for path/directory discovery using a custom AI-generated wordlist matched to the detected tech stack.

---

### Type C — Parameter Name Files (`params.txt`)

A list of parameter names to test against target endpoints. These are used for hidden parameter discovery — testing whether a parameter exists on an endpoint that doesn't advertise it.

**Format:**
```
id
user_id
userId
account_id
debug
admin
token
file
path
redirect
callback
action
cmd
format
export
include
fields
```

**Rules:**
- One parameter name per line
- No values — only names
- No `=` signs
- Names are injected as `?<name>=test` against each target URL

**Naming convention tip:** Name this file `params.txt` or `parameters.txt` so UltraFuzz auto-classifies it correctly, though any name works.

---

### Type D — Custom Domain Wordlists (`custom_words.txt`)

Words specific to your company's technology, product names, or internal naming conventions. These are merged with AI-generated wordlists for maximum coverage.

**Format:**
```
# Internal service names
myapp
myapp-api
myapp-admin
legacy
v3
v4
uat
ppe
sandbox

# Internal product names
projectname
projectname-api
projectname-portal

# Tech-specific paths you know exist
spring
actuator
graphql
internal
devops
```

**Rules:**
- One word or path segment per line
- Can include path separators: `admin/settings`, `api/v3`
- Lines starting with `#` are comments
- Include your company's internal naming patterns — these are the words generic wordlists will never have

---

### Type E — Mixed Files

UltraFuzz handles files that contain a mix of URLs, plain hosts, and words. It will classify each line independently:

- Lines starting with `http://` or `https://` → treated as URLs
- Lines containing `FUZZ` → treated as FUZZ URLs
- All other lines → treated as wordlist entries

---

### Complete Example Folder

```
/home/secteam/targets/q2-internal-audit/
│
├── fuzz_urls.txt
│   # https://api.corp.com/v1/FUZZ
│   # https://api.corp.com/v2/users/FUZZ
│   # https://portal.corp.com/FUZZ
│   # https://internal.corp.com/admin/FUZZ
│
├── plain_urls.txt
│   # https://api.corp.com
│   # https://portal.corp.com
│   # https://manage.corp.com
│   # http://192.168.10.15:8080
│   # http://192.168.10.22:9090
│
├── api_params.txt
│   # id
│   # user_id
│   # token
│   # debug
│   # action
│   # format
│
├── custom_words.txt
│   # projectname
│   # corpname-api
│   # internal-v2
│   # legacy
│   # sandbox
│
└── old_endpoints.txt
    # Any legacy endpoints your team still wants verified
    # https://old.corp.com/FUZZ
    # https://v1.api.corp.com/FUZZ
```

---

## 7. How to Run a Scan

Once the skill is installed, open Claude Code and simply say:

```
Fuzz the targets in /home/secteam/targets/q2-internal-audit/
```

Or more explicitly:

```
Load UltraFuzz and scan everything in /home/secteam/targets/q2-internal-audit/
```

UltraFuzz will immediately:

1. Ask you to confirm the folder path
2. Parse all files and show a summary of what it found
3. Ask if you want Quick Scan (fast, high-signal) or Deep Scan (thorough, full wordlists)
4. Begin scanning autonomously

You do not need to specify wordlists, flags, or commands — Claude handles everything.

---

## 8. What Happens Automatically

### Phase 1 — Target Parsing
- All `.txt` / `.lst` files in the folder are read
- URLs with `FUZZ` are extracted as direct targets
- Plain URLs get `/FUZZ` appended
- Parameter files are identified and stored separately
- Up to 500 unique targets per file are queued

### Phase 2 — Host Profiling
For each unique hostname, Claude runs `curl -sI` to:
- Detect server software (`Server:` header)
- Identify frameworks (`X-Powered-By:`, `Set-Cookie:` names, `X-Generator:`)
- Classify target type: `api | admin | web_app | internal | unknown`
- Measure the 404 response size (used to auto-derive `-fs` noise filter)

### Phase 3 — AI Wordlist Generation
Claude generates wordlists in 5 phases per target:
1. Universal high-value paths (admin, config, backup, debug, git, env files)
2. Framework-specific paths (PHP/WordPress, Java/Spring, Node.js, Python/Django, ASP.NET)
3. API endpoint discovery (v1/v2, REST verbs, resource names)
4. Domain-specific enrichment (derived from hostname and known path segments)
5. Quick-scan top-200 prioritised list for first pass

### Phase 4 — ffuf Execution
Claude constructs the exact `ffuf` command per target with:
- Rate limiting appropriate to the target type
- `-fs` set to the measured 404 baseline size
- Correct status code matching (`-mc`) for the scan type
- JSON output (`-of json`) for structured triage
- Auth headers if provided

### Phase 5 — Triage & Follow-up
Each result is classified by severity. High-signal findings automatically trigger follow-up scans:
- Found `/api/v1/` → enumerate `/api/v1/FUZZ`
- Found `/.git/` → fetch config, HEAD, COMMIT_EDITMSG
- Found `/admin/` → enumerate `/admin/FUZZ` with admin wordlist
- Found GraphQL endpoint → attempt introspection query
- Found Swagger/OpenAPI → parse spec and extract all endpoints

### Phase 6 — Reporting
A markdown report is written to `/tmp/ultrafuzz_report_<timestamp>.md` containing:
- Executive summary with total finding counts
- Per-finding: URL, status, risk description, remediation steps
- Prioritised remediation checklist
- Full technical metadata (commands run, wordlists used, scan duration)

---

## 9. Vulnerability Classes Detected

### Critical
- Exposed `.env`, `.env.production`, secret/credential files
- PHP info pages (`phpinfo.php`, `info.php`)
- Web shells and backdoors (`shell.php`, `cmd.php`)

### High
- Exposed Git repositories (`/.git/config`)
- Database backup files (`.sql`, `.db`, `.sqlite`)
- Archive files containing source code (`.zip`, `.tar.gz`)
- Database admin interfaces (`phpmyadmin`, `adminer`)
- Spring Boot Actuator endpoints (`/actuator/heapdump`, `/actuator/env`)
- Admin panels accessible without authentication

### Medium
- Debug and development endpoints (`/debug`, `/dev`, `/test`)
- API documentation exposed (`swagger.json`, `openapi.yaml`)
- Log files accessible (`error.log`, `debug.log`)
- Configuration files in web root (`.conf`, `.yaml`, `.ini`)
- Server errors (500) indicating potential injection points

### Low
- Directory listing enabled
- Version disclosure files (`readme.html`, `CHANGELOG.txt`)
- Default server pages

### Informational
- API versions discovered (enumerate further)
- GraphQL endpoints (test introspection)
- Auth-protected endpoints (test with credentials)
- New path segments (queue targeted follow-up scans)

---

## 10. Output & Reports

All output is written to `/tmp/`:

```
/tmp/
├── ultrafuzz_wordlists/
│   ├── api-corp-com_universal.txt
│   ├── api-corp-com_framework.txt
│   └── api-corp-com_quickscan.txt
├── ultrafuzz_results/
│   ├── api-corp-com_v1.json        ← Raw ffuf output
│   └── portal-corp-com_admin.json
└── ultrafuzz_report_20260601_143022.md  ← Final assessment report
```

The markdown report follows this structure:

```
# UltraFuzz Security Assessment Report
  Date, targets, total requests, duration

## Executive Summary
  Finding counts by severity, urgent alerts if Critical

## Critical Findings
  Per-finding: URL, status, risk, remediation

## High Findings
  ...

## Medium Findings
  ...

## Recommendations
  Prioritised numbered remediation checklist

## Technical Details
  Full command log, wordlists used, scan config
```

---

## 11. Safety & Authorization Policy

**This skill is built exclusively for authorized internal security testing.**

- Only scan systems your organisation owns or has explicit written authorisation to test
- Never run against production systems without change-management approval and a rollback plan
- Default rate limit is 50 requests/second — Claude will not override this without explicit instruction
- All scans are logged with timestamps for audit trail compliance
- Claude will ask for authorisation confirmation before scanning any target that appears external

**Claude will refuse to fuzz targets that:**
- Are not on your internal network (without explicit authorisation statement)
- Belong to third parties
- Are not listed in your authorised scope

---

## 12. Troubleshooting

**ffuf not found:**
```bash
which ffuf
# If empty: go install github.com/ffuf/ffuf/v2@latest
# Ensure $GOPATH/bin is in your $PATH
```

**Skill not triggering:**
- Confirm `SKILL.md` is at `~/.claude/skills/ultrafuzz-skill/SKILL.md`
- Restart Claude Code after installation
- Try explicit phrasing: "Use UltraFuzz to scan..."

**0 results from scan:**
- Check network connectivity to target from the scan machine
- Verify the URL is reachable with `curl -I <target>`
- The baseline 404 filter (`-fs`) may be too aggressive — Claude will detect this and retry

**Rate limiting (429 responses):**
- Claude will automatically detect 429 patterns and reduce the rate
- You can also instruct: "use a slower scan rate for this target"

**Python script errors:**
```bash
python3 --version   # Requires 3.8+
python3 ~/.claude/skills/ultrafuzz-skill/scripts/setup.py /your/folder --probe
```

---

## Legal Disclaimer

UltraFuzz is provided for **defensive security purposes only**. Use exclusively on systems you own or have explicit written authorisation to test. Unauthorised testing of computer systems is illegal in most jurisdictions. The authors accept no liability for misuse.

---

*UltraFuzz — Built for security engineers who need results, not configuration overhead.*
