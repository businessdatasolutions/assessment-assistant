# Implementatieplan: Generieke Assessment Tool

## Context

De bestaande PROMEF Assessment Tool (`example-code.html`, ~2150 regels) is een bewezen single-file HTML-app met hardcoded criteria, scoreniveaus en cesuur voor één specifiek beoordelingsformulier. Het doel is deze om te bouwen naar een generieke app (`index.html`) waarin assessoren elk willekeurig beoordelingsformulier (PDF) kunnen uploaden, AI het formulier parst, en de app automatisch wordt ingericht. Alle bewezen patronen (kalender, tags, duo-mode, CSV, print) blijven intact.

**Aanpak**: Kopieer `example-code.html` → `index.html`, transformeer incrementeel in 5 fasen. Elke fase levert een werkende app op.

---

## Fase 1: Fundament (datamodel, config-engine, generalisatie)

Vervangt alle hardcoded constanten door een configuratie-engine. Na deze fase werkt de app identiek aan PROMEF maar leest alles uit `state.configs[]`.

### Taak 1.1 — Kopieer en hernoem
- Kopieer `example-code.html` → `index.html`
- localStorage key: `'promef_state'` → `'assessment_tool_state'` (in `saveState`, `loadState`, `fullReset`)
- Pagina-titel en nav-logo: "PROMEF" → "Assessment Tool"
- **Verificatie**: App laadt, data wordt opgeslagen onder nieuwe key

### Taak 1.2 — State uitbreiden met `configs[]` en `settings{}`
- Voeg `configs: []` en `settings: {}` toe aan het `state` object
- `loadState()`: defaults voor ontbrekende velden
- Auto-creëer een PROMEF legacy-config uit de bestaande constanten als `state.configs` leeg is
- **Verificatie**: Console: `state.configs.length === 1`, `state.configs[0].title` bevat "PROMEF"

### Taak 1.3 — Config-engine bouwen
- `getActiveConfig(groupId)` — config ophalen voor groep of actieve
- `getLevelLabel(config, level, short)` — dynamische labels
- `calculateGrade(config, total)` — dynamische scoresleutel
- `studentTotal(studentId, config)` — dynamisch criteria-count
- **Verificatie**: `calculateGrade(getActiveConfig(), 24) === 10` en `=== calculateGradeOriginal(24)`

### Taak 1.4 — `getPartner()` → `getTeammates()`
- Vervang `getPartner(student)` door `getTeammates(student)` (retourneert array)
- Update `setScore()` duo-sync: loop over array i.p.v. enkele partner
- Update `renderFeedback()`: "met X, Y" i.p.v. enkele partner
- **Verificatie**: Duo-mode werkt nog, feedback toont partnernamen

### Taak 1.5 — `senior1/senior2` → `assessors[]`
- `migrateState()`: converteer `{senior1, senior2}` naar `{assessors: [...]}`
- Update `buildGroupsLookup()`, dashboard, feedback, studenten-view
- Groep-modal: vervang 2 vaste inputs door dynamische assessor-lijst (toevoegen/verwijderen)
- `saveGroup()`: verzamel assessors uit dynamische inputs
- `DEMO_GROUPS`: `assessors: ['Docent A', 'Docent B']`
- `importCSV()`: detecteer assessor-kolommen dynamisch
- **Verificatie**: Groep bewerken toont dynamische assessor-velden, 3e assessor toevoegen werkt

### Taak 1.6 — `configId` aan groepen koppelen
- Groepen krijgen `configId` veld
- `migrateState()`: groepen zonder configId krijgen eerste config
- `DEMO_GROUPS`: `configId: 'config_promef_legacy'`
- **Verificatie**: `state.groups.every(g => g.configId)` is `true`

### Taak 1.7 — Config-engine doorvoeren in alle render-functies
Dit is de kern-refactoring. Overal waar `CRITERIA`, `SCORE_LABELS`, `SCORE_FULL`, `calculateGrade()`, `studentTotal()` worden gebruikt, vervangen door config-lookups:
- `renderAssessment()`: criteria, scoreknoppen, labels, totalen uit config
- `renderResults()`: kolommen uit config.criteria
- `renderFeedback()`: criteria, labels uit config
- `getTeamStatus()`: criteria-count uit config
- `renderDashboard()`: progress-berekening uit config
- `exportCSV()`: dynamische kolommen en bestandsnaam
- `toggleDuoMode()`: `length === 2` → `length >= 2`
- **Verificatie**: App werkt identiek aan PROMEF, maar alles leest uit `state.configs[0]`

### Taak 1.8 — Legacy constanten opruimen
- Verwijder `CRITERIA`, `SCORE_LABELS`, `SCORE_FULL`, oude `calculateGrade()`, oude `studentTotal()`, `getPartner()`
- **Verificatie**: Geen console errors, app werkt volledig

---

## Fase 2: Configuratie-UI

Voegt de twee nieuwe views toe (Configuratie + Instellingen), de configuratie-editor en onboarding.

