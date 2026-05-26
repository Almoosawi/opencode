---
title: "OpenCode v1.15.10 — Security Assessment & Hardening Report"
subtitle: "Fortify SSC Triage and Remediation Summary for Corporate Deployment"
author: "Application Security Team"
date: "2026-05-26"
---

# 1. Purpose

This document summarizes the security assessment of **OpenCode v1.15.10**, an open-source AI coding agent (MIT license), performed to support corporate deployment approval. It covers Fortify SSC static analysis findings, applied hardening measures, and residual risk.

# 2. Scope

| Item | Detail |
|---|---|
| Product | OpenCode Desktop v1.15.10 (Electron + TypeScript) |
| Source | `anomalyco/opencode` (GitHub, MIT license) |
| Hardened Fork | `Almoosawi/opencode` (private), branch `hardening/runtime` |
| Scan Tool | Fortify Static Code Analyzer via Fortify SSC |
| SSC Project Version | 16664 |
| Total Findings | 158 |
| Assessment Date | May 2026 |

# 3. Executive Summary

Fortify SSC identified 158 static analysis findings. After manual source-level triage:

- **127 (80.4%) are false positives** — mock API keys in test files and placeholder credentials in documentation
- **16 (10.1%) are mitigated** by hardening changes committed to the fork
- **15 (9.5%) are accepted risk** with documented justification (RFC-compliant patterns, standard UI conventions)
- **0 require code fixes**

All hardening measures have been implemented and committed. The fork is ready for deployment pipeline integration.

# 4. Finding Disposition Summary

| Disposition | Count | Percentage |
|---|---|---|
| False Positive — test fixture | 86 | 54.4% |
| False Positive — documentation | 38 | 24.1% |
| False Positive — test support file | 3 | 1.9% |
| Mitigated by hardening | 16 | 10.1% |
| Accepted risk (documented) | 15 | 9.5% |
| Requires fix | 0 | 0.0% |
| **Total** | **158** | **100%** |

## 4.1 False Positive Breakdown (127 findings)

86 findings flag mock API keys (`"sk-test"`, `"anthropic-key"`) in `.test.ts` files. 38 flag example configuration in `.mdx` documentation. 3 flag test infrastructure files in `test/` directories. None contain real credentials.

**Recommendation:** Add exclusion patterns to future Fortify scan profiles (`**/*.test.ts`, `**/*.mdx`, `**/test/**`, `packages/containers/**`, `packages/docs/**`) to eliminate 80% false positive noise.

## 4.2 Mitigated Findings (16 findings)

| Finding Category | Count | Hardening Measure |
|---|---|---|
| Privacy Violation (telemetry) | 2 | Sentry SDK fully removed from desktop + app packages |
| Empty Password (share feature) | 2 | Share feature controlled by enterprise managed config |
| Dockerfile: Default User Privilege | 6 | `USER` directives added to all 6 Dockerfiles |
| Dockerfile: Dependency Confusion | 6 | Production image pinned to `alpine:3.21`; CI images use versioned registry tags |

## 4.3 Accepted Risk (15 findings)

| Finding Category | Count | Justification |
|---|---|---|
| Insecure Transport (localhost HTTP) | 6 | RFC 8252 Section 7.3 — `http://localhost` is the standard pattern for OAuth 2.0 PKCE loopback redirects. Traffic never leaves the local machine. PKCE + state validation provide CSRF protection. |
| Empty Password (UI form init) | 5 | Standard UI form pattern — password fields initialize empty. Not a credential bypass. |
| Path Manipulation | 1 | Input is internally sourced (editor/terminal app detection). `execFileSync` with array arguments prevents injection. No shell interpolation. |
| Hardcoded Password (help text) | 1 | `"your-client-secret"` is example text in CLI help output. Actual secret collection uses masked `prompts.password()`. |
| OpenAPI Misconfiguration | 2 | Authentication is enforced at HTTP middleware layer (Basic auth + per-session random password). OpenAPI spec is for SDK code generation, not API gateway enforcement. |

# 5. Hardening Measures Applied

