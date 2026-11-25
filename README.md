# Grootboek Sorteerder

Zelfgebouwde tool voor de boekhouding van Café Cafetaria 't Hoekje (Lemelerveld, boekhouding in Exact Online).
Leveranciersfacturen (Bidfood, Horesca Meppel, …) hebben ongesorteerde productregels. De tool leest de PDF, koppelt
elke regel aan een grootboeknummer en telt per grootboek (en BTW-code) de subtotalen op, klaar om in te boeken.

Eigenaar: Jasmijn (boddie91). Alle tekst in de interface is Nederlands. Doel van de tool: **zo min mogelijk nadenkwerk**.
Bij elke wijziging eerst vragen: "bespaart dit haar een beslissing of een handeling?"

## Bestanden

| Bestand | Wat |
|---|---|
| `index.html` / `taskpane.html` | De hele tool in één bestand. **Moeten altijd identiek zijn** (`cmp index.html taskpane.html`). `taskpane.html` is wat het Outlook-paneel laadt; `index.html` is hetzelfde, bruikbaar als losse webpagina. |
| `manifest.xml` | Outlook-invoegtoepassing (XML-manifest, Mailbox 1.8, leesvenster van een bericht). Alle urls wijzen naar `https://boddie91.github.io/grootboek-sorteerder/`. |
| `icon-16/32/64/80/128.png` | Pictogrammen voor het manifest. |

Hosting: GitHub Pages van de `main`-branch (root). Een push naar `main` is na ongeveer een minuut live
(controleer onder Actions of "pages build and deployment" groen is). Het manifest zelf hoeft alleen opnieuw
in Outlook geladen te worden als het manifest verandert, niet bij wijzigingen in de HTML.

## Techniek

- Eén `<script type="module">` met Preact + htm (via jsdelivr, `htm@3.1.1/preact/standalone.module.js`), Tailwind via CDN,
  pdf.js 3.11.174 (cdnjs) voor tekstuitlezing, Office.js voor Outlook. Geen build-stap.
- Kleuren: marineblauw `#1F3A5F`, crème `#FAFAF7`, randen `#E4E1D6`/`#DCD8CB`, groen `#3D6B5C`, rood `#B34D4D`.
- Tabbladen: **Facturen sorteren**, **Leveranciers**, **Boekingsregels**, **Instellingen**.

### Hoe een factuur verwerkt wordt
1. PDF (slepen, kiezen of uit Outlook-bijlage) → pdf.js reconstrueert tekstregels op y-positie → `parseInvoiceText`.
2. `extractAmount`: laatste bedrag op de regel (`1.234,56`, `123,45`, `123.45`; min ervoor of erachter = negatief).
   Datums (`30.10.2026`) en artikelnummers (`92.22.94`) tellen niet als bedrag. Een losse letter erachter is de BTW-letter (L/H).
3. `cleanDescription`: artikelnummer + omschrijving, zonder aantal, verpakking en prijzen.
   `92.22.94 1 ZK VERSZO TROSTOMATEN 750GR 4,25 ZK 4,25 L` → `92.22.94 VERSZO TROSTOMATEN 750GR`. Dit is ook het automatisch voorgestelde trefwoord.
4. `matchRule`: een regel matcht als het trefwoord (hoofdletterongevoelig) in de opgeschoonde of originele omschrijving staat; langste trefwoord wint. Bedragen doen niet mee in de matching.
5. Regels zonder bedrag, en samenvattingsregels (totaal, BTW, te betalen, …) zonder artikelnummer, staan onder "Standaard genegeerde regels" (inklapbaar, handmatig toe te wijzen).
6. Resultaat: niet-herkende regels bovenaan, daaronder subtotalen per grootboek + BTW-code (standaard ingeklapt), totaal, CSV-export
   (`;`-gescheiden, komma als decimaal; kolommen Grootboeknummer, Omschrijving, Bedrag, BTW-code, Factuurnummer, Factuurdatum).
   Controle: som van de regels wordt vergeleken met het uitgelezen factuurtotaal.
