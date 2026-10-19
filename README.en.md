# Verification-First Governance for LLM Agent Systems

> Treating agent output with the skepticism of continuous integration.

[Português](README.md) · **English**

[![version](https://img.shields.io/github/v/release/tedfernandes/squad-research?label=version&color=1f7a50&style=flat-square)](https://github.com/tedfernandes/squad-research/releases)
[![DOI](https://img.shields.io/badge/DOI-10.5281%2Fzenodo.23112880-696d75?style=flat-square)](https://doi.org/10.5281/zenodo.23112880)
[![license CC BY 4.0](https://img.shields.io/badge/license-CC_BY_4.0-696d75?style=flat-square)](https://creativecommons.org/licenses/by/4.0/)
[![ORCID](https://img.shields.io/badge/ORCID-0009--0006--7522--326X-696d75?style=flat-square&logo=orcid&logoColor=white)](https://orcid.org/0009-0006-7522-326X)
[![paper online](https://img.shields.io/badge/paper-online-696d75?style=flat-square)](https://tedfernandes.github.io/squad-research/)

This repository holds a preprint and sanitized illustrative material for a design pattern
observed in a production multi-agent system we call **squad-harness**: a **verification
substrate** that sits beneath the (conventional) company-of-agents organization and keeps it
honest.

The central claim is deliberately narrow:

> The organizational layer of multi-agent systems (roles, coordinators, personas) is commodity.
> The verification layer beneath it is under-explored, and making **"not measured" a first-class
> outcome** is the single most valuable design decision in the system.

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="figures/panel-dark.en.svg">
  <img alt="Real verification-substrate figures: third state (10 of 21 projects with no tests), structural ratchet (50 invariants, 7 execute the artifact), and evaluation coverage (37 cases defined, 17 scored)." src="figures/panel-light.en.svg">
</picture>

## The five mechanisms

1. **The third state.** Every gate returns pass / fail / *indeterminate*. "Not measured" is never
   rounded to "passed"; indeterminate is a distinct exit code and never auto-merges.
2. **The defect-ledger ratchet.** An LLM-free gate of 50 invariants, each provenance-linked to
   a real, previously confirmed defect. Verification prefers *executing the artifact* over
   *string-matching its text* ("mention does not prove existence").
3. **The closing loop.** Each confirmed defect is compiled into either a new mechanical invariant
   or a new behavioral evaluation case, so audits raise a floor instead of aging into a report.
4. **Prompt-change governance.** Prompt edits ship only through a property-based
   evaluation-regression gate. Prompts are code under test.
5. **Untrusted context.** Agent-authored content injected back into the model is framed as
   reference, not instruction, and scanned for injection patterns.

## Read the paper

- Português (primary): [`paper.md`](paper.md) - full preprint (v1.0).
- English: [`paper.en.md`](paper.en.md) - full preprint (v1.0).
- Online (GitHub Pages): https://tedfernandes.github.io/squad-research/

## Status and honesty note

This is a **preprint / experience report**, not peer-reviewed work, from a **single-operator**
deployment. The evaluation separates mechanisms that are *fully exercised* from those merely
*wired*, and states threats to validity plainly (N=1, self-reported, author is also evaluator).
See Sections 5 and 6 of the paper.

No client data, production-security detail, or verbatim operational configuration appears here.
All figures are anonymized and all code excerpts are sanitized illustrations of mechanism.

## Update, 19 October 2026

The paper (v1.0) describes the system at the measured commit and **has not been changed**. The source has grown since; the numbers below were checked against the plugin repository, not rounded:

- **35 agents in 4 teams** (11 dev, 8 marketing, 8 commercial and 8 board), instead of the paper's 27 in 3 teams. The commercial team is new.
- **47 skills** and **54 invariants** in the ratchet (the paper measures 40 skills and 50 invariants).
- **41 golden cases defined across 8 agents.** How many have a recorded score is still lower than that; the paper reports 17 scored across 4 agents at the measured commit and that figure is not updated here.
- **An example of mechanism 2 at work.** The global-prospecting skill promised a do-not-contact list, automatic opt-out and a send brake, but it was only prompt text. This was treated as "a mention does not prove existence": the rules became code (`tools/prospeccao/`), and the ratchet gained invariant `PROSP1`, which executes the test suite. Mechanism 4 showed up too: the first evaluation case for the prospecting agent scored 0.00 (anti-example violated, it filled the quota with accounts from an excluded country); after a minimal prompt change it rose to 0.80, with no regression on the other three cases.
- **The same principle applied to payment.** In the same skill, the Pix receipt the client sends **does not confirm** payment: only a human validation counts, and outside Brazil only reading the state in Stripe (`paid`). The Pix key is sent only after the client confirms in writing that they want the service. It is "not measured never becomes passed" applied to money. This is a prompt and procedure rule, still without an end-to-end automated test.

This remains a single-operator report with no peer review.

## Citing

Author: Ted Fernandes ([ORCID 0009-0006-7522-326X](https://orcid.org/0009-0006-7522-326X)).

A `CITATION.cff` is provided (GitHub renders a "Cite this repository" button). DOI (concept, all
versions): [10.5281/zenodo.23112880](https://doi.org/10.5281/zenodo.23112880).

## License

- **Prose** (`paper.md`, `paper.en.md`, this README): [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).
- **Code snippets** (illustrative): [MIT](LICENSE).
