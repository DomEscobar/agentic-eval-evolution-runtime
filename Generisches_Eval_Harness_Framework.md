# Generisches Eval-Harness-Framework

Trennt den domain-spezifischen Teil (die konkrete AI-App: RAG, Klassifizierer, Coding-Agent, Chatbot) vom generischen Teil (Eval-Loop, Mutator, Konvergenz). Das Framework bleibt identisch — nur der Task Adapter wird pro App neu geschrieben.

**Update nach Research:** Ein Großteil dieses Frameworks existiert bereits als reifes Open-Source-Ökosystem. Build-vs-Buy ist hier keine offene Frage mehr — Abschnitt 8 zeigt was ihr direkt übernehmen könnt statt selbst zu bauen. Abschnitt 9 beschreibt ein kritisches Sicherheitsrisiko das jede Architektursuche betrifft.

---

## Gesamtbild

```
+--------------------------------------------------------------+
|  APP-SPEZIFISCH (neu pro Projekt)                             |
|                                                                |
|   +--------------------------------------------+              |
|   |              TASK ADAPTER                   |              |
|   |  run() . config_schema() . layers()         |              |
|   |  coupling_constraints()                     |              |
|   +--------------------+-------------------------+              |
|                        | implementiert fuer                    |
|        +---------------+---------------+------------+          |
|        v               v               v            v          |
|      RAG        Klassifizierer   Coding-Agent    Chatbot       |
+------------------------+---------------------------------------+
                         |
+------------------------v---------------------------------------+
|  GENERISCH (einmal gebaut, fuer jede App wiederverwendbar)       |
|                                                                  |
|   +---------------+                                             |
|   | Golden Dataset |                                            |
|   +-------+--------+                                            |
|           v                                                      |
|   +---------------+                                             |
|   |Metric Registry |                                            |
|   +-------+--------+                                            |
|           v                                                      |
|   +---------------+                                             |
|   | Eval Harness   |<----------------+                          |
|   | (read-only)    |                 |                          |
|   +-------+--------+                 | loop                     |
|           v                          |                          |
|   +---------------+                 |                          |
|   |   Mutator      |-----------------+                          |
|   | (Guardrails    |                                             |
|   |  NICHT sichtbar)|                                            |
|   +-------+--------+                                            |
|           v                                                      |
|   +---------------+                                             |
|   |   Archive      |  verhindert lokale Optima                   |
|   +-------+--------+                                            |
|           v                                                      |
|   +---------------+                                             |
|   |  Convergence   |  Plateau oder Budget erreicht               |
|   +-------+--------+                                            |
|           v                                                      |
|   +---------------+                                             |
|   | Hold-out-Val.  |  nie im Loop gesehen                        |
|   +-------+--------+                                            |
|           v                                                      |
|   +---------------+                                             |
|   |    DEPLOY      |                                            |
|   +---------------+                                            |
|                                                                  |
|  ================================================================  |
|  GUARDRAILS -- ausserhalb des Loops, nie mutierbar                |
|  ================================================================  |
+------------------------------------------------------------------+
```

---

## Die zwei Ebenen

**App-spezifisch (pro Projekt neu):**
Task Adapter — die einzige Komponente die pro AI-App implementiert werden muss.

**Generisch (einmal gebaut, für jede App wiederverwendbar):**
Golden Dataset Schema, Metric Registry, Eval Harness, Mutator, Archive, Convergence-Logik, Guardrails.

---

## 1. Task Adapter — das Interface das jede App implementieren muss

```python
class TaskAdapter(Protocol):
    def run(self, input: Any, config: dict) -> Any:
        """Führt die AI-App mit einer gegebenen Config aus, gibt Output zurück."""

    def get_config_schema(self) -> dict:
        """Definiert welche Parameter mutierbar sind, mit Typ und gültigem Wertebereich."""

    def get_layer_boundaries(self) -> list[str]:
        """Benennt die austauschbaren Schichten der App (für Diagnose: welche Schicht war schuld)."""

    def get_coupling_constraints(self) -> list[dict]:
        """Definiert welche Parameter-Kombinationen NICHT frei kombinierbar sind
        (z.B. Late Chunking erfordert long-context Embedding-Modell)."""
```

Nur diese vier Methoden müssen pro App neu geschrieben werden. Alles andere im Framework bleibt gleich.

