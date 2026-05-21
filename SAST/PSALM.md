# TryHackMe — SAST with Psalm

> **Module:** Static Application Security Testing (SAST)
> **Tool:** Psalm (PHP Static Analysis Linting Machine)
> **Target:** `simple-webapp` (PHP application)
> **Author:** Hlaing Minn Paing

---

## 1. Overview

Static Application Security Testing (SAST) analyzes source code **without executing it** to find security vulnerabilities and coding flaws. This note covers using **Psalm** on a PHP application to detect issues such as **SQL Injection (SQLi)** and **Local File Inclusion (LFI)**.

**Key idea:**
- **Manual review** → thorough but slow.
- **SAST tools** → fast but produce false positives/negatives.
- **Best practice:** Use SAST as a **complement** to manual review, never a replacement.

---

## 2. Setting Up Psalm

Psalm is configured via a `psalm.xml` file at the project root.

```xml
<?xml version="1.0"?>
<psalm
    errorLevel="3"
    resolveFromConfigFile="true"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns="https://getpsalm.org/schema/config"
    xsi:schemaLocation="https://getpsalm.org/schema/config vendor/vimeo/psalm/config.xsd"
    findUnusedBaselineEntry="true"
>
    <projectFiles>
        <directory name="html/" />
        <ignoreFiles>
            <directory name="vendor" />
        </ignoreFiles>
    </projectFiles>
</psalm>
```

**Configuration breakdown:**

| Option | Purpose |
|---|---|
| `errorLevel="3"` | Strictness level (lower = more issues reported) |
| `<directory name="html/" />` | Scan target directory |
| `<ignoreFiles>vendor</ignoreFiles>` | Exclude third-party dependencies |

---

## 3. Running Psalm

### 3.1 Default (Structural) Analysis

```bash
cd /home/ubuntu/Desktop/simple-webapp/
./vendor/bin/psalm --no-cache
```

**Finds coding errors**, e.g.:

```
ERROR: TypeDoesNotContainType - html/hidden-panel.php:10:5
  if ($result->num_rows = 0) {
```

→ Assignment (`=`) used instead of comparison (`==`). Always evaluates false.

### 3.2 Taint Analysis (Security)

```bash
./vendor/bin/psalm --no-cache --taint-analysis
```

**Finds security vulnerabilities** by tracing data flow from **source** → **sink**.

**Example finding — LFI:**
```
ERROR: TaintedInclude - html/view.php:22:9
  $_GET['img'] - html/view.php:22:28
  include('./gallery-files/'.$_GET['img']);
```

---

## 4. Core Concept: Source → Sink

| Term | Definition | Example |
|---|---|---|
| **Source** | Untrusted input entry point | `$_GET`, `$_POST`, `$_COOKIE` |
| **Sink** | Dangerous function consuming data | `include()`, `mysqli_query()`, `eval()` |
| **Taint flow** | Data path from source to sink | `$_GET['img']` → `include()` |

The --taint-analysis is the security one. "Taint" means "dirty data from the user." Psalm traces user input from the source (where it enters) to the sink (where it does damage).

**Vulnerability = Tainted data reaches a sink without sanitization.**

---

## 5. False Positives vs. False Negatives

| Term | Meaning | Risk |
|---|---|---|
| **False Positive** | Tool flags a vulnerability that **does not exist** | Wasted triage time |
| **False Negative** | Tool **misses** a real vulnerability | Critical — undetected risk in production |

**Rule of thumb:** False negatives are more dangerous than false positives.

---

## 6. Improving Detection with Annotations

Psalm may miss vulnerabilities when custom wrapper functions are used. Example — `db_query()` wraps `mysqli_query()`, but Psalm doesn't know it's a sink.

### Fix — Add annotations:

```php
/**
 * @psalm-taint-sink sql $query
 * @psalm-taint-specialize
 */
function db_query($conn, $query){
    $result = mysqli_query($conn, $query);
    return $result;
}
```

| Annotation | Purpose |
|---|---|
| `@psalm-taint-sink sql $query` | Marks `$query` as a SQL sink — flag any tainted input reaching it |
| `@psalm-taint-specialize` | Treat each function call as a separate finding |

### Result after annotation
Psalm now reports **3 TaintedSQL errors**:
1. `$_GET['guest_id']` → `db_query()` via `$sql`
2. `$_GET['log_id']` → `db_query()` via `$sql2`
3. Original `mysqli_query()` alert (built-in sink still triggers)

---

## 7. Manual vs Tool Comparison

| Variable | Manual Review | Psalm | Verdict |
|---|---|---|---|
| `$sql` | Vulnerable | Vulnerable | ✅ Correct |
| `$sql2` | Not Vulnerable | Vulnerable | ⚠️ False Positive |
| `$sql3` | Vulnerable | Not Vulnerable | ❌ False Negative |

**Conclusion:** Tools and humans both make mistakes — combine both for reliable coverage.

---

## 8. Vulnerabilities Found in This Module

| File | Line | Vulnerability | Source | Sink |
|---|---|---|---|---|
| `html/view.php` | 22 | Local File Inclusion (LFI) | `$_GET['img']` | `include()` |
| `html/hidden-panel.php` | 6 | SQL Injection | `$_GET['guest_id']` | `mysqli_query()` |
| `html/hidden-panel.php` | 19 | SQL Injection (FP) | `$_GET['log_id']` | `mysqli_query()` |

---

## 9. Key Takeaways

- SAST tools accelerate vulnerability discovery but require tuning.
- Always pair tool output with **manual verification**.
- Use **annotations / custom rules** to teach the tool about your codebase.
- Track both **False Positives** and **False Negatives** as part of SAST program maturity.
- A successful SAST program is measured by **signal-to-noise ratio**, not raw finding count.

---

## 10. Related Tools in My Environment

| Language | SAST Tool |
|---|---|
| PHP (Laravel) | SonarQube + Psalm |
| Java (Spring Cloud Gateway) | SonarQube + Semgrep |
| Generic | Semgrep, CodeQL, Snyk Code |

---

## References

- [Psalm Documentation](https://psalm.dev/docs/)
- [Psalm Taint Analysis](https://psalm.dev/docs/security_analysis/)
- [OWASP Source Code Analysis Tools](https://owasp.org/www-community/Source_Code_Analysis_Tools)
- TryHackMe — SAST Room
