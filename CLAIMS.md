# Claims register

Every factual statement on sengol.io traces to a decision record. Copy that
makes a claim about the product, the licence, the trial, hosting, security or
an install command cites a row here; a change to the source updates the copy
in the same change, and a claim with no source does not ship.

Sources, in order of authority:

- `sengol/PRODUCT.md` — what the product is and is not, the two ways to start,
  the user journey, the feature list, what is deliberately excluded.
- `sengol/docs/adr/ADR-0001` … `ADR-0010` — the invariants.
- `sengol-ops/decisions/REPOSITORY-RESET-2026-09.md` — licence signing key,
  trust anchors, custody, staging.
- `sengol/docs/install/*` and `sengol/docs/reference/cli.mdx` — install
  commands, paths, CLI names. Where these disagree with the code, the code
  wins and the docs are the bug.

| Claim on the site | Pages | Source |
|---|---|---|
| Self-hosted, single-tenant appliance; one install is one tenant; not a multi-tenant SaaS | index, security, trial-terms | PRODUCT.md "What it is"; ADR-0006 |
| Two ways to start: hosted 14-day trial on Sengol's AWS, or download with a 14-day licence file | index, signup, trial-terms | PRODUCT.md "The two ways to start" |
| Trial is 14 days of full capability; nothing gated, metered or watermarked | index, signup, trial-terms | PRODUCT.md "The two ways to start"; ADR-0007 (no per-feature gates) |
| Hosted trial holds synthetic data only and is destroyed at day 14 | trial-terms | PRODUCT.md "The two ways to start" |
| Corporate email only, one trial per domain | signup, trial-terms | sengol-portal `portal/licensing.py` (issuance rules) |
| Proprietary from 2.0; no free tier and no feature tiers; one licence unlocks the whole appliance | index FAQ, trial-terms | product `decisions/LICENSING-MODEL-PROPRIETARY-2026-08-17.md`; ADR-0007 |
| Licence is a signed JWT (ES256) issued under a P-256 key held in Sengol's AWS KMS; only KMS-held anchors ship in a build | install | REPOSITORY-RESET-2026-09 §6; ADR-0007 |
| Licence verified offline at boot and re-read hourly; no phone-home, no telemetry | index, install, security, trial-terms | ADR-0007; PRODUCT.md "Licensing" |
| Expiry stops evaluation and gating; it never blocks reading, exporting or verifying evidence | install, trial-terms | PRODUCT.md "Licensing" and journey step 13 |
| `SENGOL_LICENSE_FILE`, default `/etc/sengol/sengol.lic`; Helm mounts the Secret named in `license.existingSecret` at `/etc/sengol/license/sengol.lic` | install | `sengol/docs/install/licence.mdx`; `sengol/helm/sengol/values.yaml` `license:` block |
| Install paths: Docker Compose from the repository, Helm chart at `oci://ghcr.io/sengol-io/charts/sengol`, Terraform stacks in `terraform/aws`, `/azure`, `/gcp` | index, install, signup | `sengol/docs/install/*`; `sengol/.github/workflows/publish.yml` (`helm-chart` job) |
| Evidence: HMAC-SHA256 hash chain, Ed25519 countersignatures, optional RFC 3161 / Rekor anchoring, append-only Postgres | index, security | PRODUCT.md "Evidence"; ADR-0005 |
| `sengol-verify` is a separate Apache-2.0 tool with no dependency on sengol and no licence check; invoked as `sengol-verify pack.json` | index, install, security, trial-terms | ADR-0010; `sengol-verify/README.md` |
| CLI names: `sengol gate`, `sengol export`, `sengol verify`; demo seeding is `make demo` | index, install | `sengol/docs/reference/cli.mdx`; `sengol/Makefile` |
| SDK entry points `sengol.configure()` and `sengol.instrument()` | index, install | `sengol/sengol/__init__.py` |
| 32 evaluators; deterministic first, LLM judges in one gather; CRITICAL blocks inline | index | PRODUCT.md "Evaluation"; ADR-0003 |
| 29 regulations in the catalog; custom policies alongside (open `PolicyId`) | index | PRODUCT.md "Policy"; ADR-0001 |
| Judge governance: reviewers' decisions export as labels, the customer trains, Sengol calibrates, qualifies and promotes by signed id; Sengol never trains a model | index | ADR-0009; PRODUCT.md "Judge governance" |
| Identity: Entra ID / OIDC app roles, SAML 2.0, SCIM users and groups with group-to-role mapping, local users with TOTP, audited break-glass, four built-in roles | index, security | ADR-0008; PRODUCT.md "Identity"; `sengol/sengol/api/v1/saml.py` |
| Segregation of duties on judge promotion (maker-checker) | index | `sengol/sengol/governance/sod.py`; PRODUCT.md excludes only the configurable SoD *policy engine* |
| Retention floors and tombstone erasure; **no legal hold, no dual-control erasure, no scheduled retention** | index, security | PRODUCT.md "Data lifecycle" and "What is deliberately not included" |
| Signing keys in the customer's KMS; SBOM and signed images; non-root containers | security | PRODUCT.md "Operations"; `sengol/.github/workflows/publish.yml` (cosign + SBOM attestation) |
| Appliance image runs Python 3.13; the SDK supports Python 3.11+ | index | `sengol/Dockerfile`; `sengol/pyproject.toml` `requires-python`; `sengol/AGENTS.md` |
| Runs in your VPC, air-gap capable, on AWS, Azure, GCP, OpenShift or on-prem | index, security | PRODUCT.md "Install"; `sengol/terraform/{aws,azure,gcp}` |
| Contact: hello@sengol.io (general), trial@sengol.io (trial provisioning) | all | founder-owned mailboxes; not a code fact |

## Known gaps in the sources (not the site)

Recorded here so the next copy change does not import them:

- `sengol/docs/install/helm.mdx` names `charts.sengol.io` and `license.secretName`; the chart publishes to `oci://ghcr.io/sengol-io/charts/sengol` and the value is `license.existingSecret`.
- `sengol/docs/install/docker-compose.mdx` clones `github.com/sengol/sengol`; the organisation is `sengol-io`.
- `sengol/docs/install/terraform.mdx` shows a `license_secret_arn` input; `terraform/aws/variables.tf` has no licence input yet, so a Terraform-deployed appliance has no documented way to receive its licence.
