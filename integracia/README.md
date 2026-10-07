# Integrácia GovBox Pro do advokátskej praxe, LAWOSS a Chevron7

Analýza a rozhodnutia zo 6. – 7. 10. 2026. Tento priečinok nie je súčasťou pôvodného projektu GovBox Pro (slovensko-digital/govbox-pro); eviduje zámer, ako ho využiť v praxi a v aplikáciách LAWOSS a Chevron7.

## 1. Čo je GovBox Pro

Webová aplikácia (Ruby on Rails, licencia EUPL 1.2) na prácu so schránkami na slovensko.sk. Autor Solver IT a Slovensko.Digital, SaaS prevádzkujú Služby Slovensko.Digital na `https://pro.govbox.sk`.

- Správy vo vláknach, odfiltrované technické správy (doručenky, potvrdenia).
- Viac schránok naraz (kancelária, jednotliví advokáti), prístupy podľa skupín používateľov.
- Štítky, filtre, automatizácie (`app/models/automation`), poznámky k vláknam, hromadné spracovanie.
- Podpisovanie cez Autogram, podávanie podaní, aj hromadne.
- Dlhodobá archivácia podpísaných dokumentov (samostatný komponent govbox-pro-archiver).
- Schránka Finančnej správy / eDane (`app/models/fs`, `app/lib/fs`) popri ÚPVS.
- Otvorené REST API: [public/openapi.yaml](../public/openapi.yaml).

GovBox Pro sa na ÚPVS nenapája sám, ide cez prostredníka (`app/lib/upvs_environment.rb`, `Upvs::GovboxApiClient`): službu GovBox API alebo vlastný [slovensko-sk-api](https://github.com/slovensko-digital/slovensko-sk-api).

## 2. Technický účet na slovensko.sk

| Cesta | Technický účet od NASES | Poznámka |
|---|---|---|
| **A) GovBox API / hosťovaný GovBox Pro** (Služby Slovensko.Digital) | **Nie.** Integráciu na ÚPVS má prevádzkovateľ, používateľ mu udelí prístup k svojej schránke. | Platená služba, rýchly štart. |
| **B) Vlastný slovensko-sk-api** | **Áno.** Integračný proces s NASES, certifikáty, testovanie na skúšobnom prostredí. | Plná nezávislosť, mesiace administratívy. |

**Rozhodnutie:** cesta A. Ceny a podmienky treba overiť na [ekosystem.slovensko.digital](https://ekosystem.slovensko.digital/sluzby/govbox-api) (neoverené).

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

API je za prepínačom tenanta `feature_flags << :api` – zrejme ho musí zapnúť prevádzkovateľ.

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
- **Licencie:** komunikácia iba cez API → EUPL sa LAWOSS (MIT) netýka. Kopírovanie kódu GovBox Pro do LAWOSS by prinieslo povinnosti podľa EUPL.
- **Tento repozitár:** názov `govbox-pro-macOS` naznačuje natívnu Mac verziu, no obsah je zatiaľ pôvodný webový Rails projekt.

## 6. Otvorené úlohy

- [ ] Opýtať sa podpora@slovensko.digital: ako sa zapína API (`:api`) pre zákazníka tretej strany, v akom pláne a za akú cenu.
- [ ] Navrhnúť Slovensko.Digital partnerstvo / uvedenie LAWOSS ako integrácie.
- [ ] Napísať špecifikáciu pluginu v repozitári LAWOSS (nástroje, schéma konfigurácie, uloženie kľúčov, potvrdzovacie kroky).
- [ ] Rozbehať GovBox Pro lokálne (staging `https://govbox-pro.staging.slovensko.digital`) a vyskúšať API.
- [ ] Návrh prepojenia Chevron7 ↔ GovBox Pro API (konverzia, podpis).
