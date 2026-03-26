# BMAD Story Format Reference

Reference for parsing BMAD-METHOD story files. Stories are markdown files created by the `bmad-create-story` workflow.

## Required Sections

### Header
```markdown
# Story {epic_num}.{story_num}: {title}

Status: ready-for-dev
```
**Parse:** Extract `epic_num`, `story_num`, `title`, and verify `Status` is `ready-for-dev`.

### Story (User Story)
```markdown
## Story

As a {role},
I want {action},
so that {benefit}.
```
**Use for:** Plan header `Goal` field.

### Acceptance Criteria
```markdown
## Acceptance Criteria

1. **Given** [precondition]
   **When** [action]
   **Then** [expected result]
```
**Use for:** Test case foundations, final verification task, Plan header summary.

### Tasks / Subtasks
```markdown
## Tasks / Subtasks

- [ ] Task 1: [description] (AC: #1, #2)
  - [ ] 1.1 [subtask description]
  - [ ] 1.2 [subtask description]
```
**Use for:** Primary input for Plan Task generation. Each Subtask typically becomes one Plan Task.

### Dev Notes
Contains multiple subsections with implementation context:

- **Architecture Constraints** — MUST-follow rules (DB choice, module boundaries, naming)
- **Implementation Guardrails** — Critical warnings about common mistakes
- **Recommended File Paths** — Table mapping files to actions and rationale
- **Reusable Assets** — Patterns from previous Stories that can be reused
- **Testing Standards** — Framework, patterns, coverage expectations
- **Project Structure Notes** — Where new code should live
- **Git Intelligence** — Recent commit context
- **References** — Source document citations

### Dev Agent Record
```markdown
## Dev Agent Record

### Agent Model Used
### Debug Log References
### Completion Notes List
### File List
```
**Note:** This section is for implementation tracking. Ignore during plan generation.