7. Factuurnummer/-datum/totaal worden best-effort uitgelezen (`extractInvoiceMeta`: labels, Horesca-patroon `26-08637`, bestandsnaam) en zijn altijd te corrigeren.

### BTW
Per leverancier een `btwMap` (letter → code), te beheren onder Boekingsregels. Bidfood: L = 3, H = 4 (startwaarde).
Prioriteit voor de BTW-code van een regel: handmatig ingevuld → letter op de factuur → BTW-code van de boekingsregel.

### Gegevens
Leveranciers (met `rules` en `btwMap`) staan in `localStorage` (`grootboek-sorteerder-leveranciers`) en worden met
Supabase gesynchroniseerd. Facturen en resultaten worden nooit bewaard. Handmatige back-up: Instellingen → opslaan/laden (JSON).

### Supabase (online opslag)
- Project: `https://kimiiwxzjcmgzpgdxkjd.supabase.co` (project "boddie91's Project", dat ook het huishouddashboard bevat: tabellen `hh_*` niet aanraken).
- Tabel `public.grootboek_instellingen (user_id uuid pk default auth.uid(), data jsonb, updated_at timestamptz)`, RLS aan,
  policy "eigen rij" (alleen eigen `user_id`). Inhoud: `{ "suppliers": [...] }`.
- De publishable key staat in de HTML (bedoeld om openbaar te zijn; de data wordt beschermd door inloggen + RLS).
- Gedrag: bij inloggen wint de online versie als die data bevat; is die leeg, dan wordt de lokale versie omhoog gezet.
  Elke wijziging wordt na 600 ms geüpload. Inloggen via e-mail + wachtwoord (REST, geen Supabase-bibliotheek).

### Outlook-invoegtoepassing
- Persoonlijk Outlook-account (hotmail/outlook.com) op het web: "Via URL" is uitgeschakeld, dus het manifest moet **Uit bestand** geladen worden.
- Het paneel moet vastgepind zijn (`SupportsPinning` in het manifest). Bij een nieuwe geselecteerde mail (`ItemChanged`) wordt de vorige factuur gewist;
  heeft de mail precies één PDF-bijlage, dan wordt die automatisch ingelezen en gesorteerd.
- Een oud manifest met een placeholder-url gaf eerder een GitHub-404 in het paneel: bij vreemd gedrag eerst het geladen manifest controleren.

### Kantoor-dashboard
In het dashboard (repo `hoekje-kantoor-dashboard`) draait de tool als gewone webpagina, zonder Office.js. Het dashboard roept
`window.grootboekLaadPdf(base64, naam)` aan om een pdf te openen die de gebruiker in Outlook (binnen het dashboard) heeft gedownload.
Let op: het dashboard heeft een eigen `localStorage` – leveranciers/regels zijn daar niet dezelfde als in de Outlook-invoegtoepassing,
tenzij ze via online opslag gesynchroniseerd worden.

## Werken aan de tool (voor Claude)
1. Haal eerst de nieuwste versie op (`git fetch`), want in een andere chat kan er iets veranderd zijn.
2. Wijzig `taskpane.html` en kopieer naar `index.html`. Controleer de syntaxis van het script (`node --check` op de inhoud van het `<script type="module">`, met de CDN-import vervangen).
3. Commit en push naar `main`. De gebruiker hoeft daarna alleen Ctrl+F5 in Outlook te doen.
4. De UI is nooit in een echte browser getest vanuit de sandbox (CDN's en npm zijn daar geblokkeerd): wees eerlijk over wat wel en niet getest is,
   en vraag de gebruiker waar mogelijk om een testresultaat.
5. Onbekend: de exacte tekstopmaak van echte Bidfood-/Horesca-PDF's. Bij herkenningsproblemen: vraag om de tekst van een factuurkop of een paar regels.

## Open wensen / ideeën
- PDF-bijlagen van Bidfood/Horesca uit hoekjelemelerveld@outlook.com per betaaldatum (kwartaal) in Dropbox archiveren (Dropbox-connector kan geen binaire bestanden schrijven; Zapier is een mogelijke route).
- Gelijksoortige tool voor de kassa-administratie.
- Dashboard (Supabase) kan dezelfde tabel lezen.
