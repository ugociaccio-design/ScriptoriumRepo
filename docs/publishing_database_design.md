# Modello dati gestionale per piccola casa editrice

## A) Sintesi del modello dati
- **Anagrafiche**: entità unificata `parties` (persona/azienda) con contatti multipli (`party_contacts`) e ruoli multipli (`party_roles`).
- **Catalogo**: separazione tra `works` (opera/contenuto) e `editions` (prodotto/ISBN). Collane/imprint (`imprints`, `series`), parole chiave normalizzate (`work_keywords`).
- **Contributori**: associazioni molti-a-molti per opera (`work_contributors`) e, se necessario, override per edizione (`edition_contributors`) con ruolo e percentuale royalty.
- **Pipeline editoriale**: proposte/manoscritti (`manuscripts`) e task/milestone (`workflow_tasks`) assegnabili a utenti/ruoli.
- **Contratti e diritti**: `rights_contracts` per autori/traduttori, termini di royalty per formato/canale (`royalty_terms`), anticipo e reporting. Diritti secondari opzionali (`secondary_rights`).
- **Produzione e stampa**: ordini a tipografia (`production_orders`) e lotti di stampa (`print_runs`) collegati alle edizioni.
- **Magazzino**: magazzini (`warehouses`) e movimenti (`inventory_movements`) con causali e documento di riferimento.
- **Vendite**: ordini B2B (`sales_orders`, `sales_order_lines`), documenti fiscali (`sales_invoices`, `sales_invoice_lines`), resi (`sales_returns`), canali (`sales_channels`).
- **Royalties**: periodi di rendicontazione (`royalty_periods`), statement per contratto/contributore (`royalty_statements`, `royalty_statement_lines`), pagamenti anticipo/recupero.
- **Marketing** (opzionale): eventi/promozioni (`marketing_events`) e invii copia stampa/omaggi (`press_shipments`).

Assunzioni: PostgreSQL 15+, valuta EUR di default, auditing leggero con `created_at/updated_at`, nessun blob in DB (solo path).

## B) ERD (Mermaid)
```mermaid
erDiagram
  parties ||--o{ party_contacts : has
  parties ||--o{ party_roles : has
  parties ||--o{ users : staff
  roles ||--o{ party_roles : defines

  imprints ||--o{ series : contains
  series ||--o{ works : groups
  works ||--o{ editions : produces
  works ||--o{ work_keywords : tags
  works ||--o{ work_contributors : has
  editions ||--o{ edition_contributors : override

  works ||--o{ manuscripts : originates
  works ||--o{ workflow_tasks : tasks
  editions ||--o{ workflow_tasks : tasks
  users ||--o{ workflow_tasks : assigned

  parties ||--o{ rights_contracts : signs
  works ||--o{ rights_contracts : covers
  rights_contracts ||--o{ royalty_terms : defines
  rights_contracts ||--o{ secondary_rights : sells

  editions ||--o{ production_orders : printed
  production_orders ||--o{ print_runs : results
  editions ||--o{ print_runs : batches

  warehouses ||--o{ inventory_movements : track
  editions ||--o{ inventory_movements : move

  sales_channels ||--o{ sales_orders : via
  parties ||--o{ sales_orders : customer
  sales_orders ||--o{ sales_order_lines : items
  sales_orders ||--o{ sales_invoices : billed
  sales_invoices ||--o{ sales_invoice_lines : lines
  sales_invoice_lines ||--o{ sales_returns : return
  editions ||--o{ sales_invoice_lines : sold

  royalty_periods ||--o{ royalty_statements : holds
  rights_contracts ||--o{ royalty_statements : due
  royalty_statements ||--o{ royalty_statement_lines : detail
  editions ||--o{ royalty_statement_lines : counted

  marketing_events ||--o{ press_shipments : uses
  editions ||--o{ press_shipments : sent
  parties ||--o{ press_shipments : to
```

## C) Tabelle, campi e vincoli
Formato: `campo (tipo) [obbligatorio] {default}`. PK/FK e indici suggeriti per tabella.

### roles
- Scopo: codifica dei ruoli (autore, traduttore, fornitore, cliente, staff, ecc.).
- Campi: `code (text) [s]` PK; `description (text)`.
- Vincoli: PK `code`.
- Indici: PK.

### parties
- Scopo: anagrafica unificata (persona/azienda).
- Campi: `party_id (serial) [s]`; `party_type (text) [s] CHECK IN ('person','organization')`; `display_name (text) [s]`; `legal_name (text)`; `tax_id (text)`; `notes (text)`; `created_at (timestamptz) {now()}`; `updated_at (timestamptz) {now()}`.
- Vincoli: PK `party_id`; UNIQUE(`tax_id`) NULL-ok.
- Indici: `display_name`, `party_type`.

### party_contacts
- Scopo: contatti multipli per soggetto.
- Campi: `contact_id (serial) [s]`; `party_id (int) [s] FK parties`; `contact_type (text) [s] CHECK IN ('email','phone','address','website')`; `label (text)`; `value (text) [s]`; `is_primary (bool) {false}`.
- Vincoli: PK `contact_id`; FK `party_id`; UNIQUE(`party_id`,`contact_type`,`value`).
- Indici: `party_id`, filtro su `is_primary`.

### party_roles
- Scopo: ruoli multipli per party.
- Campi: `party_id (int) [s] FK parties`; `role_code (text) [s] FK roles`.
- Vincoli: PK composita(`party_id`,`role_code`).
- Indici: `role_code`.

### users
- Scopo: utenti interni collegati a party.
- Campi: `user_id (serial) [s]`; `party_id (int) [s] FK parties`; `email (text) [s]`; `password_hash (text)`; `is_active (bool){true}`; `created_at (timestamptz){now()}`.
- Vincoli: PK; UNIQUE(`email`).
- Indici: `party_id`, `is_active`.

### imprints
- Scopo: marchio/casa editrice interna.
- Campi: `imprint_id (serial) [s]`; `name (text) [s]`; `description (text)`.
- Vincoli: PK; UNIQUE(`name`).
- Indici: `name`.

### series
- Scopo: collane/linee editoriali.
- Campi: `series_id (serial) [s]`; `imprint_id (int) FK imprints`; `name (text) [s]`; `description (text)`.
- Vincoli: PK; UNIQUE(`imprint_id`,`name`).
- Indici: `imprint_id`, `name`.

