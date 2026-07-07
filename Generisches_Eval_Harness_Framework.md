
# Generisches Eval-Harness-Framework

Trennt den domain-spezifischen Teil (die konkrete AI-App: RAG, Klassifizierer, Coding-Agent, Chatbot) vom generischen Teil (Eval-Loop, Mutator, Konvergenz). Das Framework bleibt identisch — nur der Task Adapter wird pro App neu geschrieben.

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
2. Extrahiert alles was NICHT RAG-spezifisch ist in eine eigene Library (Harness, Mutator, Archive, Convergence)
3. Beim nächsten AI-Projekt: nur einen neuen Task Adapter schreiben, Rest wiederverwenden
4. Coupling Constraints und Metric Registry wachsen mit jedem neuen App-Typ — das wird mit der Zeit zum wertvollsten Teil des Frameworks