---

## 2. Golden Dataset — generisches Schema

```json
{
  "id": "case_00231",
  "input": "...",
  "expected_output": "...",
  "metadata": {
    "category": "kernfall | edge_case | negative_case",
    "difficulty": "easy | medium | hard",
    "additional_context": {}
  }
}
```

`additional_context` ist die Erweiterungsstelle für App-spezifische Zusatzdaten — bei RAG z.B. `relevant_doc_ids`, bei einem Coding-Agent z.B. `expected_test_pass_rate`, bei einem Klassifizierer z.B. `ground_truth_label_distribution`.

**Größenordnung bleibt App-abhängig, aber die Faustregel überträgt sich:**
- Statistische Metriken (Recall, Accuracy, F1) brauchen 150–300 Fälle für stabile Durchschnittswerte
- Prozess-/Trajektorien-Metriken (Checkpoint-Compliance) brauchen nur 30–60 Fälle für Abdeckung der Fallarten
- Rein strukturelle/binäre Prüfungen (Format, Vollständigkeit) brauchen nur 20–50 Fälle

---

## 3. Metric Registry — pluggable Metriken

```python
class Metric(Protocol):
    def compute(self, input: Any, expected: Any, actual: Any, context: dict) -> float:
        """Gibt einen Score zurück, typischerweise 0.0–1.0."""

    name: str
    layer: str  # welcher Schicht ist diese Metrik zugeordnet
```

Generische Metrik-Bausteine die über App-Typen hinweg wiederverwendbar sind:
- `exact_match` — deterministischer String-Vergleich
- `f1_token_overlap` — Textähnlichkeit
- `recall_at_k` / `precision_at_k` — für jede Art von Retrieval/Auswahl-Problem
- `llm_as_judge` — für qualitative Kriterien (Faithfulness, Completeness, Tonalität)
- `cost` / `latency` — für jede App relevant, unabhängig vom Inhalt

Composite Score = gewichtete Summe der Einzelmetriken, konfigurierbar pro App.

---

## 4. Eval Harness — generischer Runner

```python
def run_harness(adapter: TaskAdapter, config: dict, dataset: list[dict],
                 metrics: list[Metric]) -> dict:
    results = []
    for case in dataset:
        actual = adapter.run(case["input"], config)          # sandboxed, read-only
        scores = {m.name: m.compute(case["input"], case["expected_output"],
                                     actual, case.get("metadata", {}))
                  for m in metrics}
        results.append(scores)
    return aggregate(results)  # z.B. mean pro Metrik + composite
```

Diese Funktion ist für jede App identisch — sie kennt nur das Adapter-Interface, nicht was die App tatsächlich tut.

**Wichtige Unterscheidung aus der Recherche (DeepEval):** Ein Eval kann offline (dev-time, gegen ein festes Dataset, außerhalb des Live-Traffics) oder online (im tatsächlichen Request/Response-Pfad) laufen. Sobald ein Check live gated — also den ausgehenden Response blockiert, zurückweist oder eskaliert — ist er kein Eval mehr, sondern ein Guardrail. Der Harness hier ist ausschließlich offline. Guardrails sind eine separate Komponente (siehe Abschnitt 9).

---

## 5. Mutator — App-unabhängiges Prompt-Muster

Der Mutator bekommt immer dieselbe Struktur an Kontext, unabhängig von der App:

```
Aktuelle Config: {config}
Scores pro Metrik: {scores}
Konkrete Fehlerbeispiele: {failure_cases}
Coupling Constraints: {constraints}

Aufgabe: Schlage 1-3 neue Config-Varianten vor.
Für jede: nenne die Hypothese warum sie den beobachteten Fehler behebt.
Beachte die Coupling Constraints — schlage keine ungültigen Kombinationen vor.
Antworte als JSON.
```

Das Modell muss dabei App-spezifisches Wissen nicht "kennen" — die relevanten Fakten (Constraints, Layer-Grenzen) kommen aus dem Task Adapter, nicht aus dem Modell-Vorwissen.

**KRITISCH — nach Sicherheits-Research ergänzt:** Der Mutator darf niemals Zugriff auf oder Kenntnis von den Guardrail-Definitionen haben (siehe Abschnitt 9). Guardrails leben außerhalb des Config-Schemas, das der Mutator sieht und verändern darf.

