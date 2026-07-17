# Changelog

All notable changes to this project will be documented in this file.

## [2.2.0] - 2026-07-17

### Added
- Every `/forge:building` contract criterion now carries an `example` field: a concrete worked case (e.g. `parse('') -> raises ValueError('empty input')`), required by the generator, rejected if missing by the lead's mechanical check, and attacked by the evaluator if it contradicts its criterion.
- The Phase 3 contract gate now renders a fixed template (grounding, residual risks, criteria grouped by issue with their example and verification method) and ends in a real interactive-question hard stop, so the gate reliably pauses on every harness instead of depending on prose alone.
- Labor-tier builders now self-check every criterion in their brief against their own build, using its example and verification method, before reporting done - cutting wasted Phase 6 evaluation rounds without weakening the evaluator's independence.

## [2.1.1] - 2026-07-17

### Added
- The building skill now forbids the run's own vocabulary from reaching the code it produces: no issue numbers, contract clause ids, `harness/` paths, or role names ("the evaluator", "eval seam") in comments, docstrings, test names, section headers, or user-facing strings. Stated as a guardrail with its reasoning, and as an instruction the lead must relay in every teammate prompt, since a builder handed `C13` in its brief will write `# C13:` in the code unless told not to. Issue bodies, commit messages, and PR descriptions are unaffected.

## [2.1.0] - 2026-07-16

### Added
- Added Google Antigravity harness binding ([docs/harness-bindings/antigravity.md](docs/harness-bindings/antigravity.md)) mapping neutral terms to Gemini 3.1 Pro (judgment tier) and Gemini 3.5 Flash (labor tier), as well as native subagents and interactive tools.
- Registered Antigravity binding in the harness bindings README.
