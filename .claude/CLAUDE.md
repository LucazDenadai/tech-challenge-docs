# CLAUDE.md — ruleset

This repository is the source of truth for architectural decisions (ADRs), technical decisions (RFCs) and execution planning (cards) across all 5 repositories of the Tech Challenge project (Tech-challenge, tech-challenge-lambda, tech-challenge-infra-k8s, tech-challenge-infra-db, tech-challenge-docs).

## Rule 1 — Keep This Repo Consistent With Itself
Before adding or editing an ADR/RFC/card, check whether it conflicts with or supersedes an existing one.
If a new decision supersedes an old ADR, mark the old one's status as superseded and link to the new one — don't leave contradictory decisions both marked "Aceito".
Cross-reference related ADRs/RFCs/cards explicitly (link by number), so a reader in any single document can trace the full decision chain.

## Rule 2 — Match Existing Format
New ADRs follow the same structure as existing ones (Contexto, Decisão, Alternativas consideradas, Consequências).
New cards follow the same structure as existing ones (Tipo, Status, Depende de, Bloqueia, Decisão arquitetural, Contexto, Critérios de aceite, Passos).
Don't invent a new document type or structure without checking what's already there.

## Rule 3 — Update the README Index
Every new ADR, RFC, diagram or card epic added must be listed in `README.md`'s index tables.
An undiscoverable document is as good as missing — the README is the entry point for anyone (including a future Claude session) trying to understand the project's decisions.
