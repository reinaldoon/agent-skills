---
name: bug-intake
description: >
  Transform non-technical CS bug reports into strong, actionable GitHub issues.
  Use when: (1) a CS agent or non-technical person reports a bug in vague language,
  (2) there are screenshots, recordings, error messages, or behavioral descriptions to parse,
  (3) the goal is to check GitHub for duplicates and open a well-structured issue,
  (4) the team title pattern `[scope] bug: short description` must be followed.
  Triggers on: bug report, CS report, user complaint, error screenshot, issue intake, bug intake.
---

# Bug Intake

Transform non-technical bug reports into structured, actionable GitHub issues.

## Role

Act as the team's Bug Reporter. Translate confused reports into clear issues, extract evidence without inventing details, check for duplicates, and open strong issues when needed.

**Never** do deep debugging, invent root cause, or assert anything unsupported by evidence.

## Mandatory Principles

1. **Never invent** — Mark unknowns as `Desconhecido`. No gap-filling with assumptions.
2. **Separate observed from inferred** — `Observed:` what the user saw/clicked/read. `Inferred:` your hypothesis (marked clearly). `Unknown:` what's missing.
3. **Prioritize visible evidence** — From screenshots/recordings extract: visible text, button names, field labels, URL/route, toast/modal/error messages, timestamps, environment, browser, device, visual state (stuck button, infinite loading, missing item, wrong badge).
4. **No weak language** — Avoid titles like "ajustes", "coisa errada", "erro no sistema", "bug geral", "problema no app". Use concrete symptoms.
5. **Controlled scope** — Pick scope from: `frontend`, `backend`, `api`, `worker`, `infra`, `db`, `auth`, `billing`, `shared`, `docs`, `mobile`, `triage`. Default to `triage` when uncertain.
6. **Fixed title pattern** — Always: `[scope] bug: short description`

## Workflow

### Step 1 — Normalize the report

Convert raw report into operational language. Extract:

- **Problem summary**
- **Observed behavior**
- **Expected behavior**
- **Reproduction steps**
- **Impact**
- **Frequency**
- **Environment**
- **Evidence**
- **Unknowns**

Rules:
- "Didn't work" → translate to the actual symptom
- "It froze" → specify what froze: screen, button, flow, upload, loading, navigation, response
- Multiple problems → separate into individual bugs

### Step 2 — Choose scope and title

Format: `[scope] bug: short description`

Rules: lead with main symptom, one idea per title, avoid presumed cause, avoid vagueness.

Good: `[frontend] bug: save button stays in infinite loading`
Bad: `[frontend] bug: fix screen issue`

### Step 3 — Check duplicates on GitHub

First identify the repository:

```bash
gh repo view --json nameWithOwner,url,defaultBranchRef
```

Fallback: `git remote get-url origin`

If no repo context, stop and report inability to proceed.

Search with multiple queries (symptom, exact error text, page+symptom, Portuguese term, English equivalent):

```bash
gh issue list --state all --search 'main symptom keywords'
gh issue list --state all --search '"exact error text"'
```

**Duplicate criteria:** same area/flow + same symptom + same conditions + same user-perceived effect.

If merely similar but not clearly identical → treat as **related**, not duplicate.

**When duplicate found:** Do not open new issue. Cite existing issue, explain why it's a duplicate, add new evidence via comment if useful:

```bash
gh issue comment ISSUE_NUMBER --body 'New CS report with additional evidence:
- Impact: [fill]
- Frequency: [fill]
- Additional evidence: [fill]
- Observed steps: [fill]'
```

### Step 4 — Build the issue body

Use this template:

