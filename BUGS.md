# Bugs Found in check-my-process (cmp) CLI v1.1.1

## Summary

| # | Severity | Bug | Command |
|---|----------|-----|---------|
| 1 | High | `init` command not implemented | `cmp init` |
| 2 | High | No config schema validation | `cmp validate` |
| 3 | Medium | `validate` reports success without config file | `cmp validate` |
| 4 | Medium | Invalid `--format` values accepted silently | `cmp check` |
| 5 | Medium | Negative/zero PR numbers accepted | `cmp check` |
| 6 | Medium | Multi-slash repo paths parsed incorrectly | `cmp check` |
| 7 | Low | Invalid `check_in` locations accepted | `cmp validate` |
| 8 | Low | Empty `check_in` array accepted | `cmp validate` |
| 9 | Low | Float values accepted for integer fields | `cmp validate` |
| 10 | Low | Unknown config fields silently ignored | `cmp validate` |
| 11 | Low | Invalid regex patterns not validated at config time | `cmp validate` |
| 12 | Low | GitHub error messages leak internal API details | `cmp check` |

---

## Bug Details

### Bug #1: `init` command not implemented

**Severity:** High
**Command:** `cmp init`

**Description:**
The `init` command is advertised in the help but does nothing useful. It just prints a placeholder message.

**Steps to Reproduce:**
```bash
cmp init
```

**Expected:** Creates a starter `cmp.toml` config file in the current directory.

**Actual:**
```
(init command coming in a future release)
```

---

### Bug #2: No config schema validation

**Severity:** High
**Command:** `cmp validate`

**Description:**
The `validate` command does not actually validate the config values. It only checks if the TOML syntax is valid. Invalid values, wrong types, and nonsensical configurations are accepted.

**Steps to Reproduce:**
```bash
cat << 'EOF' > /tmp/invalid.toml
[settings]
default_severity = "invalid_severity"

[pr]
max_files = -10
max_lines = "not a number"
min_approvals = 0

[branch]
pattern = "[invalid(regex"

[ticket]
pattern = "[A-Z"
check_in = ["invalid_location"]
EOF

cmp validate --config /tmp/invalid.toml
```

**Expected:** Validation errors for:
- Invalid severity value (should be "error" or "warning")
- Negative `max_files` value
- Non-numeric `max_lines` value
- Invalid regex patterns
- Invalid `check_in` location values

**Actual:**
```
Config is valid

Settings:
  default_severity: invalid_severity

PR rules:
  max_files: -10
  max_lines: not a number
...
```

---

### Bug #3: `validate` reports success without config file

**Severity:** Medium
**Command:** `cmp validate`

**Description:**
When no `cmp.toml` file exists, `validate` reports "Config is valid" and shows the default config. This is misleading because it suggests a config file was found and validated.

**Steps to Reproduce:**
```bash
cd /tmp && cmp validate
```

**Expected:** Either an error saying no config file was found, or a clear message indicating defaults are being used.

**Actual:**
```
Config is valid

Settings:
  default_severity: error
...
```

---

### Bug #4: Invalid `--format` values accepted silently

**Severity:** Medium
**Command:** `cmp check`

**Description:**
The `--format` option accepts any value, not just "text" or "json". Invalid formats fall through to text output without warning.

**Steps to Reproduce:**
```bash
GITHUB_TOKEN=fake cmp check --repo "owner/repo" --pr 1 --format invalid
GITHUB_TOKEN=fake cmp check --repo "owner/repo" --pr 1 --format XML
```

**Expected:** Error message saying format must be "text" or "json".

**Actual:** Proceeds with the command (fails on token validation, but format is not validated).

---

### Bug #5: Negative/zero PR numbers accepted

**Severity:** Medium
**Command:** `cmp check`

**Description:**
PR numbers of 0 or negative values are accepted and sent to the GitHub API, which will fail with a confusing error.

**Steps to Reproduce:**
```bash
GITHUB_TOKEN=fake cmp check --repo "owner/repo" --pr -1
GITHUB_TOKEN=fake cmp check --repo "owner/repo" --pr 0
```

**Expected:** Error message saying PR number must be a positive integer.

**Actual:**
```
GET /repos/owner/repo/pulls/-1 - 401 with id ...
Error: Invalid GitHub token
```

The error message is misleading - it blames the token when the real issue is the invalid PR number.

---

### Bug #6: Multi-slash repo paths parsed incorrectly

**Severity:** Medium
**Command:** `cmp check`

**Description:**
Repo paths with more than one slash (e.g., `a/b/c`) are parsed incorrectly. Only the first two segments are used, and the rest is silently ignored.

**Steps to Reproduce:**
```bash
GITHUB_TOKEN=fake cmp check --repo "a/b/c/d" --pr 1
```

