# CMS Profile: Joomla 5/6 (senza page builder)

> **Compatibilita testata**: Joomla 5.x, 6.x senza page builder (contenuto HTML puro)
> **Ultimo aggiornamento**: vedi history git

---

## Architettura del contenuto

In Joomla senza page builder, ogni articolo ha due campi nel DB nella tabella `{prefix}_content`:

- **`introtext`**: HTML del contenuto, renderizzato sul frontend
- **`fulltext`**: solitamente vuoto; se presente, viene mostrato dopo il "read more"

Il contenuto e HTML puro con tag standard: `<p>`, `<strong>`, `<br>`, `<h2>`, `<ul>`, `<li>`, `<span>`, `<div>`.

### Contenuto dinamico

Alcuni articoli possono contenere codice dinamico tramite plugin (es. Sourcerer):
```
{source}<?php ... ?>{/source}
```
Questi blocchi **non vanno tradotti** — generano contenuto a runtime.

Anche i tag `{module id="..."}` non vanno tradotti: caricano un modulo Joomla per ID.

---

## Tabelle DB coinvolte

| Entita | Tabella | Campi chiave |
|--------|---------|-------------|
| Articoli | `{prefix}_content` | id, title, alias, introtext, fulltext, catid, state, language, publish_up, modified, created_by, ordering |
| Menu item | `{prefix}_menu` | id, title, alias, link, params (JSON), language, published, menutype, parent_id, component_id, type |
| Categorie | `{prefix}_categories` | id, title, alias, description, language, published, parent_id, extension, path |
| Moduli | `{prefix}_modules` | id, title, module, position, content, params, language, published |
| Moduli - Menu | `{prefix}_modules_menu` | moduleid, menuid |
| Associazioni | `{prefix}_associations` | id, context, key |
| Menu types | `{prefix}_menu_types` | id, menutype, title |

### Prefisso tabelle

Il prefisso `{prefix}` varia per installazione (es. `jos`, `te7qc`, `abc`). Va identificato dal dump.

---

## Sistema multilingua

### Come funziona

Joomla gestisce il multilingua nativamente. Ogni entita ha un campo `language` con il codice lingua (es. `de-DE`, `it-IT`, `en-GB`). Le associazioni tra entita equivalenti in lingue diverse passano dalla tabella `{prefix}_associations`.

### Tabella associations

Ogni record ha:
- `id`: l'ID dell'entita (articolo, menu item, modulo)
- `context`: il tipo di entita
- `key`: una chiave condivisa tra tutte le entita che rappresentano lo stesso contenuto nelle diverse lingue

Tutte le entita dello stesso "gruppo" multilingue condividono la stessa `key`.

### Context per entita

| Entita | Context |
|--------|---------|
| Articoli | `com_content.item` |
| Menu item | `com_menus.item` |
| Categorie | `com_categories.item` |
| Moduli | `com_modules.module` |

### Struttura menu per lingua

Joomla usa un menutype separato per ogni lingua:
- `menu-it`, `menu-de`, `menu-en` per i menu principali
- `footer-it`, `footer-de`, `footer-en` per i footer
- `hidden-it`, `hidden-de`, `hidden-en` per pagine nascoste (404, thank you, ecc.)

---

## Routing e link interni

### Formato link interni

I link interni in Joomla usano tipicamente il formato:

```
index.php?option=com_content&view=article&id={article_id}&Itemid={menu_item_id}&lang={codice_lingua}
```

Per le categorie blog:
```
index.php?option=com_content&view=category&layout=blog&id={category_id}
```

### Alias dei menu item

L'alias del menu item determina lo slug nell'URL della pagina. Va aggiornato con l'ultimo segmento dell'URL dal file SEO (es. `/de/wanderungen-dolomiten` -> `wanderungen-dolomiten`).

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

## Creazione entita per lingua target (Fase 0c)

### Articoli (`{prefix}_content`)

