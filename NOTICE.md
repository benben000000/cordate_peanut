# Attribution Notice

This repository implements concepts from the following work:

> Garcia, Benedict. *Invisible Pedagogy: An Algorithmic and Neuroplasticity Model for Ambient
> Scaffolding and Implicit Skill Acquisition.* Working paper, v1.0, Zenodo, 30 June 2026.
> DOI: [10.5281/zenodo.21052875](https://doi.org/10.5281/zenodo.21052875)
> License: [CC-BY-4.0](https://creativecommons.org/licenses/by/4.0/)

## Paper-to-implementation mapping

| Paper construct | Implementation artifact |
| --- | --- |
| Human–AI interaction as POMDP | Two-speed loop: fast local observer + slow semantic belief update (`docs/03-SYSTEM-FLOW.md` §3.2) |
| EKF personalization | On-device weight adapter; state = 16 classifier weights + bias + latent fatigue; observation = Gemini label when confidence ≥ 0.70 (`docs/02-TRD.md` §2.2.4) |
| Synaptic Tagging & Capture consolidation | Spaced-recall scheduler: tag cue at +10 min, capture cue at +24 h, max 2 items per friction episode (`docs/02-TRD.md` §2.2.4) |
| Adaptive scaffolding policy | Cue selection FSM with per-channel cooldown 4000 ms and intensity ceiling 0.35 (`docs/05-UIUX-SPEC.md` §5.3) |
| Alternative conceptual framework to standard AI tutoring trajectories | Ambient injection instead of modal/chat tutoring; measured by the mastery curve in `docs/01-PRD.md` §1.5 |

## Required attribution text (if reused)

"Invisible Pedagogy" model by Benedict Garcia (2026), DOI 10.5281/zenodo.21052875, CC-BY-4.0.
