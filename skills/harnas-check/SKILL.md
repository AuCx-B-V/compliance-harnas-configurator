---
name: harnas-check
description: Pre-flight check tegen het actieve compliance-harnas. Leest configs/ACTIVE.json, lijst geactiveerde controls per tier (soft/medium/hard/org) en waarschuwt over open of N.v.t. runtime-controls. Gebruik aan het begin van een sessie of voordat je impactvolle acties uitvoert (Bash, Edit, Write op productie-paden, e-mails versturen, deployen). Activeer ook bij vragen als "ben ik compliant", "check mijn harnas", "welke controls zijn open", "wat moet ik nog doen voor compliance".
---

# Harnas-check — pre-flight tegen het actieve compliance-harnas

Je voert een check uit tegen het actieve harnas dat de gebruiker via de Compliance-Harnas Configurator heeft vastgesteld.

## Wat te doen

### 1. Lees het actieve harnas

Zoek `ACTIVE.json` in deze volgorde (eerste die bestaat, wint):

1. Pad in environment variable `HARNAS_CONFIG` (als gezet)
2. `./configs/ACTIVE.json` (relatief aan current working directory)
3. `~/.claude/harnas/ACTIVE.json` (standaard user-config-locatie)

Als geen van deze bestaat: stop en geef terug:

> Geen actief harnas gevonden. Open de Compliance-Harnas Configurator
> (`Compliance-Harnas-Configurator.html`), ga naar de **Toon Harnas**-tab en
> klik **"Sla op als ACTIVE.json"** — sla op in één van de drie locaties hierboven.

### 2. Parse en interpreteer

Het JSON-bestand bevat:
- `name`, `owner`, `dpo`, `ciso`, `purpose`, `aiActClass`
- `L01-L04` keuzes
- `L05` (Agent & Autonomie): `agent` (true/false) en bij `agent: true` een `level` (`A1`-`A4`). Ontbreekt in configs van vóór L05.
- `controlStatus` — object met per harnas-code: `{status, owner, date, evidence, note}`
  - status-waarden: `open` · `prog` (in uitvoering) · `done` (geïmplementeerd) · `na` (n.v.t.) · `risk` (geaccepteerd risico)

### 3. Categoriseer controls per tier

Gebruik deze tier-mapping (gelijk aan `harnasSteering` in de configurator):

| Tier | Controls | Betekenis |
|---|---|---|
| **soft** | G.01, G.02, G.03, G.04, M.03, O.03, T.01, T.03, T.05 | hoort in CLAUDE.md |
| **medium** | D.03, D.06, M.04, I.02, I.03, I.06, T.02, A.01, A.02 | afgedwongen via hook in settings.json |
| **hard** | D.04, I.01, I.04, I.05, O.05, O.06 | infra/contractueel — buiten Claude |
| **org** | overige | puur audit-dossier — geen runtime |

De agent-normen `X.01`-`X.30` hebben geen tier. Deel ze niet in bij `org` en neem ze niet op in de blokken hierboven; ze krijgen een eigen blok (stap 3b).

### 3b. Agent-normen (L05)

Alleen als `L05.agent` true is en `L05.level` gezet. Bepaal eerst het toetsingsniveau: verwerkt de use-case persoonsgegevens (`L03.pg` bestaat en is niet `pg_none`), dan geldt één niveau hoger dan gekozen (A1 → A2, A2 → A3, A3 → A4; A4 blijft A4). Dit is gelijk aan `l05Effective` in de configurator.

Actieve normen per toetsingsniveau:

| Niveau | Actieve normen |
|---|---|
| **A1** | X.01-X.06, X.13-X.15, X.28-X.30 (12) |
| **A2** | X.01-X.18, X.28-X.30 (21) |
| **A3 / A4** | X.01-X.30 (30) |

Een actieve norm zonder regel in `controlStatus` heeft status `open`.

Andere gevallen:
- `L05.agent` is false: meld "geen agent" en sla het blok over.
- `L05` ontbreekt of heeft geen `level`, terwijl `L04.agents` true is: meld dat het autonomieniveau ontbreekt en dat de agent-normen daardoor niet geactiveerd zijn (wizard, stap 6).

### 4. Geef een gestructureerd rapport

Gebruik dit format (Nederlands, kort):

```
## Harnas-check — {name}

**Use-case:** {name} · eigenaar {owner} · {aiActClass}
**Doel:** {purpose}
**Modified:** {modified}

### ✓ Geïmplementeerd ({aantal})
- [tier · code] naam — {date} {owner} {evidence-link}

### ⚠ Open runtime-controls ({aantal})  ← actie nodig
- [medium · I.02] Prompt-injection defense — geen status, geen eigenaar
  → vereist hook in settings.json (PostToolUse op tool-input)
- ...

### ◔ In uitvoering ({aantal})
- [soft · T.01] AI-kenbaarheid — eigenaar René, sinds 2026-04-15

### — Geaccepteerd risico / N.v.t. ({aantal})
- ...

### Hard-controls (infra) — niet door Claude beheerd
- I.01 Authenticatie — verifieer dat YubiKey-MFA op Anthropic-account aan staat
- I.05 Netwerksegmentatie — egress-whitelist actief?
- ...

### Agent-normen (L05) — {gekozen niveau}, getoetst als {toetsingsniveau}
{aantal} actief · {aantal} geïmplementeerd · {aantal} in uitvoering · {aantal} open · {aantal} n.v.t./risico
- Open: X.07 Eigen agent-identiteit, X.10 Toolallowlist, ...
Telt niet mee in het oordeel hieronder: de agent-normen hebben nog geen tier.
```

### 5. Geef ten slotte een **GO / WAIT / STOP**-oordeel

- **GO** — alle medium-tier controls zijn `done`, `na` of `risk`. Geen open runtime-controls.
- **WAIT** — er staan medium-tier controls op `open` of `prog`. Lijst ze expliciet en vraag of de gebruiker wil doorgaan.
- **STOP** — er staat een gap-flag (⚑) op `open` (= AI Act-trigger nog niet afgedekt). Toon welke trigger het betreft.

## Belangrijke regels

- **Lees alleen, schrijf niet.** Deze skill verandert geen state — alleen rapportage.
- **Citeer de control-codes letterlijk** zodat de gebruiker ze kan terugvinden in de configurator.
- **Wees expliciet over wat ontbreekt.** Geen voldoende informatie = open, niet groen.
- **Toon datums.** "Sinds 2026-04-15" is informatiever dan "in uitvoering".
- **Geen oordeel over `org`-controls.** Die horen in audit-dossiers, niet in pre-flight.

## Optioneel — herinneringen

Als de gebruiker **"sessie-context"** of **"laatste sessie"** noemt, integreer dit in de standaard sessie-startroutine van CLAUDE.md (lees ook `LAATSTE_SESSIE.md`).

Als er sinds de vorige sessie controls zijn `done` geworden, prijs kort. Als er controls zijn teruggezet naar `open` of `risk`: vraag waarom.

## Tips voor de gebruiker

- Stel `ACTIVE.json` opnieuw in via de configurator wanneer je een nieuwe use-case start
- Update statussen liefst in de configurator (Controls-tab) — dat houdt JSON canoniek
- Gebruik de `evidence`-URL voor links naar SharePoint, NOREA-dossier, of git-commit