### works
- Scopo: opera/contenuto.
- Campi: `work_id (serial) [s]`; `series_id (int) FK series`; `title (text) [s]`; `subtitle (text)`; `synopsis (text)`; `genre (text)`; `language_original (text)`; `status (text) [s] CHECK IN ('development','in_review','in_contract','in_production','published','out_of_print')`; `planned_pub_date (date)`; `publication_date (date)`; `created_at (timestamptz){now()}`; `updated_at (timestamptz){now()}`.
- Vincoli: PK; FK `series_id`.
- Indici: `status`, `title`, `publication_date`.

### work_keywords
- Scopo: parole chiave normalizzate.
- Campi: `work_id (int) [s] FK works`; `keyword (text) [s]`.
- Vincoli: PK composita(`work_id`,`keyword`).
- Indici: `keyword` (ricerca).

### editions
- Scopo: prodotto/edizione con ISBN/format.
- Campi: `edition_id (serial) [s]`; `work_id (int) [s] FK works`; `imprint_id (int) FK imprints`; `format (text) [s] CHECK IN ('print','ebook','audiobook')`; `isbn13 (char(13)) UNIQUE`; `ean (text)`; `internal_code (text)`; `status (text) [s] CHECK IN ('announced','in_printing','published','out_of_print')`; `publication_date (date)`; `list_price (numeric(10,2))`; `currency (char(3)){'EUR'}`; `page_count (int)`; `file_format (text)`; `dimensions (text)`; `stock_min (int)`; `created_at (timestamptz){now()}`; `updated_at (timestamptz){now()}`.
- Vincoli: PK; FK `work_id`, `imprint_id`; UNIQUE(`internal_code`) NULL-ok.
- Indici: `work_id`, `format`, `status`, `publication_date`, `isbn13` (unique).

### work_contributors
- Scopo: contributori per opera con percentuali base.
- Campi: `contrib_id (serial) [s]`; `work_id (int) [s] FK works`; `party_id (int) [s] FK parties`; `role_code (text) [s] FK roles`; `royalty_share (numeric(5,2)) CHECK (royalty_share BETWEEN 0 AND 100)`.
- Vincoli: PK; UNIQUE(`work_id`,`party_id`,`role_code`).
- Indici: `party_id`, `role_code`.

### edition_contributors
- Scopo: override contributori/percentuali per edizione.
- Campi: `edition_id (int) [s] FK editions`; `party_id (int) [s] FK parties`; `role_code (text) [s] FK roles`; `royalty_share (numeric(5,2)) CHECK (royalty_share BETWEEN 0 AND 100)`.
- Vincoli: PK composita(`edition_id`,`party_id`,`role_code`).
- Indici: `party_id`.

### manuscripts
- Scopo: gestione proposte/manoscritti.
- Campi: `manuscript_id (serial) [s]`; `work_id (int) FK works`; `title (text) [s]`; `source (text)`; `status (text) [s] CHECK IN ('submitted','in_review','rejected','optioned','contracted')`; `submission_date (date)`; `decision_date (date)`; `notes (text)`; `file_path (text)`.
- Vincoli: PK; FK `work_id` (null se non ancora collegato).
- Indici: `status`, `submission_date`.

### workflow_tasks
- Scopo: task/milestone editoriali.
- Campi: `task_id (serial) [s]`; `work_id (int) FK works`; `edition_id (int) FK editions`; `manuscript_id (int) FK manuscripts`; `task_type (text) [s] CHECK IN ('reading','evaluation','editing','copyediting','proof','layout','cover','print_order','marketing','publication')`; `status (text) [s] CHECK IN ('todo','in_progress','blocked','done')`; `assigned_to (int) FK users`; `due_date (date)`; `completed_at (timestamptz)`; `notes (text)`; `created_at (timestamptz){now()}`.
- Vincoli: PK; almeno uno tra `work_id`,`edition_id`,`manuscript_id`; FK.
- Indici: `status`, `due_date`, `assigned_to`.

### rights_contracts
- Scopo: contratti con autori/traduttori.
- Campi: `contract_id (serial) [s]`; `work_id (int) [s] FK works`; `party_id (int) [s] FK parties`; `role_code (text) [s] FK roles`; `start_date (date) [s]`; `end_date (date)`; `territory (text)`; `advance_amount (numeric(12,2)) {0}`; `reporting_frequency (text) CHECK IN ('annual','semiannual','quarterly')`; `payment_terms (text)`; `status (text) CHECK IN ('active','expired','terminated') { 'active' }`; `created_at (timestamptz){now()}`.
- Vincoli: PK; UNIQUE(`work_id`,`party_id`,`role_code`,`start_date`).
- Indici: `party_id`, `status`, `end_date`.

### royalty_terms
- Scopo: regole di royalty per formato/canale.
- Campi: `term_id (serial) [s]`; `contract_id (int) [s] FK rights_contracts`; `format (text) CHECK IN ('print','ebook','audiobook','all') { 'all' }`; `channel (text)`; `rate (numeric(5,2)) [s] CHECK (rate BETWEEN 0 AND 100)`; `threshold_units (int)`; `notes (text)`.
- Vincoli: PK; UNIQUE(`contract_id`,`format`,`channel`,`threshold_units`).
- Indici: `contract_id`, `format`, `channel`.

### secondary_rights
- Scopo: vendite diritti secondari/opzioni.
- Campi: `secondary_id (serial) [s]`; `contract_id (int) [s] FK rights_contracts`; `right_type (text)`; `licensee (text)`; `territory (text)`; `amount (numeric(12,2))`; `signature_date (date)`; `notes (text)`.
- Vincoli: PK; FK `contract_id`.
- Indici: `right_type`, `signature_date`.

### production_orders
- Scopo: ordini a tipografia.
- Campi: `order_id (serial) [s]`; `edition_id (int) [s] FK editions`; `supplier_id (int) [s] FK parties`; `specs (text)`; `quantity (int) [s]`; `unit_cost (numeric(10,2))`; `setup_cost (numeric(10,2))`; `due_date (date)`; `status (text) CHECK IN ('requested','approved','in_progress','delivered','cancelled')`; `created_at (timestamptz){now()}`.
- Vincoli: PK; FK `edition_id`,`supplier_id`.
- Indici: `supplier_id`,`due_date`,`status`.

