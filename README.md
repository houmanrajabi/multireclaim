# MultiRECLAIM Annotation Platform & Reclamation-Aware Augmentation (RAA)

Community-based annotation system for LGBTQ+ slur reclamation research.

**Live platform:** https://multireclaim.study · https://annotate.multireclaim.study  
**Contact:** contact@multireclaim.study

---

## Overview

- MultiRECLAIM is a self-hosted, privacy-first annotation platform that collects community judgements on slur reclamation from the LGBTQ+ community and allies. It implements an asymmetric annotation schema based on the FATA framework, and zero-trust annotator privacy.

- Reclamation-Aware Augmentation: Synthetic contrastive pair generation. For every accepted reclaimed example the pipeline generates a matching non-reclaimed counterpart using the same slur.

---
## Reclamation-Aware Augmentation (RAA) Architecture

```
StanceConfig (slur, speaker, pragmatic_role, community, language)
    │
    ▼  Stage 1 — Base model generation (no RLHF)
    ▼  Stage 2 — Instruct validator (reclaimed only)
    ▼  Stage 3 — MPEC gate  (empowerment factors)
    ▼  Stage 4 — Feedback-driven regeneration (max 5 attempts)
    │
    └─ Counterpart: base model + surface-mirroring prompt + MPEC
```
## MultiRECLAIM Annotation Platform Architecture

Five Docker Compose services:

```
Internet
    │
    ▼
 Nginx
    ├── multireclaim.study        → Registration
    └── annotate.multireclaim.study → Label Studio
                                           │
                                       PostgreSQL
                                           │
                                        Cron
                                   (sync + export + backup)
```

| Service | Image | Role |
|---|---|---|
| `postgres` | postgres:15-alpine | Annotation database |
| `labelstudio` | heartexlabs/label-studio:latest | Annotation interface |
| `registration` | custom FastAPI | Annotator intake + privacy gate |
| `nginx` | nginx:alpine | Reverse proxy + SSL termination |
| `cron` | python:3.11-slim | Nightly automation |

---

## Annotation Schema

Asymmetric 4-step schema — Text A (reclaimed candidate) gets Steps 1–3, Text B (generated counterpart) gets Steps 4a–4b only.

---

## Project Configuration

434 raw pairs generated across 11 English slurs from MultiPRIDE Table 7 (`fag, bitch, gay, puss, queer, drag, whore, queen, slut, sissy, slay`), 40 target pairs per slur, on A40 GPU, bf16. After deduplication: 369 clean pairs, and after inverted removal **337 final pairs**.

| Project | ID | Tasks | Overlap | Purpose |
|---|---|---|---|---|
| Calibration Task | 4 | 5 | 999 | All annotators — schema familiarisation |
| Main Task | 5 | 332 | 3 | Research dataset — majority vote |

Calibration overlap=999 means every annotator sees all 5 calibration tasks regardless of how many others have completed them. Main overlap=3 means each task needs exactly 3 independent annotations before closing.

---

## Privacy Architecture

- **No PII collected**: names, real emails, IP addresses, device info — never stored
- **Anonymous IDs**: `annotator_XXXXXX` hash — no reverse lookup possible
- **Generated credentials**: `annotator_XXXXXX@multireclaim.study` email + random password, shown once
- **Registry stores only**: hash ID + gate responses + timestamp
- **Zero-trust design**: even if the registry is leaked, no individual is identifiable
- **Session isolation**: annotator sessions cleared on org assignment to prevent stale state

---

## Security

- UFW: only ports 22, 80, 443 open
- Services bound to `127.0.0.1` (Docker/UFW bypass prevention)
- Nginx: default_server returns 444 for unknown hosts
- Nginx: rate limiting on registration (5r/m) and annotation (30r/m)
- Nginx: token injection on `/user/signup/` for Label Studio DISABLE_SIGNUP_WITHOUT_LINK
- HSTS, X-Frame-Options, X-Content-Type-Options, Referrer-Policy headers
- SSH: key-only authentication
- Fail2ban: ban after 5 failed attempts
- Unattended-upgrades: automated OS security patches

---
