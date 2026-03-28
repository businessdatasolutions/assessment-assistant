# Technical Design Document: Assessment Tool (Generiek)

## 1. Overzicht

Dit document beschrijft het technisch ontwerp voor de generieke Assessment Tool — een single-file HTML-applicatie waarmee assessoren een beoordelingsformulier (PDF) uploaden, AI het formulier parst tot een werkende configuratie, en de assessor vervolgens kan scoren, observeren en feedback genereren.

**Voorloper**: De PROMEF Assessment Tool (`example-code.html`, ~2150 regels) dient als bewezen referentie-implementatie. De generieke versie breidt deze uit met dynamische configuraties, AI-parsing en multi-formulier ondersteuning.

**Technologie**: Single `index.html` met inline `<style>` en `<script>`. Vanilla JS, geen frameworks, geen build tools, geen externe dependencies.

---

## 2. Architectuur

### 2.1 High-level structuur

```
index.html
├── <style>           CSS (~400 regels)
│   ├── Base / Design tokens
│   ├── Component styles (nav, cards, calendar, assessment, etc.)
│   ├── Responsive breakpoints (768px, 480px)
│   └── Print stylesheet
│
├── <body>            HTML (~150 regels)
│   ├── <nav>         Navigatie met tabs + hamburger
│   ├── <main>        View-secties (empty containers, filled by JS)
│   └── Modals        Overlay modals voor CRUD + configuratie-editor
│
└── <script>          JavaScript (~2500+ regels)
    ├── Config & Constants
    ├── State Management & Persistentie
    ├── AI/API Integration (NIEUW)
    ├── Configuration Engine (NIEUW)
    ├── Navigation & View System
    ├── Render Functions (per view)
    ├── CRUD Operations
    ├── CSV Parser & Import/Export
    ├── Calendar & Planning Logic
    ├── Assessment Logic (scores, tags, notes)
    ├── Grade Calculation
    └── Responsive Helpers
```

### 2.2 View-systeem

Views zijn `<section>` elementen in `<main>`, getoggeld via `.active` class. Elke view heeft een bijbehorende render-functie die `innerHTML` schrijft.

| View ID | Render-functie | Beschrijving | Status |
|---------|---------------|--------------|--------|
| `view-config` | `renderConfig()` | PDF upload, AI-parsing, configuratie-editor | **NIEUW** |
| `view-dashboard` | `renderDashboard()` | Kalender met tijdslots en teamblokken | Bestaand |
| `view-assessment` | `renderAssessment()` | Scoren, tags, notities per team | Bestaand (generaliseren) |
| `view-results` | `renderResults()` | Overzichtstabel scores en cijfers | Bestaand (generaliseren) |
| `view-feedback` | `renderFeedback()` | Per-student feedbackrapport | Bestaand (generaliseren) |
| `view-students` | `renderStudents()` | Groepen, teams, studenten beheer | Bestaand (uitbreiden) |
| `view-export` | `renderExport()` | Export, backup, import, reset | Bestaand (uitbreiden) |
| `view-settings` | `renderSettings()` | API-key, configuratiebeheer | **NIEUW** |
| `view-guide` | `renderGuide()` | Handleiding | Bestaand (herschrijven) |

Navigatie via `showView(viewName)`:

```javascript
function showView(view) {
  currentView = view;
  document.querySelectorAll('.view').forEach(v => v.classList.remove('active'));
  document.querySelectorAll('nav .tab').forEach(t => t.classList.remove('active'));
  document.getElementById('view-' + view)?.classList.add('active');
  document.querySelector(`nav .tab[data-view="${view}"]`)?.classList.add('active');
  document.getElementById('navMenu')?.classList.remove('open');

  const renderers = {
    config: renderConfig, dashboard: renderDashboard,
    assessment: renderAssessment, results: renderResults,
    feedback: renderFeedback, students: renderStudents,
    export: renderExport, settings: renderSettings, guide: renderGuide
  };
  renderers[view]?.();
}
```

---

## 3. Datamodel

### 3.1 Globale state

Alle applicatiedata leeft in een enkel `state` object dat geserialiseerd wordt naar localStorage.

```javascript
let state = {
  configs:   [],   // Assessment-configuraties (formulieren)
  groups:    [],   // Groepen (assessmentdagen)
  teams:     [],   // Teams (gekoppeld aan groep)
  students:  [],   // Studenten (gekoppeld aan groep + team)
  scores:    {},   // { studentId: { critId: levelValue } }
  notes:     {},   // { studentId: { critId: "tekst" } }
  duoMode:   {},   // { teamKey: { critId: bool } }
  slots:     {},   // { teamKey: "HH:MM" }
  settings:  {}    // { apiKey, activeConfigId, ... }
};
```

### 3.2 Entity-schema's

#### Config

```javascript
{
  id: "config_1679...",           // Uniek ID (timestamp-based)
  title: "PROMEF eindassessment", // Titel van het assessment
  criteria: [
    {
      id: 1,
      name: "Criterium naam",
      desc: "Beschrijving van het criterium",
      tags: [
        { text: "observatie tekst", level: 1, custom: false },
        { text: "eigen observatie", level: 2, custom: true }
      ]
    }
  ],
  levels: {
    count: 4,
    labels:      { 1: "Onder niveau", 2: "Op niveau", 3: "Boven niveau", 4: "Excellent" },
    shortLabels: { 1: "Onder", 2: "Op", 3: "Boven", 4: "Excellent" },
    colors:      { 1: "red", 2: "orange", 3: "green", 4: "purple" }
  },
  grading: {
    scale: [
      { minPoints: 24, grade: 10 },
      { minPoints: 20, grade: 9 },
      { minPoints: 16, grade: 8 },
      { minPoints: 14, grade: 7 },
      { minPoints: 12, grade: 6 },
      { minPoints: 10, grade: 5 }
    ],
    defaultGrade: 4,
    passRule: "Alle criteria minimaal niveau 2",
    passThreshold: 2
  },
  createdAt: "2026-03-26T10:00:00Z",
  updatedAt: "2026-03-26T10:00:00Z"
}
```

#### Group

```javascript
{
  id: "BKN-F01",              // Naam = ID
  name: "BKN-F01",
  configId: "config_1679...",  // NIEUW: welk formulier
  assessors: ["Docent A", "Docent B"],  // NIEUW: flexibel aantal (was: senior1/senior2)
  date: "2026-04-14",
  startTime: "09:00",
  endTime: "14:30",
  slotDuration: 30,
  breaks: ["12:00"]
}
```

**Migratie t.o.v. PROMEF**: `senior1`/`senior2` velden worden samengevoegd tot `assessors[]` array. Bestaande data wordt gemigreerd via `migrateState()`.

#### Team

```javascript
{
  id: "BKN-F01-1",      // teamKey(groupId, num)
  groupId: "BKN-F01",
  num: 1
}
```

#### Student

```javascript
{
  id: 1,                  // Auto-increment integer
  name: "Anna",
  groupId: "BKN-F01",
  teamId: "BKN-F01-1"
}
```

#### Settings

```javascript
{
  apiKey: "sk-ant-...",         // Anthropic API key (alleen localStorage)
  activeConfigId: "config_...", // Laatst geselecteerde configuratie
  appTitle: ""                  // Optionele app-titel
}
```

### 3.3 ID-strategie

| Entity | ID-formaat | Voorbeeld |
|--------|-----------|-----------|
| Config | `config_` + timestamp | `config_1679832000000` |
| Group | Naam (string) | `BKN-F01` |
| Team | `teamKey(groupId, num)` | `BKN-F01-1` |
| Student | Auto-increment integer | `1`, `2`, `3` |

### 3.4 Persistentie

```javascript
const STORAGE_KEY = 'assessment_tool_state';

function saveState() {
  localStorage.setItem(STORAGE_KEY, JSON.stringify(state));
}

function loadState() {
  const saved = localStorage.getItem(STORAGE_KEY);
  if (saved) {
    try { state = JSON.parse(saved); } catch(e) { console.error('Load error', e); }
  }
  // Defaults voor ontbrekende velden
  if (!state.configs) state.configs = [];
  if (!state.groups) state.groups = [];
  if (!state.teams) state.teams = [];
  if (!state.students) state.students = [];
  if (!state.scores) state.scores = {};
  if (!state.notes) state.notes = {};
  if (!state.duoMode) state.duoMode = {};
  if (!state.slots) state.slots = {};
  if (!state.settings) state.settings = {};
  migrateState();
}
```

### 3.5 Migratie

`migrateState()` handelt automatisch schema-wijzigingen af:

