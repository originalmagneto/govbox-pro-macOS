# Špecifikácia pluginu GovBox pre LAWOSS

Verzia návrhu 0.1, 7. 10. 2026. Nadväzuje na [README.md](README.md) (analýza a rozhodnutia). Po schválení presunúť do `lawoss-marketplace/plugins/govbox/` a príslušnej špecifikácie v koordinačnom repozitári LAWOSS.

Značky v texte:
- **[overené]** – overené v kóde GovBox Pro (tento repozitár) alebo LAWOSS (`~/PROJECTS/MikeOSS-SLOVAKIA-AI/LAWOSS`, vetva `dev`, commit `7a26d9f5`).
- **[návrh]** – nové rozhodnutie tejto špecifikácie.
- **[otvorené]** – treba rozhodnúť (zoznam v časti 14).

---

## 1. Cieľ a rozsah

**Cieľ.** Advokát pripojí do LAWOSS vlastný prístup ku GovBox Pro, vyberie schránky na synchronizáciu a správy z nich sa automaticky zaradia do správnych spisov v štruktúre OKF. Čo sa zaradiť nedá, skončí v jednej triediacej schránke, nie v náhodnom priečinku.

**V rozsahu MVP**
- Pripojenie cez GovBox Pro API (situácie 1 a 2 z README, časť 2).
- Výber schránok.
- Ručne spustená synchronizácia.
- Pravidlá mapovania na spisy.
- Zápis do OKF (`00_Na_zatriedenie`, `VSTUPY.md`, `KOMUNIKACNE-KANALY.md`).
- Prehľad čakajúcich doručeniek.

**Mimo MVP (ďalšie fázy, časť 13)**
- Prevzatie doručenky.
- Návrh a odoslanie podania.
- Pravidelná synchronizácia na pozadí.
- Schránky Finančnej správy.
- Adaptér slovensko-sk-api.
- Natívna karta v UI LAWOSS.
- Windows.

**Nie je cieľom**
- Nahradiť webové rozhranie GovBox Pro: podpisovanie cez Autogram, správa používateľov a automatizácie zostávajú tam.
- Ukladať obsah schránok mimo počítača advokáta.

## 2. Obmedzenia z LAWOSS, ktoré návrh určujú

| Obmedzenie | Zdroj | Dôsledok pre plugin |
|---|---|---|
| Plugin = balík z LAWOSS Marketplace: skilly + lokálny stdio MCP server + CLI. Neexistuje API pre UI panely, plánovače ani schému nastavení. | ADR 0015 bod 2; `apps/server/src/lawoss/marketplace-global.ts` **[overené]** | Všetko ovládanie ide cez MCP nástroje, skill a CLI. Konfigurácia je súbor, nie formulár v aplikácii. |
| Okrem týždennej kontroly marketplace nesmie byť v LAWOSS žiadne plánované ani automatické sieťové volanie. CI to stráži. | `lawoss/scripts/check-no-eigenwelt.mjs:27-31` (pravidlo 9); ADR 0015 bod 6 **[overené]** | Synchronizácia sa spúšťa **iba na pokyn človeka**. Pravidelný beh si vyžaduje nové ADR. |
| LAWOSS nemá Keychain ani Electron `safeStorage`. Tajomstvá dnes ležia v `~/.config/legalwork/env.json` (0600, nešifrované). | `apps/server/src/env-file.ts` **[overené]** | Plugin si súkromný kľúč spravuje sám v macOS Keychain (časť 4). |
| Potvrdenie rizikového kroku nie je pole manifestu. Deklaruje sa v katalógu (`capabilities`, `humanGate`) a v texte skillu. Vynútiť ho musí plugin. | `apps/app/src/lawoss/domains/marketplace/catalog.ts:15-17`; vzor `plugins/cz-agents/scripts/runtime-policy.mjs` **[overené]** | Pre právne úkony vlastná potvrdzovacia brána mimo agenta (časť 8). |
| Prichádzajúce písomnosti majú v OKF register `VSTUPY.md` (`IN-NNN`, `pending` → `processed`) a priečinok podľa roly `inbox` (`00_Na_zatriedenie`). | `lawoss/okf/inputs.ts`; `apps/app/src/lawoss/lite/matter-intake.ts`; `lawoss/okf/src/profile.ts:46-49` **[overené]** | Plugin zapisuje presne do týchto miest a v tomto formáte, aby ďalej fungovalo existujúce triedenie (`roztried-spis`). |
| Kanály komunikácie majú register `KOMUNIKACNE-KANALY.md` so stavom a kurzorom. Pravidlá šablóny: zdieľaný kanál má **jednu** evidenciu, kurzor sa nekopíruje do viacerých evidencií. Komunikácia bez určenej veci zostáva **u klienta** ako `pending`. Kurzor úplnej kontroly sa posúva až po úspešnom prechode všetkých stránok. | `lawoss/okf/templates/spis/KOMUNIKACNE-KANALY.md` **[overené]** | Jeden register na schránku, nie na spis (časť 7.4). Nepriradené správy so známym klientom idú ku klientovi. |
| Nedostupný obsah alebo príloha sa eviduje vo `VSTUPY.md` ako `pending` s vysvetlením. Každá správa vo vlákne má vlastné `IN-NNN`. Duplicity sa určujú podľa kanála, účtu a stabilného ID správy. | `lawoss/okf/templates/spis/VSTUPY.md` **[overené]** | Čakajúca doručenka ide do `VSTUPY.md` ako `pending` s lehotou na prevzatie (časť 8.1). Stĺpec Zdroj nesie ID správy GovBox. |
| Hľadanie spisu podľa spisovej značky už existuje (`findCaseNumber()`, normalizácia na kľúč typu `8C-123-2023`). Hľadanie podľa IČO neexistuje. | `lawoss/okf/src/triage/rules.ts:76`, `triage/plan.ts:85-86` **[overené]** | Mapovanie podľa spisovej značky použije rovnakú normalizáciu. Pravidlo podľa IČO je vlastné pravidlo pluginu. |