### print_runs
- Scopo: lotti di stampa e ristampe.
- Campi: `print_run_id (serial) [s]`; `edition_id (int) [s] FK editions`; `production_order_id (int) FK production_orders`; `batch_number (text)`; `quantity (int) [s]`; `total_cost (numeric(12,2))`; `received_date (date)`.
- Vincoli: PK; FK; UNIQUE(`production_order_id`) NULL-ok.
- Indici: `edition_id`,`received_date`.

### warehouses
- Scopo: magazzini fisici/virtuali.
- Campi: `warehouse_id (serial) [s]`; `name (text) [s]`; `location (text)`; `is_default (bool){false}`.
- Vincoli: PK; UNIQUE(`name`).
- Indici: `is_default`.

### inventory_movements
- Scopo: movimenti di magazzino.
- Campi: `movement_id (serial) [s]`; `edition_id (int) [s] FK editions`; `warehouse_id (int) [s] FK warehouses`; `movement_type (text) [s] CHECK IN ('in','out','return_in','promo_out','damaged','adjustment')`; `quantity (int) [s]`; `unit_cost (numeric(10,2))`; `document_type (text)`; `document_id (int)`; `reason (text)`; `movement_date (date) [s]`;
  `created_at (timestamptz){now()}`.
- Vincoli: PK; CHECK `quantity > 0`; FK `edition_id`,`warehouse_id`.
- Indici: `edition_id`, `movement_date`, `movement_type`, `document_type,document_id`.

### sales_channels
- Scopo: catalogo canali di vendita.
- Campi: `channel_code (text) [s]`; `description (text)`.
- Vincoli: PK `channel_code`.
- Indici: PK.

### sales_orders
- Scopo: ordini clienti B2B.
- Campi: `order_id (serial) [s]`; `customer_id (int) [s] FK parties`; `channel_code (text) FK sales_channels`; `order_date (date) [s]`; `status (text) CHECK IN ('draft','confirmed','shipped','invoiced','cancelled')`; `currency (char(3)){'EUR'}`; `notes (text)`.
- Vincoli: PK; FK `customer_id`,`channel_code`.
- Indici: `customer_id`,`order_date`,`status`.

### sales_order_lines
- Scopo: righe ordine.
- Campi: `order_line_id (serial) [s]`; `order_id (int) [s] FK sales_orders`; `edition_id (int) [s] FK editions`; `quantity (int) [s]`; `unit_price (numeric(10,2))`; `discount_pct (numeric(5,2)) {0}`.
- Vincoli: PK; CHECK `quantity>0`; FK `order_id`,`edition_id`.
- Indici: `order_id`, `edition_id`.

### sales_invoices
- Scopo: fatture/note di credito.
- Campi: `invoice_id (serial) [s]`; `order_id (int) FK sales_orders`; `customer_id (int) [s] FK parties`; `document_type (text) [s] CHECK IN ('invoice','credit_note')`; `invoice_date (date) [s]`; `total_amount (numeric(12,2))`; `currency (char(3)){'EUR'}`; `notes (text)`.
- Vincoli: PK; FK `order_id`,`customer_id`.
- Indici: `customer_id`,`invoice_date`,`document_type`.

### sales_invoice_lines
- Scopo: dettaglio documento di vendita.
- Campi: `invoice_line_id (serial) [s]`; `invoice_id (int) [s] FK sales_invoices`; `edition_id (int) [s] FK editions`; `quantity (int) [s]`; `net_unit_price (numeric(10,2))`; `discount_pct (numeric(5,2)){0}`; `tax_rate (numeric(5,2)) {0}`.
- Vincoli: PK; CHECK `quantity <> 0`; FK `invoice_id`,`edition_id`.
- Indici: `invoice_id`,`edition_id`.

### sales_returns
- Scopo: resi collegati a vendite.
- Campi: `return_id (serial) [s]`; `invoice_line_id (int) [s] FK sales_invoice_lines`; `quantity (int) [s]`; `return_date (date) [s]`; `reason (text)`.
- Vincoli: PK; CHECK `quantity>0`; FK `invoice_line_id`.
- Indici: `return_date`, `invoice_line_id`.

### royalty_periods
- Scopo: periodi di rendicontazione.
- Campi: `period_id (serial) [s]`; `name (text) [s]`; `start_date (date) [s]`; `end_date (date) [s]`; `status (text) CHECK IN ('open','closed') { 'open' }`.
- Vincoli: PK; UNIQUE(`start_date`,`end_date`).
- Indici: `status`.

### royalty_statements
- Scopo: statement per contratto/periodo.
- Campi: `statement_id (serial) [s]`; `period_id (int) [s] FK royalty_periods`; `contract_id (int) [s] FK rights_contracts`; `generated_at (timestamptz){now()}`; `amount_due (numeric(12,2))`; `advance_applied (numeric(12,2)) {0}`; `notes (text)`.
- Vincoli: PK; UNIQUE(`period_id`,`contract_id`).
- Indici: `contract_id`.

### royalty_statement_lines
- Scopo: dettaglio calcolo royalties.
- Campi: `line_id (serial) [s]`; `statement_id (int) [s] FK royalty_statements`; `edition_id (int) [s] FK editions`; `channel_code (text)`; `units_sold (int) [s]`; `units_returned (int) {0}`; `net_units (int) GENERATED ALWAYS AS (units_sold - COALESCE(units_returned,0)) STORED`; `net_sales_amount (numeric(12,2))`; `royalty_rate (numeric(5,2))`; `royalty_amount (numeric(12,2))`.
- Vincoli: PK; FK `statement_id`,`edition_id`; CHECK `units_sold>=0`; CHECK `royalty_rate>=0 AND royalty_rate<=100`.
- Indici: `statement_id`,`edition_id`.

### marketing_events
- Scopo: eventi/fiere/presentazioni.
- Campi: `event_id (serial) [s]`; `name (text) [s]`; `event_date (date)`; `location (text)`; `notes (text)`.
- Vincoli: PK.
- Indici: `event_date`.