1. **PROMEF legacy** (`senior1`/`senior2` → `assessors[]`): detecteert old format en converteert.
2. **Ontbrekende `configId`**: groepen zonder configId krijgen de eerste beschikbare config of een lege.
3. **Oude `promef_state` key**: importeert bestaande PROMEF-data vanuit de oude localStorage key.

```javascript
function migrateState() {
  // 1. Import oude PROMEF data als die er nog is
  const oldData = localStorage.getItem('promef_state');
  if (oldData && !localStorage.getItem(STORAGE_KEY)) {
    try {
      const old = JSON.parse(oldData);
      // ... migratie-logica
    } catch(e) {}
  }

  // 2. senior1/senior2 → assessors[]
  for (const g of state.groups) {
    if (g.senior1 !== undefined && !g.assessors) {
      g.assessors = [g.senior1, g.senior2].filter(Boolean);
      delete g.senior1;
      delete g.senior2;
    }
    if (!g.configId && state.configs.length > 0) {
      g.configId = state.configs[0].id;
    }
  }

  // 3. Ensure slotDuration/breaks defaults
  for (const g of state.groups) {
    if (!g.slotDuration) g.slotDuration = 30;
    if (!g.breaks) g.breaks = [];
  }
}
```

---

## 4. Configuratie-engine (NIEUW)

### 4.1 Actieve configuratie ophalen

In de PROMEF-app zijn criteria, scoreniveaus en de cijferberekening hardcoded constanten (`CRITERIA`, `SCORE_LABELS`, `calculateGrade()`). In de generieke versie worden deze dynamisch opgehaald uit de actieve configuratie.

```javascript
// Vervangt: const CRITERIA = [...]
function getActiveConfig(groupId) {
  if (groupId) {
    const group = getGroup(groupId);
    if (group?.configId) {
      return state.configs.find(c => c.id === group.configId);
    }
  }
  // Fallback: settings.activeConfigId of eerste config
  const activeId = state.settings.activeConfigId;
  return state.configs.find(c => c.id === activeId) || state.configs[0] || null;
}

// Vervangt: const SCORE_LABELS, SCORE_FULL
function getLevelLabel(config, level, short = false) {
  if (!config) return String(level);
  return short
    ? (config.levels.shortLabels[level] || String(level))
    : (config.levels.labels[level] || String(level));
}

// Vervangt: function calculateGrade(total)
function calculateGrade(config, total) {
  if (!config?.grading?.scale) return null;
  const sorted = [...config.grading.scale].sort((a, b) => b.minPoints - a.minPoints);
  for (const entry of sorted) {
    if (total >= entry.minPoints) return entry.grade;
  }
  return config.grading.defaultGrade || null;
}

// Vervangt: hardcoded count === 6
function studentTotal(studentId, config) {
  if (!config) return { total: 0, count: 0, complete: false, hasUnder: false, grade: null };
  const criteriaCount = config.criteria.length;
  const threshold = config.grading.passThreshold || 2;
  let total = 0, count = 0, hasUnder = false;

  for (const c of config.criteria) {
    const s = getScore(studentId, c.id);
    if (s !== null) {
      total += s;
      count++;
      if (s < threshold) hasUnder = true;
    }
  }
  const complete = count === criteriaCount;
  const maxPoints = criteriaCount * config.levels.count;
  return {
    total, count, complete, hasUnder,
    grade: complete ? calculateGrade(config, total) : null,
    maxPoints
  };
}
```

### 4.2 Dynamische CSS-kleuren

Scoreniveau-kleuren worden dynamisch gegenereerd op basis van het aantal niveaus in de configuratie. Bij exact 4 niveaus worden de standaardkleuren gebruikt; bij een afwijkend aantal worden kleuren geinterpoleerd over een HSL-gradiënt (rood → oranje → groen → paars).

```javascript
const DEFAULT_LEVEL_COLORS = {
  1: { main: '#ef4444', bg: '#fef2f2', text: '#991b1b', border: '#fecaca' },
  2: { main: '#f59e0b', bg: '#fffbeb', text: '#92400e', border: '#fde68a' },
  3: { main: '#22c55e', bg: '#f0fdf4', text: '#166534', border: '#bbf7d0' },
  4: { main: '#8b5cf6', bg: '#f5f3ff', text: '#5b21b6', border: '#ddd6fe' }
};

function applyConfigColors(config) {
  const root = document.documentElement;
  const count = config?.levels?.count || 4;

  if (count === 4 && !config?.levels?.colors) {
    // Standaardkleuren — via CSS custom properties uit de stylesheet
    return;
  }

  // Genereer kleuren via HSL-interpolatie
  const hueStops = [0, 35, 140, 270]; // rood, oranje, groen, paars
  for (let i = 1; i <= count; i++) {
    const t = count === 1 ? 0 : (i - 1) / (count - 1);
    const hue = interpolateHue(hueStops, t);
    root.style.setProperty(`--score${i}`, `hsl(${hue}, 75%, 55%)`);
    root.style.setProperty(`--score${i}-bg`, `hsl(${hue}, 80%, 96%)`);
    root.style.setProperty(`--score${i}-text`, `hsl(${hue}, 70%, 25%)`);
  }
}
```

### 4.3 Config CRUD

```javascript
function createConfig(data) {
  const config = {
    id: 'config_' + Date.now(),
    ...data,
    createdAt: new Date().toISOString(),
    updatedAt: new Date().toISOString()
  };
  state.configs.push(config);
  state.settings.activeConfigId = config.id;
  saveState();
  return config;
}

function updateConfig(configId, updates) {
  const config = state.configs.find(c => c.id === configId);
  if (!config) return;
  Object.assign(config, updates, { updatedAt: new Date().toISOString() });
  saveState();
}

function deleteConfig(configId) {
  // Check of er groepen aan gekoppeld zijn
  const linkedGroups = state.groups.filter(g => g.configId === configId);
  if (linkedGroups.length > 0) return false; // Niet toestaan
  state.configs = state.configs.filter(c => c.id !== configId);
  if (state.settings.activeConfigId === configId) {
    state.settings.activeConfigId = state.configs[0]?.id || null;
  }
  saveState();
  return true;
}

function duplicateConfig(configId) {
  const original = state.configs.find(c => c.id === configId);
  if (!original) return null;
  const copy = JSON.parse(JSON.stringify(original));
  copy.id = 'config_' + Date.now();
  copy.title += ' (kopie)';
  copy.createdAt = new Date().toISOString();
  copy.updatedAt = new Date().toISOString();
  state.configs.push(copy);
  saveState();
  return copy;
}

function exportConfig(configId) {
  const config = state.configs.find(c => c.id === configId);
  if (!config) return;
  const json = JSON.stringify(config, null, 2);
  downloadFile(json, `${config.title.replace(/\s+/g, '-')}-config.json`, 'application/json');
}

function importConfig(jsonText) {
  const config = JSON.parse(jsonText);
  config.id = 'config_' + Date.now(); // Nieuw ID om conflicten te voorkomen
  config.createdAt = new Date().toISOString();
  config.updatedAt = new Date().toISOString();
  state.configs.push(config);
  saveState();
  return config;
}
```

---

## 5. AI-integratie (NIEUW)

### 5.1 API-aanroep

De app roept de Claude API direct aan vanuit de browser via `fetch()`. De PDF wordt base64-encoded meegestuurd als document-type content block.

```javascript
async function parsePDFWithAI(pdfBase64) {
  const apiKey = state.settings.apiKey;
  if (!apiKey) throw new Error('Geen API-key ingesteld');

  const response = await fetch('https://api.anthropic.com/v1/messages', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'x-api-key': apiKey,
      'anthropic-version': '2023-06-01',
      'anthropic-dangerous-direct-browser-access': 'true'
    },
    body: JSON.stringify({
      model: 'claude-sonnet-4-20250514',
      max_tokens: 8192,
      system: AI_SYSTEM_PROMPT,
      messages: [{
        role: 'user',
        content: [
          {
            type: 'document',
            source: {
              type: 'base64',
              media_type: 'application/pdf',
              data: pdfBase64
            }
          },
          {
            type: 'text',
            text: 'Analyseer dit beoordelingsformulier en extraheer de assessment-configuratie als JSON volgens het gespecificeerde formaat.'
          }
        ]
      }]
    })
  });

  if (!response.ok) {
    const err = await response.json().catch(() => ({}));
    throw new Error(err.error?.message || `API-fout: ${response.status}`);
  }

  const result = await response.json();
  const text = result.content[0]?.text || '';

  // Extraheer JSON uit response (kan in markdown code block staan)
  const jsonMatch = text.match(/```json\s*([\s\S]*?)```/) || text.match(/(\{[\s\S]*\})/);
  if (!jsonMatch) throw new Error('Geen JSON gevonden in AI-response');

  return JSON.parse(jsonMatch[1]);
}
```