## 3. Architektúra

```
┌──────────────── Mac advokáta ────────────────────────────────────────────┐
│                                                                           │
│  LAWOSS (Electron) ── agent (OpenCode) ── skill „govbox-schranka“         │
│                              │                                            │
│                              │ MCP (stdio)                                │
│                              ▼                                            │
│                    MCP server „govbox“ ──── macOS Keychain (súkromný kľúč)│
│                    │        │                                             │
│      stav (SQLite) ┘        └──► OKF priečinok praxe                      │
│  LAWOSS_STATE_DIR/govbox        Office/govbox.yaml, Office/Schranky/…     │
│                                 Klienti/…/Spisy/…/00_Na_zatriedenie, …    │
└──────────────────────────────┬────────────────────────────────────────────┘
                               │ HTTPS + JWT (RS256, sub = tenant)
                               ▼
                 GovBox Pro (pro.govbox.sk alebo vlastná inštancia)
                               │
                               ▼
              GovBox API / vlastný slovensko-sk-api ──► ÚPVS (slovensko.sk)
```

### 3.1 Balík v marketplace **[návrh]**

Vzor je `lawoss-marketplace/plugins/crz` (rozloženie) a `plugins/cz-agents` (stav v SQLite, poskytovateľ s prihlasovacími údajmi, politika volaní).

```
lawoss-marketplace/plugins/govbox/
  .claude-plugin/plugin.json        name: govbox, license: MIT, skills: ./skills/
  .codex-plugin/plugin.json         interface.displayName: "GovBox Pro – schránky slovensko.sk"
                                    capabilities: ["Read", "Write"]
  .mcp.json                         { "mcpServers": { "govbox": { "command": "node", "args": ["scripts/run.mjs", "mcp"] } } }
  runtime-config.json               { "name": "govbox", "entrypoint": "dist/index.js",
                                      "minimumNode": "22.14.0", "nativeBuilds": ["better-sqlite3"] }
  runtime/provenance.json           hash každého súboru (generuje build)
  scripts/run.mjs                   spoločný spúšťač (nemeniť)
  skills/govbox-schranka/SKILL.md
  src/                              TypeScript → dist/
    adapters/govbox-pro.ts          adaptér GovBox Pro API (MVP)
    adapters/types.ts               rozhranie adaptéra (časť 3.2)
    auth/jwt.ts, auth/keychain.ts
    sync/, mapping/, okf-writer/, gate/
  docs/SETUP.md
```

Položka v katalógu `lawoss-catalog.json`:
- Jurisdikcia: SK.
- `capabilities: ["network", "local-write", "external-action"]`. Hodnota `external-action` platí až od fázy 2.
- `humanGate: true`.
- `policy.authentication: "ON_INSTALL"`.

### 3.2 Rozhranie adaptéra **[návrh]**

Aby sa neskôr dal doplniť adaptér slovensko-sk-api (README, situácie 3 a 4), zvyšok pluginu nepozná GovBox Pro priamo, iba toto rozhranie:

```ts
interface MailboxAdapter {
  listBoxes(): Promise<Box[]>;                         // id, uri, name, type, active
  fetchNewMessages(cursor: Cursor): Promise<{ messages: MessageMeta[]; next: Cursor; done: boolean }>;
  getMessage(id: string): Promise<MessageFull>;        // vrátane objektov (originály)
  getObjectPdf(objectId: string): Promise<Uint8Array>; // PDF vizualizácia
  getThread(id: string): Promise<{ messageIds: string[]; tags: string[] }>;
  // fáza 2+
  authorizeDelivery?(messageId: string): Promise<{ threadId: string }>;
  createDraft?(draft: DraftInput): Promise<{ id: string; threadId: string }>;
  submitDraft?(id: string): Promise<void>;
}
```

`MessageMeta` musí vždy obsahovať `boxId`. V GovBox Pro ho adaptér zatiaľ odvodzuje (časť 6.2).

## 4. Pripojenie a kľúče

### 4.1 Predpoklady u používateľa
1. Platený tenant v GovBox Pro (alebo vlastná inštancia).
2. Prevádzkovateľ zapol tenantovi príznak `:api` **[overené]**, `app/models/tenant.rb:67`. Bez neho v menu nie je „API Prístup“. Podmienky a cenu treba overiť **[otvorené]**.
3. Administrátor tenanta má prístup na stránku „API Prístup“, kde sa vkladá verejný kľúč (`Admin::ApiAccessesController#update`, pole `api_token_public_key`) **[overené]**.

### 4.2 Nastavenie (`govbox setup`) **[návrh]**

Nastavenie sa robí **v termináli cez CLI pluginu, nie cez agenta**. Súkromný kľúč tak nikdy neprejde cez model ani históriu chatu.

1. `node scripts/run.mjs call govbox_setup_begin '{}'` alebo príkaz `govbox setup` sa spýta na:
   - URL inštancie (predvolene `https://pro.govbox.sk`, staging `https://govbox-pro.staging.slovensko.digital`),
   - ID tenanta.
2. Vygeneruje RSA kľúč 3072 bit lokálne.
3. Súkromný kľúč uloží do Keychain: `security add-generic-password -s lawoss.govbox -a <url>#<tenant> -w <PEM>`.
4. Vypíše **verejný kľúč**. Používateľ ho vloží v GovBox Pro → Admin → API Prístup.
5. Overí spojenie volaním `GET /api/boxes` a výsledok zapíše do `Office/govbox.yaml` (časť 5).

