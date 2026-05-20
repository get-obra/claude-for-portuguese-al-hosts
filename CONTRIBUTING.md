# Contributing

Thank you for considering a contribution to **Claude for Portuguese AL hosts**. This pack is open source under Apache 2.0 because the operators it serves (independent Portuguese vacation rental hosts and the small businesses adjacent to them) benefit when the work is shared.

## What we want

Contributions from people who actually know this domain are the most valuable kind:

- **Portuguese AL hosts** describing edge cases the canonical workflow does not cover yet.
- **Accountants** familiar with AT month-end reporting and SAF-T PT format who can sharpen the accountant-handoff skill.
- **Compliance professionals** with views on GDPR Article 9 handling, CNPD posture, and SEF reporting nuance.
- **Developers** familiar with the SEF portal, Playwright automation, MCP connectors, and agentic-runtime composition.
- **Translators** who can review the multilingual welcome and in-stay support copy for naturalness in their language.

## How to contribute

1. **Open an issue first** for anything beyond a one-line typo fix or wording suggestion. The issue lets us discuss scope before you write the code or content. Issues tagged `good-first-issue` are explicitly inviting first-time contributors.
2. **Fork, branch, pull request.** Standard GitHub flow. Reference the issue your PR addresses.
3. **One concern per PR.** A skill update is a different PR from a connector update is a different PR from a documentation update.
4. **Tests where applicable.** Connectors ship with test fixtures. Skills ship with example transcripts. If you change one, update the other.
5. **No real customer data, ever.** Anything in this repo (issues, PRs, fixtures, examples) must use synthetic data only. Real passport numbers, real guest names, real reservation amounts do not belong here.

## What we will not accept

- Contributions that route customer data through hosted services other than Anthropic's Claude API. The local-first principle is non-negotiable.
- Contributions that remove or weaken the gating, agreement-gate, or audit-chain primitives. These are architectural commitments.
- Contributions that add hosted dependencies on services with weaker compliance posture than the rest of the pack.
- Marketing material for other AI services. This pack is built on Claude, exclusively.

## Code of conduct

Participation in this project is subject to the [Code of Conduct](CODE_OF_CONDUCT.md). Be kind. Disagreements about technical or compliance choices are normal and welcome; disagreements about whether other contributors deserve respect are not.

## Licensing

By contributing to this project, you agree that your contributions are licensed under the [Apache License 2.0](LICENSE), the same license as the rest of the repository.

## Recognition

Contributors are acknowledged in the project's CHANGELOG once it lands, and in the eventual v1.0 release notes. Substantial contributors who want a direct conversation about the work are welcome to reach out at `team@get-obra.com`.

## Maintainer

This project is maintained by [Obra](https://get-obra.com). For commercial deployments of these skills as a managed service, see the Obra website. The open-source pack remains free for anyone to use, fork, and adapt.
