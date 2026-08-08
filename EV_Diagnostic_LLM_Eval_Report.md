# Domain-Expert-Verified LLM Diagnostics Benchmark (EV/HV Systems)
### Llama vs. Gemma on 20 Heavy-Vehicle EV Fault Scenarios

**Role:** Domain expert reviewer (BMS/HV systems background)

---

## 1. What this project is

Two open models (Llama, Gemma) were each given the same 20 real-world diagnostic
prompts covering HV contactors, BMS, CAN communication, insulation faults, thermal
management, drive inverters, and related heavy-vehicle EV subsystems. Each answer
was independently scored by a reviewer with hands-on BMS/HV systems experience,
using three fields: a verdict (Yes/Partial/No), what the answer was missing, and a
confidence rating (1–5) in that verdict.

This is **not** an LLM-as-judge eval. The scoring came from domain expertise, not
from another model grading the outputs. That's the actual contribution: a small,
carefully-reasoned, human-verified comparison, rather than a large but shallow
automated one.

## 2. Headline numbers

| Model | Yes | Partial | No | Avg. reviewer confidence |
|---|---|---|---|---|
| Llama | 2 | 15 | 3 | 3.5 / 5 |
| Gemma | 4 | 13 | 3 | 3.55 / 5 |

Both models land in roughly the same place on a raw scorecard — mostly "Partial,"
a handful of outright wrong answers, similar confidence. **The raw numbers alone
undersell what's actually different between them.** The value of this eval is in
the failure taxonomy below, not the tally.

## 3. Failure taxonomy

Five distinct failure patterns showed up repeatedly. Naming them separately matters
more than lumping everything into "wrong," because they have different causes and
different real-world consequences.