**JWT pri každom volaní** **[overené]**, `app/lib/api_token_authenticator.rb`:
- Algoritmus RS256.
- `sub` = ID tenanta.
- `exp` = teraz + 4 minúty (limit servera je 5).
- `jti` = UUID v4 (36 znakov vyhovuje vzoru `[0-9a-z\-_]{32,256}`).
- Token sa necacheuje dlhšie ako 1 minútu.

**Rozsah prístupu.** Kľúč otvára **celého tenanta** (`app/lib/api_environment.rb:16`) **[overené]**. Výber schránok v časti 5 je preto iba filter pluginu, nie obmedzenie na serveri. Plugin na to používateľa upozorní pri nastavení aj v `govbox_doctor`.

**Rotácia a zrušenie.**
- `govbox rotate-key` vygeneruje nový pár. Používateľ vloží nový verejný kľúč a starý v Keychain sa zmaže až po úspešnom overení.
- Zrušenie prístupu: vymazať verejný kľúč v GovBox Pro (prázdne pole = odstránenie, **[overené]**) a spustiť `govbox forget` (zmaže položku v Keychain).

**Windows** – mimo MVP. Neskôr Credential Manager. Žiadna záložná cesta cez nešifrovaný súbor sa v MVP neposkytuje.

## 5. Výber schránok

### 5.1 Postup **[návrh]**
1. `govbox_boxes_list` vráti všetky schránky tenanta z `GET /api/boxes` s poľami `id`, `name`, `short_name`, `uri`, `type`, `active`, `obo`.
2. Agent ich ukáže v tabuľke. Používateľ povie, ktoré synchronizovať a odkedy.
3. `govbox_boxes_select` zapíše výber do `Office/govbox.yaml`.

Pravidlá výberu:
- Schránky typu `Fs::Box` (Finančná správa) sa ukážu, ale **nedajú sa vybrať**, kým upstream API nevracia `box_id` (časť 12). Dôvod je v časti 6.2.
- Neaktívna schránka (`active: false`, napr. vypršalo oprávnenie na zastupovanie) sa dá vybrať, ale `govbox_doctor` hlási varovanie.

### 5.2 Konfiguračný súbor `Office/govbox.yaml` **[návrh]**

Konfigurácia leží v priečinku praxe, nie v cache pluginu. Zálohuje sa spolu so spismi, advokát ju vidí a môže upraviť a agent ju vie prečítať. **Neobsahuje tajomstvá**, iba odkaz na položku v Keychain.

```yaml
version: 1
connection:
  adapter: govbox-pro                 # neskôr: slovensko-sk-api
  base_url: https://pro.govbox.sk
  tenant_id: "123"                    # sub v JWT – nie je tajomstvo
  key_ref: keychain:lawoss.govbox     # odkaz, nie kľúč

boxes:
  - id: 42
    uri: ico://sk/12345678
    name: "Mgr. Ján Novák, advokát"
    type: Upvs::Box
    sync: true
    since: 2026-09-01                 # staršie správy sa preskočia
    mode: auto                        # auto | navrh  (časť 7.3)
  - id: 43
    uri: ico://sk/87654321
    name: "ACME s. r. o."             # schránka klienta (zastupovanie)
    type: Upvs::Box
    sync: true
    since: 2026-10-01
    mode: auto
    default_client: "Klienti/ACME s. r. o."
  - id: 44
    name: "Daňová schránka"
    type: Fs::Box
    sync: false                       # zatiaľ nepodporované

rules: []                             # časť 7.2
```

## 6. Synchronizácia

### 6.1 Spúšťanie **[návrh]**
- Iba na pokyn používateľa:
  - fráza pre agenta („skontroluj schránky“, „čo prišlo do schránky“),
  - príkaz `govbox sync`,
  - neskôr tlačidlo v UI (časť 13).
- Žiadny časovač, žiadna synchronizácia pri štarte aplikácie. Pravidelný beh je fáza 4 a potrebuje ADR.
- Jedna synchronizácia má dva kroky: **plán** (`govbox_sync_plan`) a **zápis** (`govbox_sync_apply`). Pri jednom pokyne „skontroluj schránky“ ich skill spustí za sebou. Zápis sa však robí len pre spoľahlivé priradenia (časť 7.3). Ostatné ostanú ako návrh.

### 6.2 Získanie správ **[overené]** + **[návrh]**

- **Kurzor.** `GET /api/messages/sync?last_id=N` vracia najviac 200 správ s `id > N`, zoradené podľa `id`, za **celého tenanta** (`Api::MessagesController#sync`).
  - Plugin si pamätá jeden kurzor pre tenanta v `state.sqlite`.
  - Pokračuje po stránkach, kým nedostane menej ako 200 správ.
  - Kurzor sa ukladá **po každej spracovanej stránke**, takže prerušenú synchronizáciu možno obnoviť.
- **Schránka správy.** Odpoveď sync neobsahuje `box_id` ani priečinok ([sync.json.jbuilder](../app/views/api/messages/sync.json.jbuilder)) **[overené]**. Adaptér ju preto odvodí z metadát ÚPVS:
  - `metadata.recipient_uri == box.uri` → prijatá správa do schránky `box`,
  - `metadata.sender_uri == box.uri` → odoslaná zo schránky `box`,
  - inak → schránku nemožno určiť: správa sa preskočí a zaloguje.
  - Správy Finančnej správy tieto polia nemajú, preto `Fs::Box` v MVP nie je podporovaný.