### Taak 2.1 — Nieuwe views en navigatie-tabs
- HTML: `<section id="view-config">` en `<section id="view-settings">`
- Nav: "Configuratie" tab (eerste positie), "Instellingen" tab (na Databeheer)
- `showView()`: cases voor 'config' en 'settings'
- Placeholder render-functies
- **Verificatie**: 9 tabs zichtbaar, klikbaar, lege secties tonen

### Taak 2.2 — Welkomstscherm (onboarding)
- `init()`: als `state.configs.length === 0` → toon `view-config` i.p.v. dashboard
- `renderConfig()`: welkomstscherm met 3 opties: Upload PDF, Handmatig, Voorbeelddata
- **Verificatie**: Lege localStorage → welkomstscherm verschijnt

### Taak 2.3 — API-key modal en Instellingen-view
- Nieuwe modal `apiKeyModal` met password-input
- `showApiKeyModal(callback)`, `saveApiKey()`, `closeApiKeyModal()`
- `renderSettings()`: gemaskeerde API-key met wijzig/verwijder, configuratie-overzicht
- **Verificatie**: Key invoeren → opgeslagen in `state.settings.apiKey`, getoond als dots

### Taak 2.4 — Configuratie-editor
De grootste nieuwe UI-component:
- `renderConfigEditor(configId)`: titel, criteria-lijst (CRUD per criterium), tags per niveau, scoreniveaus (labels bewerken, toevoegen/verwijderen), cesuur-tabel, live preview
- Helper-functies: `updateConfigTitle()`, `addCriterion()`, `removeCriterion()`, `moveCriterion()`, `addTagToEditor()`, `removeTagFromEditor()`, `updateLevel()`, `addLevel()`, `removeLevel()`, `updateGradingRow()`
- Nieuwe CSS: `.config-editor`, `.config-criterion-card`, `.config-tag-editor`, `.config-preview`
- **Verificatie**: Alle criteria bewerkbaar, tags toevoegen/verwijderen, niveaus wijzigen, cesuur aanpassen

### Taak 2.5 — Handmatige configuratie
- `startManualConfig()`: maakt blanco config (2 criteria, 3 niveaus) en opent editor
- **Verificatie**: "Handmatig instellen" → editor met blanco formulier, opslaan en gebruiken werkt

### Taak 2.6 — Config CRUD functies
- `createConfig(data)`, `updateConfig()`, `deleteConfig()` (weigert bij gekoppelde groepen), `duplicateConfig()` (kopie-suffix), `exportConfig()` (JSON download), `importConfig()` (JSON laden)
- **Verificatie**: Dupliceren → "(kopie)" in titel, exporteren → JSON download, importeren → nieuwe config

### Taak 2.7 — Config-dropdown in groep-modal
- Dropdown "Beoordelingsformulier" in groep-modal, gevuld uit `state.configs`
- `saveGroup()` slaat geselecteerde configId op
- **Verificatie**: Verschillende groepen kunnen verschillende configs hebben

### Taak 2.8 — Export-view uitbreiden
- Nieuwe kaarten: "Configuratie exporteren" (met config-dropdown) en "Configuratie importeren" (file input)
- JSON backup: controleer dat API-key NIET wordt meegeexporteerd
- **Verificatie**: Config export/import roundtrip werkt, API-key niet in exports

---

## Fase 3: AI-integratie

PDF-parsing via de Claude API.

### Taak 3.1 — PDF upload UI
- Upload-zone met drag & drop in config-view
- `readFileAsBase64(file)` — FileReader → base64
- `startPDFUpload()` — checkt API-key, toont upload-zone
- Nieuwe CSS: `.upload-zone`, `.upload-zone.drag-over`, `.parsing-progress`
- **Verificatie**: PDF selecteren genereert base64 string (controleer in console)

### Taak 3.2 — Claude API aanroep
- `const AI_SYSTEM_PROMPT` — extractie-instructies (uit TDD §5.2)
- `parsePDFWithAI(pdfBase64)` — fetch naar `api.anthropic.com/v1/messages` met PDF als document block
- Headers: `x-api-key`, `anthropic-version`, `anthropic-dangerous-direct-browser-access`
- Response: extraheer JSON uit markdown code block
- **Verificatie**: Upload `test-formulier.pdf` → response bevat 6 criteria, 4 niveaus

### Taak 3.3 — Response-validatie
- `validateConfigData(data)` — controleert titel, criteria, levels, grading
- Normaliseert ontbrekende velden (criterium-IDs, lege tags-arrays)
- **Verificatie**: Geldige data → geen error; `{}` → "Titel ontbreekt"; `{title:"X",criteria:[]}` → "Geen criteria"

### Taak 3.4 — Volledige upload-flow
- `handlePDFUpload(file)` — orchestreert: base64 → loading → API → validatie → createConfig → editor
- Loading-indicator tijdens parsing
- Foutmelding met retry-optie bij API-fouten
- **Verificatie**: Upload `test-formulier.pdf` → loading → editor opent met PROMEF-config → 6 criteria, correcte scoresleutel

