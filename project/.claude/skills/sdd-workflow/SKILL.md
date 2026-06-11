# SDD Workflow Skill

## Description

Guides the user through the complete SDD (Spec-Driven Development) workflow for implementing features in SpecTask. Orchestrates the five phases: Constitution review, Specify, Plan, Tasks, Implement.

## Trigger

- User says "new feature", "implement", "add feature", or "sdd"
- User wants to add functionality to SpecTask

## Workflow

### Phase 1: Constitution Review

1. Read `CLAUDE.md` to confirm project conventions are current
2. If adding a new domain or pattern, update CLAUDE.md first
3. Confirm with user before proceeding

### Phase 2: Specification

1. Ask user to describe the feature in plain language
2. Transform into EARS-formatted requirements:
   - Ubiquitous: "The system shall..."
   - Event-Driven: "WHEN [event], the system shall..."
   - State-Driven: "WHILE [state], the system shall..."
   - Optional: "WHERE [feature], the system shall..."
   - Unwanted: "IF [condition], THEN the system shall..."
3. Add acceptance criteria in Given/When/Then format
4. Define out-of-scope explicitly
5. Save to `specs/{feature-name}-spec.md`
6. **GATE**: User reviews and approves spec

### Phase 3: Planning

1. Read the approved spec
2. Make architecture decisions (document rationale)
3. Design data model changes
4. Plan file structure additions
5. Identify risks and dependencies
6. Save to `plans/{feature-name}-plan.md`
7. **GATE**: User reviews and approves plan

### Phase 4: Task Decomposition

1. Read the approved plan
2. Create tasks following ordering rules:
   - Infrastructure before application code
   - Types/interfaces before implementation
   - Tests before implementation (TDD)
   - Inner layers before outer layers
3. Each task must have:
   - Spec requirement reference
   - Clear acceptance criteria
   - Verification command
   - Dependency list
4. Verify all spec requirements are covered by tasks
5. **GATE**: User reviews and approves task list

### Phase 5: Implementation

For each task in order:

1. **Read** the task description and linked spec requirement
2. **Execute** the task:
   - If test task (RED): Write tests, verify they compile, confirm they fail
   - If implementation task (GREEN): Write code, verify tests pass
3. **Verify** using the task's verification command
4. **Commit** with conventional commit message:
   - test: ... (for RED phase tasks)
   - feat: ... (for GREEN phase tasks)
   - refactor: ... (for cleanup tasks)

### Phase 6: Verification

1. Run all tests: `pnpm test`
2. Check coverage: `pnpm test --coverage` (target: 80%+)
3. Type check: `pnpm tsc --noEmit`
4. Run verifier agent against all specs
5. Fix any CRITICAL or HIGH issues
6. Re-run verifier until PASS

## Quality Gates

| Gate | Criteria | Blocker |
|------|----------|---------|
| Spec -> Plan | All requirements EARS-formatted, acceptance criteria testable | Missing acceptance criteria |
| Plan -> Tasks | Architecture addresses all requirements | Unaddressed requirement |
| Tasks -> Implement | All requirements covered by tasks, tests ordered first | Missing requirement coverage |
| Implement -> Done | All tests pass, coverage >= 80%, verifier PASS | CRITICAL issues in verifier report |
