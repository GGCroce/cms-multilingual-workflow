# Workflow: Inserimento testi multilingua in siti CMS via AI

Questo documento descrive la metodologia CMS-agnostica per creare e popolare le versioni multilingue di un sito web. Il lavoro è svolto da un'AI che legge, comprende e sostituisce i testi semanticamente, una pagina alla volta. Il dump DB contiene **solo la lingua sorgente**; l'AI crea da zero tutta la struttura necessaria per ogni lingua target e la popola con i testi tradotti.

Il workflow funziona con qualsiasi CMS e page builder. Le informazioni specifiche sul CMS in uso vengono acquisite in due modi:
- **CMS Known Profiles** (`cms-known-profiles/`): se esiste un profilo per il CMS in uso, l'AI lo legge per avere basi solide e tranelli noti
- **Discovery online**: se non esiste un profilo, l'AI consulta la documentazione ufficiale del CMS per scoprire architettura, tabelle e logiche multilingua

---

## Prima cosa da fare (ogni sessione)

**Prima di qualsiasi altra azione**, cercare nella directory di lavoro se esistono i file `TODO.md` e `STRUTTURA-PROGETTO.md`:

- **Se esistono entrambi** → il lavoro è già stato iniziato in una sessione precedente. Leggere TODO.md per sapere a che punto siamo (fase corrente, ultima operazione completata, lingua corrente) e STRUTTURA-PROGETTO.md per avere il contesto del progetto. Riprendere dal primo punto non completato nel TODO.
- **Se esiste solo TODO.md** → la Fase 1 (esplorazione) era in corso ma non è stata completata. Riprendere la Fase 1.
- **Se non esistono** → è un progetto nuovo. Crearli e partire dalla Fase 0a.

> Questo controllo rende il flusso autonomo: l'utente dice "riprendiamo" e l'AI sa esattamente cosa fare senza ulteriori istruzioni.

---

## Fase 0a: Identificazione CMS e caricamento profilo

L'utente comunica quale CMS e page builder usa il sito (es. "Joomla con YOOtheme", "WordPress con Elementor", "Drupal").

### L'AI verifica se esiste un profilo noto

Cercare nella cartella `cms-known-profiles/` (nella stessa directory di questo workflow) un file corrispondente al CMS in uso.

- **Se esiste** (es. `joomla-yootheme.md`) → leggerlo. Contiene: architettura del contenuto, tabelle DB coinvolte, formato del contenuto strutturato, sistema multilingua, formato link interni, direttive per INSERT/UPDATE, e i **tranelli noti** scoperti in sessioni precedenti. Usarlo come base per tutto il lavoro.
- **Se non esiste** → eseguire **discovery online**: cercare nella documentazione ufficiale del CMS e dei plugin multilingua le informazioni equivalenti. In particolare scoprire:
  1. **Dove il CMS memorizza il contenuto strutturato** — quale tabella, quale campo, quale formato (JSON, HTML serializzato, blocchi, ecc.)
  2. **Come è strutturato il contenuto** — albero di nodi, blocchi piatti, widget, ecc. e come identificare quelli con testo visibile vs quelli dinamici/template
  3. **Come funziona il routing** — permalink, alias, menu item, path aliases
  4. **Come funziona il sistema multilingua** — quale plugin/modulo, quale tabella gestisce le associazioni tra contenuti nelle diverse lingue
  5. **Quali entità vanno duplicate per lingua** — articoli/post, menu/navigation, categorie/tassonomie, moduli/widget, blocchi riutilizzabili
  6. **Come si scrivono le entità nel DB** — campi obbligatori, valori di default per stato/pubblicazione, formato delle query

> Il costo token del discovery online si paga **una volta sola**. Tutte le informazioni finiscono in STRUTTURA-PROGETTO.md e non serve più consultare la documentazione.

---

## Contesto architetturale (generale)

Indipendentemente dal CMS, il pattern è lo stesso:

- Ogni pagina ha del **contenuto strutturato** memorizzato nel DB in un formato specifico (JSON, HTML con blocchi, serializzato, ecc.)
- Questo contenuto ha una **struttura ad albero o a blocchi** con elementi che contengono testo visibile, link e configurazioni
- Il CMS ha un **sistema di routing** che mappa URL a contenuti
- Il CMS ha (o può avere tramite plugin) un **sistema multilingua** che associa contenuti equivalenti tra lingue diverse
- Oltre ai contenuti principali, ci sono **entità secondarie** (moduli, widget, blocchi) con testo traducibile

Le specifiche di *come* tutto questo funziona per il CMS in uso sono nel profilo CMS o vengono scoperte in Fase 0a.

### Entità tipiche coinvolte

Il flusso lavora tipicamente su queste entità per ogni lingua target (i nomi esatti dipendono dal CMS):

- **Contenuti principali** (articoli, post, nodi) — il testo delle pagine
- **Menu/navigazione** — routing, alias/slug, metadati SEO
- **Categorie/tassonomie** — organizzazione dei contenuti
- **Moduli/widget/blocchi** — contenuti in posizioni del template (footer, sidebar, banner, ecc.)
- **Associazioni multilingue** — la tabella che lega le entità corrispondenti tra le lingue

---

## File di lavoro da creare all'inizio

Appena si inizia il lavoro su un progetto, creare due file .md nella directory di lavoro. Questi file sono fondamentali per la **resilienza al cambio sessione**: il lavoro può essere interrotto in qualsiasi momento (esaurimento token, interruzione, cambio contesto) e una nuova sessione AI può riprendere esattamente da dove si è fermata leggendo questi due file.

### TODO.md

Traccia lo stato di avanzamento del lavoro. Va aggiornato **dopo ogni fase e dopo ogni singolo articolo/entità completata** — mai accumulare lavoro senza aggiornare il TODO. Il TODO è **unico per tutto il progetto** e contiene sezioni per ogni lingua target. Struttura:

```markdown
# TODO - Traduzione multilingua - {nome progetto}

## Info progetto
CMS: {nome CMS + page builder}
Profilo CMS: {nome file profilo, o "discovery" se non esisteva}

## Stato attuale
Fase corrente: [0a / 0b / 0c / 1 / 2 / 3 / 4 / 5]
Lingua corrente: {lingua}

## Lingue target
- [x] IT (italiano)
- [ ] EN (english)
- [ ] FR (français)

## Fase 0a: Identificazione CMS
- [x] CMS identificato: {nome}
- [x] Profilo caricato / Discovery completato

## Fase 0b: Spezzettamento file SEO
- [x] Seo-IT.docx → seo-it-sezioni/
- [x] Seo-EN.docx → seo-en-sezioni/
- [ ] Seo-FR.docx → seo-fr-sezioni/

## Fase 0c: Creazione struttura lingue target
- [x] IT: contenuti, menu, categorie, moduli creati
- [ ] EN: contenuti, menu, categorie, moduli creati
- [ ] FR: contenuti, menu, categorie, moduli creati

## Fase 1: Esplorazione
- [x] Letto dump DB
- [x] Mappata struttura contenuto (tipi nodo/blocco, props testo)
- [x] Identificati elementi dinamici/template (da non tradurre)
- [x] Mappati moduli/widget con contenuto testuale
- [x] Costruito mapping routing/link interni (tutte le lingue)
- [x] Verificati metadati SEO
- [x] Scritto STRUTTURA-PROGETTO.md

## Fase 2: Sostituzione testi (pagina per pagina)

### Italiano (IT)
- [x] {id} - home
- [x] {id} - chi-siamo
- [ ] {id} - camere          ← prossimo
- [ ] ...
- [ ] Moduli/widget IT completati

### English (EN)
- [ ] {id} - home
- [ ] ...
- [ ] Moduli/widget EN completati

## Fase 3: Generazione SQL
- [ ] Generato update-{progetto}.sql (tutte le lingue)

## Fase 4: Importazione e verifica
- [ ] Importato SQL
- [ ] Spot-check frontend (tutte le lingue)

## Fase 5: Retrospettiva
- [ ] Chiesto all'utente se ci sono problemi da documentare
- [ ] Aggiornato profilo CMS (se necessario)
```

