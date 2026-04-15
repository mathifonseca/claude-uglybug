# Contributing to /uglybug

Thanks for your interest! Contributions of all sizes are welcome — prompt refinements, new heuristics, additional post-game agents, bug reports, documentation fixes.

## How to contribute

1. **Open an issue first** for any non-trivial change so we can align on scope before you write code.
2. **Fork and branch**: create a feature branch from `main`.
3. **Make your change**: keep it focused. One logical change per PR.
4. **Test locally**: install your fork into `~/.claude/skills/uglybug` and run `/uglybug` against a real codebase with known bugs. Record the scoreboard and finding quality.
5. **Open a PR** with a clear description of what changed and why. If you changed role prompts, include before/after scoreboards from the same target codebase.

## What makes a good contribution

- **Prompt refinements**: the role prompts in `references/roles/` are load-bearing. Subtle phrasing changes can shift agent behavior meaningfully — share concrete before/after evidence.
- **New heuristics**: additions to `references/heuristics.md` should come with a real-world example of a bug class that existing heuristics miss.
- **New post-game agents**: the current set is Consolidator + Hardener. If you add a third (e.g. a Root-Causer), justify the marginal value over rerunning an existing one.
- **Bug reports**: include the target codebase (or a minimal repro), the command you ran, the resulting `state.json`, and what went wrong.
- **Documentation**: README fixes, clearer diagrams, additional examples are always welcome.

## Code of conduct

Be kind. See [CODE_OF_CONDUCT.md](CODE_OF_CONDUCT.md).

## License

By contributing, you agree that your contributions will be licensed under the MIT License.