- **Rozpracované správy.** Sync vracia aj rozpracované podania (`MessageDraft < Message`). Plugin preskočí správy, ktoré majú `status` v stavoch rozpracovania (`being_loaded`, `being_validated`, `invalid`, `created`, `being_submitted`, `temporary_submit_fail`, `submit_fail`). Správy so stavom `submitted` spracuje ako odoslané. **[otvorené]** – overiť na stagingu, či odoslané podanie neprichádza aj ako samostatná správa (duplicita).
- **Filtre.** Správy zo schránok s `sync: false` a správy s `delivered_at < since` sa preskočia. Ich metadáta sa nesťahujú ďalej a objekty vôbec.
- **Obsah.** Pre správy, ktoré prejdú filtrom, plugin stiahne:
  - `GET /api/messages/{id}`: objekty vrátane `data` (Base64) = originály,
  - `GET /api/message_objects/{id}/pdf` pre každý objekt: PDF vizualizácia.
- **Neskoršie zmeny.** Kurzor `last_id` neprináša zmeny už stiahnutých správ, napr. štítok pridaný neskôr alebo prevzatá doručenka. Plugin preto pri každej synchronizácii znova načíta:
  - všetky **čakajúce doručenky** (časť 8.1) cez `GET /api/messages/{id}`,
  - vlákna, ktoré majú čakajúcu doručenku, cez `GET /api/message_threads/{id}`. Tak zistí, či po prevzatí pribudla správa s obsahom.
  - Ostatné zmeny MVP ignoruje. Riešenie je upstream (časť 12).
- **Chyby a obmedzenia.**
  - Chyby 5xx a sieť: 3 pokusy s exponenciálnym odstupom.
  - 401: neplatný kľúč, synchronizácia sa zastaví so stavom `error` a odkazom na `govbox_doctor`.
  - Žiadne paralelné sťahovanie nad 4 súbežné požiadavky.

### 6.3 Stav pluginu `LAWOSS_STATE_DIR/govbox/state.sqlite` **[návrh]**

SQLite je **iba cache a index**. Dá sa celý zmazať a znovu postaviť z `Office/govbox.yaml` a registrov `VSTUPY.md` (`govbox reindex`).

| Tabuľka | Obsah |
|---|---|
| `cursor` | `tenant_id`, `last_id`, `last_run_at` |
| `messages` | `govbox_id`, `uuid`, `box_id`, `thread_id`, `direction`, `delivered_at`, `title`, `sender_name`, `business_refs`, `tags`, `status` (`skipped`, `assigned`, `client_inbox`, `office_inbox`, `pending_delivery`), `target`, `input_id` |
| `threads` | `thread_id`, `target` (spis), `assigned_by` (`rule:<id>`, `manual`, `case_number`), `assigned_at` |
| `runs` | priebeh synchronizácie, počty, chyby (pre `KOMUNIKACNE-KANALY.md`) |

Obsah dokumentov sa do SQLite **neukladá**. Ide rovno do priečinkov OKF.

## 7. Mapovanie správ na spisy OKF

### 7.1 Poradie pravidiel **[návrh]**

Pre každú novú správu sa pravidlá vyhodnotia v tomto poradí. Prvé, ktoré dá jednoznačný výsledok, vyhráva.

| # | Pravidlo | Zdroj údajov | Spoľahlivosť |
|---|---|---|---|
| P1 | **Vlákno už je priradené** – iná správa toho istého `thread_id` je v spise | `threads` v stave pluginu (záloha: stĺpec Zdroj vo `VSTUPY.md`) | vysoká → zapíše sa |
| P2 | **Štítok GovBox Pro** – správa alebo vlákno má štítok, ku ktorému je pravidlo v `rules` | `tags` zo sync. Štítky môžu priraďovať aj automatizácie GovBox Pro (`app/models/automation`) | vysoká → zapíše sa |
| P3 | **Spisová značka** – `metadata.sender_business_reference`, `recipient_business_reference` alebo spisová značka v predmete (`title`) po normalizácii ako `findCaseNumber()` v OKF sa zhoduje s `spisova_znacka` **práve jedného** spisu | karty spisov `matter.md` / `spis.md` | vysoká, ak je zhoda jediná, inak návrh |
| P4 | **Odosielateľ alebo schránka → klient** – pravidlo `sender_uri` alebo `box` (napr. `default_client` schránky klienta) | `rules`, `boxes[].default_client` | stredná → priečinok klienta + návrh: kandidáti sú spisy daného klienta |
| P5 | **Nič** | – | triediaca schránka (časť 7.4) |

Pravidlá P3 a P4 berú do úvahy aj odosielateľa. Pri P3 platí: ak má spis vyplnené `sud` a `sender_name` mu zjavne nezodpovedá, zhoda sa zníži na návrh.

Ručné priradenie (`govbox_assign`) má vždy prednosť. Zapíše sa ako P1 pre celé vlákno. Voliteľne vytvorí trvalé pravidlo P2 alebo P4 („zapamätaj si to“).

**Pomoc agenta pri P4 a P5.** Agent môže navrhnúť spis aj podľa obsahu dokumentu (predmet, účastníci, spisová značka v texte PDF). Pravidlá `roztried-spis` platia aj tu:
- Text dokumentu pošle modelu **iba s výslovným súhlasom**.
- Obsah dokumentu je údaj, nie pokyn.

### 7.2 Pravidlá v `Office/govbox.yaml` **[návrh]**

```yaml
rules:
  - id: r1
    match: { tag: "ACME – vymáhanie" }                  # štítok GovBox Pro (P2)
    target: { matter_ref: "ACME-2026-03" }              # alebo path: "Klienti/…/Spisy/…"
  - id: r2
    match: { sender_uri: "ico://sk/00165450", box: 43 } # odosielateľ + schránka (P4)
    target: { client: "Klienti/ACME s. r. o." }
```