### press_shipments
- Scopo: invii stampa/omaggi.
- Campi: `shipment_id (serial) [s]`; `event_id (int) FK marketing_events`; `edition_id (int) [s] FK editions`; `recipient_id (int) [s] FK parties`; `quantity (int) [s]`; `ship_date (date)`; `purpose (text)`; `tracking_code (text)`.
- Vincoli: PK; CHECK `quantity>0`; FK `event_id`,`edition_id`,`recipient_id`.
- Indici: `ship_date`, `edition_id`.

## D) SQL DDL (PostgreSQL)
```sql
-- Estensioni opzionali
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- Ruoli
CREATE TABLE roles (
  code text PRIMARY KEY,
  description text
);

CREATE TABLE parties (
  party_id SERIAL PRIMARY KEY,
  party_type text NOT NULL CHECK (party_type IN ('person','organization')),
  display_name text NOT NULL,
  legal_name text,
  tax_id text UNIQUE,
  notes text,
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE party_contacts (
  contact_id SERIAL PRIMARY KEY,
  party_id int NOT NULL REFERENCES parties(party_id) ON DELETE CASCADE,
  contact_type text NOT NULL CHECK (contact_type IN ('email','phone','address','website')),
  label text,
  value text NOT NULL,
  is_primary boolean NOT NULL DEFAULT false,
  UNIQUE (party_id, contact_type, value)
);

CREATE TABLE party_roles (
  party_id int NOT NULL REFERENCES parties(party_id) ON DELETE CASCADE,
  role_code text NOT NULL REFERENCES roles(code),
  PRIMARY KEY (party_id, role_code)
);

CREATE TABLE users (
  user_id SERIAL PRIMARY KEY,
  party_id int NOT NULL REFERENCES parties(party_id),
  email text NOT NULL UNIQUE,
  password_hash text,
  is_active boolean NOT NULL DEFAULT true,
  created_at timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE imprints (
  imprint_id SERIAL PRIMARY KEY,
  name text NOT NULL UNIQUE,
  description text
);

CREATE TABLE series (
  series_id SERIAL PRIMARY KEY,
  imprint_id int REFERENCES imprints(imprint_id),
  name text NOT NULL,
  description text,
  UNIQUE (imprint_id, name)
);

CREATE TABLE works (
  work_id SERIAL PRIMARY KEY,
  series_id int REFERENCES series(series_id),
  title text NOT NULL,
  subtitle text,
  synopsis text,
  genre text,
  language_original text,
  status text NOT NULL CHECK (status IN ('development','in_review','in_contract','in_production','published','out_of_print')),
  planned_pub_date date,
  publication_date date,
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE work_keywords (
  work_id int NOT NULL REFERENCES works(work_id) ON DELETE CASCADE,
  keyword text NOT NULL,
  PRIMARY KEY (work_id, keyword)
);

CREATE TABLE editions (
  edition_id SERIAL PRIMARY KEY,
  work_id int NOT NULL REFERENCES works(work_id) ON DELETE CASCADE,
  imprint_id int REFERENCES imprints(imprint_id),
  format text NOT NULL CHECK (format IN ('print','ebook','audiobook')),
  isbn13 char(13) UNIQUE,
  ean text,
  internal_code text UNIQUE,
  status text NOT NULL CHECK (status IN ('announced','in_printing','published','out_of_print')),
  publication_date date,
  list_price numeric(10,2),
  currency char(3) NOT NULL DEFAULT 'EUR',
  page_count int,
  file_format text,
  dimensions text,
  stock_min int,
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE work_contributors (
  contrib_id SERIAL PRIMARY KEY,
  work_id int NOT NULL REFERENCES works(work_id) ON DELETE CASCADE,
  party_id int NOT NULL REFERENCES parties(party_id),
  role_code text NOT NULL REFERENCES roles(code),
  royalty_share numeric(5,2) CHECK (royalty_share >= 0 AND royalty_share <= 100),
  UNIQUE (work_id, party_id, role_code)
);

CREATE TABLE edition_contributors (
  edition_id int NOT NULL REFERENCES editions(edition_id) ON DELETE CASCADE,
  party_id int NOT NULL REFERENCES parties(party_id),
  role_code text NOT NULL REFERENCES roles(code),
  royalty_share numeric(5,2) CHECK (royalty_share >= 0 AND royalty_share <= 100),
  PRIMARY KEY (edition_id, party_id, role_code)
);

CREATE TABLE manuscripts (
  manuscript_id SERIAL PRIMARY KEY,
  work_id int REFERENCES works(work_id),
  title text NOT NULL,
  source text,
  status text NOT NULL CHECK (status IN ('submitted','in_review','rejected','optioned','contracted')),
  submission_date date,
  decision_date date,
  notes text,
  file_path text
);

CREATE TABLE workflow_tasks (
  task_id SERIAL PRIMARY KEY,
  work_id int REFERENCES works(work_id),
  edition_id int REFERENCES editions(edition_id),
  manuscript_id int REFERENCES manuscripts(manuscript_id),
  task_type text NOT NULL CHECK (task_type IN ('reading','evaluation','editing','copyediting','proof','layout','cover','print_order','marketing','publication')),
  status text NOT NULL CHECK (status IN ('todo','in_progress','blocked','done')),
  assigned_to int REFERENCES users(user_id),
  due_date date,
  completed_at timestamptz,
  notes text,
  created_at timestamptz NOT NULL DEFAULT now(),
  CHECK (work_id IS NOT NULL OR edition_id IS NOT NULL OR manuscript_id IS NOT NULL)
);

CREATE TABLE rights_contracts (
  contract_id SERIAL PRIMARY KEY,
  work_id int NOT NULL REFERENCES works(work_id) ON DELETE CASCADE,
  party_id int NOT NULL REFERENCES parties(party_id),
  role_code text NOT NULL REFERENCES roles(code),
  start_date date NOT NULL,
  end_date date,
  territory text,
  advance_amount numeric(12,2) DEFAULT 0,
  reporting_frequency text CHECK (reporting_frequency IN ('annual','semiannual','quarterly')),
  payment_terms text,
  status text DEFAULT 'active' CHECK (status IN ('active','expired','terminated')),
  created_at timestamptz NOT NULL DEFAULT now(),
  UNIQUE (work_id, party_id, role_code, start_date)
);

CREATE TABLE royalty_terms (
  term_id SERIAL PRIMARY KEY,
  contract_id int NOT NULL REFERENCES rights_contracts(contract_id) ON DELETE CASCADE,
  format text DEFAULT 'all' CHECK (format IN ('print','ebook','audiobook','all')),
  channel text,
  rate numeric(5,2) NOT NULL CHECK (rate >= 0 AND rate <= 100),
  threshold_units int,
  notes text,
  UNIQUE (contract_id, format, channel, threshold_units)
);

CREATE TABLE secondary_rights (
  secondary_id SERIAL PRIMARY KEY,
  contract_id int NOT NULL REFERENCES rights_contracts(contract_id) ON DELETE CASCADE,
  right_type text,
  licensee text,
  territory text,
  amount numeric(12,2),
  signature_date date,
  notes text
);

CREATE TABLE production_orders (
  order_id SERIAL PRIMARY KEY,
  edition_id int NOT NULL REFERENCES editions(edition_id) ON DELETE CASCADE,
  supplier_id int NOT NULL REFERENCES parties(party_id),
  specs text,
  quantity int NOT NULL,
  unit_cost numeric(10,2),
  setup_cost numeric(10,2),
  due_date date,
  status text CHECK (status IN ('requested','approved','in_progress','delivered','cancelled')),
  created_at timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE print_runs (
  print_run_id SERIAL PRIMARY KEY,
  edition_id int NOT NULL REFERENCES editions(edition_id) ON DELETE CASCADE,
  production_order_id int UNIQUE REFERENCES production_orders(order_id),
  batch_number text,
  quantity int NOT NULL,
  total_cost numeric(12,2),
  received_date date
);

CREATE TABLE warehouses (
  warehouse_id SERIAL PRIMARY KEY,
  name text NOT NULL UNIQUE,
  location text,
  is_default boolean NOT NULL DEFAULT false
);

CREATE TABLE inventory_movements (
  movement_id SERIAL PRIMARY KEY,
  edition_id int NOT NULL REFERENCES editions(edition_id) ON DELETE CASCADE,
  warehouse_id int NOT NULL REFERENCES warehouses(warehouse_id),
  movement_type text NOT NULL CHECK (movement_type IN ('in','out','return_in','promo_out','damaged','adjustment')),
  quantity int NOT NULL CHECK (quantity > 0),
  unit_cost numeric(10,2),
  document_type text,
  document_id int,
  reason text,
  movement_date date NOT NULL,
  created_at timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE sales_channels (
  channel_code text PRIMARY KEY,
  description text
);

CREATE TABLE sales_orders (
  order_id SERIAL PRIMARY KEY,
  customer_id int NOT NULL REFERENCES parties(party_id),
  channel_code text REFERENCES sales_channels(channel_code),
  order_date date NOT NULL,
  status text CHECK (status IN ('draft','confirmed','shipped','invoiced','cancelled')),
  currency char(3) NOT NULL DEFAULT 'EUR',
  notes text
);

CREATE TABLE sales_order_lines (
  order_line_id SERIAL PRIMARY KEY,
  order_id int NOT NULL REFERENCES sales_orders(order_id) ON DELETE CASCADE,
  edition_id int NOT NULL REFERENCES editions(edition_id),
  quantity int NOT NULL CHECK (quantity > 0),
  unit_price numeric(10,2),
  discount_pct numeric(5,2) DEFAULT 0
);

CREATE TABLE sales_invoices (
  invoice_id SERIAL PRIMARY KEY,
  order_id int REFERENCES sales_orders(order_id),
  customer_id int NOT NULL REFERENCES parties(party_id),
  document_type text NOT NULL CHECK (document_type IN ('invoice','credit_note')),
  invoice_date date NOT NULL,
  total_amount numeric(12,2),
  currency char(3) NOT NULL DEFAULT 'EUR',
  notes text
);

CREATE TABLE sales_invoice_lines (
  invoice_line_id SERIAL PRIMARY KEY,
  invoice_id int NOT NULL REFERENCES sales_invoices(invoice_id) ON DELETE CASCADE,
  edition_id int NOT NULL REFERENCES editions(edition_id),
  quantity int NOT NULL CHECK (quantity <> 0),
  net_unit_price numeric(10,2),
  discount_pct numeric(5,2) DEFAULT 0,
  tax_rate numeric(5,2) DEFAULT 0
);

CREATE TABLE sales_returns (
  return_id SERIAL PRIMARY KEY,
  invoice_line_id int NOT NULL REFERENCES sales_invoice_lines(invoice_line_id) ON DELETE CASCADE,
  quantity int NOT NULL CHECK (quantity > 0),
  return_date date NOT NULL,
  reason text
);

CREATE TABLE royalty_periods (
  period_id SERIAL PRIMARY KEY,
  name text NOT NULL,
  start_date date NOT NULL,
  end_date date NOT NULL,
  status text NOT NULL DEFAULT 'open' CHECK (status IN ('open','closed')),
  UNIQUE (start_date, end_date)
);

CREATE TABLE royalty_statements (
  statement_id SERIAL PRIMARY KEY,
  period_id int NOT NULL REFERENCES royalty_periods(period_id),
  contract_id int NOT NULL REFERENCES rights_contracts(contract_id),
  generated_at timestamptz NOT NULL DEFAULT now(),
  amount_due numeric(12,2),
  advance_applied numeric(12,2) DEFAULT 0,
  notes text,
  UNIQUE (period_id, contract_id)
);

CREATE TABLE royalty_statement_lines (
  line_id SERIAL PRIMARY KEY,
  statement_id int NOT NULL REFERENCES royalty_statements(statement_id) ON DELETE CASCADE,
  edition_id int NOT NULL REFERENCES editions(edition_id),
  channel_code text,
  units_sold int NOT NULL CHECK (units_sold >= 0),
  units_returned int DEFAULT 0,
  net_units int GENERATED ALWAYS AS (units_sold - COALESCE(units_returned,0)) STORED,
  net_sales_amount numeric(12,2),
  royalty_rate numeric(5,2) CHECK (royalty_rate >= 0 AND royalty_rate <= 100),
  royalty_amount numeric(12,2)
);

CREATE TABLE marketing_events (
  event_id SERIAL PRIMARY KEY,
  name text NOT NULL,
  event_date date,
  location text,
  notes text
);

CREATE TABLE press_shipments (
  shipment_id SERIAL PRIMARY KEY,
  event_id int REFERENCES marketing_events(event_id),
  edition_id int NOT NULL REFERENCES editions(edition_id),
  recipient_id int NOT NULL REFERENCES parties(party_id),
  quantity int NOT NULL CHECK (quantity > 0),
  ship_date date,
  purpose text,
  tracking_code text
);
```

