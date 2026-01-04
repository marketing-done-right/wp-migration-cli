# WordPress Migration Runbook

**Breakdance Builder – Large Site URL Replacement (WP-CLI)**

| Field | Value |
|-------|-------|
| Audience | Engineering / DevOps / Web Team |
| Stack | DigitalOcean LEMP (Nginx, PHP-FPM, MySQL) |
| Builder | Breakdance Builder |
| Use case | Dev → Production migrations on large WordPress sites |
| Validated on | StormGuardRC.com (~3.2 GB DB, 2,000+ pages) |

---

## Purpose

This runbook documents the exact, repeatable procedure to migrate a large WordPress site built with Breakdance Builder from a development domain to production using WP-CLI only.

It avoids:

- wp-admin timeouts
- partial URL replacements
- broken Breakdance layouts

---

## Preconditions

- Database already imported (`wp db import`)
- WP-CLI available
- Correct document root (`/site`)
- Site not yet publicly switched (safe to run)

---

## Domains Used (Example)

| Environment | URL |
|-------------|-----|
| Old (dev) | `https://dev.stormguardrc.com` |
| New (prod) | `https://stormguardrc.com` |

> Replace with your own domains as needed.

---

## Step 1 — Confirm WordPress is detected

```bash
wp core is-installed
```

**Why:** Confirms WP-CLI is operating against a valid WordPress installation. This command is silent on success.

---

## Step 2 — Check current site URLs

```bash
wp option get siteurl
wp option get home
```

**Expected (before migration):**

```
https://dev.stormguardrc.com
```

**Why:** Verifies the imported database still points to the dev domain.

---

## Step 3 — Increase PHP limits for WP-CLI

```bash
export WP_CLI_PHP_ARGS='-d memory_limit=2048M -d max_execution_time=0'
```

**Why:** Large databases + serialized data can exceed default PHP limits. This applies only to WP-CLI, not PHP-FPM.

---

## Step 4 — Update core WordPress URLs

```bash
wp option update siteurl 'https://stormguardrc.com'
wp option update home 'https://stormguardrc.com'
```

**Why:** WordPress core must point to the new domain before global replacements.

---

## Step 5 — Replace HTTPS URLs (dry-run)

```bash
wp search-replace 'https://dev.stormguardrc.com' 'https://stormguardrc.com' \
  --all-tables-with-prefix \
  --skip-columns=guid \
  --precise \
  --recurse-objects \
  --report-changed-only \
  --dry-run
```

**Expected output:**

```
Success: 18885 replacements to be made.
```

**Why this matters:**

- Validates scope
- Shows exact impact
- Prevents accidental corruption

---

## Step 6 — Execute HTTPS replacement

```bash
wp search-replace 'https://dev.stormguardrc.com' 'https://stormguardrc.com' \
  --all-tables-with-prefix \
  --skip-columns=guid \
  --precise \
  --recurse-objects \
  --report-changed-only
```

**Expected output:**

```
Success: Made 18885 replacements.
```

**Important flags:**

| Flag | Purpose |
|------|---------|
| `--all-tables-with-prefix` | includes plugin tables |
| `--skip-columns=guid` | preserves WordPress GUIDs |
| `--precise` + `--recurse-objects` | safe for serialized / JSON data (critical for Breakdance) |

---

## Step 7 — Handle legacy HTTP URLs

```bash
wp search-replace 'http://dev.stormguardrc.com' 'https://stormguardrc.com' \
  --all-tables-with-prefix \
  --skip-columns=guid \
  --precise \
  --recurse-objects \
  --report-changed-only
```

**Expected output:**

```
Success: Made 5 replacements.
```

**Why:** Catches any leftover non-HTTPS references.

---

## Step 8 — Verify remaining dev references

```bash
wp db query "SELECT COUNT(*) AS c FROM wp_mdr_posts WHERE post_content LIKE '%dev.stormguardrc.com%';"
```

**Result:** `0`

```bash
wp db query "SELECT COUNT(*) AS c FROM wp_mdr_postmeta WHERE meta_value LIKE '%dev.stormguardrc.com%';"
```

**Result:** `21641`

```bash
wp db query "SELECT COUNT(*) AS c FROM wp_mdr_options WHERE option_value LIKE '%dev.stormguardrc.com%';"
```

**Result:** `5`

**Interpretation:** Breakdance stores many URLs as host-only values (no `http://` or `https://`). These are not handled by standard replacements.

---

## Step 9 — Replace host-only URLs (critical for Breakdance)

### Dry-run

```bash
wp search-replace 'dev.stormguardrc.com' 'stormguardrc.com' \
  --all-tables-with-prefix \
  --skip-columns=guid \
  --precise \
  --recurse-objects \
  --report-changed-only \
  --dry-run
```

**Expected:**

```
Success: 23535 replacements to be made.
```

### Execute replacement

```bash
wp search-replace 'dev.stormguardrc.com' 'stormguardrc.com' \
  --all-tables-with-prefix \
  --skip-columns=guid \
  --precise \
  --recurse-objects \
  --report-changed-only
```

**Expected:**

```
Success: Made 23535 replacements.
```

**Why this step is essential:** This replaces Breakdance internal references and avoids using the Breakdance admin URL replacement tool (which often times out on large sites).

---

## Step 10 — Final verification (must be zero)

```bash
wp db query "SELECT COUNT(*) AS c FROM wp_mdr_postmeta WHERE meta_value LIKE '%dev.stormguardrc.com%';"
```

**Expected:** `0`

```bash
wp db query "SELECT COUNT(*) AS c FROM wp_mdr_options WHERE option_value LIKE '%dev.stormguardrc.com%';"
```

**Expected:** `0`

> ⚠️ **If not zero → stop and investigate before proceeding.**

---

## Step 11 — Flush rewrites and cache

```bash
wp rewrite flush
wp cache flush
```

**Notes:**

- Safe on Nginx (no `.htaccess` required)
- Ensures clean routing and cached content

---

## Step 12 — Final confirmation

```bash
wp option get siteurl
wp option get home
```

**Expected:**

```
https://stormguardrc.com
```

---

## Outcome

- ✅ All URLs updated correctly
- ✅ Breakdance layouts intact
- ✅ No wp-admin timeouts
- ✅ Fully CLI-based and repeatable
- ✅ Safe for large databases
- ✅ Compatible with Sucuri WAF + Nginx

---

## Key Takeaway

> **For Breakdance migrations, always replace the host without scheme.**
> Replacing only `https://` URLs is not sufficient.
