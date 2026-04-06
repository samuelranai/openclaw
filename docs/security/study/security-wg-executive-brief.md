---
title: "OpenClaw Security WG — Executive Brief"
summary: "One-page executive summary of the Security WG mission, problem statement, objectives, roadmap, and success metrics — for management review and approval"
status: draft
---

# OpenClaw Security WG — Executive Brief

## Mission

Establish a security-first plugin and identity architecture for OpenClaw that enables
enterprise adoption, protects against supply chain threats, and supports partner
security teams operating at scale — without breaking the existing operator-trust model.

---

## Problem Statement

| Area              | Current Gap                                                                                                                     |
| ----------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| Plugin Trust      | All plugins run with equal in-process privilege; no enforcement boundary exists between a security plugin and a compromised one |
| Credential Mgmt   | `SecretRef` cannot handle token refresh, rotation events, or federated identity (SPIFFE/OIDC)                                   |
| Identity & Access | No built-in sender identity model; tool policy cannot be scoped by role, team, or group membership                              |
| Supply Chain      | Static scanner produces false positives on legitimate security skills; no signing or provenance                                 |

---

## Objectives

1. Define and ship a **plugin trust tier** that allows security plugins to enforce
   policy with block semantics, protected from subversion by lower-trust plugins.
2. Enable **enterprise credential management** via a plugin-extensible credential
   provider supporting token lifecycle, rotation, and federated identity.
3. Introduce a **built-in identity model** (sender roles, group context) enabling
   team-scoped access policies without replacing the single-operator trust model.
4. Harden the **supply chain** with scan suppression, provenance receipts, and
   eventual signing — unblocking security skill developers today.

---

## Roadmap

### Ongoing Short-Term (Active)

| ID  | Feature                          | Workstream   | Value                                       |
| --- | -------------------------------- | ------------ | ------------------------------------------- |
| T01 | Plugin trust model decision memo | Plugin Trust | Unblocks all enforcement work               |
| T02 | Ship `before_skill_install` hook | Plugin Trust | Gate skill install; supply chain control    |
| T04 | Hook execution semantics spec    | Plugin Trust | Deterministic behavior for security plugins |
| T05 | Runtime plugin disablement       | Plugin Trust | Revoke malicious plugin without restart     |
| T06 | RFC #59165 partner review        | Credentials  | Validate enterprise credential scenarios    |
| T11 | Group context in hook events     | IAM          | Enable per-sender security policy in groups |
| T12 | Scan suppression M1 + M2         | Supply Chain | Unblock security skill developers today     |

### Planning Near-Term (Next)

| ID  | Feature                                   | Workstream   | Depends on |
| --- | ----------------------------------------- | ------------ | ---------- |
| T03 | Enforcement hook tier config key          | Plugin Trust | T01        |
| T15 | Enforcement hook tier shipped             | Plugin Trust | T01, T03   |
| T07 | Credential Provider API design            | Credentials  | T06        |
| T19 | `"plugin"` SecretRefSource implementation | Credentials  | T06, T07   |
| T20 | Plugin SDK `registerCredentialProvider`   | Credentials  | T19        |
| T08 | Sender identity primitives                | IAM          | —          |
| T09 | Identity-based access policy              | IAM          | T08        |
| T10 | Identity enrichment in `before_dispatch`  | IAM          | T08        |
| T13 | AST-level and custom scanner rules        | Supply Chain | T02        |
| T14 | Pre-download scan for npm skills          | Supply Chain | T02        |

### Backlog (Goal Mid/Long Term)

Watcher process API · `before_plugin_install` hook · Network egress policy hook ·
Vault reference credential provider · Credential handling security audit ·
IAM full specification · LDAP/OIDC external IdP integration · RBAC-lite resource-scoped
access control · Session-scoped identity · Per-sender `groupScope` session partitioning ·
Group role model (replace `senderIsOwner`) · Skill/plugin signing scheme ·
Provenance receipt (`.scan-receipt.json`) · ClawHub registry scan UI ·
Supply chain scanner plugin capability

---

## Success Metrics

| Metric                                       | Target                                                           |
| -------------------------------------------- | ---------------------------------------------------------------- |
| Partner security team feedback collected     | Before 4/11 (Q1, Q2, Q3 in WG Open Questions)                    |
| Trust model decision memo approved by WG     | End of Ongoing Short-Term phase                                  |
| `before_skill_install` hook shipped          | End of Ongoing Short-Term phase                                  |
| Security skill false-positive rate (scanner) | Zero unacknowledged critical/warn for legitimate security skills |
| Enforcement hook tier available to partners  | End of Planning Near-Term phase                                  |
| Credential Provider RFC merged               | End of Planning Near-Term phase                                  |
| Enterprise deployment with identity policy   | First customer pilot during Planning Near-Term                   |

---

## Scope and Boundaries

- **In scope:** Plugin enforcement tiers, credential provider extensibility, sender
  identity and group access policy, supply chain scanner hardening.
- **Out of scope:** OS-level per-user multi-tenant isolation; adversarial group-chat
  isolation between untrusted users (consistent with current `SECURITY.md`).
- **Reference:** Full technical details and task breakdowns in
  [security-working-plan.md](security-working-plan.md).
