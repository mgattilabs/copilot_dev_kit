# Copilot Dev Kit

Toolkit multi-agent per GitHub Copilot, orientato a pianificazione e implementazione guidata tramite agenti specializzati.

[![GitHub Copilot](https://img.shields.io/badge/GitHub-Copilot-blue?logo=github)](https://github.com/features/copilot)
[![License](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)

---

## Overview

`copilot_dev_kit` definisce un flusso operativo con 4 agenti:

- `Skynet`: orchestrazione del workflow, non implementa mai codice
- `Spock`: analisi e pianificazione in due fasi (`interview` -> `plan`)
- `Neo`: implementazione codice, con scope obbligatorio `backend` o `frontend`
- `Woz`: design UI/UX (invocato da Spock durante il planning frontend)

L'obiettivo e garantire coerenza nelle decisioni tecniche, separare i ruoli e ridurre le ambiguita durante le implementazioni.

---

## Cosa e stato aggiornato

Il repository e stato allineato alle modifiche recenti:

- rimosso ogni riferimento ad agenti/componenti non piu presenti
- `Skynet` delega solo a `Spock` e `Neo` (Woz passa da Spock)
- `Neo` e un agente unico con `scope` esplicito (`backend` | `frontend`)
- istruzioni semplificate e centralizzate nei file attualmente presenti in `.github/instructions`

---

## Quick Start

### 1. Aggiungi il kit al progetto

```bash
# dalla root del tuo progetto
git clone https://github.com/mgattilabs/copilot_dev_kit .github/copilot_dev_kit
```

Oppure come submodule:

```bash
git submodule add https://github.com/mgattilabs/copilot_dev_kit .github/copilot_dev_kit
```

### 2. Avvia il flusso con Skynet

In Copilot Chat (VS Code):

```text
@Skynet implementa autenticazione JWT
```

Flusso atteso:

1. `Skynet -> Spock (interview)`
2. `Skynet -> Spock (plan)`
3. approvazione piano
4. `Skynet -> Neo (scope: backend)`
5. eventuale `Skynet -> Neo (scope: frontend)`

---

## Struttura attuale del repository

```text
.github/
  agents/
    neo.agent.md
    skynet.agent.md
    spock.agent.md
    woz.agent.md
  instructions/
    context-engineering.instructions.md
    object-calisthenics.instructions.md
    performance-optimization.instructions.md
README.md
```

---

## File principali

- `.github/agents/skynet.agent.md`: orchestrazione, routing e regole di delega
- `.github/agents/spock.agent.md`: workflow `interview/plan`, checklist architetturali
- `.github/agents/neo.agent.md`: regole di implementazione e autodetect stack
- `.github/agents/woz.agent.md`: design system e linee guida UI Angular Material
- `.github/instructions/*.instructions.md`: vincoli trasversali su contesto, performance e object calisthenics

---

## Note operative

- Per task full-stack, `Neo` va invocato in due passaggi separati: prima `backend`, poi `frontend`.
- Lo scope di `Neo` e obbligatorio: senza `scope` l'agente si blocca per chiarimento.
- La fase di design UI passa da `Spock -> Woz`, non viene chiamata direttamente da Skynet.

---

## License

MIT License. Vedi `LICENSE` per i dettagli.