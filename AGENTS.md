        # AGENTS.md

- This repo is currently requirements-first; treat `project_requirements.md` as the source of truth until code and manifests exist.
- Do not guess the stack or install commands. Derive commands from the actual root manifests, lockfiles, CI, and scripts once they are added.
- `project_requirements.md` may be edited by the user; do not overwrite or “fix” it unless explicitly asked.
- Prefer the smallest v1 that satisfies the spec. Avoid adding optional endpoints, queues, or dependencies unless needed for acceptance criteria.
- Before implementing, inspect any new root manifests, build/test/lint config, CI workflows, and repo-local instruction files.
- If the repo gains source code, keep API service, worker, queue integration, templates, and migrations separated by clear directory boundaries.
- Verify changes with the project’s own commands once they exist; do not invent replacement workflows.