**Expected:** Error message saying repo format is invalid.

**Actual:** Parses as owner=`a`, repo=`b`, silently ignores `/c/d`:
```
GET /repos/a/b/pulls/1/reviews - 401 ...
```

---

### Bug #7: Invalid `check_in` locations accepted

**Severity:** Low
**Command:** `cmp validate`

**Description:**
The `check_in` array for ticket validation accepts any string values, not just the valid locations: "title", "branch", "body".

**Steps to Reproduce:**
```bash
cat << 'EOF' > /tmp/test.toml
[ticket]
pattern = "TEST-[0-9]+"
check_in = ["invalid_field", "not_a_location"]
EOF

cmp validate --config /tmp/test.toml
```

**Expected:** Validation error for invalid `check_in` values.

**Actual:**
```
Config is valid
...
  check_in: invalid_field, not_a_location
```

---

### Bug #8: Empty `check_in` array accepted

**Severity:** Low
**Command:** `cmp validate`

**Description:**
An empty `check_in` array is accepted, which would make ticket validation always fail (or pass vacuously).

**Steps to Reproduce:**
```bash
cat << 'EOF' > /tmp/test.toml
[ticket]
pattern = "TEST-[0-9]+"
check_in = []
EOF

cmp validate --config /tmp/test.toml
```

**Expected:** Warning that `check_in` is empty.

**Actual:**
```
Config is valid
...
Ticket rules:
  pattern: TEST-[0-9]+
  check_in:
```

---

### Bug #9: Float values accepted for integer fields

**Severity:** Low
**Command:** `cmp validate`

**Description:**
Fields that should be integers (`max_files`, `max_lines`, `min_approvals`) accept float values.

**Steps to Reproduce:**
```bash
cat << 'EOF' > /tmp/test.toml
[pr]
min_approvals = 1.5
max_files = 10.7
EOF

cmp validate --config /tmp/test.toml
```

**Expected:** Validation error or warning that these should be integers.

**Actual:**
```
Config is valid
...
  min_approvals: 1.5
```

---

### Bug #10: Unknown config fields silently ignored

**Severity:** Low
**Command:** `cmp validate`

**Description:**
Unknown fields and sections in the config file are silently ignored without any warning. This could lead to typos going unnoticed.

**Steps to Reproduce:**
```bash
cat << 'EOF' > /tmp/test.toml
[settings]
default_severity = "error"
unknwon_typo_field = "value"

[unknown_section]
foo = "bar"

[pr]
max_fiels = 10
EOF

cmp validate --config /tmp/test.toml
```

**Expected:** Warning about unknown fields/sections (e.g., `max_fiels` is likely a typo for `max_files`).

**Actual:**
```
Config is valid
...
```
The typo `max_fiels` is ignored and default `max_files: 20` is used instead.

---

### Bug #11: Invalid regex patterns not validated at config time

**Severity:** Low
**Command:** `cmp validate`

**Description:**
Invalid regex patterns in `branch.pattern` or `ticket.pattern` are not validated by `cmp validate`. They will only fail at runtime during `cmp check`.

**Steps to Reproduce:**
```bash
cat << 'EOF' > /tmp/test.toml
[branch]
pattern = "[invalid(regex"
EOF

cmp validate --config /tmp/test.toml
```

**Expected:** Validation error for invalid regex pattern.

**Actual:**
```
Config is valid
...
  pattern: [invalid(regex
```

---

### Bug #12: GitHub error messages leak internal API details

**Severity:** Low
**Command:** `cmp check`

**Description:**
When GitHub API errors occur, the raw error messages including internal request IDs and API paths are shown to the user.

**Steps to Reproduce:**
```bash
GITHUB_TOKEN=fake cmp check --repo "owner/repo" --pr 1
```

**Expected:** Clean error message like "Invalid GitHub token" or "Authentication failed".

**Actual:**
```
GET /repos/owner/repo/pulls/1 - 401 with id A95B:1AA5C9:507FAC:5BF0F5:6939BD8B in 174ms
Error: Invalid GitHub token
```

The first line exposes internal API details that aren't useful to end users.

---

## Recommendations

1. **Implement the `init` command** - This is a core feature that users expect to work.

2. **Add proper config schema validation** - Validate:
   - `default_severity` is "error" or "warning"
   - Numeric fields are positive integers
   - Regex patterns are valid
   - `check_in` values are from allowed list
   - Warn on unknown fields/sections

3. **Improve error messages** -
   - Validate `--format` option choices
   - Validate PR number is positive
   - Validate repo format more strictly
   - Suppress internal API details from error output

4. **Add explicit "using defaults" message** - When no config file is found, clearly indicate that defaults are being used rather than saying "Config is valid".
