# power-automate-flows-skill

Skill (formato [Agent Skills](https://agentskills.io/specification) / `SKILL.md`) per lavorare con i **Power Automate cloud flow su Dataverse** da CLI/API invece che dal portale maker: pull della definizione live, edit del JSON, deploy via PATCH di `clientdata`, e verifica eseguendo un run reale e leggendone gli output grezzi. Copre la generazione documenti (Word/Excel → PDF) e i tranelli di connettore/identità che fanno perdere più tempo.

## Skill inclusa

Una sola skill, `power-automate-flows`, con la documentazione di riferimento divisa per funzionalità:

| File | Cosa contiene |
|---|---|
| [`SKILL.md`](SKILL.md) | Punto di ingresso: overview, golden rules, loop deploy+test, indice verso i file di riferimento. |
| [`deploy-and-test.md`](deploy-and-test.md) | Loop ad alta frequenza pull → edit → deploy → trigger → verify; token `az`, PATCH di `clientdata`, **ri-registrazione del trigger** (lo spreca-tempo n.1), lettura output run via `outputsLink`. |
| [`document-generation.md`](document-generation.md) | Generazione documenti: matrice connettori (Word/Excel/OneDrive/PDF), content control Word per `w:id`, workaround liste a lunghezza variabile, tranelli Excel→PDF e allegati email. |
| [`auth-environments.md`](auth-environments.md) | Auth e identità: drift `pac`/`az`, quale identità accede a quale drive (il tranello OneDrive), connection reference & binding, mappa errore → causa. |

> La skill è generica: nessun org id, GUID, tenant o nome cliente hardcoded — solo placeholder (`<org>`, `<WF>`, `<ENV>`, `<expected>`). Adattabile a qualsiasi environment.

## Installazione

La skill va messa in una cartella chiamata `power-automate-flows` dentro la directory skill dell'agente. Il modo più portabile è clonare il repo direttamente in quella posizione.

### Claude Code

Skill personale (disponibile in tutti i progetti):

```bash
git clone git@github.com:MeStakes/power-automate-flows-skill.git ~/.claude/skills/power-automate-flows
```

Oppure solo per il progetto corrente: clona in `.claude/skills/power-automate-flows`. Claude Code la scopre in automatico; invocala con il tool `Skill`.

### Codex CLI

Skill a livello utente:

```bash
git clone git@github.com:MeStakes/power-automate-flows-skill.git ~/.codex/skills/power-automate-flows
```

Oppure per progetto: `.agents/skills/power-automate-flows`. Codex rileva le nuove skill in automatico; se non compare, riavvia Codex per forzare il rescan.

### GitHub Copilot CLI

Skill personale (tutti i progetti):

```bash
git clone git@github.com:MeStakes/power-automate-flows-skill.git ~/.copilot/skills/power-automate-flows
```

Oppure per repo: clona in `.github/skills/power-automate-flows`. Le skill sono scoperte in automatico tramite la convenzione `*/SKILL.md`; usa il tool `skill` per invocarla.

> Su gh CLI ≥ 2.90.0 esiste anche `gh skill install <org/repo>`, ma il `git clone` qui sopra è il metodo più affidabile e identico su tutte le piattaforme.

## Aggiornare

```bash
cd <skills-dir>/power-automate-flows && git pull
```

## Prerequisiti d'uso

`pac` (Power Platform CLI), `az` (Azure CLI, per i token Graph / Dataverse / Flow API) e Python (urllib). Tutte le operazioni sono eseguibili headless.