## E) Dati di esempio (INSERT)
```sql
-- Ruoli base
INSERT INTO roles(code, description) VALUES
 ('author','Autore'),('translator','Traduttore'),('editor','Editor'),('illustrator','Illustratore'),
 ('supplier','Fornitore'),('customer','Cliente B2B'),('staff','Staff'),('agent','Agente');

-- Parties
INSERT INTO parties(party_type, display_name, legal_name) VALUES
 ('person','Luca Bianchi',NULL), --1 autore
 ('person','Maria Rossi',NULL), --2 autore
 ('person','Giulia Verdi',NULL), --3 traduttore
 ('organization','Tipografia Alfa','Tipografia Alfa Srl'), --4 fornitore
 ('organization','Distribuzioni Centro','Distribuzioni Centro Srl'), --5 cliente/distributore
 ('organization','Libreria Aurora','Libreria Aurora Snc'); --6 cliente/libreria

-- Ruoli
INSERT INTO party_roles VALUES
 (1,'author'),(2,'author'),(3,'translator'),(4,'supplier'),(5,'customer'),(6,'customer');

-- Canali
INSERT INTO sales_channels(channel_code, description) VALUES
 ('distributor','Distributore'),('bookstore','Libreria'),('direct','E-commerce diretto'),('event','Fiera/evento');

-- Utente interno (per assegnazione task)
INSERT INTO users(party_id,email,password_hash) VALUES (1,'luca@editore.it','hash');

-- Imprint e collana
INSERT INTO imprints(name, description) VALUES ('Scriptorium','Narrativa e saggistica');
INSERT INTO series(imprint_id,name,description) VALUES (1,'Aurora','Collana narrativa');

-- Works
INSERT INTO works(series_id,title,subtitle,synopsis,genre,language_original,status,planned_pub_date,publication_date)
VALUES
 (1,'Il Viaggio Segreto','Romanzo di formazione','Un giovane scopre un mondo nascosto','Narrativa','it','published','2024-03-01','2024-04-15'),
 (1,'Mappe del Tempo',NULL,'Saggi brevi su storia e scienza','Saggistica','en','in_production','2024-09-01',NULL);

-- Keywords
INSERT INTO work_keywords VALUES (1,'avventura'),(1,'crescita'),(2,'storia'),(2,'scienza');

-- Editions (3: due per opera1, una per opera2)
INSERT INTO editions(work_id,imprint_id,format,isbn13,internal_code,status,publication_date,list_price,page_count)
VALUES
 (1,1,'print','9781234567890','VIA-PR','published','2024-04-15',18.00,320),
 (1,1,'ebook','9781234567891','VIA-EB','published','2024-04-15',9.99,320),
 (2,1,'print','9781234567892','MAP-PR','announced',NULL,22.00,280);

-- Contributori base
INSERT INTO work_contributors(work_id,party_id,role_code,royalty_share) VALUES
 (1,1,'author',60.00), (1,2,'author',40.00), (1,3,'translator',10.00),
 (2,2,'author',100.00);

-- Manoscritto per opera 2
INSERT INTO manuscripts(work_id,title,source,status,submission_date) VALUES
 (2,'Mappe del Tempo','invio spontaneo','in_review','2024-05-10');

-- Task pipeline
INSERT INTO workflow_tasks(work_id,task_type,status,assigned_to,due_date,notes) VALUES
 (2,'reading','in_progress',1,'2024-06-01','Prima lettura'),
 (2,'editing','todo',1,'2024-07-01','Da assegnare editor');

-- Contratti
INSERT INTO rights_contracts(work_id,party_id,role_code,start_date,end_date,territory,advance_amount,reporting_frequency,payment_terms)
VALUES
 (1,1,'author','2023-10-01',NULL,'mondo',2000,'semiannual','bonifico 30gg'),
 (1,2,'author','2023-10-01',NULL,'mondo',1500,'semiannual','bonifico 30gg'),
 (1,3,'translator','2023-11-01',NULL,'mondo',500,'annual','bonifico 30gg');

INSERT INTO royalty_terms(contract_id,format,channel,rate) VALUES
 (1,'print','bookstore',8.00),(1,'ebook','direct',25.00),
 (2,'print','bookstore',5.00),(2,'ebook','direct',15.00),
 (3,'print','bookstore',2.00),(3,'ebook','direct',10.00);

-- Produzione e stampa
INSERT INTO production_orders(edition_id,supplier_id,specs,quantity,unit_cost,setup_cost,due_date,status)
VALUES (1,4,'brossura, 14x21, carta 90gr',1500,3.00,500.00,'2024-03-20','delivered');

INSERT INTO print_runs(edition_id,production_order_id,batch_number,quantity,total_cost,received_date)
VALUES (1,1,'PR23-001',1500,5000.00,'2024-03-25');

-- Magazzino e movimenti
INSERT INTO warehouses(name,location,is_default) VALUES ('Principale','Via Roma 1, Milano',true);
INSERT INTO inventory_movements(edition_id,warehouse_id,movement_type,quantity,unit_cost,document_type,document_id,reason,movement_date)
VALUES
 (1,1,'in',1500,3.33,'print_run',1,'Ingresso lotto PR23-001','2024-03-25'),
 (1,1,'out',200,3.33,'invoice',1,'Vendita a distributore','2024-04-16'),
 (1,1,'out',50,3.33,'invoice',2,'Vendita a libreria','2024-04-18'),
 (1,1,'return_in',20,3.33,'return',1,'Reso libreria','2024-05-10');

-- Ordini e vendite
INSERT INTO sales_orders(customer_id,channel_code,order_date,status,notes) VALUES
 (5,'distributor','2024-04-10','invoiced','Prima fornitura'),
 (6,'bookstore','2024-04-12','invoiced','Ordine lancio');

INSERT INTO sales_order_lines(order_id,edition_id,quantity,unit_price,discount_pct) VALUES
 (1,1,200,12.00,50.00),
 (2,1,50,12.00,45.00);

INSERT INTO sales_invoices(order_id,customer_id,document_type,invoice_date,total_amount) VALUES
 (1,5,'invoice','2024-04-15',1200.00),
 (2,6,'invoice','2024-04-17',330.00);

INSERT INTO sales_invoice_lines(invoice_id,edition_id,quantity,net_unit_price,discount_pct,tax_rate) VALUES
 (1,1,200,6.00,0,4.00),
 (2,1,50,6.60,0,4.00);

-- Reso
INSERT INTO sales_returns(invoice_line_id,quantity,return_date,reason) VALUES
 (2,20,'2024-05-05','Reso lancio lento');

-- Periodo royalty e statement
INSERT INTO royalty_periods(name,start_date,end_date,status) VALUES
 ('2024-H1','2024-01-01','2024-06-30','open');

INSERT INTO royalty_statements(period_id,contract_id,amount_due,advance_applied,notes)
VALUES (1,1,300.00,200.00,'Calcolo H1 autore 1');

INSERT INTO royalty_statement_lines(statement_id,edition_id,channel_code,units_sold,units_returned,net_sales_amount,royalty_rate,royalty_amount)
VALUES
 (1,1,'bookstore',250,20,1500.00,8.00,120.00),
 (1,2,'direct',80,0,640.00,25.00,160.00);
```

