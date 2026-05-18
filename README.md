# compliancedoc Compliance Documenter

compliancedoc is a VS Code extension plus backend service for producing compliance-aware code explanations, documentation, refactoring guidance, and audit reports for financial-services software.

It analyzes selected code against configured frameworks such as FINRA, SEC, SOX, PCI-DSS, GLBA, CFTC, and GDPR, then returns structured output that can be reviewed, copied, inserted into source code and stored as audit evidence.

> Output should be reviewed by a qualified compliance officer before it is relied on for regulatory submissions or production sign-off.
---

## Contents

- [Product Overview](#product-overview)
- [compliancedoc Features](#vs-code-extension-features)
- [Backend Features](#backend-features)
- [Supported Compliance Frameworks](#supported-compliance-frameworks)
- [Core Workflows](#core-workflows)
- [Commands](#commands)
- [API Surface](#api-surface)
- [Plans and Limits](#plans-and-limits)
---

## Product Overview

The project has two main parts:

| Part | Path | Responsibility |
| --- | --- | --- |
| VS Code extension | `finance/documenter-extension` | Captures selected code, manages sign-in, runs compliance actions, displays results, inserts generated docs, stores local history, and exposes commands/status UI. |
| Backend API | Authenticates users, enforces quotas, queues generation jobs, stores documents and rules, handles billing, and serves audit/history endpoints. |

The extension sends selected code and metadata to the configured backend. The backend validates the request, applies the user's active compliance frameworks and rules, generates the result asynchronously, stores the document, and returns it to the extension.

---

## compliancedoc Features

### compliancedoc `CD:` Actions

compliancedoc `CD:` provides four primary code-analysis actions:

| Feature | Output | Purpose |
| --- | --- | --- |
| Explain | Markdown | Plain-English explanation for compliance officers, auditors, and non-technical reviewers. |
| Document | JSDoc | Insertable compliance documentation block for the selected function or code path. |
| Refactor | Markdown plus code | Specific remediation guidance and a compliant refactored code example. |
| Audit | Markdown report | Formal audit-style report for internal review or regulatory preparation. |

### Explain Code

Explains selected code in plain English for compliance officers, auditors, product owners, and other reviewers who do not want to read implementation details line by line.

Use this when you need to understand what a function does, what data it touches, and whether it appears to create compliance risk.

The output includes:

- What the code does
- Data handled and sensitivity classification
- Compliance flags with rule references
- Audit trail assessment

The explanation avoids developer-only shorthand where possible and calls out visible controls such as logging, authorization checks, validation, masking, encryption, or retention behavior. If a control is not visible in the selected code, the feature should treat it as missing or not evidenced.

### Generate Docs

Generates permanent compliance documentation for the selected function. This feature is designed to produce a strict JSDoc block that can be inserted directly above source code and kept in version control as review evidence.

Use this when a regulated code path needs source-level documentation that explains its business purpose, regulatory context, data classification, audit expectations, and change-control concerns.

The extension validates and normalizes the returned block before insertion.

The generated documentation includes:

- `@function` and `@description`
- Compliance tags with rule codes and severity
- Data classification, PII, and financial data notes
- Risk level and audit-trail status
- Parameters, return value, throws, and compliant usage example

If the AI response does not return valid JSDoc, the extension builds a fallback JSDoc block from the analysis so the user still receives insertable documentation.

For successful Document generations, the extension inserts the JSDoc above the selected function, preserves indentation, and replaces an existing adjacent JSDoc block when one is already present.

### Suggest Refactoring

Reviews the selected code for compliance gaps and proposes concrete remediation steps. Unlike Explain, this feature is developer-facing: it focuses on what should change and includes a refactored code example.

Use this before commit, during audit remediation, or when planning compliance-related technical debt work.

The output includes:

- Compliance risks found
- PII/PHI handling issues
- Recommended changes
- Refactored code
- Changes requiring compliance officer sign-off
- Testing recommendations

Recommendations are tied to specific rule references when applicable and should prioritize higher-severity gaps first. The generated refactored code is intended as a starting point for developer review, not an automatic patch.

### Generate Audit Report

Produces a formal audit-style report for internal audit, compliance review, regulator preparation, or sign-off discussions. This is the most comprehensive of the four features.

Use this when selected code needs to be assessed as part of an examination, control review, release gate, or evidence package.

The report includes:

- Executive summary
- System under review
- Regulatory mapping table
- Compliance gaps
- Audit trail analysis
- Access control assessment
- Data protection assessment
- Sign-off readiness
- Recommended actions
- Examiner questions the code should answer

The report is written for compliance and audit audiences. It maps the selected code to applicable frameworks, identifies missing evidence, and states whether the code appears ready for audit or needs remediation.

### Result Panel

The side panel displays generation results beside the editor. It supports:

- Compliance flag badges
- Token usage and cache indicator
- Copy output to clipboard
- Submit thumbs-up or thumbs-down feedback

### Automatic JSDoc Insertion

For the Document feature, the extension automatically inserts validated JSDoc above the selected function.

Insertion behavior:

- Locates the nearest function-like declaration above the selection
- Preserves indentation
- Replaces an existing JSDoc block immediately above the function when present
- Falls back to inserting at the selection start if no declaration is found
- Shows diagnostics in the `AI Compliance Documenter` output channel if insertion fails

Detected function patterns include JavaScript/TypeScript functions, arrow functions, class methods, Python `def`, Go `func`, and common Java/C# style method declarations.

### Framework Selection

Users can configure active frameworks: `CD: Set Compliance Frameworks`.

Supported framework values:

- `finra`
- `sec`
- `sox`
- `pci-dss`
- `glba`
- `cftc`
- `gdpr`

The backend only generates documents when at least one valid framework is configured.

### Status Bar

The extension adds a status bar item that shows:

- Sign-in state
- Plan tier
- Monthly usage
- Active framework list in the tooltip

The status refreshes at activation and every five minutes.

### History

The extension has both remote and local history support:

- Remote history comes from `/documents/history`
- Local history is stored in VS Code global state
- Local entries are scoped by a hash of the current license key
- The extension keeps up to 50 local history items
- History details can be reopened

### Custom Rules

Pro users can create and manage personal compliance rules from the extension panel.

Rule fields:

- Rule name
- Applicability: global or framework-specific
- Framework
- Optional rule code
- Description
- Prompt hint
- Severity
- Active/inactive state

Custom rules are stored by the backend and can be created, edited, toggled, or deleted.

---

## Backend Features

### Quotas

- Free tier has a `fixed` 10/month operations limit
- Pro tier reports `Unlimited`
- The extension displays usage in the status panel and status bar

### Asynchronous Document Generation

Document generation is queued instead of handled synchronously.

Flow:

1. Extension calls
2. Backend validates license, quota, request body, and frameworks
3. Backend enqueues a `generate-document` job
4. Job status is persisted through the job store
5. Completed result is returned and shown in VS Code

### Generation

The backend uses the `Anthropic SDK` to generate compliance-aware output. Generation uses:

- Selected source code
- Feature type: explain, document, refactor, or audit
- Programming language
- User's configured frameworks
- Built-in and personal compliance rules

The extension also supplies a strict output contract so results can be validated and inserted reliably.

### Feedback

Users can submit feedback for a generated document:

- `thumbs_up`
- `thumbs_down`
- Optional comment


### Compliance Rules

The backend serves both `built-in` and `user-created` rules.

It supports:

- Listing all applicable rules grouped by framework
- Listing personal rules
- Creating personal rules
- Updating personal rules
- Deleting personal rules
- Enabling/disabling rules

## Supports 7 Compliance Frameworks

The product recognizes these framework families:

| Framework | Examples of covered concerns |
| --- | --- |
| FINRA | Supervisory controls, business continuity, trade review workflows |
| SEC | Records retention, immutable financial records, auditability |
| SOX | Executive certification, internal controls, financial reporting changes |
| PCI-DSS | Cardholder data protection, secure development, masking/tokenization |
| GLBA | Consumer financial data safeguards and privacy controls |
| CFTC | Swap records, transaction information availability, business conduct records |
| GDPR | Security of processing, privacy by design, erasure/privacy implications |

The prompt catalog in the extension includes rule references such as `FINRA-3110`, `SEC-17a-4`, `SOX-302`, `SOX-404`, `PCI-3.4`, `GLBA-501`, `CFTC-1.31`, `CFTC-23.400`, `CFTC-23.606`, and `GDPR-Art32`.

---

## Core Workflows

### First Use

1. Install or run the VS Code extension.
2. Open `CD: Register / Sign In`.
3. Create an account or Sign In.
4. Select your compliance frameworks.
5. Select code in the editor.
6. Run one of the `CD:` Compliance commands.

### Generate Insertable Documentation

1. Select a function or code block.
2. Run `CD: Generate Docs (Compliance)`.
3. The backend generates JSDoc.
4. The extension validates the JSDoc.
5. The extension inserts it above the selected function.

### Produce an Audit Report

1. Select the code under review.
2. Run `CD: Generate Audit Report`.
3. Review the report in the side panel.
4. Provide feedback.

### Manage Custom Rules

1. Open `CD: Manage Custom Compliance Rules`.
2. Add, edit, disable, or delete personal rules (`Global` or `framework-specific`).
3. Future generations include active applicable rules.

---

## Commands

All commands are available from the Command Palette. The four analysis commands also appear `CD:` in the editor context menu when code is selected.

---

## API Surface

### Auth

Authenticate with email/password.

### Documents

Generate a document. |
List the user's most recent documents. |
Fetch stored documents. |
feedback for a generated document. |

### Compliance

|Personal rules | `Global` or `framework-specific`. | List the user's personal rules. |
| Create a personal rule. | Update a personal rule. |
| Delete a personal rule. |
| Enable or disable a personal rule. |

---

## Plans and Limits

| Capability | Free | Pro |
| --- | --- | --- |
| Explain, Document, Refactor, Audit | Yes | Yes |
| Supported 7 compliance frameworks | Yes | Yes |
| JSDoc insertion | Yes | Yes |
| History | Yes | Yes |
| Monthly generation quota | Configured by backend | Unlimited |
| Custom personal rules (`Global` or `framework-specific`) | No | Yes |

---