### Taak 3.5 — Tag-regeneratie
- `regenerateTags(configId)` — stuurt criteria naar AI, ontvangt nieuwe tags, behoudt custom tags
- "Tags regenereren (AI)" knop in config-editor
- **Verificatie**: Criterium aanpassen → tags regenereren → nieuwe tags verschijnen, custom tags blijven

---

## Fase 4: Views generaliseren

Resterende UI-aanpassingen voor volledige genericiteit.

### Taak 4.1 — Dynamische CSS scorekleuren
- `applyConfigColors(config)` — HSL-interpolatie voor N niveaus
- `interpolateHue(stops, t)` — helper voor kleurberekening
- Bij 4 niveaus: standaardkleuren; anders: dynamisch gegenereerd
- Injecteer `.s5`, `.s6` etc. als `<style>` block wanneer nodig
- **Verificatie**: Config met 5 niveaus → 5 knoppen met unieke kleuren

### Taak 4.2 — Eigen tags toevoegen tijdens assessment
- "+ tag" knop per criterium in assessment-view
- `showAddTagForm(critId)` — popup met tekst-input en niveau-keuze
- `addCustomTag(configId, critId, text, level)` — slaat op met `custom: true`
- Custom tag verschijnt bij alle teams/studenten
- **Verificatie**: Tag toevoegen → chip verschijnt → persistent na pagina-reload

### Taak 4.3 — Resultaten-view voor meerdere configs
- Groepeer resultaten per config
- Elke config-groep krijgt eigen header-rij met correcte criteria-kolommen
- **Verificatie**: 2 groepen met verschillende configs → elk juiste kolommen

### Taak 4.4 — CSV-export dynamische kolommen
- Kolomnamen uit `config.criteria`
- Bestandsnaam uit `config.title`
- **Verificatie**: Export met PROMEF → 6 kolommen; export met 3-criteria config → 3 kolommen

### Taak 4.5 — CSV-import dynamische assessoren
- `importCSV()`: detecteer "Assessor N" en "Senior N" kolommen automatisch
- Maak `assessors[]` array aan per groep
- **Verificatie**: CSV met 3 assessor-kolommen → groep met 3 assessoren

### Taak 4.6 — Dashboard assessor-weergave
- Overal `assessors.join(' & ')` i.p.v. `senior1 & senior2`
- Dashboard, kalender-print, buildGroupsLookup
- **Verificatie**: Dashboard toont alle assessor-namen

---

## Fase 5: Afwerking en testen

### Taak 5.1 — HSL-interpolatie afronden
- `interpolateHue()` met stops `[0, 35, 140, 270]` (rood → oranje → groen → paars)
- **Verificatie**: 2 niveaus → rood + paars; 5 niveaus → 5 unieke kleuren

### Taak 5.2 — Demo-data met generieke config
- `loadDemoDataWithConfig()` — maakt generieke demo-config + demo-data
- **Verificatie**: "Voorbeelddata laden" → werkende demo zonder PROMEF-verwijzingen

### Taak 5.3 — Handleiding herschrijven
- `renderGuide()` volledig herschrijven: PDF upload, AI-parsing, handmatige config, flexible assessoren, config delen
- Geen PROMEF-specifieke verwijzingen
- **Verificatie**: Handleiding beschrijft correct de huidige app

### Taak 5.4 — Foutafhandeling
- Ongeldige API-key → duidelijke melding
- Netwerkfout → retry-optie
- localStorage vol → waarschuwing
- PDF > 10MB → weigeren voor API-call
- Config verwijderd terwijl groepen eraan gekoppeld → graceful fallback
- **Verificatie**: Elke edge case handmatig testen per TDD §14.11

### Taak 5.5 — `runTests()` voor console
- Unit tests uit TDD §14.2-14.8 als `runTests()` functie
- Config-engine (C1-C16), pariteitscheck PROMEF (G1-G15), scores (S1-S12), tags (T1-T8), CRUD (CR1-CR8), CSV (V1-V6), migratie (M1-M6)
- **Verificatie**: `runTests()` in console → "Tests: N/N passed"

### Taak 5.6 — Eindvalidatie
- Upload `test-formulier.pdf` → config moet matchen met hardcoded PROMEF-constanten
- E2E-scenario's uit TDD §14.9 doorlopen (E2E-1 t/m E2E-4)
- Responsive layout testen op 3 breakpoints
- Print CSS testen voor feedback en kalender
- **Verificatie**: Alle tests slagen, app is volledig generiek

---

## Kritieke bestanden

| Bestand | Rol |
|---------|-----|
| `example-code.html` | Bronbestand om te kopiëren en transformeren |
| `index.html` | Output — de generieke assessment tool |
| `TDD.md` | Technisch ontwerp met exacte code voor config-engine, AI-integratie, testplan |
| `PRD-generic.md` | Producteisen en acceptatiecriteria |
| `test-formulier.pdf` | Gouden referentie-PDF voor AI-parsing validatie |
| `.env` | Anthropic API-key voor testen van AI-integratie |
