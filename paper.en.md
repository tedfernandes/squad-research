# Verification-First Governance for LLM Agent Systems: Treating Agent Output with the Skepticism of Continuous Integration

**Ted Fernandes**
Independent researcher and practitioner
`tedfernandes@gmail.com`
ORCID: [0009-0006-7522-326X](https://orcid.org/0009-0006-7522-326X)

*Preprint, version 1.0 (2026-09). Experience report / systems paper.*

*(Versão em português, principal: [`paper.md`](paper.md).)*

---

## Abstract

Multi-agent frameworks built on large language models (LLMs) have converged on a common design: a company of role-specialized agents coordinated toward a goal. This organizational metaphor is now well established (MetaGPT, ChatDev, CrewAI, AutoGen). What remains under-addressed is a harder problem: **how does such a system verify its own output, and how does it avoid reporting success when nothing was actually checked?** We report on a production agent system we call "squad-harness" operating a real portfolio of ~36 software and marketing projects across 6 clients, run by a single human operator. The system's distinguishing contribution is not its 27-agent, three-team organization, which is conventional, but a **verification substrate** layered beneath it. We describe five mechanisms: (1) an explicit *third outcome state* ("indeterminate") enforced across every gate, encoding the rule that absence of measurement is never a pass; (2) a defect-ledger *structural ratchet*, an LLM-free gate of 50 invariants in which every invariant is provenance-linked to a previously confirmed defect, and where verification prefers *executing the artifact* over string-matching its text; (3) a *closed learning loop* in which each confirmed defect is compiled into either a new mechanical invariant or a new behavioral evaluation case, so that audits monotonically raise a floor; (4) *prompt-change governance* via property-based evaluation-regression gates, treating prompts as code under test; and (5) *untrusted-by-default* handling of agent-authored context injected back into the model. We give a candid evaluation grounded in the system's own artifacts, including the mechanisms that are fully exercised and those that are not, and we discuss threats to validity honestly: this is a single-operator, self-reported deployment. Our claim is narrow and, we argue, defensible: the organizational layer of agent systems is commodity; the verification layer is where engineering discipline, and the interesting design space, actually lives.

**Keywords:** LLM agents, multi-agent systems, software engineering, verification, evaluation, agent governance, experience report.

---

## 1. Introduction

The dominant pattern in LLM agent systems is orchestration: a coordinator decomposes a task, routes sub-tasks to role-specialized agents, and composes their outputs. Frameworks such as MetaGPT [1], ChatDev [2], AutoGen [3], CrewAI [4], and LangGraph [5] make this pattern easy to build. The metaphor of "a software company staffed by agents" is, at this point, a solved design exercise.

There is a quieter failure mode that these frameworks largely inherit rather than fix. **An agent system that orchestrates but does not verify will confidently report success on work it never checked.** When an agent finishes a coding task in a repository that has no test suite, the naive outcome is binary: pass or fail. "Nothing was verified" is silently rounded to "passed," a pull request opens, and, with automatic merge enabled, unverified code ships. The problem is not that agents are unreliable; it is that the *reporting contract* has only two states where it needs three.

This paper reports on a production system, which we call *squad-harness*, that was built to run a real, revenue-bearing portfolio rather than a benchmark. Over roughly a year of daily use, its design center of gravity migrated away from the agents and toward the machinery that keeps the agents honest. We make an explicit, deliberately narrow claim:

> The organizational layer of multi-agent systems (roles, coordinators, personas) is commodity and not a research contribution. The **verification-first governance substrate** beneath it, however, is under-explored, and the specific discipline of making "not measured" a first-class outcome is both generalizable and, in our experience, the single most valuable design decision in the system.

We are candid that this is an *experience report* from a single-operator deployment, not a controlled study. Section 6 treats the resulting threats to validity directly. Our contribution is a design pattern and a working existence proof, supported by the system's own longitudinal artifacts, not a benchmarked performance claim.

## 2. Background and related work

**Multi-agent orchestration.** MetaGPT [1] encodes standard operating procedures over roles (product manager, architect, engineer, QA), and ChatDev [2] frames development as a virtual company progressing through phases. AutoGen [3] provides conversable agents and group chat; CrewAI [4] offers role-based "crews" with sequential or hierarchical processes; LangGraph [5] exposes agent/state orchestration as a graph. Our system's three-team, coordinator-led structure is squarely within this space and we do not claim novelty for it.

**Reflection and memory.** Reflexion [6] has agents verbally reflect on failures to improve subsequent attempts; Generative Agents [7] maintain a memory stream with reflection; MemGPT [8] manages long-term memory as a tiered system. These establish that agents can learn from their own traces. Our learning loop differs in *where the lesson lands*: rather than into a retrievable memory store, a confirmed defect is compiled into a mechanical invariant or a regression test, so the improvement is enforced by a gate, not merely available for recall.

**Evaluation of agents and prompts.** SWE-bench [9] and similar benchmarks measure task success rates; tooling such as promptfoo and LangSmith [10] provides prompt/agent evaluation harnesses. These are typically applied *externally*, to measure a system. We instead wire property-based evaluation *into the release path* of the agents themselves: a prompt change that regresses a golden case cannot ship.

**Software-engineering gates.** Continuous integration (CI), invariants, and regression suites are standard practice in software engineering. The contribution here is not inventing gates but *turning them inward*: applying CI-grade skepticism to the configuration and output of an agent system, and treating the agent's own written artifacts as untrusted input.

The gap we address sits between these literatures: orchestration frameworks assume verification is the user's problem; evaluation tooling sits outside the running system; reflection improves behavior without hard-enforcing it. We report what it looks like to make verification the spine of the system.

## 3. System context (deployment, not contribution)

*squad-harness* is a plugin for an agent runtime (Claude Code) and comprises 27 agents in three teams (11 engineering, 8 marketing, 8 executive/"board"), each team led by a coordinator, all reporting to a single human operator who acts as the decision authority. It ships 40 procedural "skills" (runnable procedures with explicit gates, not prompt snippets) and a body of tooling: a structural gate, a headless task runner, git and lifecycle hooks, an evaluation harness, and a publish-parity gate. The system runs a live portfolio of approximately 36 projects across 6 clients.

We describe this only as the environment in which the mechanisms below were forged. The organizational design is intentionally *not* the claim. To protect third parties, all figures in this paper are anonymized: no client, project, or production-security detail from the deployment appears here, and code excerpts are sanitized illustrations of mechanism, not verbatim configuration.

## 4. The verification substrate

### 4.1 The third state: absence of measurement is never a pass

Every gate in the system reports one of **three** outcomes, not two: pass, fail, or *indeterminate*. Indeterminate is the outcome when the check could not actually run, for example a coding task in a repository that defines no lint, type-check, or test scripts. The governing maxim, repeated verbatim across the codebase, is *"skipping is not passing"* and *"absence of measurement is never a positive signal."*

The consequences are mechanical, not advisory. The headless task runner, when it finds no gate to execute, does not return success; it returns an indeterminate verdict, labels the resulting pull request as unverified, and refuses to enable automatic merge. Indeterminate is a distinct process exit code (2), separate from pass (0) and fail (1), specifically so that wrapping automation and CI cannot mistake "not checked" for "checked and fine."

```text
# Sanitized decision core of the task runner
gate = run_gate(project)          # {passed, indeterminate}
verdict = read_reviewer_verdict(review_text)  # APPROVE | APPROVE_WITH_NOTES | REJECT | ILLEGIBLE | ABSENT

blocks   = verdict in {REJECT, ILLEGIBLE, ABSENT}
deliver  = (gate.passed or gate.indeterminate) and not blocks
automerge = gate.passed and verdict == APPROVE   # indeterminate NEVER auto-merges

exit_code = 0 if deliver else (1 if blocks else 2)   # 2 == indeterminate, a real state
```

Two details matter for honesty of reporting. First, the reviewer-verdict parser is *fail-closed*: an unreadable or absent verdict is treated as a block, not a pass, because a permissive regex once matched only bare `REJECT` and missed formatted variants such as `**VERDICT:** REJECT`, a defect found by adversarial review. Second, indeterminate never enters the auto-fix loop, because there is no error output to repair; the system does not manufacture activity to disguise the absence of a signal.

### 4.2 The defect-ledger ratchet: mention does not prove existence

The structural gate ("the ratchet") is an LLM-free program that runs in seconds and checks **50 invariants** over the system's own source. Its defining property is stated in its header: *every invariant exists because a real, confirmed defect was closed, and every mechanically verifiable finding is added here.* The ratchet is thus a machine-readable ledger of the system's past failures; the number of invariants grows monotonically as defects are found and fixed.

Invariants span structural integrity (every agent declared in the manifest exists on disk and vice versa; every command declared in the manifest parses; no invisible control characters in source), gate integrity (the task runner never returns success on an empty gate; protective permission rules remain intact against a versioned snapshot), and deployment hygiene (published-branch hooks exist; local and remote main are the same commit).

The most important sub-property is expressed in the codebase as *"match the property, not the phrase."* Several invariants **execute the real artifact** rather than string-matching source text. The verdict-parser invariant runs the parser's own test suite; the safety-guard invariant executes the guard against both a list of operations it must block and a list it must not; the end-to-end invariant runs the full task flow against a real temporary repository with a stubbed model. This design choice was itself defect-driven: a string-matching invariant twice reported "green" on a correct refactor whose text had merely changed, and, more seriously, a code mutation once passed 22 of 22 unit tests while failing end-to-end. The lesson, that citing a behavior does not prove the behavior exists, recurs so often in the ledger that its occurrences are counted in comments. Invariants added later follow the same rule: those guarding a generated artifact (the README top badges, the git-derived changelog table) do not match text, they run the generator and the verifier and compare the output against the source of truth.

### 4.3 The closing loop: audits raise a floor

The system records lessons through a retrospective procedure that distills an incident into one to three atomic rules (`symptom -> cause -> rule`), written to a per-project or portfolio-level learnings file. A start-of-session hook injects these learnings back into the model's context, and a stop hook non-blockingly nudges the operator if a code-touching session recorded no lesson.

The step that closes the loop, and that we consider the crux, is this: **a lesson derived from a confirmed defect is given a second, mechanical destination.** If the defect is mechanically verifiable, it becomes a new invariant in the ratchet (Section 4.2). If it is behavioral, it becomes a new golden case in the evaluation suite (Section 4.4). A retrospective that ends only in prose is treated as incomplete. The effect is that each audit does not merely produce a report that ages; it raises a floor that the next session starts above. The learnings file is also pruned against a numeric ceiling by a dedicated tool that never deletes autonomously (the cost of losing a real lesson is asymmetric), and that returns *indeterminate*, again the third state, when it finds nothing measurable.

### 4.4 Prompt-change governance: prompts as code under test

Agents cannot edit their own definitions. A prompt change moves through a single gated path: the evaluation harness records a baseline, the change is applied, and the harness re-scores. Comparison is **by property, not by literal text**: each golden case carries a rubric of three to eight verifiable items and an "anti-example" that zeroes the case if violated; a case's numeric score must not drop. Rewording a correct answer must still pass; reintroducing the original defect must still fail; and *removing coverage counts as regression*. The harness also warns when the prompt hash is unchanged across two measurements, i.e., when a prompt has accidentally been compared to itself. A regression blocks the change from shipping.

We report the state of this mechanism honestly in Section 5: the harness and the golden-case corpus exist and are enforced structurally, but the corpus of *recorded scored runs* at the measured commit is small.

### 4.5 The delivery envelope: structured, self-skeptical handoff

Every agent returns work in a fixed nine-line envelope validated by a script the *consumer* runs before acting ("do not check by eye, run it"). Beyond provenance and verification fields, the envelope forces a three-way partition of every deliverable: (1) what is done, (2) what was deliberately left out of scope, and (3) what is pending or uncertain. The system's own documentation names the third part "the most important." Work that arrives without its promised evidence is not rounded up to done; it enters the delivery explicitly labeled "unverified," attributed to the agent that produced it. This is verification discipline expressed at the interface between agents, not only at the gates.

## 5. Evaluation

![Real verification-substrate figures: third state (10 of 21 projects with no tests), structural ratchet (50 invariants, 7 execute the artifact) and evaluation coverage (37 cases defined, 17 scored).](figures/panel-light.en.svg)

*Figure 1. Real figures of the verification substrate at the measured commit. Nothing fabricated.*

We evaluate against the system's own artifacts. We separate mechanisms that are *fully exercised* from those that are *defined but lightly exercised*, because conflating the two would violate the very principle the system embodies.

**Structural ratchet (fully exercised).** At the measured commit the ratchet enforces 50 invariants, each provenance-linked to a confirmed defect. Because the ledger only grows, the recurrence rate of a *fixed-and-ledgered* defect is, by construction, zero: a regression re-trips its invariant before publication. A subset of invariants execute the real artifact (parser suite, guard suite, end-to-end flow) rather than matching text; these exist specifically because text-matching variants produced false "green" results in the past.

**The third state (fully exercised, high impact).** The third state is the mechanism we can most concretely defend. In a measured snapshot of the portfolio, **10 of 21 assessed projects contained zero test files.** A two-state agent runner would therefore have reported "green" for work it could not verify on roughly 48% of assessed projects. Making indeterminate a first-class outcome converts that silent lie into an explicit, visible label and blocks automatic merge on exactly those projects. We regard this single number as the strongest empirical argument in the paper.

**Execute-not-mention (fully exercised, defect-attested).** Two concrete defects are attributable to this principle: (a) a reviewer-verdict parser that missed formatted rejections, caught by adversarial review and now guarded by an executed test; and (b) a mutation that passed all unit tests but failed the end-to-end flow, motivating the end-to-end invariant. Both are recorded in the codebase, not reconstructed for this paper.

**Evaluation-regression harness (defined, partially exercised).** The harness, the golden-case format, and the property-based comparison are implemented and enforced (a structural invariant requires at least 30 golden cases across agents; 37 are defined at the measured commit). The recorded-score store grew from a single case to **17 scored cases across 4 agents** (solutions architect, dev orchestrator, prompt engineer and QA engineer), each with a rubric and a baseline. The mechanism is real and wired into the release path; its *exercised coverage* has grown but does not yet reach all 27 agents. We report both sides deliberately: it is precisely the kind of number the system's own third-state discipline forbids us to round, up or down.

**Learning loop (fully exercised at the mechanism level).** The retrospective-to-invariant compilation is observable directly: the ratchet's growth is the loop's output, since invariants are the compiled form of confirmed defects. The behavioral half of the loop (retrospective-to-eval-case) shares the coverage limitation noted above.

## 6. Threats to validity

This is a single-operator, self-reported deployment; the author is also the evaluator. There is no controlled baseline and no independent replication, so all claims are existence-and-mechanism claims, not comparative performance claims. The portfolio is heterogeneous but small (~36 projects, 6 clients) and drawn from one operator's practice, limiting external validity. The system is built on one agent runtime, so some mechanisms may be substrate-specific rather than universal. The evaluation-regression harness is under-exercised (Section 5), so claims about its effect on prompt quality are limited to its design, not its measured impact. Finally, because the author designed both the system and its self-checks, invariants may encode the author's blind spots as readily as the author's lessons; the ledger records defects that were *found*, which is not the same as the defects that *exist*. We state these limits plainly because the paper's thesis is verification honesty, and it would be self-refuting to overstate.

## 7. Discussion

Two observations generalize beyond this deployment. First, the three-state discipline is cheap and portable: any agent pipeline that emits a binary pass/fail can add an indeterminate state and a rule that it never auto-approves, and doing so converts a class of silent failures into visible ones. Second, a defect-ledger ratchet reframes agent reliability as an accumulating asset rather than a per-run gamble: the interesting artifact is not any single agent's output but the growing set of machine-checks that fixed defects cannot silently return.

We also observe what is *not* a defensible moat. The agent roster, personas, and procedural skills are configuration and prompt engineering; they are replicable. The runtime is not ours. What compounds, and what is hard to copy, is the ledger of real defects and the discipline that compiles each one into an enforced check. That is operational capital, not a technical primitive, and we think naming it as such is more useful than claiming otherwise.

## 8. Conclusion

The company-of-agents metaphor is settled. The open, and more consequential, question is how such systems verify themselves. We reported a production system whose center of gravity is a verification substrate: a first-class indeterminate state so that "not measured" is never "passed," an LLM-free ratchet of 50 defect-provenanced invariants that prefer executing an artifact to citing it, a loop that compiles each confirmed defect into an enforced check, prompt changes gated by property-based regression, and agent-authored context treated as untrusted. We evaluated candidly, separating what is exercised from what is merely wired, and we located the real, non-portable value in the accumulated defect ledger rather than in the agents. Our recommendation to builders of agent systems is simple and, we believe, underpriced: treat your agents' output the way a mature CI system treats a build, and make "I did not check" a state you are structurally forbidden to hide.

## Availability

Sanitized illustrative snippets and this paper are available in the accompanying repository. No client data, production-security detail, or verbatim operational configuration is included, by design.

## References

[1] S. Hong et al. "MetaGPT: Meta Programming for a Multi-Agent Collaborative Framework." ICLR, 2024.
[2] C. Qian et al. "ChatDev: Communicative Agents for Software Development." 2023.
[3] Q. Wu et al. "AutoGen: Enabling Next-Gen LLM Applications via Multi-Agent Conversation." 2023.
[4] CrewAI: Framework for orchestrating role-playing, autonomous AI agents. Software framework, 2023-.
[5] LangGraph: Low-level orchestration framework for stateful multi-actor LLM applications. LangChain, 2024-.
[6] N. Shinn et al. "Reflexion: Language Agents with Verbal Reinforcement Learning." NeurIPS, 2023.
[7] J. S. Park et al. "Generative Agents: Interactive Simulacra of Human Behavior." UIST, 2023.
[8] C. Packer et al. "MemGPT: Towards LLMs as Operating Systems." 2023.
[9] C. E. Jimenez et al. "SWE-bench: Can Language Models Resolve Real-World GitHub Issues?" ICLR, 2024.
[10] promptfoo and LangSmith: open-source and commercial LLM evaluation tooling. Web resources, 2023-.

---

*This is a preprint and has not been peer reviewed. Feedback via the repository's issues is welcome.*