### 3.1 Scope hallucination — wrong subsystem, not just a wrong fact
**Llama, Q5 (CAN fault, VCU↔BMS):** the question asks about the Battery Management
System. Llama's answer silently substitutes "Body Control Module (BCM)" instead —
and does it inconsistently within the same sentence ("Body Control Module (BCM) or
Body Control Module (BCM)"), suggesting the model itself wasn't tracking which
subsystem it meant. BMS never appears in the answer. This is more serious than a
wrong spec value: a technician following this answer would inspect the wrong ECU
end-to-end.

**Severity:** highest in this eval. A wrong number is checkable against a datasheet;
a silently wrong target subsystem isn't, until the fix doesn't work.

### 3.2 Backwards causal mechanism
**Gemma, Q9 (pre-charge circuit failure):** the answer states the pre-charge
circuit exists to "slowly charge the battery... before the main battery is fully
charged" — which doesn't hold together even on its own terms. The actual function
is the reverse: the pre-charge resistor limits inrush current into the downstream
DC-link capacitors, using the battery as the source, before the main contactor
closes. The battery is never the target of pre-charge; it's always the source.
Getting a causal mechanism backwards is worse than omitting it, because it actively
misdirects troubleshooting.

### 3.3 Diagnostic-tool / fault-class conflation
**Both models, Q5 and elsewhere:** CAN bus signal integrity gets reduced to "measure
DC voltage with a multimeter." A multimeter can catch a dead short or open circuit
on the bus, but it can't tell you whether frames are being sent, received, or
malformed — that needs termination-resistance measurement (bus unpowered) or a
scope/protocol analyzer. This is a **shared** blind spot across both models, which
is more interesting than either model's individual mistakes — it suggests a gap in
how both were trained to reason about data-bus vs. power-bus diagnostics, not a
one-off error.

### 3.4 Template/safety-boilerplate bias overriding fault-type reasoning
**Gemma, most rows:** nearly every answer opens with "Disconnect the Battery / PPE
/ Ground Yourself" — including for faults that are purely communication, software,
or sensor-calibration issues (Q5, Q9) where full HV de-energization isn't the
correct first step. Llama does this more selectively but still misapplies
ICE-era concepts: "alternator" (Q14 — EVs don't have one) and "Throttle Position
Sensor/Switch" (Q8 — EVs use accelerator pedal position sensors, not throttles).
Both patterns point the same way: the model is pattern-matching "EV fault → generic
safety/terminology template" rather than reasoning from what kind of fault it is.

### 3.5 PPE-before-diagnosis sequencing bias (both models, quantified)

This deserves to be separated out from 3.4 because it's not just an impression —
it holds at nearly identical rates across both models when checked against all 20
rows.

| Model | Answers where PPE/"disconnect the battery" is mentioned before any fault-code/scan-tool step | Answers where a scan/code step comes first |
|---|---|---|
| Llama | 13 / 20 | 3 / 20 |
| Gemma | 13 / 20 | 1 / 20 |

That's already a striking result on its own — two different models, same rate.
But it's worse than a generic safety-first habit once you separate the questions
by type. **Nine of the twenty prompts describe a purely code- or state-reported
symptom with no physical work implied yet**: Q3 (BMS reports imbalance), Q5 (CAN
fault appears), Q9 (precharge fails during init), Q13 (overvoltage warning), Q14
(EPS loses power intermittently), Q17 (SOC drops under load), Q18 (isolation fault
vanishes/reappears), Q19 (BMS blocks drive command), Q20 (APPS mismatch code). For
every one of these, the correct first action is to **read the log or pull the
fault code** — that tells you whether the vehicle needs to be touched at all. Both
models default to donning insulating gloves and disconnecting the battery before
they've looked at the code that would tell them whether that's even necessary.

This is the strongest single finding in the eval, precisely because it's shared
and quantifiable rather than a one-off scoring judgment: **both models sequence
safety procedure ahead of diagnosis on the majority of rows, including on rows
where diagnosis requires no physical access.** It suggests the "EV fault → PPE
first" association is baked in at a level that overrides reasoning about what
*kind* of fault is being described (data/software/reported-state vs. physical/
hands-on).

### 3.6 Domain-nuance blindness (both models, same gap)
Two questions had a specific clue embedded in the prompt that neither model picked
up on:
- **Q13** (overvoltage on a single cell block **under light load**): the "light
  load" detail is the tell — low current means low IR-drop, so a real overvoltage
  under light load more likely points to a loose sense-wire or bad connection
  (a measurement fault) than an actual cell fault. Both models gave generic
  overvoltage-cause lists without engaging with "light load" at all.
- **Q17** (SOC drops 40%→10% under load): both models treat this as a hardware
  fault. Neither raises the more common real-world explanation — that this is often
  a **SOC-estimation artifact**, where voltage sag under high current in an aged or
  cold pack causes the voltage-based SOC algorithm to misreport, rather than an
  actual 30-point loss of charge.

## 4. Comparative read: Llama vs. Gemma

| Category | Llama | Gemma |
|---|---|---|
| Getting the target subsystem right | Weaker — one severe scope hallucination (Q5), ICE-terminology carryovers (Q8, Q14) | Stronger — no subsystem substitution found |
| Getting the underlying mechanism right | Vaguer but rarely flatly wrong | More detailed, but wrong in a way that reads confidently (Q9, Q18) |
| Procedural sequencing | Reasonable, but not cost-ordered (Q4) | Frequently opens with a disconnect-battery ritual regardless of fault type |
| Technical specificity when correct | Q6 IMD test method is fabricated | Q6, Q7 show genuinely better grounded, system-specific answers |

Neither model is categorically better. **Gemma tends to be more confidently wrong
when it's wrong** (Q9, Q14's internal contradiction), while **Llama tends to be
vaguer, with fewer sharp errors but also less specific, actionable detail.** That's
a meaningfully different risk profile for a diagnostic-support tool: Gemma's errors
are more dangerous per-incident; Llama's answers are safer but less useful.

## 5. What neither model does, across all 20 questions

- Neither ever asks for the actual fault code (SPN/FMI for J1939 heavy-vehicle CAN,
  or an OEM DTC) before generating a generic checklist — a real technician's first
  move.
- Neither references applicable standards (ISO 6469 for HV safety, J1939 for
  heavy-vehicle CAN) despite the heavy-vehicle framing in the prompts.
- Both default to near-identical bolded-header, numbered-list formatting regardless
  of whether the fault is electrical, mechanical, sensor, or software — a sign of
  matching output *format* to "EV troubleshooting" rather than tailoring depth to
  fault type.

## 6. Section highlight for a portfolio summary

If you only lead with one finding when sharing this project, make it 3.5. It's the
one backed by a clean count across all 20 rows for both models rather than a
single-question judgment call, which makes it the most defensible claim in the
whole eval and the easiest for someone else to independently verify against the
raw CSVs.

## 7. Suggested next steps

1. **Score remaining ambiguity:** a few "Partial" verdicts above (Q3, Q10, Q11,
   Q15, Q16, Q20) are genuinely close calls between models — worth a second pass
   if this eval is going into a portfolio piece, since reviewer confidence on those
   sits at 3/5.
2. **Add a third model** (e.g., a frontier model) as a reference point — right now
   this is a comparison between two open models with no anchor for "what does a
   strong answer look like."
3. **Turn Section 3 into the paper's spine.** The taxonomy (scope hallucination,
   backwards mechanism, tool conflation, template bias, domain-nuance blindness) is
   the reusable contribution — it would transfer to evaluating any model on
   technical-diagnostic tasks, not just these two on this dataset.

## 8. Files

- `llama_ev_troubleshooting_eval_scored.csv` — full scored rows, Llama
- `gemma_ev_troubleshooting_eval_scored.csv` — full scored rows, Gemma
