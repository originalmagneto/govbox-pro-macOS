# Integrácia GovBox Pro do advokátskej praxe, LAWOSS a Chevron7

Analýza a rozhodnutia zo 6. – 7. 10. 2026, doplnené 7. 10. 2026 po overení v kóde. Tento priečinok nie je súčasťou pôvodného projektu GovBox Pro (slovensko-digital/govbox-pro); eviduje zámer, ako ho využiť v praxi a v aplikáciách LAWOSS a Chevron7.

## 1. Čo je GovBox Pro

Webová aplikácia (Ruby on Rails, licencia EUPL 1.2) na prácu so schránkami na slovensko.sk. Autor Solver IT a Slovensko.Digital, SaaS prevádzkujú Služby Slovensko.Digital na `https://pro.govbox.sk`.

- Správy vo vláknach, odfiltrované technické správy (doručenky, potvrdenia).
- Viac schránok naraz (kancelária, jednotliví advokáti), prístupy podľa skupín používateľov.
- Štítky, filtre, automatizácie (`app/models/automation`), poznámky k vláknam, hromadné spracovanie.
- Podpisovanie cez Autogram, podávanie podaní, aj hromadne.
- Dlhodobá archivácia podpísaných dokumentov (samostatný komponent govbox-pro-archiver).
- Schránka Finančnej správy / eDane (`app/models/fs`, `app/lib/fs`) popri ÚPVS.
- Otvorené REST API: [public/openapi.yaml](../public/openapi.yaml).

