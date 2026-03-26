# PRD: Assessment Tool (Generiek)

## Context

Docenten en assessoren in het hoger onderwijs gebruiken beoordelingsformulieren om studentprestaties te scoren tijdens assessments, presentaties en gesprekken. Elk vak, opleiding of toetsmoment heeft een eigen formulier met specifieke criteria, scoreniveaus en cesuur. Het probleem: elke assessment-app is ofwel te generiek (spreadsheet) ofwel te specifiek (hardcoded voor een formulier).

**Doel**: Een lokale browser-app (single HTML file) waarin een assessor een beoordelingsformulier (PDF) uploadt, waarna AI het formulier uitleest en de app automatisch inricht. De assessor hoeft alleen te scoren — de app regelt de rest: planning, dataverzameling, verwerking en terugkoppeling.

**Kernprincipe**: Van PDF naar werkende assessment in minder dan 2 minuten.

---

## Gebruikersflow

```
┌─────────────────────────────────────────────────────┐
│  1. OPZETTEN                                        │
│  ┌───────────┐    ┌──────────┐    ┌──────────────┐  │
│  │ Upload PDF │───▶│ AI parst │───▶│ Review/edit  │  │
│  │ formulier  │    │ formulier│    │ configuratie │  │
│  └───────────┘    └──────────┘    └──────────────┘  │
│                                          │          │
│  2. VOORBEREIDEN                         ▼          │
│  ┌───────────┐    ┌──────────┐    ┌──────────────┐  │
│  │ Groepen & │───▶│ Studenten│───▶│  Kalender    │  │
│  │ sessies   │    │ & teams  │    │  inplannen   │  │
│  └───────────┘    └──────────┘    └──────────────┘  │
│                                          │          │
│  3. UITVOEREN                            ▼          │
│  ┌──────────────────────────────────────────────┐   │
│  │  Assessment-view: scoren, observeren, noteren │   │
│  └──────────────────────────────────────────────┘   │
│                                          │          │
│  4. AFRONDEN                             ▼          │
│  ┌───────────┐    ┌──────────┐    ┌──────────────┐  │
│  │ Resultaten│───▶│ Feedback │───▶│   Export      │  │
│  │ overzicht │    │ per stud.│    │  CSV/PDF/JSON │  │
│  └───────────┘    └──────────┘    └──────────────┘  │
└─────────────────────────────────────────────────────┘
```

---

## 1. Assessment-configuratie

### 1.1 PDF Upload & AI-parsing

De assessor uploadt een beoordelingsformulier als PDF. De app stuurt het document naar de Claude API (vanuit de browser, met de API-key van de gebruiker) en ontvangt een gestructureerde assessment-configuratie.

**API-key beheer**:
- Bij eerste gebruik vraagt de app om een Anthropic API-key
- Key wordt opgeslagen in localStorage (niet in de config-export)
- Optie om key te verwijderen/wijzigen via Instellingen
- Duidelijke uitleg dat de key alleen lokaal wordt opgeslagen en alleen voor PDF-parsing wordt gebruikt

**Wat de AI extraheert uit het PDF**:
- Assessmenttitel (bijv. "PROMEF eindassessment F-cluster")
- Criteria: nummer, naam, beschrijving
- Scoreniveaus: labels en puntwaarden (bijv. Onder niveau=1, Op niveau=2, etc.)
- Beschrijving per scoreniveau (als aanwezig)
- Scoresleutel/cesuur: mapping van totaalpunten naar cijfer
- Zak/slaag-regel (bijv. "alle onderdelen minimaal Op niveau")
- Observatie-indicatoren per criterium per niveau (als aanwezig in het formulier)
- Metadata: doelgroep, toetsvorm, bijzonderheden

**AI-output formaat** (intern JSON):
```json
{
  "title": "Assessment titel",
  "criteria": [
    {
      "id": 1,
      "name": "Criterium naam",
      "desc": "Beschrijving",
      "tags": [
        { "text": "indicator tekst", "level": 1 }
      ]
    }
  ],
  "levels": {
    "count": 4,
    "labels": { "1": "Onder niveau", "2": "Op niveau", "...": "..." },
    "shortLabels": { "1": "Onder", "2": "Op", "...": "..." },
    "colors": { "1": "red", "2": "orange", "3": "green", "4": "purple" }
  },
  "grading": {
    "scale": [{ "minPoints": 24, "grade": 10 }, { "...": "..." }],
    "passRule": "Alle criteria minimaal niveau 2",
    "passThreshold": 2
  }
}
```

