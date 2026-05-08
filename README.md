# quarkus-testing

An opinionated [Claude Skill](https://agentskills.io/specification) for testing Quarkus applications. When an AI coding agent is writing or modifying tests in a Quarkus project, this skill loads into context and tells it which mocking mechanism to use, how to wire up integration tests, what not to mock, and which rationalizations to ignore under deadline pressure.

The full guidance lives in [`SKILL.md`](./SKILL.md). The short version:

- `@InjectMock` is **always** the default for replacing a CDI bean in tests.
- `QuarkusMock.installMockForType` is reserved for three specific cases: programmatic control (hand-rolled fakes), normal-scoped bean replacement that must be visible to other injection points, and dynamic mocking within a test.
- `QuarkusMock.installMockForInstance` only ever pairs with `installMockForType`, for per-test overrides.
- `QuarkusMock` is **incompatible with parallel test execution** — race conditions are guaranteed.
- Integration tests use the `*IT extends *Test` companion pattern.
- Don't mock domain types (aggregates, value objects, entities); mock collaborators only.
- Don't test the framework.
- Endpoint paths in tests must match the resource's `@Path`.

The skill also ships an Excuse / Reality table for resisting the rationalizations that show up at 4:55 PM on a Friday: "ship the one-line fix, refactor Monday," "belt-and-braces never hurts," "a senior reviewer said so."

## Installation

The skill is a single `SKILL.md` file with YAML frontmatter. Drop the directory containing it into your agent's skills location.

### Claude Code

User-level (active across every project):

```bash
git clone https://github.com/jeremyrdavis/quarkus-skill-testing.git ~/.claude/skills/quarkus-testing
```

Project-level (active only in one project):

```bash
git clone https://github.com/jeremyrdavis/quarkus-skill-testing.git <your-project>/.claude/skills/quarkus-testing
```

### Other agents

| Agent | Skills directory |
|---|---|
| Codex | `~/.agents/skills/quarkus-testing/` |
| Copilot CLI | follow Copilot's plugin install docs; the skill is loaded by the same `Skill` tool |
| Gemini CLI | follow Gemini's `activate_skill` docs |

Only the directory containing `SKILL.md` is required. The other files in this repo (`README.md`, `LICENSE`) are repo metadata and don't affect skill loading.

## How triggering works

Agents read the `description:` field in `SKILL.md`'s YAML frontmatter to decide whether to load the body of the skill into context for a given task. The current trigger fires when the agent is:

- writing or modifying a `@QuarkusTest` / `@QuarkusIntegrationTest` class
- naming an integration test class
- choosing between `@InjectMock`, `QuarkusMock.installMockForType`, and `QuarkusMock.installMockForInstance`
- mocking a CDI bean
- setting up PostgreSQL for tests via Dev Services
- deciding whether different test methods need different mock behaviors

If you want to broaden or narrow that trigger, edit the `description:` field — that's the part agents read every time.

## Authoring methodology

This skill was developed and verified using the [`superpowers:writing-skills`](https://github.com/anthropics/claude-code-plugins) RED-GREEN-REFACTOR workflow:

1. **RED** — run a multi-pressure scenario against a subagent without the skill loaded; capture the verbatim rationalizations the agent uses to justify the wrong choice.
2. **GREEN** — write or edit the skill addressing those specific rationalizations; re-run; verify the agent now picks the right option and cites the rule.
3. **REFACTOR** — capture any new rationalizations that surface and add them to the rule list, the Anti-patterns table, or the Excuse / Reality table.

The Excuse / Reality table in `SKILL.md` is built from rationalizations that real subagents produced under simulated time / authority / sunk-cost pressure.

## License

MIT — see [`LICENSE`](./LICENSE).

## Contributions

PRs welcome, especially:

- Pressure-test scenarios that surface rationalizations the current skill doesn't address.
- Coverage for newer Quarkus testing APIs.
- Variants for adjacent Quarkus test domains (Kafka, gRPC, scheduled tasks).

When proposing a rule change, please include the pressure scenario you used to validate it — the methodology section above describes the format.
