# CMS Profile: Joomla 5/6 + YOOtheme Builder

> **Compatibilità testata**: Joomla 5.x, 6.x con YOOtheme Builder 4.x
> **Ultimo aggiornamento**: vedi history git

---

## Architettura del contenuto

In Joomla con YOOtheme Builder, ogni articolo ha due campi nel DB nella tabella `{prefix}_content`:

- **`introtext`**: HTML semplificato, usato per anteprime e listing
- **`fulltext`**: JSON del page builder YOOtheme, racchiuso in un commento HTML `<!-- {...} -->`. **Questo è il campo che YOOtheme renderizza sul frontend.**

Aggiornare solo `introtext` non basta. Bisogna estrarre il JSON dal wrapper `<!-- -->`, modificarlo, e riavvolgerlo nel wrapper.

### Struttura del JSON fulltext

Il JSON ha struttura ad albero:

```
layout → section → row → column → elementi foglia
```

Ogni elemento ha:
- `type`: il tipo di nodo (headline, text, button_item, ecc.)
- `props`: oggetto con proprietà, incluse quelle con testo visibile e link
- `children`: array di nodi figli

La struttura esatta (quali type esistono e quali props contengono testo) **varia da progetto a progetto** e va scoperta in Fase 1.

### Tipi di nodo comuni

Questi sono i tipi di nodo più frequenti nei progetti YOOtheme. La lista non è esaustiva — la Fase 1 potrebbe trovare tipi aggiuntivi specifici del progetto.

| Tipo nodo        | Props testo tipiche   | Props link | Note                        |
|------------------|-----------------------|------------|-----------------------------|
| headline         | content               | link       |                             |
| text             | content               | —          | Può contenere HTML ricco    |
| html             | content               | —          | Codice HTML arbitrario      |
| button_item      | content               | link       |                             |
| list_item        | content               | link       |                             |
| grid_item        | title, content, meta  | link       |                             |
| accordion_item   | title, content        | —          | FAQ, sezioni espandibili    |
| slideshow_item   | title, content, meta  | link       |                             |
| panel            | title, content, meta  | link       |                             |
| map_item         | title, content        | —          | Marker su mappa             |
| overlay_item     | title, content, meta  | link       |                             |

---

## Tabelle DB coinvolte

| Entità             | Tabella                        | Campi chiave                                                           |
|--------------------|--------------------------------|------------------------------------------------------------------------|
| Articoli           | `{prefix}_content`             | id, title, alias, introtext, fulltext, catid, state, language, publish_up, modified |
| Menu item          | `{prefix}_menu`                | id, title, alias, link, params (JSON), language, published             |
| Categorie          | `{prefix}_categories`          | id, title, alias, language, published                                  |
| Moduli             | `{prefix}_modules`             | id, title, module, position, content, params, language, published      |
| Moduli ↔ Menu      | `{prefix}_modules_menu`        | moduleid, menuid                                                       |
| Associazioni       | `{prefix}_associations`        | id, context, key                                                       |

### Prefisso tabelle

Il prefisso `{prefix}` varia per installazione (es. `jos`, `jml5`, `abc`). Va identificato dal dump.

---

## Sistema multilingua

### Come funziona

Joomla gestisce il multilingua nativamente. Ogni entità ha un campo `language` con il codice lingua (es. `de-DE`, `it-IT`, `en-GB`). Le associazioni tra entità equivalenti in lingue diverse passano dalla tabella `{prefix}_associations`.

### Tabella associations

Ogni record ha:
- `id`: l'ID dell'entità (articolo, menu item, modulo)
- `context`: il tipo di entità (es. `com_content.item`, `com_menus.item`, `com_modules.module`)
- `key`: una chiave condivisa tra tutte le entità che rappresentano lo stesso contenuto nelle diverse lingue

Tutte le entità dello stesso "gruppo" multilingue condividono la stessa `key`.

```sql
-- Esempio: articolo "Home" in 3 lingue
INSERT INTO `{prefix}_associations` (`id`, `context`, `key`)
VALUES
  (301, 'com_content.item', 'abc123'),  -- DE
  (334, 'com_content.item', 'abc123'),  -- IT
  (340, 'com_content.item', 'abc123');  -- EN
```

### Context per entità

| Entità      | Context                |
|-------------|------------------------|
| Articoli    | `com_content.item`     |
| Menu item   | `com_menus.item`       |
| Categorie   | `com_categories.item`  |
| Moduli      | `com_modules.module`   |

---

## Routing e link interni

### Formato link interni

I link interni in Joomla + YOOtheme usano tipicamente il formato:

```
index.php?Itemid={menu_item_id}&lang={codice_lingua}
```

Dove `{codice_lingua}` è nel formato breve (es. `it`, `de`, `en`).

Questi link si trovano in due posti nel fulltext JSON:
- Nelle **props `link`** dei nodi (es. `button_item`, `headline`)
- Nei **tag `<a href="...">`** dentro i contenuti HTML dei nodi `text` e `html`

### Alias dei menu item

L'alias del menu item determina lo slug nell'URL della pagina. Va aggiornato con l'ultimo segmento dell'URL dal file SEO (es. `/it/qualcosa/chi-siamo/` → `chi-siamo`).

**Attenzione**: il titolo del menu e l'alias non hanno sempre corrispondenza diretta. La fonte per l'alias è sempre l'URL nel file SEO, mai il titolo.

