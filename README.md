# 🌐 CMS Multilingual Workflow — AI-Powered Translation Insertion

Workflow operativo per inserire testi tradotti nei database di siti CMS multilingua, tramite AI (Claude Code). L'AI legge, comprende e sostituisce i testi semanticamente, creando da zero la struttura multilingua se necessaria.

## Come funziona

1. Dai all'AI il **dump del DB** (solo lingua sorgente) + i **file SEO tradotti** per ogni lingua target
2. L'AI **esplora** la struttura del progetto, **crea** le entità mancanti, **traduce** pagina per pagina
3. Output: un **file SQL** pronto da importare con tutti gli INSERT e UPDATE

Il flusso è progettato per essere **resiliente alle interruzioni**: ogni sessione può riprendere esattamente da dove si è fermata.

## Struttura del repo

```
help-ai/
├── workflow.md                        ← Il flusso operativo (CMS-agnostico)
├── cms-known-profiles/                ← Profili specifici per CMS
│   ├── joomla-yootheme.md            ← Joomla 5/6 + YOOtheme Builder
│   └── ...                            ← Cresce con l'uso
├── README.md
├── LICENSE
└── .gitignore
```

## CMS supportati

| CMS | Page Builder | Profilo | Stato |
|-----|-------------|---------|-------|
| Joomla 5/6 | YOOtheme Builder 4.x | [joomla-yootheme.md](cms-known-profiles/joomla-yootheme.md) | ✅ Testato |

> Per CMS senza profilo, l'AI fa **discovery automatico** dalla documentazione online e propone la creazione del profilo a fine sessione.

## Quick start

### 1. Prepara i file

Nella root del progetto, clona questo repo come cartella `help-ai/` e aggiungi:

- `dump-db.json` — dump del DB nella lingua sorgente (articoli, menu, categorie, moduli, associazioni)
- `Seo-IT.docx` — file SEO tradotto in italiano (o qualsiasi lingua target)
- `Seo-EN.docx` — file SEO tradotto in inglese (ecc.)

I file di lavoro vanno direttamente in `help-ai/` — il `.gitignore` li esclude dal repo.

### 2. Avvia Claude Code

```bash
claude
```

### 3. Dai le istruzioni

```
Leggi il file workflow.md in help-ai/ e il profilo CMS appropriato,
poi esegui il flusso. Il CMS è Joomla con YOOtheme.
Lingua sorgente: DE. Lingue target: IT, EN.
```

### 4. Lascia lavorare

Claude Code:
- Spezzetta e sanitizza i file SEO
- Esplora il dump DB
- Crea le strutture per le lingue target (se mancano)
- Traduce pagina per pagina con matching semantico
- Genera il file SQL finale

### 5. Importa e verifica

Importa `update-{progetto}.sql` nel DB e verifica sul frontend.

## Come contribuire

Questo workflow **migliora con l'uso**. Alla fine di ogni sessione (Fase 5: Retrospettiva), Claude Code chiede se ci sono problemi da documentare e propone:

- **Aggiornamenti ai profili esistenti** → `fix(joomla-yootheme): descrizione`
- **Nuovi profili CMS** → `feat(wordpress-elementor): aggiunto profilo iniziale`
- **Miglioramenti al workflow** → `docs(workflow): aggiunta regola per...`

Fai review del commit proposto e push.

## Fasi del workflow

| Fase | Cosa fa | Quando |
|------|---------|--------|
| **0a** | Identifica il CMS, carica profilo o fa discovery | Una volta per progetto |
| **0b** | Spezzetta e sanitizza i file SEO | Una volta per progetto |
| **0c** | Crea struttura lingue target (articoli, menu, categorie, moduli) | Se mancanti |
| **1** | Esplora la struttura del dump DB | Una volta per progetto |
| **2** | Sostituisce i testi pagina per pagina, lingua per lingua | Cuore del lavoro |
| **3** | Genera il file SQL finale | Una volta per progetto |
| **4** | Importazione e verifica frontend | Manuale |
| **5** | Retrospettiva e aggiornamento profilo CMS | Fine sessione |

## Principi di design

- **CMS-agnostico**: il workflow funziona con qualsiasi CMS. Le specifiche sono nei profili.
- **Resiliente**: `TODO.md` e `STRUTTURA-PROGETTO.md` permettono di riprendere da qualsiasi interruzione.
- **Economico in token**: il dump si legge una volta, poi si lavora con file piccoli e mirati.
- **Semantico**: l'AI capisce *cosa* dice un testo e lo abbina alla traduzione corretta, non per posizione.
- **Organico**: il workflow migliora ad ogni utilizzo grazie alla Fase 5.

## Licenza

Vedi [LICENSE](LICENSE).