## F) Query utili (12 esempi)
1. **Catalogo titoli pubblicati per collana e data**
```sql
SELECT s.name AS collana, w.title, e.format, e.isbn13, e.publication_date
FROM editions e
JOIN works w ON e.work_id = w.work_id
LEFT JOIN series s ON w.series_id = s.series_id
WHERE e.status = 'published'
ORDER BY s.name, e.publication_date;
```
2. **Scheda titolo completa (opera + edizioni + contributori)**
```sql
SELECT w.title, w.subtitle, w.synopsis, e.edition_id, e.format, e.isbn13, e.list_price,
       c.display_name, wc.role_code, wc.royalty_share
FROM works w
JOIN editions e ON e.work_id = w.work_id
LEFT JOIN work_contributors wc ON wc.work_id = w.work_id
LEFT JOIN parties c ON c.party_id = wc.party_id
WHERE w.work_id = $1
ORDER BY e.format, wc.role_code;
```
3. **Giacenza attuale e valore (costo medio)**
```sql
WITH movements AS (
  SELECT edition_id,
         SUM(CASE WHEN movement_type IN ('in','return_in') THEN quantity ELSE -quantity END) AS qty,
         SUM(CASE WHEN movement_type IN ('in','return_in') THEN quantity * COALESCE(unit_cost,0) ELSE 0 END) AS cost
  FROM inventory_movements
  GROUP BY edition_id
)
SELECT e.edition_id, e.isbn13, e.format,
       m.qty AS stock_qty,
       CASE WHEN m.qty > 0 THEN m.cost / NULLIF(m.qty,0) ELSE NULL END AS avg_cost
FROM editions e
JOIN movements m ON m.edition_id = e.edition_id;
```
4. **Movimento magazzino per periodo**
```sql
SELECT e.isbn13, e.format, im.movement_type, im.quantity, im.movement_date, im.document_type, im.document_id
FROM inventory_movements im
JOIN editions e ON im.edition_id = e.edition_id
WHERE im.movement_date BETWEEN $1 AND $2
ORDER BY im.movement_date;
```
5. **Vendite per canale e mese**
```sql
SELECT DATE_TRUNC('month', si.invoice_date) AS month, sc.channel_code,
       SUM(sil.quantity) AS units,
       SUM(sil.quantity * sil.net_unit_price) AS net_revenue
FROM sales_invoices si
JOIN sales_invoice_lines sil ON sil.invoice_id = si.invoice_id
LEFT JOIN sales_orders so ON so.order_id = si.order_id
LEFT JOIN sales_channels sc ON sc.channel_code = so.channel_code
WHERE si.document_type = 'invoice'
GROUP BY month, sc.channel_code
ORDER BY month;
```
6. **Top 10 titoli per copie vendute (netto resi)**
```sql
WITH sold AS (
  SELECT sil.edition_id,
         SUM(sil.quantity) AS sold_qty
  FROM sales_invoice_lines sil
  JOIN sales_invoices si ON si.invoice_id = sil.invoice_id AND si.document_type='invoice'
  GROUP BY sil.edition_id
), returned AS (
  SELECT sil.edition_id, SUM(sr.quantity) AS returned_qty
  FROM sales_returns sr
  JOIN sales_invoice_lines sil ON sil.invoice_line_id = sr.invoice_line_id
  GROUP BY sil.edition_id
)
SELECT e.isbn13, e.format, COALESCE(s.sold_qty,0) - COALESCE(r.returned_qty,0) AS net_units
FROM editions e
LEFT JOIN sold s ON s.edition_id = e.edition_id
LEFT JOIN returned r ON r.edition_id = e.edition_id
ORDER BY net_units DESC
LIMIT 10;
```
7. **Resi per cliente e titolo**
```sql
SELECT p.display_name AS customer, e.isbn13, SUM(sr.quantity) AS returned_qty
FROM sales_returns sr
JOIN sales_invoice_lines sil ON sil.invoice_line_id = sr.invoice_line_id
JOIN sales_invoices si ON si.invoice_id = sil.invoice_id
JOIN parties p ON p.party_id = si.customer_id
JOIN editions e ON e.edition_id = sil.edition_id
GROUP BY p.display_name, e.isbn13
ORDER BY returned_qty DESC;
```
8. **Calcolo royalties per autore e periodo (netto resi, esclusi omaggi)**
```sql
WITH sales_net AS (
  SELECT sil.edition_id, so.channel_code,
         SUM(sil.quantity) AS units_sold
  FROM sales_invoice_lines sil
  JOIN sales_invoices si ON si.invoice_id = sil.invoice_id AND si.document_type='invoice'
  LEFT JOIN sales_orders so ON so.order_id = si.order_id
  WHERE si.invoice_date BETWEEN $1 AND $2
  GROUP BY sil.edition_id, so.channel_code
), returns AS (
  SELECT sil.edition_id, SUM(sr.quantity) AS units_returned
  FROM sales_returns sr
  JOIN sales_invoice_lines sil ON sil.invoice_line_id = sr.invoice_line_id
  WHERE sr.return_date BETWEEN $1 AND $2
  GROUP BY sil.edition_id
)
SELECT p.display_name AS contributor,
       w.title,
       rt.rate,
       (sn.units_sold - COALESCE(r.units_returned,0)) AS net_units,
       ((sn.units_sold - COALESCE(r.units_returned,0)) * e.list_price * rt.rate/100) AS royalty_estimate
FROM sales_net sn
LEFT JOIN returns r ON r.edition_id = sn.edition_id
JOIN editions e ON e.edition_id = sn.edition_id
JOIN works w ON w.work_id = e.work_id
JOIN rights_contracts rc ON rc.work_id = w.work_id
JOIN parties p ON p.party_id = rc.party_id
LEFT JOIN royalty_terms rt ON rt.contract_id = rc.contract_id AND (rt.channel = sn.channel_code OR rt.channel IS NULL)
WHERE rc.role_code IN ('author','translator');
```
9. **Stato pipeline editoriale (titoli in lavorazione per milestone)**
```sql
SELECT w.title, wt.task_type, wt.status, wt.due_date, u.email AS assigned_to
FROM workflow_tasks wt
JOIN works w ON w.work_id = wt.work_id
LEFT JOIN users u ON u.user_id = wt.assigned_to
WHERE w.status IN ('development','in_review','in_contract','in_production')
ORDER BY wt.due_date;
```
10. **Scadenze task prossimi 14 giorni**
```sql
SELECT wt.task_id, w.title, wt.task_type, wt.due_date, wt.status, u.email
FROM workflow_tasks wt
JOIN works w ON w.work_id = wt.work_id
LEFT JOIN users u ON u.user_id = wt.assigned_to
WHERE wt.due_date BETWEEN CURRENT_DATE AND CURRENT_DATE + INTERVAL '14 days'
ORDER BY wt.due_date;
```
11. **Titoli sotto scorta minima**
```sql
WITH stock AS (
  SELECT edition_id,
         SUM(CASE WHEN movement_type IN ('in','return_in') THEN quantity ELSE -quantity END) AS qty
  FROM inventory_movements
  GROUP BY edition_id
)
SELECT e.isbn13, e.format, e.stock_min, s.qty AS current_qty
FROM editions e
JOIN stock s ON s.edition_id = e.edition_id
WHERE e.stock_min IS NOT NULL AND s.qty < e.stock_min;
```
12. **Margine stimato per titolo**
```sql
WITH sales AS (
  SELECT sil.edition_id,
         SUM(sil.quantity * sil.net_unit_price) AS revenue
  FROM sales_invoice_lines sil
  JOIN sales_invoices si ON si.invoice_id = sil.invoice_id AND si.document_type='invoice'
  GROUP BY sil.edition_id
), returns AS (
  SELECT sil.edition_id, SUM(sr.quantity * sil.net_unit_price) AS return_value
  FROM sales_returns sr
  JOIN sales_invoice_lines sil ON sil.invoice_line_id = sr.invoice_line_id
  GROUP BY sil.edition_id
), cost AS (
  SELECT edition_id,
         SUM(CASE WHEN movement_type IN ('in','return_in') THEN quantity * COALESCE(unit_cost,0) ELSE 0 END) AS total_cost,
         SUM(CASE WHEN movement_type IN ('in','return_in') THEN quantity ELSE 0 END) AS units_in
  FROM inventory_movements
  GROUP BY edition_id
), royalty_est AS (
  SELECT rc.work_id, SUM(rt.rate) AS blended_rate
  FROM rights_contracts rc
  LEFT JOIN royalty_terms rt ON rt.contract_id = rc.contract_id
  WHERE rc.status='active'
  GROUP BY rc.work_id
)
SELECT w.title, e.isbn13,
       (COALESCE(s.revenue,0) - COALESCE(r.return_value,0)) AS net_revenue,
       CASE WHEN c.units_in>0 THEN c.total_cost / c.units_in ELSE 0 END AS avg_cost,
       (COALESCE(s.revenue,0) - COALESCE(r.return_value,0)) - (CASE WHEN c.units_in>0 THEN c.total_cost / c.units_in ELSE 0 END * COALESCE((s.revenue/ NULLIF(e.list_price,0)),0))
         - COALESCE(re.blended_rate,0)/100 * COALESCE(s.revenue,0) AS estimated_margin
FROM editions e
JOIN works w ON w.work_id = e.work_id
LEFT JOIN sales s ON s.edition_id = e.edition_id
LEFT JOIN returns r ON r.edition_id = e.edition_id
LEFT JOIN cost c ON c.edition_id = e.edition_id
LEFT JOIN royalty_est re ON re.work_id = w.work_id;
```

