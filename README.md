# claude-skills

A collection of useful [Claude Code skills](https://docs.claude.com/en/docs/claude-code) — reusable, on-demand guidance that Claude loads when a task matches its trigger conditions.

## Skills

| Skill | Description |
| --- | --- |
| [solid-principles](solid-principles/SKILL.md) | Applies the SOLID principles (Single Responsibility, Open/Closed, Liskov Substitution, Interface Segregation, Dependency Inversion) when designing, writing, reviewing, or refactoring object-oriented code — flags concrete design smells rather than enforcing the principles dogmatically. |
| [event-processor-pattern](event-processor-pattern/SKILL.md) | Documents a layered pattern for consuming messages off a broker (RabbitMQ/Kafka/etc.) and routing them to per-event handlers via a registry. Use when adding a new event type or wiring up a new queue consumer. |

## Usage

Copy a skill directory into `~/.claude/skills/` (available globally) or a project's `.claude/skills/` (available in that project), then restart Claude Code. Claude will invoke the skill automatically when a request matches its description.
