# Website Improvement Backlog

## Erledigt

- ~~Social Links bereinigen~~ — Steam, Telegram, Meetup entfernt
- ~~Meta-Descriptions für alle Blogposts~~ — alle 5 Posts haben jetzt `description` im Front-Matter
- ~~Navigation optimieren~~ — "Other" Dropdown für Chess/Cheatsheets, About Me umbenannt
- ~~Schema.org Structured Data~~ — Person JSON-LD seitenweit, BlogPosting JSON-LD pro Post
- ~~Interne Verlinkung~~ — Cross-Links zwischen Posts und zu Cheatsheets hinzugefügt
- ~~Google Search Console vorbereitet~~ — `google_site_verification` in `_config.yml` bereit, Code muss noch eingetragen werden

---

## Prio 1 - Quick Wins

### Freelance-Seite veröffentlichen
- `freelance.markdown` hat aktuell `published: false`
- Seite publishen und in die Hauptnavigation einbauen
- Sollte erster Eintrag in der Navigation sein ("Services" oder "Angebot")

### Hero-Section auf Startseite
- Aktuell zeigt die Homepage nur Blogposts ohne Kontext
- Hero-Bereich mit Kurzvorstellung: Name, Rolle, Kernkompetenz
- Klarer Call-to-Action Button ("Projekt anfragen" / "Mich buchen")
- Link zur Freelance/Services-Seite
- Bestehende Terminal-Animation kann Teil davon bleiben

---

## Prio 2 - Content & SEO

### Open Graph Default-Image
- Aktuell kein `og:image` gesetzt
- Wenn Blog auf LinkedIn/Twitter geteilt wird, fehlt das Preview-Bild
- Default-Bild erstellen (z.B. Name + Titel + Branding)
- In `_config.yml` als `defaults` für Posts und Pages setzen
- Optional: individuelle OG-Images pro Post

### Projekt-Portfolio / Case Studies
- Neue Seite oder Section: "Projekte" / "Portfolio"
- 2-3 anonymisierte Projektbeschreibungen mit:
  - Branche / Kontext
  - Herausforderung
  - Lösung / Technologien
  - Ergebnis / Impact (messbar wenn möglich)
- Zeigt echte Erfahrung, nicht nur Zertifikate

### Testimonials / Social Proof
- Kundenzitate oder LinkedIn-Empfehlungen einbinden
- Auf der Freelance-Seite oder als eigene Section auf der Startseite
- 2-3 kurze Quotes reichen für den Anfang

### Blogfrequenz erhöhen
- Ziel: mindestens 2 Posts pro Monat
- Content-Ideen:
  - Java-Neuheiten bei jedem Release
  - Kubernetes/Cloud Praxistipps
  - Buchreviews (bereits begonnen)
  - Lessons Learned aus Projekten
  - Tool-Vergleiche und Evaluationen

---

## Prio 3 - Strategisch

### Sprach-Strategie klären
- Aktuell: Mix aus Deutsch und Englisch ohne Trennung
- Optionen:
  - A) Alles Deutsch (Nische, weniger Konkurrenz, Schweizer Markt)
  - B) Alles Englisch (mehr Reichweite, härterer Wettbewerb)
  - C) Zweisprachig mit hreflang-Tags und klarer Trennung
- Entscheidung beeinflusst SEO und Zielgruppe

### Kontaktformular
- Aktuell nur E-Mail-Link als Kontaktmöglichkeit
- Kontaktformular senkt die Hürde für Anfragen
- Optionen: Formspree, Netlify Forms, oder eigene Lösung
- Felder: Name, E-Mail, Projektbeschreibung, Budget (optional)
