# con-claude-bridge

con-claude-bridge lets the [Con terminal](https://github.com/nowledge-co/con-terminal)'s
AI agent use Claude Code as its model. Con stays in charge. Every command or
file edit Claude wants still appears as a Con tool call, and Con still asks you
before running it.

The project is at the design stage, with no code yet. These documents
describe it:

- [`docs/architecture.md`](docs/architecture.md): how the bridge works.
- [`docs/adr/`](docs/adr/): the decision records that explain why.
- [`docs/configuration.md`](docs/configuration.md): the settings and files
  the bridge will use.
- [`docs/implementation-plan.md`](docs/implementation-plan.md): the order in
  which it gets built.