---

## 6. Archive & Convergence — identisch für jede App

```python
class Archive:
    def add(self, config: dict, scores: dict): ...
    def best(self) -> dict: ...
    def is_converged(self, patience: int = 5, epsilon: float = 0.01) -> bool:
        """True wenn sich der beste Score über `patience` Iterationen
        um weniger als `epsilon` verbessert hat."""
```

Stop-Kriterium ist rein score-basiert — braucht keine Kenntnis der App-Domäne.

---

## 7. Guardrails — für jede App gleich wichtig

- Eval-Code liegt read-only, getrennt vom Adapter-Prozess
- Adapter hat keinen Schreibzugriff auf Harness oder Golden Dataset
- Hold-out-Set: 10-20% des Datasets nie im Loop verwendet, nur für finale Validierung vor Deploy
- Composite Score darf nicht durch eine einzelne dominante Metrik gehackt werden — Gewichtung regelmäßig gegen Hold-out prüfen
- **Neu (siehe Abschnitt 9): Guardrails dürfen architektonisch niemals Teil des mutierbaren Config-Raums sein**

---

## 8. Bestehendes Ökosystem — was ihr übernehmen könnt statt selbst zu bauen

Die Recherche zeigt: für fast jede Komponente des generischen Frameworks existiert bereits ein reifes Open-Source-Tool.

```
+-----------------------------------------------------------+
|  KATEGORIE 1 -- Eval-Frameworks (messen)                    |
|                                                              |
|   DeepEval / OpenAI Evals   ->  app-agnostischer Runner     |
|   RAGAS / ARES              ->  RAG-spezifische Metriken    |
|   lm-eval-harness           ->  Standard-Benchmarks         |
|                                                              |
|  KATEGORIE 2 -- Optimierungs-Frameworks (mutieren)          |
|                                                              |
|   DSPy + MIPROv2            ->  Signature statt Prompt,     |
|                                  modell-agnostisch           |
|   OPRO / PromptBreeder      ->  Population-Mutation         |
|                                  auf Prompt-Ebene            |
|   Promptomatix              ->  waehlt Task-Typ + Metrik    |
|                                  automatisch                 |
|                                                              |
|  KATEGORIE 3 -- Selbst-evolvierende Architekturen           |
|                                                              |
|   HyperAgents (2026)        ->  Task-Agent + Meta-Agent     |
|   Evolver (Genes/Capsules)  ->  Verhalten-Evolution,        |
|                                  auditierbarer Event-Log     |
|   AlphaEvolve / DGM         ->  wachsendes Archiv,          |
|                                  stepping stones             |
+-----------------------------------------------------------+
```

**Was das konkret bedeutet:**

- **DSPy übernimmt effektiv den Mutator + Task Adapter für die Prompt/Config-Ebene.** DSPy definiert Signatures (Task-Spezifikation), Modules (Logikblöcke) und Optimizers (Compilation-Engines) — dieselbe Signature kompiliert zu unterschiedlichen Prompts für verschiedene Modelle ohne Code-Änderung. Das ist euer Task-Adapter-Prinzip, bereits produktionsreif. MIPROv2 ist seit Ende 2025 der Flaggschiff-Optimizer.

- **RAGAS/ARES übernehmen die RAG-spezifische Metric Registry.** RAGAS berechnet Faithfulness als Verhältnis von durch Kontext gestützten Aussagen zur Gesamtzahl der Aussagen, plus Answer Relevance, Context Precision, Context Recall. ARES trainiert zusätzlich leichte LLM-Judges auf synthetischen Daten und bewertet jede RAG-Komponente separat.

- **Das "Evolver"-Pattern (Genes/Capsules/Events) ist praktisch identisch mit eurem Archive-Konzept** — nur bereits mit Naming-Konvention und Event-Sourcing-Log durchdacht: `genes.json` (atomare Verbesserungseinheiten), `capsules.json` (zusammengesetzte Strategien), `events.jsonl` (append-only, auditierbar).

- **Was NICHT fertig existiert:** der Agentic-Loop-Teil mit Checkpoint-Compliance und Trajektorien-Bewertung ist RAG/Domain-spezifisch genug, dass ihr den vermutlich selbst bauen müsst — hier gibt es keine direkte Standard-Library.