**Als het formulier geen observatie-indicatoren bevat**: de AI genereert suggesties voor 3-4 observatie-tags per criterium per niveau, gebaseerd op de criteriumbeschrijving. De assessor kan deze aanpassen of verwijderen.

### 1.2 Configuratie-editor (review & aanpassen)

Na AI-parsing toont de app een bewerkbare preview van de configuratie:

- **Assessmenttitel**: bewerkbaar tekstveld
- **Criteria**: lijst met per criterium:
  - Naam en beschrijving (bewerkbaar)
  - Observatie-tags per niveau: toevoegen, verwijderen, herformuleren
  - Criterium verwijderen of toevoegen
  - Volgorde wijzigen (drag & drop of pijltjes)
- **Scoreniveaus**: labels en kleuren aanpassen, niveaus toevoegen/verwijderen
- **Cesuur**: scoresleutel bewerkbaar (puntgrenzen en bijbehorende cijfers)
- **Zak/slaag-regel**: drempel aanpassen of uitschakelen
- **"Opnieuw genereren"**: laat AI de tags/indicatoren opnieuw genereren (bijv. na het aanpassen van criteria)

Visuele preview: naast de editor een live voorbeeld van hoe een criterium er in de assessment-view uit zal zien (scoreknoppen + tags).

### 1.3 Handmatige configuratie (zonder AI)

Alternatief voor assessoren zonder API-key:
- Blanco configuratie aanmaken met zelfgekozen aantal criteria en niveaus
- Alle velden handmatig invullen
- Dezelfde editor-UI als na AI-parsing

### 1.4 Configuratiebeheer

- **Opslaan**: configuratie krijgt een naam en wordt opgeslagen in localStorage
- **Meerdere configuraties**: een assessor kan meerdere beoordelingsformulieren beheren
- **Actieve configuratie**: per groep wordt een configuratie gekoppeld (bij groep aanmaken)
- **Export/import**: configuratie als JSON exporteren/importeren (zonder API-key) — voor delen met collega's
- **Dupliceren**: bestaande configuratie kopieren als startpunt voor een variant

---

## 2. Gegevens

### Datamodel

```
state.configs:  [{ id, title, criteria, levels, grading, createdAt, updatedAt }]
state.groups:   [{ id, name, configId, assessors[], date, startTime, endTime, slotDuration, breaks }]
state.teams:    [{ id, groupId, num }]
state.students: [{ id, name, groupId, teamId }]
state.scores:   { studentId: { critId: levelValue } }
state.notes:    { studentId: { critId: "tekst met checkmark tag-regels" } }
state.duoMode:  { teamKey: { critId: bool } }
state.slots:    { teamKey: "HH:MM" }
state.settings: { apiKey, activeConfigId, ... }
```

**Wijzigingen t.o.v. PROMEF-app**:
- `state.configs` is nieuw — bevat assessment-configuraties
- `state.groups` krijgt `configId` (welk formulier) en `assessors[]` (flexibel aantal assessoren i.p.v. vast senior1/senior2)
- `state.settings` is nieuw — API-key en app-voorkeuren
- `CRITERIA` constante verdwijnt — criteria komen uit de actieve config
- `SCORE_LABELS`, `SCORE_FULL` constanten verdwijnen — komen uit config.levels
- `calculateGrade()` wordt dynamisch op basis van config.grading.scale

### Groep

- Naam (vrij tekstveld)
- **Configuratie**: welk beoordelingsformulier wordt gebruikt (dropdown)
- **Assessoren**: 1 of meer namen (dynamisch, geen vast aantal)
- Datum, starttijd, eindtijd, slotduur, pauzes

### Team

- Teamnummer, gekoppeld aan groep
- **1-N studenten** per team (geen vaste duo-aanname)

### Student

- Naam, gekoppeld aan groep en team

---

## 3. Functionele eisen

### 3.1 Onboarding (eerste gebruik)

Bij eerste bezoek (geen data in localStorage):
1. Welkomstscherm met twee opties:
   - **"Upload beoordelingsformulier"** — PDF-upload flow (vraagt API-key als die ontbreekt)
   - **"Handmatig instellen"** — blanco configuratie-editor