```md
## Summary
[Short objective summary]

## Observed behavior
[What actually happened]

## Expected behavior
[What should happen]

## Reproduction steps
1. [step 1]
2. [step 2]
3. [step 3]

## Impact
- Affected user/group: [who or Desconhecido]
- Business impact: [blocks flow / degrades flow / confuses user / trust loss / Desconhecido]
- Frequency: [always / intermittent / once / Desconhecido]

## Evidence
- Reporter statement: [faithful summary]
- Screenshot/video evidence: [what's visible]
- Visible error text: [exact text or Desconhecido]
- Route/page/feature: [route/screen or Desconhecido]

## Environment
- Environment: [prod / staging / local / Desconhecido]
- Device: [Desktop / Mobile / Desconhecido]
- Browser/App: [name/version if available / Desconhecido]
- Timestamp: [when it occurred or Desconhecido]

## Scope guess
[chosen scope and why, 1 line]

## Duplicate check
- Queries used:
  - `[query 1]`
  - `[query 2]`
  - `[query 3]`
- Result: [no strong duplicate found / related issue found: #123]

## Unknowns
- [item 1]
- [item 2]

## Notes
- Report consolidated from CS report.
- Root cause not confirmed.
```

Rules: be factual, be short, preserve uncertainty, no blame, no technical cause without evidence.

### Step 5 — Labels

List existing labels first:

```bash
gh label list
```

Use only labels that already exist. Common candidates: `bug`, `needs-triage`, `source:cs`, `frontend`, `backend`, `api`, `infra`. Never invent labels.

### Step 6 — Upload images

If the bug report includes screenshots or images, upload them to S3 before creating the issue.

**Pre-requisite:** AWS CLI configured with `s3:PutObject` permission on the bucket.

**Bucket:** `s3://attachments.riasistemas.com.br/github-issues/`
**Public URL:** `https://attachments.riasistemas.com.br/github-issues/`

#### PII check (mandatory before upload)

Before uploading, inspect the screenshot for sensitive data: tokens, passwords, CPF, client names, OAB numbers, process numbers, financial data. If PII is present, describe the evidence textually instead of uploading.

#### Validation

- Allowed extensions: `.png`, `.jpg`, `.jpeg`, `.gif`, `.webp`
- Maximum file size: 10MB
- If file fails validation, describe textually and add label `needs-screenshot`

#### Upload

```bash
# Extract real extension from source file
FILE="/path/to/image.png"
EXT="${FILE##*.}"
FILENAME="$(date +%Y%m%d)-$(uuidgen | cut -d'-' -f1 | tr '[:upper:]' '[:lower:]').${EXT}"
aws s3 cp "${FILE}" "s3://attachments.riasistemas.com.br/github-issues/${FILENAME}"
```

Verify the upload is accessible:

```bash
curl -s -o /dev/null -w "%{http_code}" "https://attachments.riasistemas.com.br/github-issues/${FILENAME}"
```

Reference in the issue body under Evidence:

```md
## Evidence
- Screenshot: ![description](https://attachments.riasistemas.com.br/github-issues/{FILENAME})
```

#### Rules

- Never commit images to the git repository
- Always use S3 for image hosting
- Always check for PII before uploading
- If `aws s3 cp` fails, describe the evidence textually and add label `needs-screenshot`

### Step 7 — Create the issue

```bash
cat > /tmp/bug-report.md <<'EOF'
[ISSUE BODY HERE]
EOF

gh issue create \
  --title "[scope] bug: short description" \
  --body-file /tmp/bug-report.md \
  --label bug \
  --label needs-triage
```

## Decision Rules

**Open new issue when:** no strong duplicate, clear enough symptom, impact justifies triage (even with gaps).

**Do NOT open when:** clearly identical issue exists, same problem without new evidence, information too insufficient to identify even the main symptom.

**Weak reports:** Do best effort. Use `scope = triage`, fill Unknowns section well. Do not block issue creation for perfectionism.

## Output Format

### Case 1 — Duplicate found

```
Issue existente encontrada: #123
Motivo: [same flow, same symptom, same condition]
Acao tomada: did not open new issue, added evidence to #123
```

### Case 2 — New issue created

```
Issue criada: #456
Titulo: [scope] bug: short description
Resumo: [1 line problem] / [1 line impact] / [1 line main unknowns]
```

### Case 3 — Cannot operate on GitHub

```
Nao foi possivel concluir a criacao da issue.
Motivo: [no repo / no gh auth / insufficient permissions]
Reporte estruturado pronto para abertura manual abaixo.
```

## Guardrails

- Never assert root cause without evidence
- Never open obvious duplicates
- Never use vague titles
- Never mix multiple bugs in one issue
- Never hide uncertainty
- Always prioritize operational clarity
- Always use `[scope] bug: short description` pattern
