# ⚡ power-automate-flows-skill

> 🇬🇧 [Read in English](README.md)

Una [Agent Skill](https://agentskills.io/specification) (formato `SKILL.md`) che trasforma **Claude** — e qualsiasi altro coding agent compatibile con le skill — in un operatore pratico per i **Power Automate cloud flow su Dataverse**. 🤖

## 💡 L'idea

Il concetto di fondo: **usare Claude per creare ed editare i flow di Power Automate da CLI/API invece di cliccare nel portale maker.** 🖱️❌

Claude scarica la definizione live del flow, la modifica come JSON, fa il deploy con un PATCH di `clientdata` e verifica la modifica lanciando un run *reale* e leggendone gli output grezzi — niente UI, niente tirare a indovinare. Copre anche la generazione documenti (Word/Excel → PDF) e i tranelli di connettore/identità che fanno perdere più tempo. 🕳️

🌱 **Questa skill cresce man mano che viene usata.** Ogni nuovo tranello, codice di errore o workaround scoperto sul campo viene reintegrato qui. È quindi volutamente un documento vivo — **le pull request con migliorie sono molto gradite!** 🙏

## 📦 Cosa contiene

Una sola skill, `power-automate-flows`, con la documentazione di riferimento divisa per funzionalità:

| File | Cosa contiene |
|---|---|
| 📄 [`SKILL.md`](SKILL.md) | Punto d'ingresso: overview, golden rules, loop deploy+test e indice verso i file di riferimento. |
| 🚀 [`deploy-and-test.md`](deploy-and-test.md) | Il loop ad alta frequenza pull → edit → deploy → trigger → verify; token `az`, PATCH di `clientdata`, il **tranello della ri-registrazione del trigger** (lo spreca-tempo n.1), lettura output run via `outputsLink`. |
| 📝 [`document-generation.md`](document-generation.md) | Generazione documenti: matrice connettori (Word/Excel/OneDrive/PDF), content control Word per `w:id`, workaround per liste a lunghezza variabile, tranelli Excel→PDF e allegati email. |
| 🔐 [`auth-environments.md`](auth-environments.md) | Auth e identità: drift `pac`/`az`, quale identità accede a quale drive (il tranello OneDrive), connection reference & binding, mappa errore → causa. |

> ✅ La skill è **generica**: nessun org id, GUID, tenant o nome cliente hardcoded — solo placeholder (`<org>`, `<WF>`, `<ENV>`, `<expected>`). Adattabile a qualsiasi environment.

## 🛠️ Installazione

La skill va messa in una cartella chiamata `power-automate-flows` dentro la directory skill dell'agente. Il modo più portabile è clonare il repo direttamente in quella posizione. 📥

### 🟣 Claude Code

Skill personale (disponibile in tutti i progetti):

```bash
git clone git@github.com:MeStakes/power-automate-flows-skill.git ~/.claude/skills/power-automate-flows
```

Oppure solo per il progetto corrente: clona in `.claude/skills/power-automate-flows`. Claude Code la scopre in automatico; invocala con il tool `Skill`. ✨

### 🟢 Codex CLI

Skill a livello utente:

```bash
git clone git@github.com:MeStakes/power-automate-flows-skill.git ~/.codex/skills/power-automate-flows
```

Oppure per progetto: `.agents/skills/power-automate-flows`. Codex rileva le nuove skill in automatico; se non compare, riavvia Codex per forzare il rescan. 🔄

### 🔵 GitHub Copilot CLI

Skill personale (tutti i progetti):

```bash
git clone git@github.com:MeStakes/power-automate-flows-skill.git ~/.copilot/skills/power-automate-flows
```

Oppure per repo: clona in `.github/skills/power-automate-flows`. Le skill sono scoperte in automatico tramite la convenzione `*/SKILL.md`; usa il tool `skill` per invocarla.

> ℹ️ Su gh CLI ≥ 2.90.0 esiste anche `gh skill install <org/repo>`, ma il `git clone` qui sopra è il metodo più affidabile e identico su tutte le piattaforme.

## ♻️ Aggiornare

```bash
cd <skills-dir>/power-automate-flows && git pull
```

## 📋 Prerequisiti d'uso

`pac` (Power Platform CLI), `az` (Azure CLI, per i token Graph / Dataverse / Flow API) e Python (urllib). Tutte le operazioni sono eseguibili headless. 💻

## 🤝 Contribuire

Hai trovato una stranezza di un connettore, un codice di errore criptico o un workaround migliore? Apri una **pull request** — questa skill migliora a ogni flow reale che tocca. 🚀