### STRUTTURA-PROGETTO.md

Viene scritto al termine della Fase 1 (esplorazione). Contiene un resoconto leggibile e compatto della struttura del DB e del CMS, in modo che per tutto il resto del lavoro **non sia mai necessario riaprire il dump DB né consultare la documentazione**. Include informazioni per **tutte le lingue target**.

Il contenuto esatto varia in base al CMS, ma deve sempre includere:

- Prefisso tabelle / nomi tabelle coinvolte
- Lingua sorgente e lingue target
- Lista contenuti principali della lingua sorgente (ID, alias, titolo)
- Mapping contenuti sorgente → target per ogni lingua (ID, alias, routing ID)
- Lista moduli/widget/blocchi con contenuto testuale
- Mapping moduli sorgente → target per ogni lingua
- Categorie/tassonomie e relativi mapping
- Tipi di nodo/blocco trovati nel contenuto strutturato e relative props con testo
- Regole per riconoscere elementi dinamici/template (da non tradurre)
- Mapping link interni / routing tra le lingue
- Gruppi di associazione multilingue
- Formato link interni
- Stato metadati SEO

> **Perché STRUTTURA-PROGETTO.md**: il dump DB è grande e costoso in token. Dopo l'esplorazione iniziale, tutto ciò che serve per il lavoro pagina per pagina è in questo file compatto. Il dump non va più riaperto. Inoltre, se la sessione cambia, la nuova AI non deve ripetere l'esplorazione: legge STRUTTURA-PROGETTO.md e ha tutto il contesto.

---

## Input necessari

### Per l'esplorazione iniziale (una volta sola)

- **Dump del database della lingua sorgente** con le tabelle rilevanti per il CMS in uso (contenuti, menu/routing, categorie, moduli/widget, associazioni multilingue). Formato preferito: JSON (meno token). Accettabile: SQL dump.

### Testi tradotti (uno per ogni lingua target)

- **File SEO tradotto** — può essere fornito come:
  - **Un file unico per lingua** (DOCX o altro) contenente tutte le pagine, separate da marcatori `Url:`. Ogni sezione va dall'`Url:` corrente fino al prossimo `Url:` (o fine file) e contiene Title, Desc e tutto il contenuto della pagina.
  - **File già spezzettati** (uno per pagina) nella cartella `seo-{lingua}-sezioni/`.

Se viene fornito il file unico, l'AI lo spezzetta prima di iniziare il lavoro (vedi Fase 0b).

### Per il lavoro pagina per pagina

- **Contenuto strutturato di un singolo articolo/pagina** — estratto dal dump, contiene il contenuto nella lingua sorgente da sostituire.
- **File SEO tradotto della pagina corrispondente** — il frammento relativo a quella sola pagina, dalla cartella `seo-{lingua}-sezioni/`.

> **Perché pagina per pagina**: il contenuto di un singolo articolo + il suo file SEO entrano comodamente nel contesto dell'AI. Nessun bisogno di caricare tutto il dump né tutto il file SEO. Ogni operazione è autonoma: un contenuto in, un contenuto aggiornato out.

---

## Fase 0b: Spezzettamento e sanitizzazione file SEO

Se i testi tradotti sono in un unico file per lingua, l'AI lo legge, lo sanitizza e lo divide in file separati per pagina. **Questo va fatto per tutti i file SEO di tutte le lingue target in un unico passaggio.**

### Sanitizzazione caratteri

I file DOCX che passano tra sistemi diversi (Mac, Windows, Google Docs) accumulano caratteri problematici. **Prima di spezzettare**, applicare queste normalizzazioni al contenuto:

- **Trattini**: em dash `—` (U+2014) e en dash `–` (U+2013) → trattino normale `-` (U+002D), a meno che non siano intenzionali nel contenuto
- **Virgolette curve**: `"` `"` → `"` dritte, `'` `'` → `'` dritte
- **Caratteri invisibili**: rimuovere BOM (U+FEFF), zero-width space (U+200B), soft hyphen (U+00AD), zero-width non-joiner (U+200C)
- **Spazi anomali**: non-breaking space (U+00A0) → spazio normale, spazi multipli → spazio singolo
- **Verificare** che il risultato sia UTF-8 pulito

### Come funziona lo spezzettamento

Il file SEO unico ha questa struttura (la formattazione può variare, ma il marcatore `Url:` delimita ogni sezione):

```
Url: /it/home/
Title: Titolo pagina home
Desc: Meta description home
[contenuto home...]

Url: /it/camere/
Title: Titolo pagina camere
Desc: Meta description camere
[contenuto camere...]
```

L'AI, **per ogni lingua target**:

1. Legge il file SEO completo
2. Applica la sanitizzazione caratteri
3. Lo divide in sezioni: ogni sezione va da un `Url:` al successivo (o alla fine del file)
4. Crea la cartella `seo-{lingua}-sezioni/`
5. Salva ogni sezione come file separato, usando lo slug dell'URL come nome: `seo-{lingua}-home.txt` (o `.md`)
6. Preserva **tutta la formattazione originale** di ogni sezione (dopo la sanitizzazione)

> Questo step si fa una volta sola per tutte le lingue, prima di iniziare qualsiasi altro lavoro. I file spezzettati e sanitizzati restano nelle cartelle e vengono usati singolarmente in Fase 2.

---

## Fase 0c: Creazione struttura lingue target (se necessaria)

**Se nel dump DB non esistono contenuti, menu, categorie e moduli/widget per una lingua target**, l'AI li crea a partire dalla struttura della lingua sorgente.

### Come procedere

Consultare il profilo CMS (o le informazioni raccolte in Fase 0a) per sapere:
- Quali tabelle sono coinvolte e quali campi sono obbligatori
- Quali valori usare per stato/pubblicazione (contenuto pubblicato, data corrente)
- Come funzionano le associazioni multilingue nel CMS in uso
- Eventuali tranelli noti per la creazione di nuove entità

Per ogni lingua target, clonare dalla lingua sorgente:

1. **Categorie/tassonomie** — duplicare, impostare lingua target, annotare mapping ID
2. **Contenuti principali** — duplicare, impostare lingua target, associare alla categoria target, stato pubblicato, data pubblicazione corrente. Il contenuto strutturato viene copiato dall'originale (verrà sovrascritto in Fase 2)
3. **Menu/navigazione** — duplicare, impostare lingua target, aggiornare link al contenuto target. L'alias/slug verrà aggiornato con quello dal file SEO in Fase 3
4. **Moduli/widget/blocchi** — duplicare quelli con contenuto testuale, impostare lingua target, aggiornare assegnazioni alle pagine
5. **Associazioni multilingue** — creare i record che legano ogni entità sorgente alla corrispondente entità target, secondo il meccanismo del CMS in uso

### Output della Fase 0c

Aggiornare **STRUTTURA-PROGETTO.md** con tutti i mapping creati e aggiornare **TODO.md**.

> **Nota**: questa fase genera query INSERT (non UPDATE). Le query vanno incluse nel file SQL finale della Fase 3, **prima** delle query UPDATE sui contenuti.

---

## Fase 1: Esplorazione della struttura (una volta per progetto)

**Ogni progetto è diverso, anche a parità di CMS.** L'AI legge il dump DB (intero, questa volta sola) per capire la struttura specifica del progetto.

### 1a. Struttura del contenuto

Percorrere il contenuto strutturato di alcuni articoli/pagine rappresentativi e scoprire:

- Quali tipi di nodo/blocco/widget esistono
- Per ogni tipo, quali proprietà contengono testo visibile (content, title, description, ecc.)
- Per ogni tipo, quali proprietà contengono link interni

### 1b. Elementi dinamici/template