### 5.2 System prompt

```javascript
const AI_SYSTEM_PROMPT = `Je bent een assistent die beoordelingsformulieren analyseert voor het hoger onderwijs.

Analyseer het bijgevoegde PDF-document en extraheer de volgende informatie als gestructureerde JSON:

1. **title**: De titel/naam van het assessment
2. **criteria**: Alle beoordelingscriteria met:
   - id: volgnummer (integer, startend bij 1)
   - name: korte naam van het criterium
   - desc: beschrijving/toelichting
   - tags: observatie-indicatoren per scoreniveau (als aanwezig in het document)
3. **levels**: Scoreniveaus met:
   - count: aantal niveaus
   - labels: per niveau het volledige label (key = niveau als string)
   - shortLabels: per niveau een kort label (max 10 karakters)
4. **grading**: Cesuur/scoresleutel met:
   - scale: array van { minPoints, grade } aflopend gesorteerd
   - defaultGrade: cijfer als minimum niet gehaald
   - passRule: tekstuele beschrijving van de zak/slaag-regel
   - passThreshold: minimaal vereist niveau per criterium (integer)

Als het document GEEN observatie-indicatoren per criterium bevat, genereer dan per criterium 3-4 suggestie-tags per scoreniveau. Baseer deze op de criteriumbeschrijving. Geef elk een level (1 = laagste, N = hoogste).

Antwoord UITSLUITEND met geldige JSON in het volgende formaat:
\`\`\`json
{
  "title": "...",
  "criteria": [
    {
      "id": 1,
      "name": "...",
      "desc": "...",
      "tags": [
        { "text": "...", "level": 1 },
        { "text": "...", "level": 2 }
      ]
    }
  ],
  "levels": {
    "count": 4,
    "labels": { "1": "...", "2": "...", "3": "...", "4": "..." },
    "shortLabels": { "1": "...", "2": "...", "3": "...", "4": "..." }
  },
  "grading": {
    "scale": [{ "minPoints": 24, "grade": 10 }],
    "defaultGrade": 4,
    "passRule": "...",
    "passThreshold": 2
  }
}
\`\`\``;
```

### 5.3 PDF upload flow

```javascript
async function handlePDFUpload(file) {
  // 1. Lees PDF als base64
  const base64 = await readFileAsBase64(file);

  // 2. Toon loading state
  showParsingProgress(true);

  try {
    // 3. Stuur naar AI
    const parsed = await parsePDFWithAI(base64);

    // 4. Valideer response
    validateConfigData(parsed);

    // 5. Maak config aan
    const config = createConfig(parsed);

    // 6. Open configuratie-editor
    showConfigEditor(config.id);
  } catch (err) {
    showParsingError(err.message);
  } finally {
    showParsingProgress(false);
  }
}

function readFileAsBase64(file) {
  return new Promise((resolve, reject) => {
    const reader = new FileReader();
    reader.onload = () => {
      const base64 = reader.result.split(',')[1]; // Strip data:...;base64, prefix
      resolve(base64);
    };
    reader.onerror = reject;
    reader.readAsDataURL(file);
  });
}

function validateConfigData(data) {
  if (!data.title) throw new Error('Titel ontbreekt');
  if (!Array.isArray(data.criteria) || data.criteria.length === 0) throw new Error('Geen criteria gevonden');
  if (!data.levels?.count) throw new Error('Scoreniveaus ontbreken');
  if (!data.grading?.scale) throw new Error('Cesuur/scoresleutel ontbreekt');

  // Normaliseer: zorg dat alle criteria tags hebben
  for (const c of data.criteria) {
    if (!c.tags) c.tags = [];
    if (!c.id) c.id = data.criteria.indexOf(c) + 1;
  }
}
```

### 5.4 Tag-regeneratie

De assessor kan AI tags opnieuw laten genereren na het aanpassen van criteria:

```javascript
async function regenerateTags(configId) {
  const config = state.configs.find(c => c.id === configId);
  if (!config) return;

  const prompt = `Genereer per criterium 3-4 observatie-tags per scoreniveau.
  Assessment: ${config.title}
  Niveaus: ${Object.values(config.levels.labels).join(', ')}
  Criteria:\n${config.criteria.map(c => `${c.id}. ${c.name}: ${c.desc}`).join('\n')}

  Antwoord als JSON array van criteria met tags. Behoud custom tags (tag.custom === true).`;

  // API call en merge met bestaande custom tags...
}
```

---

## 6. View-ontwerp (NIEUW en GEWIJZIGD)

### 6.1 Configuratie-view (`renderConfig`)

De configuratie-view heeft drie modi:

#### A. Welkomstscherm (eerste gebruik / geen configs)

```
┌──────────────────────────────────────────┐
│  Welkom bij de Assessment Tool           │
│                                          │
│  ┌──────────────────┐  ┌──────────────┐  │
│  │  Upload PDF      │  │  Handmatig   │  │
│  │  formulier       │  │  instellen   │  │
│  │  (AI-parsing)    │  │              │  │
│  └──────────────────┘  └──────────────┘  │
│                                          │
│  [Voorbeelddata laden]                   │
└──────────────────────────────────────────┘
```

#### B. Upload + parsing

```
┌──────────────────────────────────────────┐
│  PDF uploaden                            │
│  ┌────────────────────────────────┐      │
│  │  Sleep PDF hierheen of klik    │      │
│  │  om bestand te kiezen          │      │
│  └────────────────────────────────┘      │
│                                          │
│  [Parsing... ████████░░ criteria...]     │
│                                          │
└──────────────────────────────────────────┘
```

#### C. Configuratie-editor

```
┌──────────────────────────────────────────┐
│  Configuratie: PROMEF eindassessment  ✎  │
│                                          │
│  ┌─ Criteria ────────────────────────┐   │
│  │ 1. Veranderen en evalueren     ✎ ✕│   │
│  │    Beschrijving: ...           ✎  │   │
│  │    Tags per niveau:               │   │
│  │    Onder: [geen theorie] [+]      │   │
│  │    Op:    [concept benoemd] [+]   │   │
│  │    ...                            │   │
│  │ 2. Communicatie                ✎ ✕│   │
│  │    ...                            │   │
│  │ [+ Criterium toevoegen]           │   │
│  └───────────────────────────────────┘   │
│                                          │
│  ┌─ Scoreniveaus ────────────────────┐   │
│  │ 1: Onder niveau  [kleur] ✎       │   │
│  │ 2: Op niveau     [kleur] ✎       │   │
│  │ ...                               │   │
│  │ [+ Niveau] [- Niveau]            │   │
│  └───────────────────────────────────┘   │
│                                          │
│  ┌─ Cesuur ──────────────────────────┐   │
│  │ 24+ → 10  |  20+ → 9  |  ...     │   │
│  │ Zak/slaag: min. niveau 2 per crit│   │
│  └───────────────────────────────────┘   │
│                                          │
│  [Opslaan]  [Tags regenereren (AI)]      │
│                                          │
│  ┌─ Live preview ────────────────────┐   │
│  │ (Voorbeeld hoe een criterium      │   │
│  │  eruitziet in de assessment-view)  │   │
│  └───────────────────────────────────┘   │
└──────────────────────────────────────────┘
```

### 6.2 Assessment-view (generalisatie)

Belangrijkste wijzigingen t.o.v. PROMEF:

| Aspect | PROMEF (hardcoded) | Generiek (dynamisch) |
|--------|-------------------|---------------------|
| Criteria | `CRITERIA` constante (6 items) | `config.criteria` (N items) |
| Scoreknoppen | `for (let lvl = 1; lvl <= 4; lvl++)` | `for (let lvl = 1; lvl <= config.levels.count; lvl++)` |
| Labels | `SCORE_LABELS[lvl]` | `config.levels.shortLabels[lvl]` |
| Totaal | `${t.total}/24` | `${t.total}/${t.maxPoints}` |
| Voortgang | `${t.count}/6 criteria` | `${t.count}/${config.criteria.length} criteria` |
| Partner | `getPartner(student)` | `getTeammates(student)` (1-N) |
| Duo-sync | `teammates.length === 2` check | `teammates.length >= 1` |
| Assessoren header | `${grp.senior1} & ${grp.senior2}` | `${grp.assessors.join(' & ')}` |

### 6.3 Studenten-view (uitbreiding)

De groep-modal krijgt twee extra velden:

```html
<!-- Configuratie dropdown -->
<div class="form-group">
  <label>Beoordelingsformulier</label>
  <select id="groupConfigSelect">
    <!-- Dynamisch gevuld met state.configs -->
  </select>
</div>

