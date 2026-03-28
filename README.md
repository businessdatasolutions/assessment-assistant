# Assessment Tool

Een webbased beoordelingstool voor HBO-docenten die studenten beoordelen op basis van een beoordelingsformulier. De tool leest automatisch het PDF-formulier in via AI en configureert zich op basis van de criteria en niveaus daarin.

## Wat doet de tool?

- **PDF-formulier inlezen**: Upload je beoordelingsformulier als PDF; de AI haalt criteria, niveaus en indicatoren er automatisch uit
- **Planning**: Maak groepen, voeg studenten toe en plan beoordelingsmomenten in een kalender
- **Beoordelen**: Score studenten per criterium met kleurgecodeerde knoppen; voeg observaties en notities toe
- **Resultaten**: Automatische berekening van totaalscore en beoordeling (voldoende/onvoldoende)
- **Feedback**: Genereer individuele feedbackoverzichten per student
- **Exporteren**: Exporteer resultaten als CSV of maak een volledige JSON-back-up

## Hoe starten?

De tool is één enkel HTML-bestand, zonder installatie of server.

1. Open `index.html` in een moderne webbrowser (Chrome, Firefox, Edge)
2. Kies een startoptie:
   - **PDF uploaden**: upload je beoordelingsformulier (vereist een API-sleutel)
   - **Handmatig configureren**: stel criteria en niveaus zelf in
   - **Demo laden**: bekijk de tool met voorbeelddata

### API-sleutel instellen

Voor het automatisch inlezen van een PDF-formulier heb je een API-sleutel van Anthropic (Claude) of Google (Gemini) nodig. De sleutel wordt lokaal opgeslagen in je browser en nooit naar een server verstuurd.

- Anthropic Claude: [console.anthropic.com](https://console.anthropic.com)
- Google Gemini: [aistudio.google.com](https://aistudio.google.com)

## Gegevensopslag

Alle data wordt opgeslagen in de lokale opslag van je browser (localStorage). Er is geen server, geen account en geen internet nodig na de eerste PDF-verwerking. Gebruik de exportfunctie regelmatig om een back-up te maken.

## Techniek

- Eén HTML-bestand, geen installatie
- Vanilla JavaScript en CSS + Tailwind CSS
- Werkt offline (na eerste PDF-verwerking)
- AI-integratie via Anthropic Claude API of Google Gemini API

## Projectstructuur

```
index.html          Volledige applicatie
docs/
  PRD-generic.md    Productspecificatie
  TDD.md            Technisch ontwerp
  tasks.md          Implementatietaken
test/               Testbestanden (PDF's en CSV's)
dev/
  review.md         Code review notities
```