Identificare gli elementi che generano contenuto dinamicamente e **non vanno tradotti**. Le regole per riconoscerli dipendono dal CMS e page builder in uso — consultare il profilo CMS se disponibile, altrimenti analizzare i pattern nel dump.

### 1c. Moduli/widget con contenuto testuale

Esaminare i moduli/widget della lingua sorgente e identificare quelli che contengono testo traducibile.

### 1d. Mapping link interni

Dalla tabella di routing/menu del CMS, costruire la tabella di mapping degli ID di routing **per tutte le lingue** (sorgente e tutte le target). Identificare il formato dei link interni.

### 1e. Gruppi di associazione multilingue

Registrare quali entità corrispondono alla stessa pagina/contenuto tra le diverse lingue.

### 1f. Metadati SEO

Verificare se i metadati SEO (page title, meta description) sono compilati o vuoti per i contenuti/menu delle lingue target.

### Output della Fase 1

Scrivere **STRUTTURA-PROGETTO.md** e aggiornare **TODO.md** segnando la Fase 1 come completata. Da questo momento in poi il dump DB non va più riaperto: tutto il necessario è in STRUTTURA-PROGETTO.md.

---

## Fase 2: Sostituzione testi (una pagina alla volta)

Questo è il cuore del lavoro. Si procede **lingua per lingua**, e per ogni lingua **contenuto per contenuto**.

### Input

1. Il **contenuto strutturato** di quell'articolo/pagina (estratto dal dump)
2. Il **file SEO tradotto** di quella pagina (dalla cartella `seo-{lingua}-sezioni/`)

### Cosa fa l'AI

1. **Legge il contenuto strutturato** — capisce la struttura: quali nodi contengono testo, quali sono dinamici/template, quali hanno link
2. **Legge il file SEO tradotto** — capisce il contenuto: titoli, paragrafi, liste, FAQ, ecc., organizzati per sezione della pagina
3. **Per ogni nodo/blocco live con testo** nel contenuto:
   - Capisce semanticamente a quale porzione del testo tradotto corrisponde
   - Sostituisce il valore nella proprietà con il testo tradotto
4. **Preserva il formato** del valore originale:
   - Se il valore sorgente è plain text → inserire plain text
   - Se contiene HTML (`<p>`, `<ul>`, `<strong>`, ecc.) → inserire HTML con la stessa struttura
5. **Non tocca** nulla che non sia testo visibile: type, children, struttura, proprietà di configurazione, immagini, stili
6. **Non tocca** gli elementi dinamici/template
7. **Aggiorna i link interni**: sostituisce i parametri lingua e rimappa gli ID di routing usando la tabella in STRUTTURA-PROGETTO.md. Questo include sia le proprietà link dei nodi, sia i tag `<a href="...">` dentro i contenuti HTML. Per widget/iframe, cambia solo i parametri lingua nel codice embedded.
8. **Annota Title, Meta Description e alias/slug** dal file SEO per l'aggiornamento dei menu/routing (Fase 3). L'alias va estratto dall'**ultimo segmento dell'URL** nel campo `Url:` del file SEO (es. `/it/qualcosa/chi-siamo/` → `chi-siamo`).

### Output

Il contenuto strutturato aggiornato (testi tradotti + link corretti), pronto per generare la query SQL UPDATE.

### Moduli/widget

Dopo aver completato tutti i contenuti principali di una lingua, procedere con i **moduli/widget** di quella lingua. Per ogni modulo con contenuto testuale:
- Leggere il contenuto del modulo sorgente
- Trovare il testo corrispondente nel file SEO (se presente) o tradurre in base al contesto
- Aggiornare il contenuto del modulo target

### Ripetere per ogni contenuto e lingua

Procedere pagina per pagina, lingua per lingua. Ogni contenuto è un'operazione **atomica e indipendente**:

1. L'AI carica STRUTTURA-PROGETTO.md + il contenuto strutturato + il file SEO della pagina
2. Lavora e produce il contenuto aggiornato
3. **Aggiorna immediatamente TODO.md** segnando il contenuto come completato
4. Passa al successivo