Campi da impostare per il nuovo articolo:
- `title`: titolo tradotto (OBBLIGATORIO - ogni articolo DEVE avere un titolo)
- `alias`: alias univoco (OBBLIGATORIO - ogni articolo DEVE avere un alias univoco nel DB)
- `introtext`: HTML tradotto
- `fulltext`: copiato dall'originale (solitamente vuoto)
- `catid`: ID della categoria target
- `state`: `1` (pubblicato)
- `language`: codice lingua (es. `de-DE`)
- `publish_up`: `NOW()`
- `created`: `NOW()`
- `created_by`: stesso dell'originale
- `access`: stesso dell'originale (tipicamente `1` = public)
- `ordering`: stesso dell'originale (per mantenere l'ordine)

### Menu item (`{prefix}_menu`)

Campi da impostare:
- `title`: titolo tradotto (OBBLIGATORIO)
- `alias`: alias dall'URL nel file SEO (OBBLIGATORIO, univoco)
- `link`: `index.php?option=com_content&view=category&layout=blog&id={category_id_target}` (o `&view=article&id={article_id_target}`)
- `language`: codice lingua
- `published`: `1`
- `parent_id`: stesso dell'originale (o l'equivalente nel menu della lingua target)
- `menutype`: il menutype della lingua target (es. `menu-de`)
- `type`: `component`
- `component_id`: stesso dell'originale
- `params`: copiati dall'originale, con page_title e menu-meta_description tradotti

### Categorie (`{prefix}_categories`)

Campi da impostare:
- `title`: titolo tradotto (OBBLIGATORIO)
- `alias`: alias univoco (OBBLIGATORIO)
- `description`: HTML tradotto della descrizione categoria
- `language`: codice lingua
- `published`: `1`
- `extension`: `com_content`
- `access`: stesso dell'originale
- `path`: uguale all'alias per categorie di primo livello

### Regola universale: titolo e alias

**Ogni entita in Joomla (articolo, categoria, menu item) DEVE avere:**
1. Un **titolo** non vuoto
2. Un **alias univoco** nel suo contesto

Per gli alias:
- Menu items: l'alias viene dall'URL nel file SEO
- Articoli e categorie: l'alias NON e fornito nel file SEO. Va generato dall'AI a partire dal titolo tradotto, in formato slug (lowercase, trattini, senza caratteri speciali). Se lo slug risulterebbe identico a un alias gia esistente, aggiungere un suffisso (es. `-de`, `-en`)

---

## Formato query SQL

### INSERT articolo

```sql
INSERT INTO `{prefix}_content`
(`id`, `title`, `alias`, `introtext`, `fulltext`, `state`, `catid`,
 `created`, `created_by`, `modified`, `modified_by`, `publish_up`,
 `access`, `ordering`, `language`)
VALUES ({id}, '{title}', '{alias}', '{introtext}', '', 1, {catid},
 NOW(), {created_by}, NOW(), {created_by}, NOW(), 1, {ordering}, '{lang}');
```

### INSERT categoria

```sql
INSERT INTO `{prefix}_categories`
(`id`, `parent_id`, `lft`, `rgt`, `level`, `path`, `extension`,
 `title`, `alias`, `description`, `published`, `access`, `params`,
 `metadesc`, `metakey`, `metadata`, `language`,
 `created_user_id`, `created_time`, `modified_user_id`, `modified_time`, `version`)
VALUES ({id}, 1, 0, 0, 1, '{alias}', 'com_content',
 '{title}', '{alias}', '{description}', 1, 1,
 '{"category_layout":"","image":"","image_alt":""}',
 '', '', '{"author":"","robots":""}', '{lang}',
 {created_by}, NOW(), {created_by}, NOW(), 1);
```

> **Nota**: `lft` e `rgt` vanno impostati a 0; dopo l'import, ricostruire l'albero da Joomla admin.

### INSERT menu item