GovBox Pro sa na ÚPVS nenapája sám. Volá prostredníka na adrese `GB_API_URL` (`app/services/upvs/govbox_api_client.rb`, `app/lib/upvs/govbox_api.rb`): buď službu GovBox API, alebo vlastný [slovensko-sk-api](https://github.com/slovensko-digital/slovensko-sk-api).

**Oba prostredníky hovoria rovnakým protokolom** (overené v kóde): endpointy `/api/edesk/*` a `/api/sktalk/*`, autentifikácia JWT `{sub, obo, exp, jti}` podpísaným RS256 súkromným kľúčom pripojenia schránky (`app/lib/upvs/api.rb:23`, model `Govbox::ApiConnection` so stĺpcami `sub`, `obo`, `api_token_private_key`). GovBox API je teda hosťovaný slovensko-sk-api; vlastná inštancia s vlastným technickým účtom sa ovláda rovnako, mení sa len URL, `sub` a kľúč.

## 2. Technický účet na slovensko.sk

Pre plugin z toho vychádzajú **dva adaptéry a štyri situácie používateľa**:

| Situácia používateľa | Technický účet od NASES | Adaptér v plugine |
|---|---|---|
| 1. Platí hosťovaný GovBox Pro (`pro.govbox.sk`) | nie – integráciu na ÚPVS má prevádzkovateľ | **GovBox Pro API** (tenký) |
| 2. Prevádzkuje vlastný GovBox Pro (EUPL) s `GB_API_URL` na vlastný slovensko-sk-api | áno | **GovBox Pro API** (ten istý) |
| 3. Platí iba GovBox API (bez Pro) | nie | **slovensko-sk-api** (ťažký) |
| 4. Prevádzkuje iba vlastný slovensko-sk-api | áno | **slovensko-sk-api** (ten istý) |

- Používateľ s vlastným technickým účtom nepotrebuje v plugine nič osobitné. Najlepšie mu poslúži situácia 2 – plugin zostáva rovnako tenký ako pri situácii 1.
- Adaptér slovensko-sk-api (situácie 3, 4) by musel sám robiť to, čo dnes robí GovBox Pro: synchronizáciu priečinkov a správ, skladanie vlákien, spracovanie XML formulárov, autorizáciu doručeniek, odosielanie cez SKTalk. To je prakticky prepis GovBox Pro.
- Priame napojenie na ÚPVS bez slovensko-sk-api (vlastná implementácia SOAP, STS, WS-Security s certifikátom technického účtu) sa neodporúča.
- Proces získania technického účtu u NASES (zmluva, certifikát, testovanie, či ho dostane aj advokát ako fyzická osoba – podnikateľ) **nie je overený**.

**Rozhodnutie:** najprv adaptér GovBox Pro API (situácie 1 a 2). Rozhranie adaptéra navrhnúť tak, aby sa dal neskôr doplniť adaptér slovensko-sk-api. Ceny a podmienky GovBox Pro / GovBox API treba overiť na [ekosystem.slovensko.digital](https://ekosystem.slovensko.digital/sluzby/govbox-api) (neoverené).

## 3. Zvolený model: plugin / App do LAWOSS, „bring your own account“

- Plugin v LAWOSS je zadarmo a open-source.
- Používateľ si sám platí GovBox Pro a do pluginu pripojí vlastný prístup.
- LAWOSS nič nepreúčtováva, nemá zmluvu so štátom, nedrží cudzie schránky ani advokátske tajomstvo.

### Prečo GovBox Pro API, nie holé GovBox API

| | **GovBox Pro API** (zvolené) | **GovBox API** |
|---|---|---|
| Používateľ platí | GovBox Pro | len GovBox API |
| Logika v plugine | tenká: správy, vlákna, PDF, štítky, podania, doručenky | ťažká: synchronizácia, vlákna, XML formuláre, doručenky – prerábanie GovBox Pro |
| Pre advokáta | to isté vidí vo webe aj v mobile, podpis cez Autogram, archív | všetko len v LAWOSS |

### Autentifikácia (overené v kóde)

Zdroj: `app/lib/api_token_authenticator.rb`, [DEVELOPER.md](../DEVELOPER.md), `securitySchemes.Tenant_Token` v openapi.yaml.

1. Používateľ vygeneruje RSA pár kľúčov, **verejný kľúč** nahrá k svojmu tenantovi v GovBox Pro.
2. **Súkromný kľúč** zostáva u neho (macOS Keychain).
3. Plugin si sám vytvára JWT: RS256, `sub` = ID tenanta, `exp` najviac 5 minút, `jti` unikátny (32–256 znakov).

**Rozsah prístupu = celý tenant.** Tenant má jeden verejný kľúč (`api_token_public_key`) a `sub` je ID tenanta (`app/lib/api_environment.rb:16`). Súkromný kľúč v Keychaine teda otvára **všetky schránky a vlákna kancelárie**, nie iba schránky jedného advokáta. Výber schránok na synchronizáciu musí preto robiť plugin a kľúč treba chrániť ako prístup k celej kancelárii.

**Zapnutie API (overené):** `:api` je v `Tenant::ALL_FEATURE_FLAGS` (`app/models/tenant.rb:67`). Zapína ho site admin cez `/api/site_admin/tenants`; kým nie je zapnuté, administrátor tenanta nevidí v menu položku „API Prístup“ (`app/lib/sidebar_menu.rb:41`, `app/policies/admin/api_access_policy.rb`).

### Rozsah MVP

- **Bez potvrdenia (čítanie):** zoznam schránok, synchronizácia a vyhľadávanie správ (`/api/messages/sync`, `/api/messages/search`), stiahnutie PDF (`/api/message_objects/{id}/pdf`) do spisu (štruktúra OKF, skill `novy-spis`).
- **Pomoc agenta:** zhrnutie rozhodnutia, vyťaženie lehôt, priradenie k spisu podľa spisovej značky, zápis lehôt do kalendára / Reminders.
- **Iba so súhlasom človeka:** návrh podania (`/api/messages/message_drafts`), odoslanie (`/api/messages/{id}/submit`), potvrdenie doručenky (`/api/messages/{id}/authorize_delivery_notification`).

## 4. Chevron7

Oba projekty stoja na Autograme, prepojenie je prirodzené:

- **Zaručená konverzia** doručených rozhodnutí podľa zákona č. 305/2013 Z. z.: dokument zo schránky cez API → listinný výstup s osvedčovacou doložkou.
- **Podpis podaní** pripravených v GovBox Pro alebo LAWOSS, odoslanie cez API.
- **Lokálna evidencia** prijatých písomností naviazaná na spis.

## 5. Riziká a pravidlá

- **Potvrdenie doručenky je právny úkon** – písomnosť je doručená a plynú lehoty. Automatizácia ani AI agent ho nesmie spustiť bez potvrdenia človekom. To isté platí pre `submit`.
- **Advokátske tajomstvo:** posielanie obsahu schránky do cloudovej AI vyžaduje jasné pravidlá; preferovať lokálne spracovanie.
- **Jeden kľúč na celú kanceláriu:** únik súkromného kľúča tenanta sprístupní všetky schránky. Kľúč patrí do Keychainu, nikdy do konfiguračných súborov ani logov.
- **Licencie:** komunikácia iba cez API → EUPL sa LAWOSS (MIT) netýka. Kopírovanie kódu GovBox Pro do LAWOSS by prinieslo povinnosti podľa EUPL.
- **Tento repozitár:** názov `govbox-pro-macOS` naznačuje natívnu Mac verziu, no obsah je zatiaľ pôvodný webový Rails projekt.

## 6. Otvorené úlohy

> **Stav: odložené (7. 10. 2026).** Integráciu teraz neriešime. Pri návrate začať rozhodnutiami nižšie a fázou 0 zo [špecifikácie](SPECIFIKACIA-PLUGINU.md#13-fázy-dodania).

**Rozhodnutia, ktoré čakajú** (podrobne v časti 14 špecifikácie):

- [ ] Nové miesta v OKF: `VSTUPY.md` a `00_Na_zatriedenie/` u klienta a triediaca schránka `Office/Schranky/<schránka>/` – odsúhlasiť a doplniť šablóny v `lawoss/okf/templates`.
- [ ] Cieľ správ zo schránky: `00_Na_zatriedenie`, alebo rovno `05_Komunikacia/Dolezita_posta`.
- [ ] Predvolený režim mapovania počas alfa verzie: `auto`, alebo `navrh`.
- [ ] Zápis do `VSTUPY.md`: plugin sám, alebo nový príkaz `okf input add` v LAWOSS (odporúčané).

**Úlohy:**

- [ ] Opýtať sa podpora@slovensko.digital: ako sa zapína API (`:api`) pre zákazníka tretej strany, v akom pláne a za akú cenu.
- [ ] Navrhnúť Slovensko.Digital partnerstvo / uvedenie LAWOSS ako integrácie.
- [x] Napísať špecifikáciu pluginu – [SPECIFIKACIA-PLUGINU.md](SPECIFIKACIA-PLUGINU.md) (neskôr presunúť do repozitára LAWOSS).
- [ ] Overiť proces a podmienky technického účtu u NASES pre advokáta.
- [ ] Fáza 0: rozbehať GovBox Pro lokálne (staging `https://govbox-pro.staging.slovensko.digital`) a vyskúšať API.
- [ ] Upstream PR do `slovensko-digital/govbox-pro`: `box_id`, `outbox`, `draft` v odpovediach `/api/messages/*` (špecifikácia, časť 12).
- [ ] Návrh prepojenia Chevron7 ↔ GovBox Pro API (konverzia, podpis).