2. Na configuratie: door naar Studenten-view om groepen en studenten aan te maken
3. Optie om demodata te laden (generiek voorbeeld)

### 3.2 Planning & Voorbereiding

**Kalenderoverzicht (dashboard)**:
- Dagweergave met tijdslots dynamisch gegenereerd per groep op basis van start-/eindtijd en slotduur (configureerbaar per groep, veelvouden van 15 min)
- Teams als kaarten met studentnamen en beoordelingsstatus (grijs/oranje/groen)
- Drag & drop voor herschikken van teams
- Lege slots klikbaar om pauze toe te voegen (hover toont "+ pauze")
- Pauze-slots als gedimde cellen met verwijderknop
- Klik op teamblok opent assessment voor dat team
- Desktop: groepen als kolommen naast elkaar
- Mobiel (<=768px): groepen gestapeld, elk met eigen grid

**Beheer (Studenten-view)**:
- Groepen aanmaken met **configuratie-keuze** (welk formulier), assessoren, datum en tijden
- **Flexibel aantal assessoren** per groep (toevoegen/verwijderen)
- Teams met **variabel aantal studenten** (1, 2, 3, ...)
- Teams bulk-aanmaken (meerdere tegelijk)
- Cascade-verwijdering: groep verwijderen wist ook teams, studenten, scores en notities
- CSV-import met auto-detectie scheidingsteken (komma of puntkomma voor Nederlands Excel)

**CSV-import formaat**:
```
Naam,Groep,Team,Assessor 1,Assessor 2,...,Datumtijd
```
- Assessor-kolommen worden dynamisch herkend
- Configuratie moet apart worden gekoppeld aan geimporteerde groepen

### 3.3 Dataverzameling tijdens assessment

**Dit is de kritieke fase — UX geoptimaliseerd voor minimale afleiding.**

Per team-gesprek:
- **Header**: teamnummer, namen studenten, groep, assessoren, datum/tijd, prev/next team navigatie
- **Duo-/groepsscore toggle**: per criterium instellen of de score gedeeld is voor het hele team of individueel per student
- **Per student (of team), per criterium**:
  - Scoreknoppen: aantal en labels komen uit de configuratie. Klik = selecteer, opnieuw klik = deselecteer. Kleurgecodeerd per niveau.
  - Observatie-tags: chips gegroepeerd per scoreniveau, uit de configuratie
  - Notitieveld: vrije tekst + tag-markers (checkmark prefix), auto-groeiend textarea

**Tag-interactie**:
- Klik op tag-chip — picker popup met studentnamen — klik = toewijzen
- Sleep tag naar notitieveld — tag wordt ingevoegd
- Tags opgeslagen als `checkmark tagText` markers in notities (geen apart tag-state)
- Gebruikte tags tonen initiaal-badges van toegewezen studenten
- Toggle-off: tag-marker verwijderd, omliggende tekst blijft behouden
- **Eigen tags toevoegen**: per criterium een "+ tag" knop waarmee de assessor ter plekke een nieuwe observatie-tag aanmaakt. De assessor typt een korte observatie, kiest optioneel een niveau (voor kleurcodering), en de tag wordt direct als chip beschikbaar. Zelfgemaakte tags worden opgeslagen in de configuratie (`tag.custom: true`) zodat ze ook bij volgende studenten/teams verschijnen.

**Notitievelden**:
- Per student per criterium een auto-groeiend tekstveld
- Vrije tekst en tag-regels (checkmark prefix) worden gecombineerd
- In de feedback-view worden tags en vrije tekst apart getoond
- Alles auto-saved naar localStorage

### 3.4 Verwerking

- Automatische totaalscore per student (som van alle criteria)
- Cijferberekening op basis van configuratie-scoresleutel
- Zak/slaag-bepaling op basis van configuratie-drempel
- Overzicht per groep met alle studenten
- Waarschuwing bij scores onder de drempel

### 3.5 Feedback

Per student:
- Alle metadata (groep, team, teamgenoten, assessoren, datum/tijd)
- Per criterium: naam, beschrijving, score (kleur + label uit config), observatie-tags apart, vrije notities apart
- Totaal, cijfer, status (voldoende/onvoldoende/incompleet)
- Print/PDF via browser (Ctrl+P / Cmd+P)
- Layout past zich aan het aantal criteria aan