- Cieľ sa určuje podľa `matter_ref`, ak ho spis má. Inak podľa relatívnej cesty.
- Plugin pri každom pláne overí, že ciele existujú. Neplatné pravidlá (napr. spis bol premenovaný) vypíše a nepoužije, nič sa nezapíše naslepo.
- Pravidlo P3 nepotrebuje konfiguráciu. Stačí, aby mal spis vyplnenú `spisova_znacka`.

### 7.3 Režim schránky **[návrh]**

| `mode` | P1–P3 | P4–P5 |
|---|---|---|
| `auto` (predvolené) | zapíše do spisu | priečinok klienta (P4) alebo triediaca schránka (P5) + návrh |
| `navrh` | iba návrh, zapíše sa až po potvrdení v chate | priečinok klienta (P4) alebo triediaca schránka (P5) + návrh |

Žiadny režim nezapisuje do spisu pri strednej alebo nízkej spoľahlivosti. **[otvorené]** – či má byť počas alfa verzie predvolené `navrh`.

### 7.4 Kam sa zapisuje **[návrh]**

**Priradená správa** ide do priečinka s rolou `inbox` daného spisu, rovnako ako ručný príjem dokumentov v LAWOSS (`matter-intake.ts`):

```
Klienti/<Klient>/Spisy/<YYYY-MM názov veci>/
  00_Na_zatriedenie/
    IN-012/
      sprava.md                                    # hlavička správy (nižšie)
      2026-10-07_Uznesenie-8C-123-2023_v01.pdf     # PDF vizualizácia hlavného objektu
      2026-10-07_Priloha-1_v01.pdf                 # PDF vizualizácie príloh
      originaly/
        Uznesenie.asice                            # originály bajt po bajte (data z API)
        form.xml
  VSTUPY.md                                        # nový riadok IN-012, Stav: pending
```

- **Názvy súborov.** PDF vizualizácie sa pomenujú podľa `document_naming` z `okf.config` (predvolene `YYYY-MM-DD_popis_v01.ext`, dátum je dátum dokumentu; pri neznámom dátume `bez-datumu`). Originály si ponechajú pôvodné meno a obsah bajt po bajte. Ak sa mená zhodujú, platí pravidlo OKF (`IN-001/priloha.pdf`).
- **Triedenie.** Ďalej do `05_Komunikacia/Dolezita_posta` alebo inam ich roztriedi existujúci postup `roztried-spis` / advokát. Pravidlo OKF `DATA_BOX_EXT` už teraz posiela `.asice` a `.zfo` do dôležitej pošty. **[otvorené]** – či má plugin správy zo schránky zapisovať rovno do `05_Komunikacia/Dolezita_posta`.
- **`sprava.md`** má frontmatter:

```yaml
type: govbox-message
input_id: IN-012
direction: in                        # in | out
box: { id: 42, uri: "ico://sk/12345678", name: "Mgr. Ján Novák, advokát" }
govbox: { message_id: 98765, uuid: "…", thread_id: 4321, correlation_id: "…" }
delivered_at: 2026-10-07T09:14:00+02:00
sender: "Okresný súd Bratislava I"
sender_uri: "…"
business_reference: { sender: "8C/123/2023", recipient: null }
tags: ["ACME – vymáhanie"]
assigned_by: case_number             # thread | tag:r1 | case_number | manual
objects:
  - { name: "Uznesenie.asice", type: ATTACHMENT, signed: true, sha256: "…", pdf: "2026-10-07_Uznesenie-8C-123-2023_v01.pdf" }
```

Pod frontmatterom nasleduje čitateľný súhrn (predmet, odosielateľ, zoznam príloh). Žiadne zhrnutie od AI sa nepridáva automaticky.

**Riadok vo `VSTUPY.md`** (formát `lawoss/okf/inputs.ts`):

```
| IN-012 | 2026-10-07T09:14:00+02:00 | govbox:42/thread:4321/msg:98765 | 00_Na_zatriedenie/IN-012/ | pending | |
```

Stĺpec Zdroj je **trvalý záznam priradenia**. Z neho `govbox reindex` obnoví tabuľku `threads`.

**Nepriradená správa so známym klientom (P4)** zostane u klienta, ako to vyžaduje šablóna kanálov („Komunikáciu bez určenej veci ponechaj u klienta ako `pending`“):

```
Klienti/<Klient>/
  00_Na_zatriedenie/IN-003/…        # rovnaká štruktúra ako vyššie
  VSTUPY.md                         # register vstupov klienta
```

**Nepriradená správa bez klienta (P5)** ide do triediacej schránky praxe:

```
Office/Schranky/<krátky názov schránky>/
  00_Na_zatriedenie/IN-001/…
  VSTUPY.md
```

Šablóna klienta ani `Office/Schranky/` dnes `VSTUPY.md` nemajú. Plugin ich vytvorí podľa šablóny spisu **[otvorené]**, otázka 1.

`govbox_assign` správu potom **presunie** (nie skopíruje) do spisu:
- V spise jej pridelí nové `IN-NNN`.
- V pôvodnom registri označí riadok `processed` s odôvodnením „presunuté do <spis> / IN-NNN“.

**`KOMUNIKACNE-KANALY.md` – jeden register na schránku.** Šablóna zakazuje kopírovať kurzor do viacerých evidencií. Register preto vedie každá schránka na jednom mieste:

| Schránka | Register |
|---|---|
| Schránka klienta (`default_client`) | `Klienti/<Klient>/KOMUNIKACNE-KANALY.md` |
| Vlastná schránka advokáta alebo kancelárie (pošta pre viacerých klientov) | `Office/Schranky/<schránka>/KOMUNIKACNE-KANALY.md` |

Spisy kurzor nedostanú. Na kanál sa odkazujú cez stĺpec Zdroj vo svojom `VSTUPY.md`.

Riadok registra po každej synchronizácii:

