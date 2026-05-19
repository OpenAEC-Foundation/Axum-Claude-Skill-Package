# Security Policy : Axum-Claude-Skill-Package

## Supported Versions

| Version | Supported |
|---------|-----------|
| 1.0.x | yes |
| < 1.0 | no |

## Reporting a Vulnerability

Skill packages contain code-generation guidance, not executable code. The primary security concern is : skill guidance that leads Claude to generate insecure code.

### How to report

- **For skill-content issues** (e.g. anti-pattern that allows SQL injection in generated code) : open a GitHub issue with `security` label
- **For sensitive issues** : email {{SECURITY_EMAIL}} with subject "SECURITY : Axum-Claude-Skill-Package"
- **For supply-chain issues** (compromised dependencies in `package.json`) : open a GitHub Security Advisory via the repo's Security tab

### Response time

- Acknowledgement : within 5 business days
- Fix or mitigation : within 30 days for high-severity, 90 days for lower-severity

## Out of scope

- Vulnerabilities in Axum itself : report to upstream
- Claude model behavior : report to Anthropic
- General LLM-prompt-injection in user input : application-side concern

## Disclosure

Once fixed, the vulnerability is disclosed in CHANGELOG.md and the GitHub Security Advisory is published.
