# ev-llm-diagnostics-eval
Domain-expert-verified comparison of Llama and Gemma on 20 EV/HV diagnostic scenarios
EV/HV Diagnostics: Domain-Expert-Verified LLM Eval

A small, human-verified comparison of two open models (Llama, Gemma) on 20 real-world heavy-vehicle EV diagnostic scenarios example HV contactors, BMS faults, CAN communication, insulation faults, thermal management, drive inverters, and related subsystems.

Each model's answers were scored by a reviewer with hands-on BMS/HV systems experience (verdict, missing information, confidence 1–5)not by another model. That's the point of the project: a small, carefully-reasoned, human-verified eval rather than a large but shallow automated one.

Headline result

Raw scores look nearly identical between the two models. The real finding is a failure taxonomy that shows up consistently across both:

Scope hallucination:one model silently substitutes the wrong subsystem for an entire answer (BMS → BCM) rather than getting a fact wrong.
Backwards causal mechanism:a pre-charge circuit explanation gets the direction of causality reversed.
Diagnostic-tool conflation:both models treat a CAN bus signal check as equivalent to a plain multimeter DC-voltage reading.
PPE-before-diagnosis sequencing bias:the strongest, most quantifiable finding. Both models put on PPE / disconnect the battery before checking the fault code on 13 of 20 answers each, including on the 9 questions that describe a purely code- or state-reported symptom with no physical work implied yet.
Domain-nuance blindness:both models miss question-specific clues (e.g. "light load" as a hint toward a sensing-wire fault rather than a real cell overvoltage).

Full write-up, methodology, and per-question reasoning: see EV_Diagnostic_LLM_Eval_Report.md.

Files
EV_Diagnostic_LLM_Eval_Report.md — full report: methodology, failure taxonomy, comparative analysis, next steps
llama_ev_troubleshooting_eval_scored.csv — all 20 questions, Llama's answers, and reviewer scoring
gemma_ev_troubleshooting_eval_scored.csv — all 20 questions, Gemma's answers, and reviewer scoring
About this project

This is my first project moving from AI research interest into hands-on model evaluation. Background is in BMS/HV systems, which is what the domain scoring in this repo is grounded in.

Next steps
Add a third (frontier) model as a reference point
Expand the PPE-before-diagnosis check into a repeatable scoring rubric
Revisit the closer-call "Partial" verdicts with a second reviewer pass
