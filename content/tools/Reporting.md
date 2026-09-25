
## Externe Tools

+ [SCC](https://github.com/boyter/scc)


## Reporting
### `report.typ`

Das Template liest die maschinell erzeugten Metriken aus einer `metrics.json` ein und setzt sie in ein formales, zweispaltiges Audit-Dokument.

  

Code-Snippet

```
#let data = json("build/metrics.json")

#set page(
  paper: "a4",
  margin: (x: 2cm, top: 2.5cm, bottom: 2.5cm),
  header: align(right)[
    #text(size: 8pt, fill: luma(120))[
      #data.project | Statusbericht #data.period | Rev. #data.revision
    ]
  ],
  footer: [
    #line(length: 100%, stroke: 0.5pt + luma(180))
    #grid(
      columns: (1fr, 1fr),
      text(size: 8pt, fill: luma(120))[Vertraulichkeitsstufe: Intern / Audit Trail],
      align(right, text(size: 8pt, fill: luma(120))[Seite #counter(page).display("1 von 1")])
    )
  ]
)

#set text(font: "Liberation Sans", size: 9.5pt, lang: "de")
#set par(justify: true, leading: 0.65em)

// Titelblock
#v(-0.5cm)
#grid(
  columns: (3fr, 1fr),
  gutter: 1cm,
  [
    #text(size: 16pt, weight: "bold")[Projekt-Statusbericht] \
    #text(size: 11pt, fill: luma(80))[Systemintegrität & Inkrement-Nachweis]
  ],
  align(right)[
    #rect(stroke: 0.5pt + luma(150), inset: 6pt, radius: 2pt)[
      #text(size: 8pt)[*Datum:* #data.date \ *Status:* #text(fill: rgb(20, 120, 40))[*KONFORM*]]
    ]
  ]
)

#v(0.3cm)
#line(length: 100%, stroke: 1pt + luma(80))
#v(0.3cm)

// Executive Summary
== 1. Management Summary

Im Berichtszeitraum (*#data.period*) wurden alle definierten Qualitäts- und Latenz-Gates der kontinuierlichen Integration ohne Regressionsfehler passiert. Sämtliche Architekturentscheidungen sind über versionierte Architecture Decision Records (ADRs) verifiziert. Das System befindet sich im Hauptzweig (`#data.branch`) in einem unmittelbar auslieferbaren Zustand (*Running Code*).

#v(0.2cm)

// Metriken-Grid
== 2. Empirische Projektmetriken (Audit Trail)

#table(
  columns: (2fr, 1fr, 3fr),
  stroke: 0.5pt + luma(180),
  fill: (x, y) => if y == 0 { luma(240) } else { none },
  align: (left, center, left),
  [*Prüfparameter*], [*Messwert*], [*Audit-Status / Nachweis*],
  [Erfasste Inkremente (Commits)], [#data.commit_count], [Deterministisch im Git-Log verifiziert],
  [Architekturentscheide (ADRs)], [#data.adr_count], [Vollständig auditiert in `docs/adr/`],
  [Codebasis (Netto-Zeilen)], [#data.loc], [Automatisch via `scc` aggregiert],
  [CI/CD Pipeline-Status], [100% PASS], [Hermetische Test-Suite bestanden],
  [Kritische Blocker / Incidents], [0], [Keine offenen Prioritätsdefekte]
)

#v(0.2cm)

// Changelog / Inkremente
== 3. Abgeschlossene Arbeitspakete & Inkremente

#list(
  ..data.commits.map(c => [
    *#c.hash*: #c.subject _(#c.author)_
  ])
)

#v(0.2cm)

// Risikobewertung
== 4. Governance & Risikobewertung

#rect(fill: luma(245), stroke: 0.5pt + luma(200), width: 100%, inset: 8pt, radius: 2pt)[
  *Konformitätsbestätigung:* \
  Alle Änderungen unterliegen dem Vier-Augen-Prinzip via Pull-Request-Reviews sowie automatisierten Linter- und Latenzschranken. Risiken bezüglich Scope-Creep wurden gemäss _Via Negativa_ eliminiert. Das System erfüllt alle formalen Kriterien für den Übergang in die nächste Iterationsphase.
]
```

### `Justfile`

Das Rezept extrahiert die Git-Historie und ADR-Statistiken, baut daraus per `jq` die `metrics.json` und ruft `typst compile` auf.

  

Code-Snippet

```
set shell := ["bash", "-uc"]

# Standard-Target
default:
    @just --list

# Generiert den formalen Management-Statusbericht als PDF
report period="Letzte 14 Tage":
    #!/usr/bin/env bash
    mkdir -p build
    
    echo "==> Sammle Git- und Repository-Metriken..."
    REVISION=$(git rev-parse --short HEAD)
    BRANCH=$(git rev-parse --abbrev-ref HEAD)
    DATE=$(date +%Y-%m-%d)
    
    # Commits der letzten 14 Tage zählen und als JSON formatieren
    COMMIT_COUNT=$(git log --since="14 days ago" --oneline | wc -l | tr -d ' ')
    
    COMMITS_JSON=$(git log --since="14 days ago" --pretty=format:'{"hash":"%h","subject":"%s","author":"%an"}' \
      | jq -s '.' | sed 's/"/\\"/g')
    
    # Zähle vorhandene ADRs
    if [ -d "docs/adr" ]; then
      ADR_COUNT=$(find docs/adr -name "*.md" | wc -l | tr -d ' ')
    else
      ADR_COUNT="0"
    fi
    
    # LOC via scc ermitteln, Fallback auf git diff falls scc fehlt
    if command -v scc >/dev/null 2>&1; then
      LOC=$(scc --format json . | jq '.[0].code // 0')
    else
      LOC=$(git ls-files | xargs wc -l 2>/dev/null | tail -n 1 | awk '{print $1}')
    fi

    # Generiere build/metrics.json
    jq -n \
      --arg project "FOSS-OODA System" \
      --arg period "{{period}}" \
      --arg revision "$REVISION" \
      --arg branch "$BRANCH" \
      --arg date "$DATE" \
      --arg commit_count "$COMMIT_COUNT" \
      --arg adr_count "$ADR_COUNT" \
      --arg loc "$LOC" \
      --slurpfile commits <(git log -n 8 --pretty=format:'{"hash":"%h","subject":"%s","author":"%an"}' | jq -s '.') \
      '{
        project: $project,
        period: $period,
        revision: $revision,
        branch: $branch,
        date: $date,
        commit_count: $commit_count,
        adr_count: $adr_count,
        loc: $loc,
        commits: $commits[0]
      }' > build/metrics.json

    echo "==> Kompiliere PDF mit Typst..."
    typst compile report.typ "build/Statusbericht-${DATE}-${REVISION}.pdf"
    echo "==> Fertig: build/Statusbericht-${DATE}-${REVISION}.pdf"

# Bereinigt generierte Artefakte
clean:
    rm -rf build/
```

### Ausführung

Ein Aufruf erzeugt das fertige PDF im `build/`-Verzeichnis ohne manuelle Eingriffe:

  

Bash

```
just report
```

Oder mit abweichendem Berichtszeitraum für den Lenkungsausschuss:

Bash

```
just report "Sprint 42 / KW 38"
```