<!-- Assessoren: dynamisch toevoegen/verwijderen -->
<div class="form-group">
  <label>Assessoren</label>
  <div id="assessorFields">
    <!-- Per assessor een input + verwijderknop -->
  </div>
  <button onclick="addAssessorField()">+ Assessor</button>
</div>
```

### 6.4 Instellingen-view (NIEUW)

```
┌──────────────────────────────────────────┐
│  Instellingen                            │
│                                          │
│  ┌─ API-key ─────────────────────────┐   │
│  │ Anthropic API-key:                │   │
│  │ [sk-ant-***...***]  [Wijzig] [✕]  │   │
│  │ Wordt alleen lokaal opgeslagen.   │   │
│  └───────────────────────────────────┘   │
│                                          │
│  ┌─ Configuraties ───────────────────┐   │
│  │ ● PROMEF eindassessment    [✎][✕] │   │
│  │   6 criteria, 4 niveaus           │   │
│  │   Gekoppeld aan: BKN-F01, BKN-F02│   │
│  │                                   │   │
│  │ ○ AOD toets               [✎][✕] │   │
│  │   4 criteria, 3 niveaus           │   │
│  │                                   │   │
│  │ [+ Nieuwe configuratie]           │   │
│  │ [Configuratie importeren (JSON)]  │   │
│  └───────────────────────────────────┘   │
└──────────────────────────────────────────┘
```

---

## 7. Gewijzigde functies t.o.v. PROMEF

### 7.1 Score-systeem

```javascript
// PROMEF: hardcoded 4 niveaus
// Generiek: dynamisch op basis van config

function setScore(studentId, critId, level) {
  if (!state.scores[studentId]) state.scores[studentId] = {};
  const current = state.scores[studentId][critId];
  state.scores[studentId][critId] = current === level ? null : level;

  // Groepsscore: sync naar alle teamgenoten
  const student = state.students.find(s => s.id === studentId);
  const tm = student ? getTeamById(student.teamId) : null;
  if (tm && isDuoMode(tm.groupId, tm.num, critId)) {
    const teammates = getTeammates(student);  // Was: getPartner()
    for (const mate of teammates) {
      if (!state.scores[mate.id]) state.scores[mate.id] = {};
      state.scores[mate.id][critId] = state.scores[studentId][critId];
    }
  }
  saveState();
}

// NIEUW: vervangt getPartner() — ondersteunt N teamleden
function getTeammates(student) {
  return state.students.filter(s => s.teamId === student.teamId && s.id !== student.id);
}

// Groepsscore toggle — gegeneraliseerd
function toggleDuoMode(group, team, critId) {
  const tk = teamKey(group, team);
  if (!state.duoMode[tk]) state.duoMode[tk] = {};
  state.duoMode[tk][critId] = !state.duoMode[tk][critId];

  if (state.duoMode[tk][critId]) {
    const teammates = getTeamStudents(group, team);
    if (teammates.length >= 2) {
      // Sync: neem eerste gevonden score
      let score = null;
      for (const s of teammates) {
        score = getScore(s.id, critId);
        if (score !== null) break;
      }
      if (score !== null) {
        for (const s of teammates) {
          if (!state.scores[s.id]) state.scores[s.id] = {};
          state.scores[s.id][critId] = score;
        }
      }
    }
  }
  saveState();
}
```

### 7.2 Eigen tags toevoegen (NIEUW)

```javascript
function addCustomTag(configId, critId, tagText, level) {
  const config = state.configs.find(c => c.id === configId);
  if (!config) return;
  const criterion = config.criteria.find(c => c.id === critId);
  if (!criterion) return;

  criterion.tags.push({
    text: tagText.trim(),
    level: level || 1,
    custom: true
  });

  config.updatedAt = new Date().toISOString();
  saveState();
}
```

Wordt aangeboden via een inline form in de assessment-view:

```javascript
// In renderAssessment(), na de tag-chips per niveau
html += `<div class="tags-level-group">
  <button class="tag-chip" style="border-style:dashed;opacity:0.6"
    onclick="showAddTagForm(${crit.id})">+ tag</button>
</div>`;
```

### 7.3 CSV-export (dynamische kolommen)

```javascript
function exportCSV() {
  const config = getActiveConfig();
  if (!config) { alert('Geen configuratie actief.'); return; }

  let csv = 'Student,Groep,Team';
  config.criteria.forEach(c => { csv += `,"${c.id}. ${c.name}"`; });
  csv += ',Totaal,Cijfer,Status\n';

  for (const s of state.students) {
    const t = studentTotal(s.id, config);
    const grp = getGroup(s.groupId);
    csv += `"${s.name}","${s.groupId}",${studentTeamNum(s)}`;

    for (const c of config.criteria) {
      const score = getScore(s.id, c.id);
      csv += `,${score !== null ? score : ''}`;
    }

    csv += `,${t.complete ? t.total : ''}`;
    csv += `,${t.grade !== null ? t.grade : ''}`;
    csv += `,${t.complete ? (t.hasUnder ? 'Onvoldoende' : 'Voldoende') : 'Incompleet'}`;
    csv += '\n';
  }

  const filename = config.title.replace(/\s+/g, '-').toLowerCase() + '-resultaten.csv';
  downloadFile(csv, filename, 'text/csv;charset=utf-8');
}
```

### 7.4 Kalender (teamstatus dynamisch)

```javascript
function getTeamStatus(group, teamNum) {
  const config = getActiveConfig(group);
  if (!config) return 'none';
  const students = getTeamStudents(group, teamNum);
  const criteriaCount = config.criteria.length;
  let scored = 0, total = students.length * criteriaCount;

  students.forEach(s => {
    config.criteria.forEach(c => {
      if (getScore(s.id, c.id) !== null) scored++;
    });
  });

  if (scored === 0) return 'none';
  if (scored === total) return 'complete';
  return 'partial';
}
```

---

## 8. CSS-architectuur

### 8.1 Design tokens

De CSS custom properties uit de PROMEF-app worden 1:1 overgenomen. Dynamische scoreniveau-kleuren worden via JS als inline `style` op `:root` gezet (zie §4.2).

```css
:root {
  /* Layout */
  --bg: #f1f5f9; --surface: #ffffff; --text: #1e293b; --text-muted: #64748b;
  --border: #e2e8f0; --primary: #2563eb; --primary-light: #dbeafe;
  --radius: 8px; --shadow: 0 1px 3px rgba(0,0,0,0.1);
  --touch-min: 44px;

  /* Score-kleuren (defaults — overschreven per config via JS) */
  --score1: #ef4444; --score1-bg: #fef2f2; --score1-text: #991b1b;
  --score2: #f59e0b; --score2-bg: #fffbeb; --score2-text: #92400e;
  --score3: #22c55e; --score3-bg: #f0fdf4; --score3-text: #166534;
  --score4: #8b5cf6; --score4-bg: #f5f3ff; --score4-text: #5b21b6;
}
```

### 8.2 Nieuwe CSS-componenten

Toevoegen voor de configuratie-view:

```css
/* Config editor */
.config-editor { max-width: 900px; }
.config-criterion-card { /* ... */ }
.config-tag-editor { /* ... */ }
.config-level-editor { /* ... */ }
.config-grading-table { /* ... */ }
.config-preview { /* ... */ }

/* API key input */
.api-key-field { /* masked input */ }
.api-key-field input { -webkit-text-security: disc; }

/* Upload dropzone */
.upload-zone { /* drag & drop area */ }
.upload-zone.drag-over { /* highlight */ }

/* Parsing progress */
.parsing-progress { /* loading indicator */ }

/* Onboarding */
.onboarding-card { /* welkomstscherm cards */ }
```

### 8.3 Responsive strategie

Ongewijzigd t.o.v. PROMEF — dezelfde breakpoints en patronen:

| Breakpoint | Gedrag |
|-----------|--------|
| > 768px | Desktop: kolommen naast elkaar, inline tabs |
| <= 768px | Tablet: hamburger menu, gestapelde groepen, bottom-sheet modals |
| <= 480px | Telefoon: compactere spacing, kleinere fonts |

---

## 9. Onboarding-flow

### 9.1 Eerste gebruik

```
init()
  └─ loadState()
       └─ Geen data in localStorage?
            └─ showView('config')  // Welkomstscherm
                 ├─ "Upload PDF" → vraagt API-key → PDF upload → AI parsing → editor
                 ├─ "Handmatig" → blanco configuratie-editor
                 └─ "Voorbeelddata" → loadDemoData() met generieke demo-config