---

## Metadati SEO dei menu item

I metadati SEO sono memorizzati nel campo `params` (JSON) dei menu item:

- `page_title`: il tag `<title>` nel browser
- `menu-meta_description`: il tag `<meta name="description">`

### Query UPDATE per metadati SEO + alias

```sql
UPDATE `{prefix}_menu`
SET
  `alias` = '{alias_from_url_slug}',
  `title` = '{titolo_menu_tradotto}',
  `params` = JSON_SET(`params`,
    '$.page_title', '{page_title_tradotto}',
    '$."menu-meta_description"', '{meta_description_tradotta}')
WHERE `id` = {menu_item_id};
```

(`JSON_SET` richiede MySQL 5.7+ / MariaDB 10.2+.)

---

## Riconoscimento nodi template (da non tradurre)

In YOOtheme, alcuni nodi nel fulltext JSON generano contenuto dinamicamente e **non vanno tradotti**. Le regole più comuni sono:

- Nodi con proprietà `"source"` nel nodo → template (contenuto generato da Joomla, es. lista articoli)
- Nodi con `props.status = "disabled"` → disabilitati, non renderizzati
- Figli di nodi template → ereditano lo stato template

> **Importante**: queste regole vanno verificate sul progetto specifico in Fase 1. Alcuni progetti possono avere pattern aggiuntivi.

---

## Creazione entità per lingua target (Fase 0c)

### Articoli (`{prefix}_content`)

Campi da impostare per il nuovo articolo:
- `title`: titolo dalla lingua sorgente (verrà eventualmente aggiornato)
- `alias`: alias localizzato
- `introtext`: copiato dall'originale
- `fulltext`: copiato dall'originale (verrà sovrascritto in Fase 2)
- `catid`: ID della categoria target
- `state`: `1` (pubblicato)
- `language`: codice lingua (es. `it-IT`)
- `publish_up`: `NOW()` (data corrente)
- `created`: `NOW()`
- `access`: stesso dell'originale (tipicamente `1` = public)

### Menu item (`{prefix}_menu`)

Campi da impostare:
- `title`: titolo tradotto
- `alias`: alias localizzato (verrà aggiornato dal file SEO in Fase 3)
- `link`: `index.php?option=com_content&view=article&id={article_id_target}`
- `language`: codice lingua
- `published`: `1`
- `parent_id`: stesso dell'originale (o l'equivalente nel menu della lingua target)
- `menutype`: il menutype della lingua target (es. `mainmenu-it`)
- `type`: `component`
- `component_id`: stesso dell'originale (ID del componente com_content)
- `params`: copiati dall'originale (verranno aggiornati con SEO in Fase 3)

### Categorie (`{prefix}_categories`)

Campi da impostare:
- `title`: titolo localizzato
- `alias`: alias localizzato
- `language`: codice lingua
- `published`: `1`
- `extension`: `com_content`
- `access`: stesso dell'originale

### Moduli (`{prefix}_modules`)

Campi da impostare:
- `title`: titolo localizzato
- `module`: stesso dell'originale (es. `mod_custom`)
- `position`: stessa dell'originale
- `content`: copiato dall'originale (verrà sovrascritto in Fase 2)
- `language`: codice lingua
- `published`: `1`
- `params`: copiati dall'originale
- `access`: stesso dell'originale

Ricordare di creare anche i record in `{prefix}_modules_menu` per le assegnazioni alle pagine.

---

## Formato query SQL

### UPDATE fulltext articolo

```sql
UPDATE `{prefix}_content`
SET `fulltext` = '{fulltext_json_escaped}',
    `modified` = NOW()
WHERE `id` = {article_id};
```

**Escape**: il JSON va riavvolto nel wrapper `<!-- ... -->` e l'intera stringa va escaped per MySQL:
- `\` → `\\`
- `'` → `\'`
- newline → `\n`

### UPDATE modulo

```sql
UPDATE `{prefix}_modules`
SET `content` = '{content_escaped}'
WHERE `id` = {module_id};
```

### INSERT associazione

```sql
INSERT INTO `{prefix}_associations` (`id`, `context`, `key`)
VALUES ({entity_id}, '{context}', '{shared_key}');
```

---

## Tranelli noti e lessons learned

<!-- Questa sezione cresce con l'uso. Ogni entry viene aggiunta in Fase 5 (Retrospettiva) quando l'utente segnala problemi. -->

### #1 — Articoli creati da AI restano in stato "sospeso"

- **Problema**: quando Claude Code crea gli articoli per la lingua target senza istruzioni esplicite, li imposta con `state = 0` (non pubblicato) o con data di pubblicazione futura
- **Causa**: comportamento conservativo dell'AI che evita di pubblicare contenuti non verificati
- **Soluzione**: specificare esplicitamente `state = 1` e `publish_up = NOW()` nelle direttive di creazione (già incluso in questo profilo nella sezione Fase 0c)

### #2 — Caratteri speciali da file DOCX cross-platform

- **Problema**: em dash, smart quotes e altri caratteri "ricchi" dai file DOCX corrompono i testi quando inseriti nel JSON
- **Causa**: file DOCX passati tra Mac, Windows, Google Docs accumulano encoding misti
- **Soluzione**: sanitizzazione obbligatoria in Fase 0b (vedi workflow.md)