| Stĺpec | Hodnota |
|---|---|
| Kanál | `GovBox Pro` |
| Účet | `<názov schránky> (<uri>)` |
| Povolený rozsah | `čítanie od <since>; zápis do 00_Na_zatriedenie` (od fázy 2: `+ prevzatie doručenky s potvrdením`) |
| Posledný pokus | dátum a čas s časovým pásmom |
| Posledná úplná kontrola | posunie sa **iba** ak prebehli všetky stránky bez chyby |
| Pokryté obdobie / kurzor | `od <since>, last_id=…` – rovnako iba po úplnej kontrole |
| Stav | `ok`, `partial` alebo `error` |
| Chyba / ďalší krok | text chyby bez tokenov, napr. „spusti govbox doctor“ |

Pri chybe zostáva posledná úplná kontrola a kurzor v registri na poslednom úspechu. Interný kurzor po stránkach v `state.sqlite` slúži iba na obnovenie prerušeného behu.

**Zápis do súborov.** Plugin zapisuje iba do priečinkov, ktoré má workspace povolené (Authorized Folders). Na konci každého zápisu spustí `okf validate` nad dotknutými spismi, ak je CLI dostupné. **[otvorené]** – či LAWOSS poskytne príkaz `okf input add`. Plugin by potom formát `VSTUPY.md` neduplikoval a nehrozil by rozchod formátov.

### 7.5 Lehoty a pamäť spisu **[návrh]**

Plugin **sám nezapisuje lehoty ani záznamy pamäte**. Zostáva to na agentovi a advokátovi podľa pravidiel `okf-pamat`:
- Po synchronizácii skill navrhne pre každý nový vstup udalosť do `## History` záznamu spisu: `- 2026-10-07 [delivery] — GovBox: <predmet> (<odosielateľ>), IN-012`. Zapíše sa cez `okf-memory write … --apply`.
- Lehotu vyťaženú z dokumentu agent zapíše iba ako **návrh** (`deadlines`). Za potvrdenú platí až so záznamom `verified: [{type: human, …}]`.

## 8. Doručenky a právne úkony

### 8.1 Čakajúce doručenky (MVP, iba čítanie)

Správa je čakajúca doručenka, ak `metadata.delivery_notification` existuje, `metadata.authorized` nie je `true` a `delivery_notification.delivery_period_end_at` je v budúcnosti. Je to rovnaká logika ako `Message#authorizable_delivery_notification?` **[overené]**, `app/models/message.rb:104`.

- Plugin priradí doručenke spis podľa pravidiel P1 až P4. Ako predmet použije `delivery_notification.consignment.subject`.
- Doručenka **ide do `VSTUPY.md` ako `pending` s vysvetlením**, podľa pravidla šablóny („Ak chýba príloha alebo obsah, zostáva `pending` s vysvetlením“). V priečinku `IN-NNN/` je iba `sprava.md` s `type: govbox-delivery-notification`. Posledný stĺpec obsahuje „čaká na prevzatie do 2026-10-22 14:00; obsah nedostupný do prevzatia“. Kokpit LAWOSS ju tak ukáže medzi nespracovanými vstupmi.
- `govbox_deliveries_pending` vráti zoznam: schránka, odosielateľ, predmet, spis, `IN-NNN`, **lehota na prevzatie** (`delivery_period_end_at`).
- Skill pri každej synchronizácii čakajúce doručenky zobrazí ako prvé a ponúkne zápis návrhu lehoty „prevziať do …“ do pamäte spisu.
- Neprevzatie môže mať právne následky. Plugin to iba oznamuje, neposudzuje. **[otvorené]** – právne overiť text upozornenia (fikcia doručenia podľa osobitných predpisov).
- Ak lehota uplynie bez prevzatia, plugin doplní do riadku „lehota na prevzatie uplynula <dátum>“. Riadok zostane `pending`, kým ho advokát nespracuje.

Po prevzatí (v GovBox Pro alebo cez plugin) pribudne vo vlákne správa s obsahom. Pravidlo P1 ju zaradí do toho istého spisu ako doručenku a dostane vlastné `IN-NNN`. Riadok doručenky plugin doplní odkazom „prevzaté <dátum>, obsah v IN-NNN“. Na `processed` ho prepne až advokát alebo agent so súhlasom, podľa pravidla šablóny.

### 8.2 Potvrdzovacia brána (fáza 2 a 3) **[návrh]**

**Prevzatie doručenky** je právny úkon: písomnosť je doručená a začínajú plynúť lehoty. **Odoslanie podania** je tiež právny úkon. Ani jeden nesmie spustiť automatizácia ani agent bez človeka. Plugin to vynúti sám, nespolieha sa na poslušnosť agenta.

1. Agent zavolá `govbox_delivery_authorize_prepare(message_id)`. Plugin vráti súhrn: schránka, odosielateľ, predmet, lehota.
2. Agent zavolá `govbox_delivery_authorize(message_id)`. Plugin **sám zobrazí natívny dialóg macOS** (`osascript`, `display dialog` s tlačidlami „Prevziať“ a „Zrušiť“, predvolene Zrušiť) s rovnakým súhrnom.
   - Dialóg agent nevie ovládať.
   - Bez kliknutia na „Prevziať“ do 120 s sa úkon nevykoná.
3. Po potvrdení plugin zavolá `POST /api/messages/{id}/authorize_delivery_notification` a zapíše výsledok do `runs` a do `KOMUNIKACNE-KANALY.md`.

Rovnaká brána platí pre `govbox_draft_submit` (`POST /api/messages/{id}/submit`).

Doplnkovo sa nástroje fázy 2 a 3 deklarujú v katalógu ako `external-action` s `humanGate: true`. Ak to OpenCode umožní pre nástroje MCP, nastavia sa aj na `ask` v mape oprávnení. **[otvorené]** – overiť v LAWOSS (`ApprovalService`, `runtime-opencode-config-store.ts`).

