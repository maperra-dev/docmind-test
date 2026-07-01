---
unique-name: schema-contenuti-correlazioni
display-name: SCHEMA CONTENUTI E CORRELAZIONI
category: GENERAL
description: Schema sintetico del contenuto di ciascun documento e delle loro correlazioni logiche/operative.
---

# Schema sintetico documenti e correlazioni

## Perimetro
Progetto: **Test Docmind** (`SDQ-DOCMIND`)  
Documenti analizzati: 13

## 1) Contenuto sintetico per documento

| Documento | uniqueName | Contenuto sintetico |
|---|---|---|
| REQUIREMENTS | `requirements` | Requisiti v1/v2 del progetto: performance, analisi errori dati, riconciliazione, tracciabilità requisito→fase, scope e out-of-scope. |
| ROADMAP | `roadmap` | Piano in 10 fasi con dipendenze, criteri di successo e stato avanzamento (fasi 1-2 completate, 3-10 pianificate). |
| STATE | `state` | Stato operativo corrente, metriche di avanzamento, decisioni cumulative, milestone completate e punto di ripartenza. |
| PROJECT | `project` | Charter del progetto: obiettivi, valore, vincoli, contesto tecnico/organizzativo, volumi, issues note e decisioni guida. |
| FEATURES | `features` | Panorama funzionale/capability del lavoro di analisi migrazione (feature map per completezza e priorità). |
| PITFALLS | `pitfalls` | Errori ricorrenti/critici nella migrazione PL/SQL (rischi di corruzione dati, performance e affidabilità) con impatto operativo. |
| STACK | `stack` | Stack tecnico Oracle PL/SQL e pattern di ottimizzazione (parallelismo, sequence, tuning, pratiche DB). |
| SUMMARY | `summary` | Sintesi esecutiva della ricerca: quadro complessivo, risultati principali e direzioni di remediation. |
| ARCHITECTURE | `architecture` | Pattern architetturali SI→NFS: schemi coinvolti, flussi dati, dipendenze e boundary tra source/staging/final. |
| COVERAGE CHECK | `coveragecheck` | Quality gate di copertura: verifica completa di tabelle/fk/query e conferma di completezza delle fasi 1-2. |
| EXECUTION GUIDE | `executionguide` | Runbook operativo: come eseguire la suite di riconciliazione, ordine di esecuzione, parametri e criteri di lettura output. |
| README | `readme` | Entry point della suite: struttura directory, quick start e rimando ai documenti guida principali. |
| TRIAGE GUIDE | `triageguide` | Guida di interpretazione risultati e gestione discrepanze; documento companion all’Execution Guide. |

## 2) Correlazioni tra documenti

### 2.1 Relazioni principali (dipendenza logica)

| Da | A | Tipo correlazione |
|---|---|---|
| PROJECT | REQUIREMENTS | Il charter definisce obiettivi/scope tradotti in requisiti operativi. |
| REQUIREMENTS | ROADMAP | I requisiti sono mappati in fasi e criteri di successo. |
| ROADMAP | STATE | Lo stato misura l’avanzamento rispetto al piano per fasi. |
| SUMMARY | PROJECT | Sintetizza outcome e stato generale rispetto agli obiettivi progetto. |
| ARCHITECTURE | FEATURES | L’architettura abilita/limita le capability analizzate nel landscape funzionale. |
| STACK | ROADMAP | Le scelte tecniche supportano soprattutto le fasi performance/ottimizzazione (3-4). |
| PITFALLS | ROADMAP | I rischi identificati guidano priorità e azioni di remediation nelle fasi analitiche. |
| README | EXECUTION GUIDE | README instrada all’esecuzione dettagliata. |
| EXECUTION GUIDE | TRIAGE GUIDE | Esecuzione prima, interpretazione/triage dopo (relazione companion). |
| EXECUTION GUIDE | COVERAGE CHECK | Le query eseguite sono validate dal controllo di copertura. |
| COVERAGE CHECK | STATE | Le evidenze di copertura alimentano la dichiarazione di completamento fase. |

### 2.2 Flusso consigliato di lettura
1. `project` → `requirements` → `roadmap` → `state`
2. `summary` → `architecture` → `stack` → `features` → `pitfalls`
3. `readme` → `executionguide` → `triageguide` → `coveragecheck`

## 3) Nota sulla correlazione automatica via tag
Il grafo tag corrente non mostra archi (documenti senza tag condivisi), quindi le correlazioni sopra sono **semantiche/operative** e derivate dal contenuto dei documenti.

---
Documento generato automaticamente da Docmind AI per mappare in modo rapido contenuti e relazioni del progetto.