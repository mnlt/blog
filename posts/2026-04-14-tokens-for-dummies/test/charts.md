# Chart test

## Context vs Usage

```mermaid
xychart-beta
    title "Context vs Usage (3 turns)"
    x-axis ["Start", "Turn 1", "Turn 2", "Turn 3"]
    y-axis "Tokens (K)" 0 --> 70
    line [18, 20, 23, 26]
    bar [0, 20, 43, 69]
```

Context (line) grows slowly: 18K → 26K.
Usage (bars) grows fast: 0 → 69K.

## Same research at different session depths

```mermaid
xychart-beta
    title "Same 14K research — different cost"
    x-axis ["Turn 1", "Turn 30", "Turn 100", "Turn 250"]
    y-axis "Usage cost (K tokens)" 0 --> 3500
    bar [200, 480, 1200, 3500]
```

Same answer. 17x more expensive at turn 250.