```sql
INSERT INTO `{prefix}_menu`
(`id`, `menutype`, `title`, `alias`, `path`, `link`, `type`, `published`,
 `parent_id`, `level`, `component_id`, `access`, `params`,
 `lft`, `rgt`, `home`, `language`, `client_id`)
VALUES ({id}, '{menutype}', '{title}', '{alias}', '{alias}',
 '{link}', 'component', 1, 1, 1, {component_id}, 1, '{params_json}',
 0, 0, 0, '{lang}', 0);
```

> **Nota**: `lft` e `rgt` vanno impostati a 0; dopo l'import, ricostruire il menu da Joomla admin.

### INSERT associazione

```sql
INSERT INTO `{prefix}_associations` (`id`, `context`, `key`)
VALUES ({entity_id}, '{context}', '{shared_key}');
```

**Escape**: le stringhe vanno escaped per MySQL:
- `\` -> `\\`
- `'` -> `''` (o `\'`)
- newline -> `\n`

---

## Post-import: operazioni obbligatorie

Dopo l'importazione del file SQL, eseguire in Joomla admin:

1. **Content > Categories > Ricostruisci** — ricalcola lft/rgt dell'albero categorie
2. **Menus > [menu interessato] > Ricostruisci** — ricalcola lft/rgt dell'albero menu

> **ATTENZIONE**: System > Maintenance > Database > Fix **NON ricrea gli asset mancanti**. Joomla non rileva entita senza asset come problema. Gli asset vanno inseriti manualmente nel file SQL (vedi sezione dedicata sotto).

---

## Tranelli noti e lessons learned

<!-- Questa sezione cresce con l'uso. Ogni entry viene aggiunta in Fase 5 (Retrospettiva) quando l'utente segnala problemi. -->

### #1 - Titolo e alias obbligatori per ogni entita

- **Problema**: se un articolo, categoria o menu item viene creato senza titolo o con alias duplicato, Joomla non lo mostra o genera errori
- **Causa**: Joomla richiede titolo non vuoto e alias univoco per ogni entita
- **Soluzione**: generare SEMPRE un titolo tradotto e un alias univoco. Per articoli e categorie l'alias non e nel file SEO: va derivato dal titolo. Aggiungere suffisso lingua (`-de`, `-en`) se serve per unicita

### #2 - lft/rgt a zero per categorie e menu items

- **Problema**: dopo INSERT con lft=0 e rgt=0, le entita non appaiono nel backend Joomla
- **Causa**: Joomla usa nested set (lft/rgt) per l'albero; senza valori corretti l'entita e "invisibile"
- **Soluzione**: inserire con lft=0, rgt=0, poi eseguire "Ricostruisci" dal manager categorie/menu in Joomla admin. Documentare questo passaggio nelle note post-import del file SQL

### #3 - Assets DEVONO essere inseriti a mano

- **Problema**: le nuove entita (articoli, categorie) non appaiono nel backend Joomla, pur essendo presenti nel DB
- **Causa**: ogni entita in Joomla richiede un record nella tabella `{prefix}_assets`. Senza asset, l'entita e invisibile. **Database Fix di Joomla NON rileva ne corregge questo problema.**
- **Soluzione**: il file SQL DEVE includere INSERT nella tabella `{prefix}_assets` per ogni nuova entita. La struttura dell'asset e:
  - `name`: `com_content.article.{id}` per articoli, `com_content.category.{id}` per categorie
  - `parent_id`: l'asset_id del parent (per articoli = asset della categoria, per categorie = asset di com_content)
  - `title`: titolo dell'entita
  - `rules`: `{}` (eredita dal parent)
  - `lft`, `rgt`: valori sequenziali oltre il max corrente; si ricostruiscono poi aprendo e salvando una qualsiasi entita in Joomla admin
- Dopo gli INSERT degli asset, aggiornare il campo `asset_id` nelle tabelle `{prefix}_content` e `{prefix}_categories` per puntare al nuovo asset

### #4 - Posizionamento intelligente di categorie e menu items