## 9. MCP nástroje

| Nástroj | Fáza | Schopnosť | Brána | Popis |
|---|---|---|---|---|
| `govbox_doctor` | 1 | network | – | Overí kľúč v Keychain, JWT, `GET /api/boxes`, príznak `:api`, platnosť cieľov pravidiel. Upozorní, že kľúč otvára celého tenanta. |
| `govbox_boxes_list` | 1 | network | – | Schránky tenanta a ich stav výberu. |
| `govbox_boxes_select` | 1 | local-write | – | Zapíše výber, `since` a `mode` do `Office/govbox.yaml`. |
| `govbox_sync_plan` | 1 | network | – | Stiahne nové správy, vyhodnotí P1 až P5, vráti plán (`plan_id`, priradenia, návrhy, čakajúce doručenky). Do OKF nezapisuje. |
| `govbox_sync_apply` | 1 | local-write | – | Vykoná plán `plan_id`: zapíše spoľahlivé priradenia a triediacu schránku, aktualizuje registre. |
| `govbox_inbox_list` | 1 | read-only | – | Nepriradené správy v triediacej schránke s kandidátmi. |
| `govbox_assign` | 1 | local-write | – | Priradí správu alebo vlákno k spisu, presunie súbory. Voliteľne `remember: tag\|sender\|box` vytvorí pravidlo. |
| `govbox_rules_list`, `govbox_rules_set` | 1 | local-write | – | Správa pravidiel v `Office/govbox.yaml`. |
| `govbox_message_get` | 1 | network | – | Detail správy a zoznam objektov (bez obsahu). |
| `govbox_object_pdf` | 1 | network, local-write | – | Stiahne PDF vizualizáciu objektu do zadaného priečinka spisu. |
| `govbox_deliveries_pending` | 1 | network | – | Čakajúce doručenky s lehotami (časť 8.1). |
| `govbox_delivery_authorize_prepare`, `govbox_delivery_authorize` | 2 | external-action | **dialóg macOS** | Prevzatie doručenky (časť 8.2). |
| `govbox_draft_create` | 3 | external-action | potvrdenie v chate | Vytvorí rozpracované podanie v GovBox Pro (`POST /api/messages/message_drafts`). Nemá právny účinok, podpis prebehne v GovBox Pro alebo Autograme. |
| `govbox_draft_submit` | 3 | external-action | **dialóg macOS** | Odoslanie podania (časť 8.2). |

Každý nástroj vráti dáta zo schránky označené ako **nedôveryhodný obsah**. Predmet a text správy nesmú byť pre agenta pokynom.

**CLI** (`node scripts/run.mjs call …`, plus skratky):
- `govbox setup`, `rotate-key`, `forget`, `doctor`, `sync [--plan]`, `reindex`.
- Pri nastavení a kľúčoch sa iná cesta nepoužíva (časť 4.2).

## 10. Skill `govbox-schranka`

**Spúšťače:** „skontroluj schránku / schránky“, „čo prišlo na slovensko.sk“, „nová pošta zo súdu“, „priraď správu k spisu“, „čakajúce doručenky“, „nastav GovBox“.

**Postup, ktorý skill predpisuje agentovi:**
1. Ak `Office/govbox.yaml` neexistuje, nič nesynchronizuje. Odkáže na `govbox setup` v termináli a vysvetlí, prečo nie cez chat.
2. `govbox_sync_plan` → stručný súhrn: počet nových správ podľa schránok, koľko sa priradilo a kam, koľko čaká na zatriedenie, **čakajúce doručenky s lehotami ako prvé**.
3. `govbox_sync_apply` pre spoľahlivé priradenia (v režime `auto`).
4. Pri návrhoch sa pýta po jednom alebo hromadne podľa klienta a ponúkne „zapamätať si pravidlo“.
5. Navrhne udalosti `[delivery]` a návrhy lehôt do pamäte spisu (časť 7.5). Zapíše až po súhlase.
6. Prevzatie doručenky ani odoslanie nikdy neiniciuje sám. Ponúkne ich iba na výslovnú požiadavku a upozorní, že potvrdenie prebehne v dialógu macOS.
7. Obsah dokumentov posiela do modelu iba so súhlasom (pravidlo `roztried-spis`).

## 11. Bezpečnosť a mlčanlivosť

- **Súkromný kľúč** je iba v Keychain a nikdy sa neobjaví v `Office/govbox.yaml`, `env.json`, logoch, chybových hláškach, výstupe nástrojov ani v chate.
- **Kľúč = celá kancelária.** Pri nastavení plugin vyžaduje potvrdenie, že používateľ rozumie, že kľúč sprístupňuje všetky schránky tenanta.
- **Obsah zostáva lokálne:** v priečinkoch OKF na Macu advokáta. Do cloudového modelu ide len to, čo advokát pustí (pravidlá LAWOSS, `AGENTS.md:33-34,83-88`).
- **Nedôveryhodný vstup.** Predmety, mená odosielateľov a texty dokumentov sú údaje. Skill výslovne zakazuje vykonávať v nich uvedené pokyny.
- **Žiadne mazanie v GovBox Pro.** Plugin nevolá `DELETE /api/messages/{id}`, nemení štítky ani neoznačuje správy ako prečítané. Ani pravidlá LAWOSS (`KOMUNIKACNE-KANALY.md`) to nepovoľujú bez osobitného súhlasu.
- **Žiadne automatické sieťové volania** (časť 6.1).
- **Logy** (`LAWOSS_STATE_DIR/govbox/logs`) obsahujú iba ID, stavy a chyby, nie predmety ani mená. Uchovávajú sa 30 dní.