Quando tutti i contenuti e moduli di una lingua sono completati, si passa alla lingua successiva.

> **Cruciale**: aggiornare TODO.md **subito dopo ogni contenuto**, prima di iniziare il prossimo. Se la sessione si interrompe, il lavoro completato non va perso: la prossima sessione AI legge TODO.md + STRUTTURA-PROGETTO.md e riprende dal primo contenuto non ancora segnato come completato.

---

## Fase 3: Generazione SQL

Dopo aver elaborato tutti i contenuti e moduli di tutte le lingue, generare un unico file SQL con tutte le operazioni, nell'ordine corretto. Le query specifiche (nomi tabelle, campi, formato escape) dipendono dal CMS — consultare il profilo CMS o le informazioni in STRUTTURA-PROGETTO.md.

### Ordine delle operazioni nel file SQL

1. **INSERT per struttura** (se Fase 0c è stata eseguita) — creazione di categorie, contenuti, menu, moduli e associazioni multilingue
2. **UPDATE per contenuti** — sostituzione del contenuto strutturato con i testi tradotti
3. **UPDATE per moduli/widget** — sostituzione del contenuto testuale dei moduli
4. **UPDATE per menu/routing** — aggiornamento alias/slug, title e metadati SEO. L'alias viene ricavato dall'**ultimo segmento dell'URL** del campo `Url:` nel file SEO. Non usare il titolo del menu come fonte per l'alias.
5. **INSERT per associazioni multilingue** — se non già inserite in Fase 0c

> Le stringhe vanno escaped correttamente per il DB in uso (MySQL, PostgreSQL, ecc.). Consultare il profilo CMS per i dettagli di formattazione specifici.

---

## Fase 4: Importazione e verifica

1. Importare il file SQL via il tool di gestione DB del progetto (phpMyAdmin, Plesk, pgAdmin, WP-CLI, Drush, ecc.)
2. Spot-check su 3-4 pagine rappresentative **per ogni lingua**:
   - Testi tradotti visibili sul frontend
   - Elementi dinamici/template ancora nella lingua sorgente (è corretto)
   - Link interni puntano alle pagine della lingua giusta
   - `<title>` e meta description corretti nel sorgente HTML
   - Alias/slug corretti nell'URL
   - Moduli/widget tradotti visibili nelle posizioni corrette
   - Associazioni multilingue funzionanti (language switcher)
3. Navigare tutte le pagine tradotte sul frontend per ogni lingua

---

## Fase 5: Retrospettiva e aggiornamento profilo CMS

**Al termine del lavoro (dopo la Fase 4)**, l'AI chiede all'utente:

> "Il lavoro è completato. Durante questa sessione sono emersi problemi, comportamenti inattesi o workaround che vorresti documentare per le prossime volte? Se sì, li formalizzo e aggiorno il profilo CMS."

### Se l'utente segnala problemi

Per ogni problema segnalato, l'AI:

1. **Formalizza** l'issue con: descrizione del problema, causa identificata, soluzione adottata, CMS/versione/plugin coinvolti
2. **Aggiorna il file profilo CMS** nella cartella `cms-known-profiles/`:
   - Se il profilo esiste → aggiunge l'issue nella sezione "Tranelli noti e lessons learned"
   - Se il profilo non esisteva (discovery) → crea un nuovo file profilo con le informazioni scoperte durante la sessione + gli issue segnalati
3. **Prepara il commit** con un messaggio descrittivo, es.:
   - `fix(joomla-yootheme): nodi accordion con source nidificato non vanno tradotti`
   - `feat(wordpress-elementor): aggiunto profilo iniziale con discovery da sessione`
   - `docs(joomla-puro): aggiunto workaround per moduli custom senza language field`

L'utente fa review e `git push`.

### Se non ci sono problemi

