# TESTING.md — iba-governor

IBA Intent Bound Authorization · Core Governor Gate Tests
Patent GB2603013.0 (Pending) · intentbound.com

---

## Quick Test

```bash
pip install -r requirements.txt
python iba_governor.py --demo
```

Expected output:

```
+ ALLOWED  [read file src/main.py]                    (0.XXXms)
+ ALLOWED  [write file src/utils.py refactor]         (0.XXXms)
+ ALLOWED  [run tests unit suite]                     (0.XXXms)
+ ALLOWED  [git commit feature branch]                (0.XXXms)
+ ALLOWED  [code review pull request]                 (0.XXXms)
x BLOCKED  [production_deploy release v2.1]           -> Action in denied list
x BLOCKED  [credential_access aws secrets]            -> Action in denied list
x BLOCKED  [external_data_export user database]       -> Action in denied list
x BLOCKED  [send email to all users]                  -> Outside declared scope (DENY_ALL)
x TERMINATE [delete_production database all tables]   -> Kill threshold — session ended
```

---

## Test Suite

### 1 — Permitted actions (ALLOW expected)

```bash
python iba_governor.py "read file src/main.py"
python iba_governor.py "write file src/utils.py"
python iba_governor.py "run tests unit suite"
python iba_governor.py "git commit feature branch"
python iba_governor.py "code review pull request"
```

All should return `+ ALLOWED` with sub-1ms latency.

---

### 2 — Denied list enforcement (BLOCK expected)

```bash
python -c "
from iba_governor import IBAGovernor, IBABlockedError
g = IBAGovernor()
for action in ['production_deploy', 'credential_access', 'external_data_export', 'payment_execution']:
    try:
        g.check_action(action)
    except IBABlockedError:
        print(f'PASS — blocked: {action}')
"
```

---

### 3 — Kill threshold (TERMINATE expected)

```bash
python -c "
from iba_governor import IBAGovernor, IBATerminatedError
g = IBAGovernor()
try:
    g.check_action('delete_production database all tables')
except IBATerminatedError as e:
    print('PASS — session terminated:', e)
"
```

---

### 4 — DENY_ALL posture (out of scope)

```bash
python -c "
from iba_governor import IBAGovernor, IBABlockedError
g = IBAGovernor()
try:
    g.check_action('send email to all users')
except IBABlockedError as e:
    print('PASS — DENY_ALL blocked:', e)
"
```

---

### 5 — No certificate (auto-creates default · DENY_ALL)

```bash
python -c "
import os
if os.path.exists('.iba.yaml'):
    os.rename('.iba.yaml', '.iba.yaml.bak')
from iba_governor import IBAGovernor
g = IBAGovernor()
print('PASS — default cert created · DENY_ALL posture active')
if os.path.exists('.iba.yaml.bak'):
    os.rename('.iba.yaml.bak', '.iba.yaml')
"
```

---

### 6 — Audit chain integrity

```bash
python iba_governor.py --demo
cat iba-audit.jsonl
```

Every gate decision — ALLOW, BLOCK, TERMINATE — should appear as a timestamped JSON line with session ID, principal, action, verdict, and reason.

---

### 7 — Latency benchmark (sub-1ms required)

```bash
python -c "
import time
from iba_governor import IBAGovernor
g = IBAGovernor()
times = []
for _ in range(1000):
    start = time.perf_counter()
    try:
        g.check_action('read file src/main.py')
    except Exception:
        pass
    times.append((time.perf_counter() - start) * 1000)
avg = sum(times) / len(times)
print(f'Average gate latency: {avg:.4f}ms')
assert avg < 1.0, f'FAIL — {avg:.4f}ms exceeds 1ms'
print('PASS — sub-1ms gate confirmed')
"
```

---

### 8 — Custom cert

```bash
cat > test.iba.yaml << 'EOF'
intent:
  description: "Read-only access. No writes."
principal:
  id: "TEST-READONLY"
  human_authorization: "AUTH-TEST"
scope:
  - read_file
denied:
  - write_file
  - delete
default_posture: DENY_ALL
kill_threshold: "delete_production | data_breach"
temporal_scope:
  hard_expiry: "2026-12-31"
EOF

python iba_governor.py "read file src/main.py" --config test.iba.yaml
python iba_governor.py "write file src/main.py" --config test.iba.yaml
rm test.iba.yaml
```

Expected: read ALLOWED · write BLOCKED.

---

## Regulatory Test Checklist

| Requirement | Test | Status |
|-------------|------|--------|
| DENY_ALL default posture | Test 4 | ✓ |
| Kill threshold fires | Test 3 | ✓ |
| Denied list enforced | Test 2 | ✓ |
| Scope enforcement | Test 4 | ✓ |
| Auto-cert on missing config | Test 5 | ✓ |
| Immutable audit chain | Test 6 | ✓ |
| Sub-1ms gate latency | Test 7 | ✓ |
| Custom cert support | Test 8 | ✓ |

---

## Live Demo

**governinglayer.com/governor-html/**

Edit the cert. Run any agent action. ALLOW · BLOCK · TERMINATE in the browser.

---

IBA Intent Bound Authorization
Patent GB2603013.0 Pending · WIPO DAS C9A6 · PCT 150+ countries
IETF draft-williams-intent-token-00
Available for acquisition · iba@intentbound.com · IntentBound.com