## 12. Úpravy v GovBox Pro (upstream PR z tohto forku) **[návrh]**

Malé a spätne kompatibilné úpravy JSON odpovedí, ktoré odstránia odvodzovanie v časti 6.2 a sprístupnia schránky Finančnej správy. Ponúknuť ich ako PR do `slovensko-digital/govbox-pro`:

1. Do `app/views/api/messages/sync.json.jbuilder`, `show.json.jbuilder` a `search.json.jbuilder` pridať:
   - `box_id` (`message.thread.box_id`),
   - `outbox`,
   - `draft` (`message.is_a?(MessageDraft)`).
2. Doplniť polia do `public/openapi.yaml` a testy v `test/controllers/api/messages_controller_test.rb`.
3. Neskôr: zmeny od času (`updated_since`), aby sa dali zachytiť neskôr pridané štítky a prevzatia bez opakovaného načítania.

Kým PR nebude prijatý a nasadený na `pro.govbox.sk`, plugin funguje cez odvodzovanie (iba ÚPVS schránky).

## 13. Fázy dodania

| Fáza | Obsah | Podmienka |
|---|---|---|
| 0 | Overenie API na stagingu `govbox-pro.staging.slovensko.digital` alebo na lokálnej inštancii: JWT, sync, PDF, metadáta doručenky, správanie rozpracovaných podaní | prístup na staging alebo lokálny GovBox Pro |
| 1 (MVP) | Časti 4 až 7, 8.1, nástroje fázy 1, skill, macOS | – |
| 2 | Prevzatie doručenky s dialógom (časť 8.2) | právne overenie textov, test na stagingu |
| 3 | Rozpracované podania a odoslanie | fáza 2 |
| 4 | Pravidelná synchronizácia na pozadí; karta GovBox v Integráciách LAWOSS (`apps/app/src/lawoss/domains/integrations`) | **nové ADR** (pravidlo 9) a rozhodnutie o zóne zmien |
| 5 | Schránky Finančnej správy; adaptér slovensko-sk-api; Windows | upstream PR (časť 12); dopyt používateľov |

## 14. Otvorené otázky

1. **Nové miesta v OKF:** `VSTUPY.md` a `00_Na_zatriedenie/` u klienta (šablóna kanálov s nimi počíta, šablóna klienta ich nemá) a triediaca schránka praxe `Office/Schranky/<schránka>/`. Treba odsúhlasiť s vlastníkom OKF a doplniť šablóny v `lawoss/okf/templates`.
2. **Formát `VSTUPY.md`:** môže ho plugin zapisovať sám, alebo LAWOSS poskytne `okf input add`? Odporúčanie: CLI v `lawoss/okf` (zelená zóna).
3. **Cieľový priečinok** správ zo schránky: `00_Na_zatriedenie` (konzistentné s ručným príjmom), alebo rovno `05_Komunikacia/Dolezita_posta`?
4. **Predvolený režim** `auto` alebo `navrh` počas alfa verzie.
5. **Ako plugin zistí koreň praxe OKF:** parameter nástroja od agenta, premenná prostredia alebo zápis v stave po prvom nastavení. MCP server beží s pracovným priečinkom v `LAWOSS_STATE_DIR`.
6. **Brána pre MCP nástroje v LAWOSS** (`ask` v mape oprávnení / `ApprovalService`) ako doplnok k dialógu macOS.
7. **Prístup k API GovBox Pro:** zapnutie `:api` pre zákazníka, plán, cena (otázka na podpora@slovensko.digital).
8. **Duplicity** pri odoslaných podaniach a správanie `status` v sync – overiť vo fáze 0.
9. **Text upozornenia** na lehotu prevzatia a následky neprevzatia – právne overiť.

## 15. Akceptačné kritériá MVP

- [ ] `govbox setup` vytvorí kľúč v Keychain, vypíše verejný kľúč. Po jeho vložení do GovBox Pro prejde `govbox_doctor`.
- [ ] Súkromný kľúč sa nevyskytuje v žiadnom súbore praxe, logu ani výstupe nástroja (test: `grep` po celom priečinku OKF a stave pluginu).
- [ ] Používateľ vyberie schránky. Do OKF sa dostanú iba správy z vybraných schránok a od dátumu `since`.
- [ ] Správa so spisovou značkou, ktorá zodpovedá jedinému spisu, skončí v jeho `00_Na_zatriedenie/IN-NNN/` s PDF, originálmi a `sprava.md`. Vo `VSTUPY.md` je riadok `pending`.
- [ ] Ďalšia správa z toho istého vlákna sa zaradí do toho istého spisu aj bez spisovej značky.
- [ ] Správa so známym klientom, ale bez spisu, skončí u klienta. Správa bez zhody skončí v triediacej schránke praxe. Po `govbox_assign` je presunutá do spisu, pôvodný riadok je `processed` s odkazom a voliteľné pravidlo funguje pri ďalšej synchronizácii.
- [ ] Každá schránka má jediný riadok kanála v jedinom `KOMUNIKACNE-KANALY.md`. Pri chybe sa posledná úplná kontrola neposunie.
- [ ] Prerušená synchronizácia pokračuje od posledného kurzora bez duplicít. `govbox reindex` po zmazaní `state.sqlite` obnoví priradenia vlákien z `VSTUPY.md`.
- [ ] Čakajúca doručenka je vo `VSTUPY.md` ako `pending` s lehotou na prevzatie. Po prevzatí v GovBox Pro pribudne obsah ako nový vstup v tom istom spise.
- [ ] `okf validate` nad dotknutými spismi neohlási chyby.
- [ ] V LAWOSS neprebehne žiadne sieťové volanie GovBox bez pokynu používateľa.
