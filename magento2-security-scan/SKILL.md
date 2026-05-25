---
name: magento2-security-scan
description: |
  Scans Magento 2 code for security vulnerabilities and compliance issues. Use when:
  - "Security audit", "find vulnerabilities", "check for XSS"
  - "CSRF protection check", "validate form security"
  - "Dependency audit", "check for outdated packages"
  - "PCI compliance check", "security review before deploy"

  DEPENDENT on magento2-dev-core for security patterns.
compatibility: claude, codex, opencode, copilot
depends: [magento2-dev-core]
metadata:
  audience: developers
  workflow: magento
---

# Magento 2 Security Scanner

This skill performs security scanning for vulnerabilities, misconfigurations, and compliance issues.

## Vulnerability Categories

### 1. Injection Vulnerabilities

| Type | Pattern | Severity |
|------|---------|----------|
| SQL Injection | Direct SQL with user input | Critical |
| XSS (Reflected) | Unescaped user input in output | High |
| XSS (Stored) | Unescaped database content | Critical |
| Command Injection | System command execution with user input | Critical |

### 2. Authentication & Authorization

| Check | Command |
|-------|---------|
| Admin path changed | Check env.php for custom admin path |
| 2FA enabled | Check admin/twofactor module |
| Password hashing | Verify EncryptorInterface usage |
| Session security | Check cookie configuration |

### 3. Data Exposure

```bash
# Check for exposed sensitive files
curl -I https://store.test/app/etc/env.php
curl -I https://store.test/var/log/system.log
curl -I https://store.test/.git/config

# Expected: All should return 403 or 404
```

### 4. CSP Configuration

```bash
# Check CSP settings
govard sh -c "bin/magento config:get cms/wysiwyg/use_static_urls"
govard sh -c "bin/magento config:get design/content-security_policy/enable_content_security_policy"
```

## Scanner Commands

### PHP Security Checker

```bash
# Install security tools
composer require --dev magento/security-package --no-interaction

# Run Magento security scan
govard sh -c "vendor/bin/magento-security-scanner scan ./app/code/Vendor/Module"
```

### Dependency Audit

```bash
# Check for known vulnerabilities
govard sh -c "composer audit --no-interaction"

# Check for outdated packages with security fixes
govard sh -c "composer outdated --direct --format=json | jq '.[] | select(.[\"security-update\"]) | .name'"
```

### File Permission Check

```bash
# Check Magento file permissions
find var/ pub/static pub/media -type f -not -perm 0644 -ls
find var/ pub/static pub/media -type d -not -perm 0755 -ls
find app/etc -type f -not -perm 0640 -ls
```

## XSS Prevention Checklist

- [ ] All output escaped with `$escaper`
- [ ] `escapeHtml` for HTML content
- [ ] `escapeHtmlAttr` for attributes
- [ ] `escapeJs` for JavaScript strings
- [ ] `escapeUrl` for URLs
- [ ] Form keys in all POST forms
- [ ] No direct `$_GET`, `$_POST`, `$_REQUEST` usage

## CSRF Protection

### Verified Patterns

```php
// Form with form key
<form action="<?= $escaper->escapeUrl($action) ?>" method="post">
    <?= $block->getBlockHtml('formkey') ?>
</form>

// AJAX with form key
$.ajax({
    url: url,
    type: 'POST',
    data: {
        form_key: $.cookie('form_key'),
        ...data
    }
});
```

### Verify Admin Protection

```bash
# Check if admin form key validation is active
govard sh -c "bin/magento config:show admin/security/use_form_key"
# Expected: 1
```

## Rate Limiting

```bash
# Configure rate limiting
govard sh -c "bin/magento config:set web/secure/open_restriction_enabled 1"
```

## Security Headers

Check in nginx/Apache config or .htaccess:

```
X-Frame-Options: SAMEORIGIN
X-Content-Type-Options: nosniff
X-XSS-Protection: 1; mode=block
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: camera=(), microphone=(), geolocation=()
```

## PCI-DSS 4.0 Checklist (E-commerce)

From April 2025, payment pages require:

- [ ] `unsafe-eval` CSP disabled
- [ ] `unsafe-inline` CSP disabled
- [ ] No inline scripts in payment forms
- [ ] TLS 1.2+ enforced

## Quick Security Scan

Run this for a rapid security assessment:

```bash
# 1. Check for SQL in code
grep -r "execute\|fetch\|fetchAll\|select\|insert\|update\|delete" \
    app/code/Vendor/Module --include="*.php" | \
    grep -v "Collection\|Repository\|ResourceModel" | head -20

# 2. Check for superglobals
grep -r "\$_GET\|\$_POST\|\$_REQUEST" \
    app/code/Vendor/Module --include="*.php"

# 3. Check for ObjectManager
grep -r "ObjectManager::getInstance" \
    app/code/Vendor/Module --include="*.php"

# 4. Check for code evaluation functions
grep -r "eval" app/code/Vendor/Module --include="*.php"
```

## Compliance Report

```markdown
# Security Audit Report

## Injection Vulnerabilities
- [ ] No SQL injection vectors
- [ ] No XSS vulnerabilities
- [ ] No command injection

## Authentication
- [ ] Admin path changed from /admin
- [ ] 2FA enabled
- [ ] Strong password policy

## Data Protection
- [ ] Sensitive files not exposed
- [ ] File permissions correct
- [ ] CSP configured

## PCI-DSS (if applicable)
- [ ] CSP no unsafe-eval on checkout
- [ ] No inline scripts in payment forms
```

## Usage

**When to use:**
- "Security audit"
- "Check for vulnerabilities"
- "PCI compliance review"
- "Before production deploy"
- "After adding payment functionality"