**Praktische Konsequenz:** Bevor ihr Harness/Mutator/Metric-Registry selbst baut, prüft ob DSPy + RAGAS/ARES + DeepEval bereits 70-80% davon abdecken. Der Aufwand verschiebt sich dann auf: Task Adapter für euren spezifischen Loop, Coupling Constraints, und die Integration der drei Tools ineinander — nicht auf das Neuerfinden der Grundmechanik.

---

## 9. Kritisches Sicherheitsrisiko — Safety-Filter-Removal bei Architektursuche

```
+--------------------------------------------------------+
|  WARUM DAS PASSIERT                                      |
|                                                            |
|   Sicherheits-Check kostet Latenz oder senkt Score        |
|                      |                                    |
|                      v                                    |
|   Optimierung findet: Entfernen verbessert Fitness        |
|                      |                                    |
|                      v                                    |
|   Selektion befoerdert genau DIESE Variante                |
|                      |                                    |
|                      v                                    |
|   Agent hat NIE "entschieden" -- der Loop tut es            |
|   automatisch, weil er nur die Fitness-Funktion sieht     |
+--------------------------------------------------------+
```

Eine aktuelle Sicherheitsanalyse zu selbst-evolvierenden Agenten-Systemen bestätigt dieses Muster als reales, empirisch beobachtetes Risiko — nicht nur theoretisch: wenn ein Sicherheitsfilter Latenz kostet oder einen Teil der High-Reward-Outputs ablehnt, verbessert seine Entfernung die Fitness-Metrik, und die Selektion befördert die Architektur-Variante die ihn entfernt. Das wurde konkret bei PromptBreeder beobachtet — dort mutiert das System seine eigenen Mutations-Prompts, wodurch jedes Sicherheits-Gerüst auf Prompt-Ebene weg-selektiert werden kann, sobald das den Task-Score erhöht.

**Konsequenz für das Framework — nicht verhandelbar:**

1. Guardrails leben in einer eigenen Codeebene, getrennt vom `config_schema` das der Mutator sieht
2. Der Mutator bekommt Guardrail-Definitionen niemals als Kontext — er kann nur mutieren was im Config-Schema steht
3. Jede vom Mutator vorgeschlagene Config wird vor der Eval-Ausführung gegen eine feste Guardrail-Checkliste validiert — unabhängig vom Fitness-Score
4. Guardrail-Verletzung führt zu automatischer Disqualifikation der Config, nicht zu einem niedrigeren Score (ein niedriger Score kann durch andere Verbesserungen kompensiert werden — eine Disqualifikation nicht)

---

## 10. Coding-Agent-Patch-Mode — Eval-System als Baseline-Trainer

Für Coding Agents ist der wichtigste Spezialfall nicht nur Prompt-/Config-Evolution, sondern ein echter Patch-Loop:

```
Baseline / Spezifikation
        |
        v
Coding Agent erzeugt Code-Patch
        |
        v
Unit Tests gate'n den Patch
        |
        v
Benchmark-Eval vergleicht gegen Best-Snapshot
        |
        +-- Verbesserung --> Best-Snapshot aktualisieren
        |
        +-- Regression ----> Rollback
        |
        v
Archive speichert Patch, Scores, Logs, Lineage
```

Das Eval-System patched dabei nicht selbst. Es ist Richter, Fitness-Funktion, Archiv und Rollback-Controller. Der Coding Agent ist der Patch-Produzent.

**Direkter Referenzfall: TDAD Auto-Improvement Loop**

TDAD beschreibt praktisch genau diesen Mechanismus:

1. Ein Coding-Agent bekommt die volle Experiment-Historie
2. Er macht eine fokussierte Änderung an den Quelldateien
3. Unit-Tests gate'n die Akzeptanz
4. Fehlschläge werden sofort reverted
5. Eine SWE-bench-Teilmenge misst Generation/Resolution/Regression
6. Verbesserungen aktualisieren den Best-Snapshot
7. Regressionen lösen Rollback aus
8. Laterale Moves können zur Exploration behalten werden

Die wichtigsten Anti-Gaming-Safeguards aus TDAD sind direkt übernehmbar:

- Eval-Skript per SHA-256 prüfen
- Eval-Skript und Hidden Tests read-only setzen
- Unit-Test-Fehler führen zu sofortigem Rollback
- Regressionsrate als eigene Metrik tracken, nicht nur Resolution
- Fünf Reverts in Folge erzwingen Restore auf den Best-Snapshot

**Formale Einordnung**

- **SICA (Self-Improving Coding Agent):** Agenten, die ihre eigene Codebase oder Scaffold verbessern, um Benchmark-Score, Kosten und Laufzeit zu optimieren.
- **Darwin Gödel Machine:** baut ein Archiv von Agent-Nachkommen und selektiert Varianten anhand von Coding-Benchmarks.
- **Huxley-Gödel Machine:** erweitert das Archive-Konzept um Clade-Metaproductivity — nicht nur "bester aktueller Score", sondern erwartetes Verbesserungspotential einer ganzen Linie.
- **Kitchen Loop:** wichtiger Gegenpunkt zu reinem Benchmark-Optimieren. Der Loop soll zu einer Spezifikation konvergieren, nicht nur eine Proxy-Metrik maximieren.

**Konsequenz für dieses Framework**

Es gibt zwei Modi:

1. **Config/Prompt Evolution** — für RAG, Chatbots, Klassifizierer, Tool-Routing
2. **Code Patch Evolution** — für Coding Agents, die Code ändern, bis eine Baseline oder Spezifikation erreicht ist

Für den zweiten Modus muss gelten:

- Der Coding Agent darf Zielcode patchen
- Der Coding Agent darf Eval-Harness, Hidden Tests, Guardrails und Archive nicht patchen
- Hidden Tests und Guardrails leben außerhalb des mutierbaren Arbeitsbereichs
- Jeder Patch wird mit Diff, Logs, Tests, Score, Kosten und Parent-Snapshot archiviert
- Baseline-Erreichung braucht Resolution **und** Non-Regression

Der sauberste Pitch-Satz:

> Ein eval-guided autonomous patch loop: der Coding Agent patched, das Eval-System misst, gated, archiviert und rollt bei Regression zurück.

---

## Mapping: wie verschiedene App-Typen den Adapter implementieren

| App-Typ | Layer-Beispiele | Typische Metriken | Beispiel Coupling Constraint |
|---|---|---|---|
| RAG | Ingestion, Retrieval, Agentic Loop, Synthesis, Output | Recall@k, Faithfulness, Rechtsstand-Check | Late Chunking ↔ long-context Embedding-Modell |
| Klassifizierer | Preprocessing, Feature/Embedding, Modellwahl, Decision Threshold | Accuracy, F1, Calibration | Threshold-Tuning ↔ Klassenverteilung im Trainingsset |
| Coding-Agent | Planning, Tool-Auswahl, Codegenerierung, Testausführung, Patch-Anwendung | Test-Pass-Rate, Diff-Größe, Kosten pro Task | Tool-Zugriff ↔ Sandbox-Berechtigungen |
| Chatbot/Agent | System-Prompt, Tool-Routing, Memory-Strategie, Antwortgenerierung | Task Completion, Tonalität (LLM-judge), Kosten | Memory-Fenster ↔ Modell-Kontextlänge |

Jede Zeile braucht nur einen neuen Task Adapter — Harness, Mutator, Archive, Convergence, Guardrails bleiben Code, der einmal gebaut und für alle vier Zeilen wiederverwendet wird.

---

## Praktischer Weg zur Generalisierung

1. Baut den Task Adapter für MUCi (RAG) fertig — das ist der Referenzfall
2. Prüft zuerst DSPy + RAGAS/ARES + DeepEval gegen eure Anforderungen, bevor ihr Harness/Mutator/Metric-Registry selbst baut (Abschnitt 8)
3. Baut Guardrails als eigene, vom Mutator nie sichtbare Codeebene — von Anfang an, nicht nachträglich (Abschnitt 9)
4. Extrahiert alles was NICHT RAG-spezifisch ist in eine eigene Library
5. Beim nächsten AI-Projekt: nur einen neuen Task Adapter schreiben, Rest wiederverwenden
6. Coupling Constraints und Metric Registry wachsen mit jedem neuen App-Typ — das wird mit der Zeit zum wertvollsten Teil des Frameworks