### 3.6 Export & Databeheer

- **CSV-export**: alle studenten met scores per criterium (kolomnamen uit config), totaal, cijfer, status. Bestandsnaam gebaseerd op configuratietitel.
- **JSON-backup**: volledige state inclusief configuratie(s)
- **JSON-restore**: backup herstellen
- **Configuratie-export**: alleen de assessment-configuratie als JSON (voor delen met collega's)
- **Configuratie-import**: JSON-bestand laden als nieuwe configuratie
- **Scores wissen**: scores/notities wissen, studenten behouden
- **Demodata laden**: generiek voorbeeld
- **Volledige reset**: alles wissen

### 3.7 Instellingen

- API-key beheer (opslaan, wijzigen, verwijderen)
- Configuraties beheren (overzicht, selecteren, verwijderen)
- App-naam/titel (optioneel, voor in de header)

### 3.8 Handleiding

Dynamische handleiding die:
- Uitlegt hoe de app werkt (onafhankelijk van specifiek formulier)
- PDF-upload en AI-parsing stap voor stap beschrijft
- Handmatige configuratie uitlegt
- Assessment-workflow beschrijft (scoren, tags, notities)
- Export en backup uitlegt
- Privacy-richtlijnen bevat (localStorage, API-key, geen server)

---

## 4. Niet-functionele eisen

- **Single HTML file**: geen server, geen externe dependencies, geen build step
- **Offline na configuratie**: na het parsen van het PDF werkt de app volledig offline. De API-call is alleen nodig bij het uploaden van een nieuw formulier of het genereren van tags.
- **localStorage**: alle data persistent in de browser
- **Responsive**: laptop-first, bruikbaar op tablet en mobiel. Breakpoints op 768px (tablet) en 480px (phone). Hamburger-menu op <=768px. Touch targets minimaal 44px.
- **Klik-first**: grote knoppen, duidelijke hover/focus states
- **Snel**: instant laden, geen loading states (behalve tijdens AI-parsing)
- **Privacy**: API-key alleen lokaal opgeslagen, PDF-inhoud wordt alleen naar de Claude API gestuurd voor parsing (niet opgeslagen door de app)
- **Alle UI-tekst in het Nederlands**

---

## 5. Technische aanpak

- Single `index.html` met inline `<style>` en `<script>`
- Vanilla JS, geen frameworks
- CSS Grid/Flexbox voor responsive layout
- CSS custom properties voor theming en scoreniveau-kleuren
- localStorage API voor persistentie
- **Anthropic Claude API** (messages endpoint) voor PDF-parsing:
  - `fetch()` vanuit de browser naar `https://api.anthropic.com/v1/messages`
  - PDF als base64-encoded document in het request
  - Gestructureerd JSON-antwoord via system prompt met extractie-instructies
- Blob API + download voor exports (CSV, JSON)
- Print CSS voor PDF-export via browser print dialog
- FileReader API voor PDF- en CSV-upload

### AI-parsing prompt (samengevat)

Het system prompt instrueert Claude om uit het PDF-document te extraheren:
1. Titel van het assessment
2. Alle criteria met nummering, naam en beschrijving
3. Scoreniveaus met labels en puntwaarden
4. Scoresleutel (punten naar cijfer mapping)
5. Zak/slaag-regels
6. Eventuele observatie-indicatoren per criterium per niveau
7. Als er geen indicatoren in het formulier staan: genereer 3-4 observatietags per criterium per niveau als suggestie

Output: strikt JSON in het gedefinieerde config-formaat.

---

## 6. Schermen / Views

1. **Configuratie** — PDF upload, AI-parsing, configuratie-editor
2. **Overzicht** (Dashboard) — kalender, planning, voortgang
3. **Assessment** — scoren tijdens gesprekken (kernview)
4. **Resultaten** — overzichtstabel scores en cijfers
5. **Feedback** — per-student rapport
6. **Studenten** — beheer groepen, teams, studenten
7. **Databeheer** — export, backup, import, reset
8. **Instellingen** — API-key, configuraties, voorkeuren
9. **Handleiding** — stapsgewijze uitleg

---

## 7. Bewezen patronen uit de PROMEF-app (hergebruik-referentie)

De PROMEF Assessment Tool (`STRAOR-ASS/index.html`) is de directe voorloper van deze generieke app. Alle onderstaande patronen zijn in de praktijk getest en bewezen. Ze moeten 1-op-1 worden overgenomen tenzij anders aangegeven.

### 7.1 Architectuur & Structuur

- **Single HTML file** met inline `<style>` en `<script>` — geen externe bestanden, geen CDN, geen build tools
- **Vanilla JS** zonder frameworks — voor snelheid en eenvoud
- **CSS custom properties** voor theming: `--score1` t/m `--score4` met `-bg` en `-text` varianten per scoreniveau, `--touch-min: 44px` voor touch targets
- **View-systeem**: view-secties in de HTML (`view-dashboard`, `view-assessment`, etc.), getoggeld via `.active` class. `showView(viewName)` roept de bijbehorende render-functie aan
- **Render-functies**: `renderDashboard()`, `renderAssessment()`, `renderResults()`, `renderFeedback()`, `renderStudents()`, `renderExport()`, `renderGuide()` — elk schrijft `innerHTML` van zijn view-sectie. Uitbreiden met `renderConfig()` en `renderSettings()`
- **Modal-systeem**: overlay-modals voor CRUD-operaties (groep, team, student). Herbruikbaar voor configuratie-editor

### 7.2 State Management & Persistentie

- **Enkele globale `state` object** met genormaliseerde entiteiten (groups, teams, students, scores, notes, duoMode, slots)
- **Auto-save**: `saveState()` wordt aangeroepen bij elke wijziging — nooit handmatig opslaan
- **localStorage** onder een key — aanpassen naar generieke key (bijv. `assessment_state`)
- **`loadState()`** bij opstarten met fallback naar defaults
- **`migrateState()`**: auto-migratie van oude formaten. Cruciaal patroon — behouden voor toekomstige schema-wijzigingen
- **ID-strategie**: Group ID = naam-string, Team ID = `teamKey(groupId, num)`, Student ID = auto-increment integer

### 7.3 Kalender & Planning (overnemen zoals is)

- **Tijdslots dynamisch gegenereerd** per groep op basis van `startTime`, `endTime`, `slotDuration` (veelvouden van 15 min)
- **`getGroupTimeSlots(group)`**: genereert alle slots voor een groep
- **`getAllTimeSlots()`**: union van alle groepen voor desktop-weergave
- **Resize listener** die dashboard opnieuw rendert bij breakpoint-overgang
- **Drag & drop** voor teams verslepen naar ander tijdslot
- **Lege slots klikbaar** — pauze toevoegen (hover toont "+ pauze")
- **Pauze-slots**: gedimde cellen met verwijderknop
- **Status-kleuren per teamblok**: grijs (niet beoordeeld), oranje (deels), groen (volledig)
- **Klik op teamblok** — opent assessment
- **`buildGroupsLookup()`**: nested object voor snelle kalender-rendering
- **`isBreakSlot(group, slot)`**, **`getTeamStatus(group, teamNum)`**: helper-functies

### 7.4 Assessment-view (kernview — overnemen en generaliseren)

- **Header**: teamnummer, studentnamen, groep, datum/tijd, prev/next team navigatie
- **Per criterium**:
  - Titel + beschrijving
  - Duo-/groepsscore toggle (checkbox)
  - Scoreknoppen: grote klikbare knoppen, kleurgecodeerd. Klik = selecteer, opnieuw klik = deselecteer
  - Als duo/groepsmode: een rij knoppen voor heel team
  - Als individueel: side-by-side per student (generaliseren naar N studenten)
  - Tags gegroepeerd per niveau met kleurcodering
  - Notitieveld per student: auto-groeiend textarea, drag-drop target
- **Footer**: per student totaal, cijfer, status, reset-knop
- **`setScore(studentId, critId, level)`**: toggle-logica + duo-sync naar alle teamleden
- **`openAssessment(groupId, teamNum)`**: navigeert naar assessment voor specifiek team

### 7.5 Tag/Observatie-systeem (overnemen en uitbreiden)

- **Tags als checkmark-markers in notities** — geen apart tag-state. Tags en vrije tekst leven samen in een veld
- **`isTagInNote(studentId, critId, tagText)`**: check via `includes`
- **`insertTagInNote()`**: toggle tag aan/uit in notitie (append of verwijder marker, behoud omliggende tekst)
- **`getTagAssignees(critId, tagText)`**: welke studenten hebben deze tag — voor badge-weergave op chips
- **`showTagPicker(event, critId, tagText)`**: popup met studentnamen, klik = toewijzen
- **`handleNoteDrop()`**: drag-to-notes interactie
- **Tag-chips**: kleurgecodeerd per niveau, tonen initiaal-badges van toegewezen studenten
- **Matching via `includes`**: tags kunnen overal in de tekst staan, met vrije tekst ervoor/erna
- **Nieuw — Eigen tags toevoegen**: per criterium een "+ tag" knop in de assessment-view. De assessor typt een korte observatie, kiest optioneel een niveau (voor kleurcodering), en de tag wordt direct als chip beschikbaar. Zelfgemaakte tags worden opgeslagen in de configuratie (`tag.custom: true`) zodat ze ook bij volgende studenten/teams verschijnen.

### 7.6 Duo/Groepsscore (generaliseren van 2 naar N)

- **Per criterium toggle**: `isDuoMode(group, team, critId)`
- **Bij inschakelen**: sync scores van alle teamleden naar dezelfde waarde
- **Bij scoren in duo-mode**: `setScore()` past automatisch score toe op alle teamleden
- **UI**: een set scoreknoppen voor het team i.p.v. per student
- **Generalisatie**: huidige code checkt `teammates.length === 2` — aanpassen naar `teammates.length >= 1`

### 7.7 Resultaten-view (overnemen en generaliseren)

- **Tabel**: Student | Team | [criterium-kolommen] | Totaal | Cijfer | Status
- **Groepering per groep** (header-rij per groep)
- **Waarschuwingscellen**: score onder drempel — `.warn` class (rood)
- **Status-logica**: complete + geen under — "Voldoende" (groen), complete + under — "Onvoldoende" (rood), incompleet — "-"
- **`studentTotal(studentId)`**: retourneert `{ total, count, complete, hasUnder, grade }` — generaliseren: `complete` check dynamisch op basis van aantal criteria in config

### 7.8 Feedback-view (overnemen en generaliseren)

- **Dropdown**: groep — studentlijst selector
- **Studentmetadata**: groep, team, teamgenoten, assessoren, datum/tijd
- **Per criterium**: naam, beschrijving, score-badge (kleurgecodeerd), observatie-tags apart, vrije notities apart
- **Tag-parsing**: notitieregels filteren op checkmark-prefix voor tags, rest = vrije tekst
- **Samenvattingsbox**: status, totaal, cijfer
- **Print/PDF**: via `window.print()` met Print CSS. Print-knop per student.

### 7.9 Studenten-beheer (overnemen en uitbreiden)

- **Groep-modal**: naam, assessoren, datum (date picker), start/eindtijd (time picker), slotduur (dropdown veelvouden van 15 min)
- **Team bulk-aanmaken**: meerdere teams tegelijk toevoegen
- **Student-modal**: naam, groep-dropdown, team-dropdown (gefilterd op geselecteerde groep)
- **Cascade-verwijdering**: groep verwijderen wist teams, studenten, scores, notities
- **Uitbreiding**: groep-modal krijgt configId-dropdown

### 7.10 CSV-systeem (overnemen en generaliseren)

- **`parseCSV(text)`**: auto-detecteert delimiter (`;` voor Nederlands Excel, `,` voor standaard)
- **`importCSV(text)`**: parst headers, maakt automatisch groepen en teams aan uit rijen
- **Verwachte kolommen**: Naam, Groep, Team + optioneel assessor-kolommen en Datumtijd
- **`formatGroupDatetime(group)`**: Nederlandse datumnotatie "di 7 apr 09:00-14:30"
- **`parseDatetimeString(str)`**: parseert Nederlandse datum/tijd-strings
- **Export CSV**: dynamische kolomnamen uit configuratie
- **Bestandsnamen**: gebaseerd op configuratietitel (niet hardcoded)

### 7.11 Responsive Design (overnemen zoals is)

- **Breakpoints**: 768px (tablet), 480px (phone)
- **Hamburger-menu**: `.nav-tabs-row` wrapper met toggle op <=768px, vervangt inline tabs door dropdown
- **`toggleMenu()`**: hamburger open/dicht
- **Kalender**: desktop = kolommen naast elkaar, mobiel = gestapeld per groep
- **Touch targets**: `--touch-min: 44px` voor alle interactieve elementen
- **Modals**: bottom-sheet op kleine schermen
- **Tabellen**: horizontaal scrollbaar op smal scherm

### 7.12 CSS Design Tokens (overnemen en uitbreiden)

```css
/* Scoreniveau-kleuren — defaults, dynamisch per config */
--score1: #ef4444;        --score1-bg: #fef2f2;    --score1-text: #991b1b;  /* Laagste */
--score2: #f59e0b;        --score2-bg: #fffbeb;    --score2-text: #92400e;  /* Drempel */
--score3: #22c55e;        --score3-bg: #f0fdf4;    --score3-text: #166534;  /* Boven */
--score4: #8b5cf6;        --score4-bg: #f5f3ff;    --score4-text: #5b21b6;  /* Excellent */
--touch-min: 44px;
```

Bij configuraties met meer of minder dan 4 niveaus: interpoleer kleuren tussen rood (laagste) en paars (hoogste) via HSL.

### 7.13 Helper-functies (overnemen)

| Functie | Doel | Aanpassing |
|---------|------|------------|
| `getGroup(groupId)` | Vind groep op ID | Geen |
| `getTeamById(teamId)` | Vind team op volledige ID | Geen |
| `getTeamStudents(group, teamNum)` | Studenten in een team | Geen |
| `getScore(studentId, critId)` | Score ophalen | Geen |
| `setScore(studentId, critId, level)` | Score zetten + duo-sync | Generaliseer duo naar N |
| `getNote(studentId, critId)` | Notitie ophalen | Geen |
| `setNote(studentId, critId, text)` | Notitie opslaan | Geen |
| `getPartner(student)` | Andere student in team | Veralgemeniseer naar `getTeammates(student)` |
| `studentTotal(studentId)` | Totaal, cijfer, status | Dynamisch op basis van config |
| `teamKey(groupId, num)` | Team-key genereren | Geen |
| `formatGroupDatetime(group)` | Nederlandse datumformat | Geen |
| `parseCSV(text)` | CSV parser met NL-support | Geen |
| `downloadFile(content, name, type)` | Blob download trigger | Geen |
| `saveState()` / `loadState()` | localStorage persistentie | Key aanpassen |
| `showView(viewName)` | View navigatie | Uitbreiden met nieuwe views |
| `toggleMenu()` | Hamburger menu | Geen |

### 7.14 Overige bewezen patronen

- **Geen loading states** behalve voor AI-parsing — alles is instant
- **Auto-save op elke interactie** — gebruiker hoeft nooit te "opslaan"
- **Alle UI-tekst in het Nederlands** — behouden
- **Print CSS** voor PDF-export via browser print dialog
- **Demo-data** als optie bij onboarding (niet standaard geladen)
- **Volledige reset** als noodknop in Databeheer

---

## 8. Verificatie / Testen

1. Open `index.html` in browser — welkomstscherm verschijnt
2. Voer API-key in, upload een beoordelingsformulier-PDF
3. Controleer of AI correct criteria, niveaus en scoresleutel extraheert
4. Pas de configuratie aan in de editor (tag toevoegen, criterium hernoemen)
5. Sla configuratie op, maak een groep aan en koppel de configuratie
6. Voeg teams en studenten toe (test ook met 1 en 3+ studenten per team)
7. Plan teams in het kalenderoverzicht
8. Open assessment en score alle criteria voor alle teamleden
9. Test observatie-tags (klik en drag-to-notes)
10. Test eigen tag toevoegen tijdens assessment
11. Test duo/groepsscore toggle
12. Controleer resultaten: totaal, cijfer, zak/slaag correct berekend
13. Controleer feedback-view: criteria, scores, tags en notities correct weergegeven
14. Exporteer CSV — controleer dat kolomnamen uit configuratie komen
15. Test JSON backup/restore (inclusief configuratie)
16. Test configuratie-export en -import (deel met collega)
17. Test handmatige configuratie (zonder AI)
18. Sluit browser, heropen — data bewaard (localStorage)
19. Test met een tweede, ander beoordelingsformulier om genericiteit te verifieren