L'AI lo segna nel TODO.md e il lavoro è concluso. Se il CMS non aveva un profilo e il discovery è andato liscio, l'AI propone comunque di creare il profilo con le informazioni base scoperte, per risparmiare il discovery alle prossime sessioni.

---

## Organizzazione dei file

La cartella `help-ai/` viene clonata dal repo nella root del progetto. I file di lavoro (dump, SEO, output) vanno direttamente dentro `help-ai/` — il `.gitignore` li esclude dal repo.

```
progetto/help-ai/
├── workflow.md                        ← dal repo (CMS-agnostico)
├── cms-known-profiles/                ← dal repo
│   ├── joomla-yootheme.md
│   ├── joomla-puro.md
│   ├── wordpress-elementor.md
│   └── ...                            ← cresce con l'uso
├── README.md                          ← dal repo
├── .gitignore                         ← dal repo (esclude i file di lavoro)
├── dump-db.json (o .sql)              ← messo dall'utente (ignorato da git)
├── Seo-IT.docx                        ← messo dall'utente (ignorato da git)
├── Seo-EN.docx                        ← messo dall'utente (ignorato da git)
├── TODO.md                            ← creato dall'AI (ignorato da git)
├── STRUTTURA-PROGETTO.md              ← creato dall'AI (ignorato da git)
├── fulltext/                          ← creato dall'AI (ignorato da git)
│   ├── {id}-home.json
│   ├── {id}-about.json
│   └── ...
├── seo-it-sezioni/                    ← creato dall'AI (ignorato da git)
│   ├── seo-it-home.txt
│   ├── seo-it-chi-siamo.txt
│   └── ...
├── seo-en-sezioni/                    ← creato dall'AI (ignorato da git)
│   ├── seo-en-home.txt
│   ├── seo-en-about-us.txt
│   └── ...
└── update-{progetto}.sql              ← output finale (ignorato da git)
```

Il dump completo e i file SEO unici si leggono **una volta sola** (Fase 0b e 1), poi si scrivono i file spezzettati e STRUTTURA-PROGETTO.md e non si riaprono più. Per ogni pagina l'AI carica **solo tre file**: STRUTTURA-PROGETTO.md + il contenuto strutturato + il file SEO di quella pagina.

---

## Errori da evitare

| Errore | Perché non funziona |
|---|---|
| Saltare l'identificazione del CMS (Fase 0a) | Senza sapere dove e come il CMS memorizza i contenuti, si lavora alla cieca |
| Saltare l'esplorazione del dump | Ogni progetto ha struttura diversa, anche a parità di CMS |
| Caricare tutto il dump per ogni pagina | Spreco di token; basta il singolo contenuto strutturato + il file SEO |
| Usare script con matching posizionale meccanico | Le strutture contenuto e testo tradotto non sono parallele; l'AI capisce meglio semanticamente |
| Fare regex/replace sul contenuto grezzo | La struttura nidificata si rompe |
| Tradurre elementi dinamici/template | Sono dinamici, non vengono renderizzati |
| Dimenticare i link interni | Il testo è tradotto ma i link puntano alla lingua sorgente |
| Dimenticare i metadati SEO | Le pagine non hanno Title/Description nel browser |
| Dimenticare alias/slug del routing | L'URL della pagina non corrisponde alla lingua target |
| Dimenticare i moduli/widget | Footer, banner e altri elementi restano nella lingua sorgente |
| Dimenticare le associazioni multilingue | Il language switcher non funziona |
| Non sanitizzare i file SEO | Caratteri speciali (em dash, smart quotes, BOM) corrompono i testi |
| Hardcodare regole da un altro progetto | Ogni progetto è specifico, anche con lo stesso CMS |
| Non aggiornare TODO.md dopo ogni contenuto | Se la sessione si interrompe, non si sa da dove riprendere |
| Rifare esplorazione per ogni lingua | STRUTTURA-PROGETTO.md e TODO.md sono unici e condivisi tra tutte le lingue |
| Non fare la retrospettiva (Fase 5) | I problemi scoperti non vengono documentati e si ripetono nel progetto successivo |