```

### 9.2 API-key flow

```
Upload PDF geklikt
  └─ state.settings.apiKey bestaat?
       ├─ Ja → direct naar PDF upload dialog
       └─ Nee → API-key modal
            ├─ Uitleg: key wordt alleen lokaal opgeslagen
            ├─ Link naar Anthropic console
            ├─ Input veld
            └─ Opslaan → state.settings.apiKey = key → saveState()
                 └─ PDF upload dialog
```

---

## 10. Modal-systeem

### 10.1 Bestaande modals (uit PROMEF)

| Modal | Trigger | Doel |
|-------|---------|------|
| `importModal` | "Importeer CSV" knop | CSV upload en import |
| `groupModal` | "+ Groep" of edit knop | Groep aanmaken/bewerken |
| `teamModal` | "+ Team" of edit knop | Team aanmaken/bewerken |
| `studentModal` | "+ Student" of edit knop | Student aanmaken/bewerken |

### 10.2 Nieuwe modals

| Modal | Trigger | Doel |
|-------|---------|------|
| `apiKeyModal` | Eerste PDF upload / Instellingen | API-key invoeren/wijzigen |
| `addTagModal` | "+ tag" knop in assessment | Eigen observatie-tag toevoegen |
| `confirmModal` | Diverse destructieve acties | Generieke bevestigingsdialog |

---

## 11. Helper-functies

### 11.1 Overgenomen uit PROMEF (ongewijzigd)

| Functie | Doel |
|---------|------|
| `getGroup(groupId)` | Vind groep op ID |
| `getTeamById(teamId)` | Vind team op ID |
| `getTeamStudents(group, teamNum)` | Studenten in een team |
| `getScore(studentId, critId)` | Score ophalen |
| `getNote(studentId, critId)` | Notitie ophalen |
| `setNote(studentId, critId, text)` | Notitie opslaan |
| `teamKey(groupId, num)` | Team-key genereren |
| `formatGroupDatetime(group)` | Nederlandse datumformat |
| `parseDatetimeString(str)` | NL datum/tijd parser |
| `parseCSV(text)` | CSV parser met NL-support |
| `downloadFile(content, name, type)` | Blob download trigger |
| `saveState()` / `loadState()` | localStorage persistentie |
| `showView(viewName)` | View navigatie |
| `toggleMenu()` | Hamburger menu |
| `getGroupTimeSlots(group)` | Tijdslots per groep |
| `isBreakSlot(group, slot)` | Check of slot pauze is |
| `getAllTimeSlots()` | Union van alle groep-slots |
| `buildGroupsLookup()` | Nested object voor kalender |
| `isTagInNote(studentId, critId, tagText)` | Tag aanwezig in notitie? |
| `insertTagInNote(studentId, critId, tagText)` | Toggle tag in notitie |
| `getTagAssignees(critId, tagText)` | Welke studenten hebben tag |
| `showTagPicker(event, critId, tagText)` | Popup met studentnamen |
| `handleNoteDrop(e, studentId, critId)` | Drag-to-notes |

### 11.2 Gewijzigde functies

| Functie | Wijziging |
|---------|-----------|
| `getPartner(student)` | Vervangen door `getTeammates(student)` — retourneert array |
| `studentTotal(studentId)` | Accepteert `config` parameter, dynamisch criteria-count |
| `calculateGrade(total)` | Accepteert `config` parameter, gebruikt `config.grading.scale` |
| `setScore(studentId, critId, level)` | Duo-sync werkt met N teamleden |
| `toggleDuoMode(group, team, critId)` | Geen `length === 2` check meer |
| `getTeamStatus(group, teamNum)` | Criteria-count uit config |
| `importCSV(text)` | Dynamische assessor-kolommen |
| `exportCSV()` | Dynamische kolomnamen uit config |
| `buildGroupsLookup()` | `assessors` i.p.v. `senior1`/`senior2` |

### 11.3 Nieuwe functies

| Functie | Doel |
|---------|------|
| `getActiveConfig(groupId)` | Config ophalen voor groep of actieve |
| `getLevelLabel(config, level, short)` | Label voor scoreniveau |
| `createConfig(data)` | Config aanmaken |
| `updateConfig(configId, updates)` | Config bijwerken |
| `deleteConfig(configId)` | Config verwijderen |
| `duplicateConfig(configId)` | Config kopieren |
| `exportConfig(configId)` | Config als JSON downloaden |
| `importConfig(jsonText)` | Config JSON importeren |
| `parsePDFWithAI(pdfBase64)` | PDF naar AI sturen en config ontvangen |
| `readFileAsBase64(file)` | File naar base64 |
| `validateConfigData(data)` | AI-output valideren |
| `addCustomTag(configId, critId, text, level)` | Eigen tag toevoegen |
| `applyConfigColors(config)` | CSS custom properties voor niveaukleuren |
| `regenerateTags(configId)` | AI-tags opnieuw genereren |

---

## 12. Geschatte omvang

| Component | PROMEF (regels) | Generiek (geschat) |
|-----------|-----------------|-------------------|
| CSS | ~365 | ~480 (+config editor, upload, onboarding) |
| HTML body | ~140 | ~200 (+nieuwe views, modals) |
| JS: Constants & State | ~100 | ~80 (constanten verdwijnen, config engine neemt over) |
| JS: Config Engine | — | ~250 |
| JS: AI Integration | — | ~150 |
| JS: Navigation | ~30 | ~40 |
| JS: Render functions | ~650 | ~900 (+config view, settings view) |
| JS: CRUD | ~250 | ~350 (+config CRUD, assessor CRUD) |
| JS: CSV/Export | ~130 | ~180 (dynamische kolommen) |
| JS: Calendar | ~200 | ~210 |
| JS: Assessment logic | ~180 | ~200 (generalisatie) |
| JS: Helpers | ~100 | ~130 |
| **Totaal** | **~2150** | **~3170** |

---

## 13. Implementatievolgorde

### Fase 1: Fundament
1. Datamodel uitbreiden met `configs[]` en `settings{}`
2. Migratie-logica (`migrateState()`)
3. Config engine (`getActiveConfig()`, `calculateGrade()`, `studentTotal()`)
4. Alle hardcoded constanten vervangen door config-lookups
5. `getPartner()` → `getTeammates()` generalisatie
6. `senior1`/`senior2` → `assessors[]` overal doorvoeren

### Fase 2: Configuratie-UI
7. Welkomstscherm / onboarding
8. Configuratie-editor (criteria, niveaus, cesuur, tags)
9. Handmatige configuratie (blanco formulier)
10. Config CRUD (opslaan, dupliceren, verwijderen, export/import)
11. Instellingen-view (API-key beheer, configuratie-overzicht)

### Fase 3: AI-integratie
12. API-key modal en opslag
13. PDF upload met base64 encoding
14. Claude API aanroep met system prompt
15. Response parsing en validatie
16. Tag-regeneratie functie

### Fase 4: Views generaliseren
17. Assessment-view: dynamische criteria, niveaus, labels
18. Resultaten-view: dynamische kolommen
19. Feedback-view: dynamische criteria
20. Dashboard: dynamische teamstatus
21. Studenten-view: configId dropdown, assessor-lijst
22. Export-view: configuratie-export/import knoppen
23. Eigen tags toevoegen in assessment-view

### Fase 5: Afwerking
24. Dynamische CSS-kleuren (meer/minder dan 4 niveaus)
25. CSV-import met dynamische assessor-kolommen
26. Handleiding herschrijven (generiek)
27. Demo-data met generieke voorbeeld-config
28. Testen met meerdere formulieren

---

## 14. Testplan

### 14.1 Referentie-testcase: PROMEF eindassessment F-cluster

Het bestand `test-formulier.pdf` is het beoordelingsformulier waarop de bestaande PROMEF-app (`example-code.html`) is gebouwd. Omdat we de verwachte output exact kennen, dient dit als **gouden referentie** voor alle tests.

#### Verwachte AI-parsing output

Wanneer `test-formulier.pdf` door `parsePDFWithAI()` wordt gestuurd, moet de geretourneerde JSON het volgende bevatten:

```javascript
const EXPECTED_PROMEF_CONFIG = {
  title: "PROMEF eindassessment F-cluster",

  criteria: [
    { id: 1, name: "Veranderen en evalueren",
      desc: /* bevat: */ "theoretisch concept uit de fasen veranderen en/of evalueren van de bedrijfskundige handelingscyclus",
      tags: [ /* 3-4 tags per level, AI-gegenereerd want PDF bevat geen indicatoren */ ] },
    { id: 2, name: "Sociaal communicatieve vaardigheden",
      desc: /* bevat: */ "communicatieve gespreksvaardigheden passend bij startende beroepsprofessional" },
    { id: 3, name: "Schakelen en verbinden",
      desc: /* bevat: */ "weerstand kan worden omgezet in veranderbereidheid" },
    { id: 4, name: "Professionaliseren",
      desc: /* bevat: */ "professionele bijdrage" /* EN */ "gegroeid als professional" },
    { id: 5, name: "Professionaliseren",
      desc: /* bevat: */ "intercultureel issue" },
    { id: 6, name: "Handelen vanuit waarden",
      desc: /* bevat: */ "morele kwestie" /* EN */ "OOA gedragsregels" }
  ],
  // criteria.length === 6

  levels: {
    count: 4,
    labels: {
      "1": "Onder niveau",
      "2": "Op niveau",
      "3": "Boven niveau",
      "4": "Excellent"
    },
    shortLabels: {
      "1": /* max 10 chars, bijv. */ "Onder",
      "2": "Op",
      "3": "Boven",
      "4": "Excellent"
    }
  },

  grading: {
    scale: [
      { minPoints: 24, grade: 10 },
      { minPoints: 20, grade: 9 },
      { minPoints: 16, grade: 8 },
      { minPoints: 14, grade: 7 },
      { minPoints: 12, grade: 6 },
      { minPoints: 10, grade: 5 },
      { minPoints: 5,  grade: 4 }
    ],
    defaultGrade: 4,
    passRule: /* bevat: */ "op niveau" /* EN */ "2",
    passThreshold: 2
  }
};
```

#### Validatiecriteria voor AI-parsing

| # | Aspect | Verwachte waarde | Validatie |
|---|--------|-----------------|-----------|
| P1 | Titel | Bevat "PROMEF" en "F-cluster" | `title.includes('PROMEF') && title.includes('F-cluster')` |
| P2 | Aantal criteria | Exact 6 | `criteria.length === 6` |
| P3 | Criterium-ID's | 1 t/m 6, uniek | `new Set(criteria.map(c => c.id)).size === 6` |
| P4 | Criterium-namen | Bevatten kernwoorden (zie boven) | Substring-match per criterium |
| P5 | Criteria hebben beschrijving | Niet leeg | `criteria.every(c => c.desc?.length > 10)` |
| P6 | Aantal niveaus | Exact 4 | `levels.count === 4` |
| P7 | Niveau-labels | "Onder niveau", "Op niveau", "Boven niveau", "Excellent" | Exacte match |
| P8 | Scoresleutel aanwezig | Minimaal 5 entries | `grading.scale.length >= 5` |
| P9 | Hoogste score → 10 | 24 punten = cijfer 10 | `scale.find(s => s.grade === 10).minPoints === 24` |
| P10 | Zak/slaag-drempel | passThreshold = 2 | `grading.passThreshold === 2` |
| P11 | Tags gegenereerd | PDF bevat geen indicatoren → AI genereert suggesties | `criteria.every(c => c.tags.length >= 8)` (min 2 per niveau × 4 niveaus) |
| P12 | Tags hebben levels | Elk tag-object heeft level 1-4 | `criteria.every(c => c.tags.every(t => t.level >= 1 && t.level <= 4))` |

---

### 14.2 Unit tests — Configuratie-engine

Tests voor de kernfuncties die hardcoded constanten vervangen. Gebruiken een handmatig opgebouwde `testConfig` (geen AI nodig).

```javascript
const testConfig = {
  id: 'test_config',
  title: 'Test Assessment',
  criteria: [
    { id: 1, name: 'Criterium A', desc: 'Beschrijving A', tags: [] },
    { id: 2, name: 'Criterium B', desc: 'Beschrijving B', tags: [] },
    { id: 3, name: 'Criterium C', desc: 'Beschrijving C', tags: [] }
  ],
  levels: {
    count: 3,
    labels: { '1': 'Onvoldoende', '2': 'Voldoende', '3': 'Goed' },
    shortLabels: { '1': 'Onv', '2': 'Vold', '3': 'Goed' }
  },
  grading: {
    scale: [
      { minPoints: 9, grade: 10 },
      { minPoints: 7, grade: 8 },
      { minPoints: 5, grade: 6 }
    ],
    defaultGrade: 4,
    passRule: 'Alle criteria minimaal Voldoende',
    passThreshold: 2
  }
};
```

| # | Test | Input | Verwacht resultaat |
|---|------|-------|--------------------|
| C1 | `calculateGrade` — maximum score | `calculateGrade(testConfig, 9)` | `10` |
| C2 | `calculateGrade` — midden | `calculateGrade(testConfig, 7)` | `8` |
| C3 | `calculateGrade` — net voldoende | `calculateGrade(testConfig, 5)` | `6` |
| C4 | `calculateGrade` — onder minimum | `calculateGrade(testConfig, 4)` | `4` (defaultGrade) |
| C5 | `calculateGrade` — null config | `calculateGrade(null, 15)` | `null` |
| C6 | `getLevelLabel` — vol label | `getLevelLabel(testConfig, 2, false)` | `"Voldoende"` |
| C7 | `getLevelLabel` — kort label | `getLevelLabel(testConfig, 2, true)` | `"Vold"` |
| C8 | `getLevelLabel` — onbekend niveau | `getLevelLabel(testConfig, 99)` | `"99"` (fallback) |
| C9 | `studentTotal` — volledig gescoord | Student met scores {1:3, 2:2, 3:3} | `{ total:8, count:3, complete:true, hasUnder:false, grade:8, maxPoints:9 }` |
| C10 | `studentTotal` — onvolledig | Student met scores {1:3} | `{ total:3, count:1, complete:false, grade:null, maxPoints:9 }` |
| C11 | `studentTotal` — heeft under | Student met scores {1:1, 2:2, 3:2} | `{ hasUnder:true, grade:6 }` (passThreshold=2, score 1 < 2) |
| C12 | `studentTotal` — geen config | `studentTotal(1, null)` | `{ total:0, count:0, complete:false, grade:null }` |
| C13 | `validateConfigData` — geldig | Volledig PROMEF config object | Geen error |
| C14 | `validateConfigData` — geen titel | `{ criteria: [...], levels: {...} }` | Throws "Titel ontbreekt" |
| C15 | `validateConfigData` — lege criteria | `{ title: "X", criteria: [] }` | Throws "Geen criteria gevonden" |
| C16 | `validateConfigData` — geen levels | `{ title: "X", criteria: [...] }` | Throws "Scoreniveaus ontbreken" |

---

### 14.3 Unit tests — Cijferberekening met PROMEF-scoresleutel

Specifiek voor het PROMEF-formulier: controleer dat de dynamische `calculateGrade()` identieke resultaten geeft als de hardcoded functie in `example-code.html`.

```javascript
const promefConfig = {
  grading: {
    scale: [
      { minPoints: 24, grade: 10 },
      { minPoints: 20, grade: 9 },
      { minPoints: 16, grade: 8 },
      { minPoints: 14, grade: 7 },
      { minPoints: 12, grade: 6 },
      { minPoints: 10, grade: 5 },
      { minPoints: 5,  grade: 4 }
    ],
    defaultGrade: 4,
    passThreshold: 2
  }
};
```

| # | Input totaal | Verwacht cijfer | Toelichting |
|---|-------------|----------------|-------------|
| G1 | 24 | 10 | Maximum (6×4) |
| G2 | 23 | 9 | Tussen 20-23 |
| G3 | 20 | 9 | Ondergrens 9 |
| G4 | 19 | 8 | Tussen 16-19 |
| G5 | 16 | 8 | Ondergrens 8 |
| G6 | 15 | 7 | 14-15 range |
| G7 | 14 | 7 | Ondergrens 7 |
| G8 | 13 | 6 | 12-13 range |
| G9 | 12 | 6 | Ondergrens 6 |
| G10 | 11 | 5 | 10-11 range |
| G11 | 10 | 5 | Ondergrens 5 |
| G12 | 9 | 4 | 5-9 range |
| G13 | 6 | 4 | Minimum in range |
| G14 | 5 | 4 | Ondergrens 4 |
| G15 | 4 | 4 | Onder laagste entry → defaultGrade |

**Pariteitscheck**: voor elk totaal van 5 t/m 24, moet `calculateGrade(promefConfig, totaal)` exact overeenkomen met de hardcoded functie uit `example-code.html`:

```javascript
function calculateGradeOriginal(total) {
  if (total >= 24) return 10;
  if (total >= 20) return 9;
  if (total >= 16) return 8;
  if (total >= 14) return 7;
  if (total >= 12) return 6;
  if (total >= 10) return 5;
  return 4;
}
```

---

### 14.4 Unit tests — Score- en teamsysteem

| # | Test | Scenario | Verwacht resultaat |
|---|------|----------|--------------------|
| S1 | `setScore` — toggle on | `setScore(1, 1, 3)` op lege score | `getScore(1, 1) === 3` |
| S2 | `setScore` — toggle off | `setScore(1, 1, 3)` twee keer | `getScore(1, 1) === null` |
| S3 | `setScore` — wissel niveau | Score 3, dan `setScore(1, 1, 2)` | `getScore(1, 1) === 2` |
| S4 | Duo-mode sync (2 studenten) | DuoMode aan, score student 1 | Student 2 krijgt zelfde score |
| S5 | Duo-mode sync (3 studenten) | DuoMode aan, team van 3, score student 1 | Student 2 en 3 krijgen zelfde score |
| S6 | Duo-mode off — geen sync | DuoMode uit, score student 1 | Student 2 behoudt eigen score |
| S7 | `getTeammates` — duo | Team met 2 studenten | Retourneert array met 1 student |
| S8 | `getTeammates` — trio | Team met 3 studenten | Retourneert array met 2 studenten |
| S9 | `getTeammates` — solo | Team met 1 student | Retourneert lege array |
| S10 | `getTeamStatus` — leeg | Geen scores voor team | `'none'` |
| S11 | `getTeamStatus` — deels | 1 van 6 criteria gescoord (1 student) | `'partial'` |
| S12 | `getTeamStatus` — volledig | Alle criteria gescoord (alle studenten) | `'complete'` |

---

### 14.5 Unit tests — Tag-systeem

| # | Test | Scenario | Verwacht resultaat |
|---|------|----------|--------------------|
| T1 | `insertTagInNote` — toevoegen | Lege notitie, insert "helder uitgelegd" | Notitie = `"✓ helder uitgelegd"` |
| T2 | `insertTagInNote` — bij bestaande tekst | Notitie "goed gesprek", insert tag | Notitie = `"goed gesprek\n✓ helder uitgelegd"` |
| T3 | `insertTagInNote` — toggle off | Tag aanwezig, insert zelfde tag | Tag verwijderd, overige tekst intact |
| T4 | `isTagInNote` — aanwezig | Notitie bevat `"✓ actief luisteren"` | `true` |
| T5 | `isTagInNote` — afwezig | Notitie bevat tag niet | `false` |
| T6 | `getTagAssignees` — meerdere | Tag bij 2 van 3 studenten | Array met 2 studenten |
| T7 | `addCustomTag` — toevoegen | Nieuwe tag "goed voorbeeld" op level 3 | Tag in config met `custom: true` |
| T8 | `addCustomTag` — persistent | Na toevoegen + `saveState` + `loadState` | Tag nog steeds in config |

---

### 14.6 Unit tests — Config CRUD

| # | Test | Scenario | Verwacht resultaat |
|---|------|----------|--------------------|
| CR1 | `createConfig` | Geldige config-data | Config in `state.configs`, uniek ID, timestamps gezet |
| CR2 | `createConfig` — wordt actief | Aanmaken eerste config | `state.settings.activeConfigId === newConfig.id` |
| CR3 | `updateConfig` | Titel wijzigen | `updatedAt` bijgewerkt, titel gewijzigd |
| CR4 | `deleteConfig` — niet gekoppeld | Config zonder groepen | Config verwijderd, `true` geretourneerd |
| CR5 | `deleteConfig` — wel gekoppeld | Config met gekoppelde groep | Config NIET verwijderd, `false` geretourneerd |
| CR6 | `duplicateConfig` | Bestaande config | Nieuwe config met "(kopie)" suffix, nieuw ID, zelfde criteria |
| CR7 | `exportConfig` → `importConfig` roundtrip | Export als JSON, import | Geimporteerde config heeft identieke criteria, niveaus, cesuur |
| CR8 | `importConfig` — krijgt nieuw ID | Import JSON met bestaand ID | Nieuw ID toegewezen (geen conflict) |

---

### 14.7 Unit tests — CSV

| # | Test | Scenario | Verwacht resultaat |
|---|------|----------|--------------------|
| V1 | `parseCSV` — puntkomma (NL) | `"Naam;Groep;Team\nAnna;G01;1"` | Correct gesplitst op `;` |
| V2 | `parseCSV` — komma (EN) | `"Naam,Groep,Team\nAnna,G01,1"` | Correct gesplitst op `,` |
| V3 | `parseCSV` — quoted velden | `"Naam;Groep\n\"De Vries, Jan\";G01"` | Naam = `"De Vries, Jan"` |
| V4 | `importCSV` — dynamische assessoren | CSV met kolommen "Assessor 1", "Assessor 2", "Assessor 3" | Groep krijgt `assessors: [3 namen]` |
| V5 | `exportCSV` — dynamische kolommen | Config met 3 criteria | CSV header bevat 3 criterium-kolommen |
| V6 | `exportCSV` — bestandsnaam uit config | Config title = "PROMEF eindassessment" | Bestandsnaam = `"promef-eindassessment-resultaten.csv"` |

---

### 14.8 Unit tests — Persistentie & migratie

| # | Test | Scenario | Verwacht resultaat |
|---|------|----------|--------------------|
| M1 | `saveState` → `loadState` roundtrip | Sla state op, laad opnieuw | Identieke state |
| M2 | Migratie `senior1/senior2` | State met `{ senior1: "A", senior2: "B" }` | Na migratie: `{ assessors: ["A", "B"] }` |
| M3 | Migratie lege senior2 | State met `{ senior1: "A", senior2: "" }` | Na migratie: `{ assessors: ["A"] }` |
| M4 | Migratie oude `promef_state` key | Data in `localStorage['promef_state']` | Geimporteerd naar `assessment_tool_state`, oude key intact |
| M5 | Ontbrekende velden | State zonder `configs` of `settings` | Na `loadState()`: lege arrays/objecten aangemaakt |
| M6 | Groep zonder configId | State met groep zonder configId, 1 config aanwezig | Na migratie: `group.configId === configs[0].id` |

---

### 14.9 Integratietests — End-to-end scenario's

#### E2E-1: Volledige flow met PROMEF-formulier

Simuleert de complete gebruikersflow met `test-formulier.pdf` als input.

| Stap | Actie | Verificatie |
|------|-------|-------------|
| 1 | Open app (lege localStorage) | Welkomstscherm verschijnt met "Upload PDF" en "Handmatig" opties |
| 2 | Voer API-key in | Key opgeslagen in `state.settings.apiKey`, niet zichtbaar in plaintext |
| 3 | Upload `test-formulier.pdf` | Loading-indicator verschijnt |
| 4 | AI-parsing voltooid | Configuratie-editor toont: titel "PROMEF eindassessment F-cluster", 6 criteria, 4 niveaus |
| 5 | Controleer criteria in editor | Alle 6 criteria herkenbaar met correcte namen en beschrijvingen (zie §14.1) |
| 6 | Controleer niveaus in editor | "Onder niveau", "Op niveau", "Boven niveau", "Excellent" |
| 7 | Controleer cesuur in editor | Scoresleutel 24→10, 20→9, 16→8, 14→7, 12→6, 10→5, 5→4 |
| 8 | Controleer zak/slaag in editor | passThreshold = 2, passRule vermeldt "op niveau" |
| 9 | Controleer tags in editor | Per criterium minimaal 8 AI-gegenereerde tags (2+ per niveau) |
| 10 | Sla configuratie op | Config verschijnt in `state.configs`, `activeConfigId` gezet |
| 11 | Maak groep "BKN-F01" aan, koppel PROMEF-config | Groep heeft `configId` van PROMEF config |
| 12 | Voeg 2 assessoren toe: "Docent A", "Docent B" | `group.assessors === ["Docent A", "Docent B"]` |
| 13 | Maak 3 teams met elk 2 studenten | 6 studenten, 3 teams in `state.teams` |
| 14 | Kalenderoverzicht | 3 teamblokken zichtbaar, alle status "none" (grijs) |
| 15 | Klik op Team 1 | Assessment-view opent met 6 criteria-blokken, 4 scoreknoppen per criterium |
| 16 | Score student 1: alle 6 criteria op niveau 3 | Totaal 18/24, cijfer 8, status "Voldoende" |
| 17 | Score student 2: criterium 1 op niveau 1, rest niveau 2 | Totaal 11/24, cijfer 5, `hasUnder: true`, status "Onvoldoende" |
| 18 | Ga terug naar kalender | Team 1 toont status "complete" (groen) |
| 19 | Open Resultaten-view | Tabel toont 6 criteriumkolommen (namen uit config), totaal, cijfer, status |
| 20 | Open Feedback voor student 1 | Rapport toont alle 6 criteria met score-badges "Boven niveau" (groen) |
| 21 | Open Feedback voor student 2 | Rapport toont criterium 1 als "Onder niveau" (rood), status "Onvoldoende" |
| 22 | Exporteer CSV | Bestandsnaam bevat "promef", 6 criteriumkolommen, correct cijfer per student |
| 23 | JSON backup → restore | Na restore: identieke state, alle scores intact |
| 24 | Sluit browser, heropen | Data bewaard in localStorage, alles intact |

#### E2E-2: Handmatige configuratie (zonder AI)

| Stap | Actie | Verificatie |
|------|-------|-------------|
| 1 | Open app, kies "Handmatig instellen" | Blanco configuratie-editor verschijnt |
| 2 | Vul titel in: "Testformulier" | Titel opgeslagen |
| 3 | Voeg 3 criteria toe | 3 criteria met id 1, 2, 3 |
| 4 | Stel 3 niveaus in: "Slecht", "Voldoende", "Goed" | `levels.count === 3`, labels correct |
| 5 | Stel cesuur in: 9→10, 7→8, 5→6 | `grading.scale` correct |
| 6 | Sla op, maak groep aan | Groep gekoppeld aan handmatige config |
| 7 | Open assessment | 3 criteria, 3 scoreknoppen per criterium |
| 8 | Score alle criteria "Goed" (3) voor 1 student | Totaal 9/9, cijfer 10 |

#### E2E-3: Meerdere configuraties

| Stap | Actie | Verificatie |
|------|-------|-------------|
| 1 | Importeer PROMEF-config (AI of JSON) | Config A in `state.configs` |
| 2 | Maak handmatig config B aan (3 criteria, 3 niveaus) | Config B in `state.configs` |
| 3 | Maak groep 1, koppel config A | Groep 1 toont 6 criteria, 4 niveaus |
| 4 | Maak groep 2, koppel config B | Groep 2 toont 3 criteria, 3 niveaus |
| 5 | Score student in groep 1 | Cijfer berekend via PROMEF-scoresleutel |
| 6 | Score student in groep 2 | Cijfer berekend via config B scoresleutel |
| 7 | Resultaten-view | Groep 1 toont 6 kolommen, groep 2 toont 3 kolommen |
| 8 | CSV-export | Kolommen afgestemd op de config van de betreffende studenten |

#### E2E-4: Config export/import (delen met collega)

| Stap | Actie | Verificatie |
|------|-------|-------------|
| 1 | Gebruiker A: upload `test-formulier.pdf`, pas tags aan | Config met custom tags |
| 2 | Gebruiker A: exporteer config als JSON | JSON-bestand gedownload, bevat GEEN API-key |
| 3 | Gebruiker B: importeer config JSON | Config verschijnt met zelfde criteria, niveaus, tags, custom tags |
| 4 | Gebruiker B: maak groep aan met geimporteerde config | Assessment werkt identiek aan gebruiker A |

---

### 14.10 UI/UX-tests

| # | Test | Verificatie |
|---|------|-------------|
| U1 | Responsive — desktop (>768px) | Navigatie als inline tabs, kalender met groepen naast elkaar |
| U2 | Responsive — tablet (<=768px) | Hamburger menu, kalender gestapeld per groep, modals als bottom-sheet |
| U3 | Responsive — telefoon (<=480px) | Compacte layout, touch targets >= 44px |
| U4 | Score-knoppen dynamisch | Aantal knoppen = `config.levels.count`, labels uit config |
| U5 | Score-knoppen kleuren | Bij 4 niveaus: rood, oranje, groen, paars (defaults) |
| U6 | Score-knoppen kleuren (afwijkend) | Bij 3 of 5 niveaus: HSL-geinterpoleerde kleuren |
| U7 | Tag-chips kleuren | Kleur correspondeert met niveau uit config |
| U8 | Drag & drop tags → notitie | Tag verschijnt als `✓`-regel in textarea |
| U9 | Tag-picker popup | Klik op tag toont studentnamen, klik wijst toe |
| U10 | Auto-save | Elke score/notitie-wijziging triggert `saveState()` |
| U11 | Print/PDF feedback | `Ctrl+P` op feedback-view toont alleen feedbackrapport |
| U12 | Kalender drag & drop teams | Team verplaatst naar ander tijdslot, swap bij bezet slot |
| U13 | API-key niet zichtbaar in exports | JSON-backup en config-export bevatten GEEN apiKey |
| U14 | Configuratie-editor live preview | Wijzigingen in editor direct zichtbaar in preview-blok |
| U15 | Eigen tag toevoegen | "+ tag" knop → input → tag verschijnt als chip met `custom: true` |

---

### 14.11 Randgevallen en foutafhandeling

| # | Test | Scenario | Verwacht gedrag |
|---|------|----------|-----------------|
| R1 | Ongeldige API-key | `parsePDFWithAI` met "invalid-key" | Duidelijke foutmelding "Ongeldige API-key", geen crash |
| R2 | Geen internetverbinding | API-call faalt met network error | Foutmelding "Geen verbinding", optie om opnieuw te proberen |
| R3 | AI retourneert onvolledige JSON | Response zonder `criteria` | `validateConfigData` gooit error, foutmelding aan gebruiker |
| R4 | AI retourneert ongeldige JSON | Response is geen geldig JSON | Parse error gevangen, foutmelding aan gebruiker |
| R5 | PDF zonder beoordelingscriteria | Upload bijv. een willekeurig PDF | AI retourneert wat het kan, validatie vangt ontbrekende velden |
| R6 | Zeer groot PDF (>10MB) | Upload groot bestand | Foutmelding over bestandsgrootte voor API-call |
| R7 | Config verwijderen met gekoppelde groepen | `deleteConfig()` op actieve config | Geweigerd, melding "Config is gekoppeld aan groep(en)" |
| R8 | Groep zonder config | Config verwijderd na groep-aanmaak (corrupte state) | Graceful fallback, waarschuwing in assessment-view |
| R9 | Score op onbekend criterium | `setScore(1, 999, 3)` | Score wordt opgeslagen maar niet getoond (geen crash) |
| R10 | Student zonder team | Student met ongeldig teamId | Assessment-view skipt, geen crash |
| R11 | localStorage vol | `saveState()` gooit QuotaExceededError | Foutmelding "Opslag vol", suggestie om data te exporteren |
| R12 | Lege CSV-import | CSV met alleen headers | Melding "Geen studenten gevonden" |
| R13 | CSV met ontbrekende kolommen | CSV zonder "Team" kolom | Melding met verwachte kolommen |
| R14 | Concurrent browser-tabs | Twee tabs met dezelfde app | Geen data-corruptie (laatste schrijft wint) |

---

### 14.12 Performancetests

| # | Test | Drempel |
|---|------|---------|
| PF1 | Initieel laden (lege state) | < 100ms tot interactief |
| PF2 | Laden met 100 studenten, 10 groepen | < 200ms tot dashboard gerenderd |
| PF3 | Assessment-view renderen (6 criteria, 2 studenten) | < 50ms |
| PF4 | Score wijzigen + re-render | < 100ms (voelt instant) |
| PF5 | `saveState()` met 100 studenten + scores | < 50ms |
| PF6 | CSV-export met 100 studenten | < 100ms |
| PF7 | PDF-parsing (API-call) | < 30s (netwerk-afhankelijk, loading indicator verplicht) |
| PF8 | Configuratie-editor met 10 criteria, 40+ tags | Soepel scrollen, geen jank |

---

### 14.13 Testuitvoering

Alle tests worden handmatig uitgevoerd in de browser (geen test-framework, conform de single-file architectuur). De aanbevolen aanpak:

1. **Unit tests (§14.2–14.8)**: voer uit via de browser-console. Bouw een `runTests()` functie die assertions uitvoert en resultaten logt:

```javascript
function runTests() {
  const results = [];
  const assert = (name, condition) => {
    results.push({ name, pass: !!condition });
    if (!condition) console.error('FAIL:', name);
  };

  // C1: calculateGrade — maximum score
  assert('C1', calculateGrade(testConfig, 9) === 10);
  // C2: calculateGrade — midden
  assert('C2', calculateGrade(testConfig, 7) === 8);
  // ... etc

  // Pariteitscheck PROMEF
  for (let t = 5; t <= 24; t++) {
    const dynamic = calculateGrade(promefConfig, t);
    const original = calculateGradeOriginal(t);
    assert(`G-parity-${t}`, dynamic === original);
  }

  const passed = results.filter(r => r.pass).length;
  console.log(`Tests: ${passed}/${results.length} passed`);
  return results;
}
```

2. **E2E-tests (§14.9)**: handmatig doorlopen als checklist, bij voorkeur na elke implementatiefase.

3. **Referentie-validatie (§14.1)**: na implementatie van AI-parsing, upload `test-formulier.pdf` en controleer de output tegen de verwachte waarden. Dit is de belangrijkste acceptatietest.
