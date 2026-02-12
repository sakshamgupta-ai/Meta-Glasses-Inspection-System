# Inspector Assist – App 1  
### Fire Extinguisher Inspection Automation (Human-First, Evidence-Backed)

---

## Overview

Inspector Assist – App 1 is a **human-centric inspection assistant** designed to automatically populate a fire extinguisher compliance checklist from **first-person inspection videos** (e.g., Meta glasses).

The system listens to what the inspector **says**, extracts **explicit inspection decisions**, and attaches **supporting visual evidence frames** from the video — without hallucinating, guessing, or overriding human judgment.

> **Core Philosophy:**  
> Human speech determines the decision.  
> Computer vision only provides supporting evidence.

---

## Problem Statement

Manual review of inspection videos is:
- Time-consuming  
- Difficult to audit  
- Prone to missed details  

Inspectors already verbalize their findings during walkthroughs.  
This system converts those verbal decisions into a **structured, auditable checklist** with visual references.

---

## Key Features

- 🎙️ **Speech-Driven Decisions** (Whisper Large-v3 / Turbo)
- 🧠 **Structured Reasoning Extraction** (LLM-based)
- 🎞️ **Evidence Frame Retrieval** (CLIP embeddings)
- ⏱️ **Temporal Grounding** (speech ↔ video alignment)
- 📊 **Confidence-Aware Output**
- 🚫 **No hallucinated evidence**
- 🚫 **No decision overrides**

---

## Checklist Items (App-1)

| Item ID | Description |
|------|------------|
| 1 | Presence of fire extinguisher |
| 2 | Annual inspection date |
| 3 | Accessibility / proper mounting |
| 4 | Locking pin & tamper seal |
| 5 | Pressure gauge operable range |
| 6 | Instruction nameplate visibility |
| 7 | Physical damage / corrosion |
| 8 | Professional service date |

---

## System Architecture

### 1. Audio Transcription (Source of Truth)
- Model: **Whisper Large-v3 / Turbo**
- Output: Timestamped transcript segments
- Purpose: Capture inspector intent and decisions

---

### 2. Decision Extraction (LLM)
- Model: **LLaMA-3.x via Groq**
- Extracts structured JSON per checklist item:
```json
{
  "item_id": 5,
  "decision": "YES",
  "reasoning": "the pressure gauge is in green",
  "confidence": 1.0,
  "date": null
}
```
## 3. Temporal Grounding

The system aligns extracted inspection reasoning to transcript segments using
**token-overlap matching**, not exact string matching.

This approach is robust to:
- Paraphrasing by the LLM
- Partial phrases
- Minor transcription differences

### Function
```python
find_reasoning_time(reasoning_text, segments)
```
**Behaviour**
-Returns  (start, end) if a transcript segment confidently matches
-Returns (None, None) if no reliable alignment exists
-Never returns fake or default timestamps

If temporal grounding fails, downstream logic must explicitly handle the
absence of timestamps.

## 4. Frame Sampling & Embedding

-Video frames are sampled uniformly across the full duration
-Image embeddings are computed once per video using CLIP
-Embeddings are stored on GPU for fast similarity search

This step is independent of checklist items and reused across all retrieval
operations.

## 5. Evidence Retrieval Strategy
Two retrieval modes are supported, selected based on item type and availability
of speech-aligned timestamps.

## A. Local (Speech-Anchored) Retrieval

Used for checklist items where speech and visual observation are closely coupled:

-Presence
-Accessibility
-Locking pin & tamper seal
-Physical damage / corrosion

Search window:
```cshap
[start_time - Δ, end_time + Δ]
```
Frames are ranked by CLIP similarity within the scoped time window.

## B. Semi-Global (Search-Required) Retrieval
Used for small or detailed visual targets that may be inspected silently or
before verbal confirmation:
-Pressure gauge
-Instruction nameplate
-Service / inspection tags

This mode is triggered only when no reliable speech timestamp exists.
Global search is restricted to avoid mixing unrelated objects.

## 6. Confidence & Status Logic
Final checklist status and confidence are determined using the following rules:

-If decision is unclear → NEEDS_REVIEW
-If visual evidence is weak or absent → confidence is capped
-Vision never overrides the inspector’s spoken decision

If no usable evidence exists → evidence_frames = [] (no fake frames)

## Output Schema

```json
{
  "item_id": 5,
  "status": "YES",
  "comment": "the pressure gauge is in green",
  "date": null,
  "evidence_frames": [
    {"timestamp": 22.79},
    {"timestamp": 24.39},
    {"timestamp": 20.79}
  ],
  "confidence_score": 1.0
}
```