## G) Scelte e trade-off
- **Normalizzazione**: contatti e ruoli separati per evitare campi multivalore; contributori gestiti con tabelle ponte; parole chiave normalizzate in `work_keywords`. Le edizioni sono separate dall’opera per supportare più formati/ISBN. Movimenti di magazzino separano causale e documento.
- **Denormalizzazione minima**: salviamo `list_price` e `currency` su `editions` (ridondante rispetto a eventuale storico prezzi) per semplicità. `net_units` in `royalty_statement_lines` è un campo generato per velocizzare reportistica.
- **Royalties**: modello basato su contratti + termini per formato/canale con soglie; eventuali regole più complesse (scaglioni multipli) richiederebbero una tabella aggiuntiva.
- **Pipeline**: `workflow_tasks` aggancia sia opera che edizione o manoscritto per flessibilità; nessun blob per allegati (solo path).
- **Magazzino**: movimenti positivi/negativi distinti da `movement_type` per semplificare controlli; costo medio calcolato da movimenti in ingresso.
- **Privacy/GDPR**: dati minimi sui contatti, note libere tenute in campi separati; cancellazione/anomizzazione possibile a livello `parties` lasciando riferimenti anonimi.
- **Indici**: proposti su colonne di ricerca frequente (ISBN, titolo, stato, date, canale) per performance su volumi piccoli/medi.