- **Problema**: le nuove categorie e menu items appaiono in fondo alla lista o in posizione casuale, non vicino ai loro equivalenti nelle altre lingue
- **Causa**: lft/rgt determinano la posizione nell'albero; con valori 0 o sequenziali alla fine, le entita finiscono fuori posto
- **Soluzione**: le nuove entita devono essere posizionate in modo intelligente rispetto ai loro cloni in altre lingue. Per esempio, se la categoria IT e in posizione 5 tra le categorie, la categoria DE va tra le altre categorie DE nella stessa posizione logica. Questo va calcolato leggendo i valori lft/rgt esistenti dal dump e inserendo le nuove entita in posizioni coerenti. In alternativa, inserire con lft=0/rgt=0 e riposizionare manualmente dal backend Joomla dopo il Ricostruisci

### #5 - Workflow associations obbligatorie per articoli (Joomla 4/5)

- **Problema**: gli articoli inseriti via SQL non appaiono nel backend Joomla (ne nella lista articoli ne nel frontend), anche se state=1, asset e categoria sono corretti
- **Causa**: Joomla 4/5 usa un sistema di workflow. Ogni articolo DEVE avere un record in `{prefix}_workflow_associations` con `stage_id` corretto, altrimenti e completamente invisibile
- **Soluzione**: per ogni nuovo articolo, inserire:
  ```sql
  INSERT INTO `{prefix}_workflow_associations` (`item_id`, `stage_id`, `extension`)
  VALUES ({article_id}, 1, 'com_content.article');
  ```
  Lo `stage_id` va verificato dalla tabella `{prefix}_workflow_stages` (tipicamente 1 = "Basic Stage" / Published). Verificare prima: `SELECT * FROM {prefix}_workflow_stages;`

### #6 - Campi JSON obbligatori per articoli

- **Problema**: aprendo un articolo nel backend Joomla esce "Error decoding JSON data: Control character error, possibly incorrectly encoded"
- **Causa**: i campi `images`, `urls`, `attribs`, `metadata` sono NULL invece di JSON valido. Joomla si aspetta JSON valido in questi campi
- **Soluzione**: copiare i valori di questi campi da un articolo esistente della stessa struttura (stesso template/layout). Il modo piu sicuro:
  ```sql
  UPDATE {prefix}_content AS dest
  JOIN {prefix}_content AS src ON src.id = {id_articolo_sorgente}
  SET dest.images = src.images, dest.urls = src.urls,
      dest.attribs = src.attribs, dest.metadata = src.metadata
  WHERE dest.id BETWEEN {primo_id} AND {ultimo_id};
  ```

### #8 - attribs (article_layout) va copiato 1:1 dall'articolo sorgente

- **Problema**: tutti gli articoli tradotti hanno lo stesso layout (es. `teresa:accordion`) anche se gli articoli sorgente IT avevano layout diversi (es. alcuni `default`, altri `accordion`)
- **Causa**: copiare `attribs` da un singolo articolo sorgente sovrascrive il layout di tutti. Ogni articolo ha il proprio `article_layout` dentro `attribs`
- **Soluzione**: copiare `attribs` articolo per articolo dal corrispondente sorgente IT, NON da un unico articolo. Usare:
  ```sql
  UPDATE {prefix}_content AS dest
  JOIN {prefix}_content AS src ON src.id = {id_articolo_IT_corrispondente}
  SET dest.attribs = src.attribs
  WHERE dest.id = {id_articolo_tradotto};
  ```
  Ripetere per ogni coppia IT→DE e IT→EN.

### #7 - publish_up con un giorno di anticipo

- **Problema**: articoli creati con `publish_up = NOW()` potrebbero non apparire sul frontend a causa di differenze di fuso orario tra server e Joomla
- **Causa**: il fuso orario configurato in Joomla puo differire da quello del server MySQL; `NOW()` usa il fuso del server
- **Soluzione**: impostare `publish_up` a un giorno prima della data corrente:
  ```sql
  publish_up = DATE_SUB(NOW(), INTERVAL 1 DAY)
  ```