All changes are on branch `hardening/runtime` in the private fork (`Almoosawi/opencode`), based on tag `v1.15.10`.

| Commit | Measure | Files Changed |
|---|---|---|
| `c0d8619d7` | Auto-updater removal — all electron-updater code deleted, IPC bridges removed, preload API stripped | 9 files |
| `8f1a9a848` | Sentry telemetry removal — `@sentry/solid` and `@sentry/vite-plugin` fully removed from desktop + app | 8 files |
| `f9d77f715` | Network egress disabled — release notes fetch short-circuited, `releaseNotes` default set to `false` | 2 files |
| `a6999e6c2` | CI build pipeline — GitHub Actions workflow with CI-only code signing gate, CycloneDX SBOM generation, build provenance manifest | 1 file |
| `707f3044c` | Dockerfile hardening — non-root users in all 6 Dockerfiles, production base image pinned to `alpine:3.21` | 6 files |

## 5.1 Enterprise Managed Config

OpenCode includes a managed configuration system that reads policy from `%ProgramData%\opencode\opencode.json`. This enables IT-controlled settings including:

- **Provider allowlist** — restrict which LLM providers can be used
- **Model restrictions** — limit available models
- **Plugin control** — disable or restrict plugin loading
- **Permission defaults** — set auto-approve policies
- **Feature toggles** — disable sharing, telemetry, experimental features

A full schema investigation with deployable GPO JSON template is available in `docs/security/managed-config-schema.md`.

## 5.2 Software Bill of Materials (SBOM)

The CI pipeline generates a CycloneDX 1.7 SBOM via `@cyclonedx/cdxgen`. Local validation confirmed 305 components catalogued. The SBOM is attached as a build artifact on each tagged release (`pdo-v*` tags).

# 6. Residual Risk Assessment

| Risk | Severity | Mitigation |
|---|---|---|
| OAuth callbacks use HTTP on localhost | Low | Per RFC 8252 spec. PKCE + state validation in place. Localhost-only. |
| Server dialog allows empty password | Informational | UI form initialization. Connection to remote servers requires explicit user action. |
| OpenAPI spec lacks security declaration | Low | Auth enforced at middleware. Cosmetic spec issue — no runtime impact. |
| CLI help text shows placeholder secret | Informational | Example text only. Not a real credential. |
| Path resolution in app detection | Low | Internally sourced input. No shell interpolation. Filesystem validation before use. |

**Overall residual risk: Low.** All accepted findings are standard patterns (RFC-compliant, UI conventions) with no exploitable attack surface in the desktop deployment model.

# 7. Deployment Readiness

| Criterion | Status |
|---|---|
| Fortify findings triaged | All 158 dispositioned |
| Critical/high findings requiring fix | 0 |
| Auto-updater removed | Yes |
| Telemetry removed | Yes |
| Network egress controlled | Yes |
| CI pipeline with signing | Yes |
| SBOM generation | Yes |
| Enterprise managed config documented | Yes |
| Dockerfiles hardened | Yes |

# 8. Recommendations

1. **Update Fortify scan profile** with exclusion patterns to reduce false positive noise by 80% in future scans.
2. **Re-scan after merge** to confirm finding remediation in the final build artifact.
3. **Configure code signing** — obtain a code signing certificate from PKI and configure `CSC_LINK`/`CSC_KEY_PASSWORD` as GitHub Actions secrets.
4. **Deploy managed config** — use the GPO JSON template from `managed-config-schema.md` to enforce provider allowlists and permission policies.
5. **Document RFC 8252 loopback pattern** in the organization's security architecture guide to streamline future OAuth-related findings.

# 9. References

- Fortify SSC detailed triage: `docs/security/fortify-triage-v1.15.10.md`
- Managed config schema: `docs/security/managed-config-schema.md`
- Hardened fork: `github.com/Almoosawi/opencode` (branch `hardening/runtime`)
- RFC 8252 — OAuth 2.0 for Native Apps, Section 7.3 (Loopback Redirect)
- CycloneDX SBOM specification v1.7
