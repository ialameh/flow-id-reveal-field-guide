# Contributing

Thanks for considering a contribution. The Agentforce Field Guide is built and maintained by people who hit these problems in production and wrote down what they learned. Every contribution that helps the next person is welcome.

## What contributions are welcome

- **Fixes.** Typos, broken links, outdated commands, factual errors.
- **New case studies.** Real incidents you have run into, anonymised.
- **New cookbook examples.** Worked-out patterns that fill a gap.
- **Diagram improvements.** Clearer SVGs, additional perspectives.
- **Updates for newer Salesforce releases.** When something the guide covers changes.
- **Translations.** If you want to translate a chapter, open an issue first so we can coordinate.

## What we ask you to keep

- **Tone.** Plain prose, conversational, technically precise. Read [Chapter 14](./14-conversation-design.md) for the writing style we are aiming at.
- **No em dashes.** Use commas, parentheses, or sentence breaks. This is a style choice for readability.
- **No marketing language.** This is a working document, not a brochure.
- **Concrete over abstract.** Examples beat principles. Code beats prose.
- **Generic, not project-specific.** This guide is meant to be useful to anyone. Anonymise project names, org names, customer names.

## How to contribute

1. **Open an issue** describing what you want to change. For typos and small fixes, you can skip this and go straight to a PR.
2. **Fork the repository.**
3. **Create a branch.** Name it descriptively, e.g. `case-study-flow-deployment-failure` or `fix-typo-chapter-3`.
4. **Make your change.** Follow the conventions in this file.
5. **Open a pull request.** Describe what you changed and why. Reference any related issue.

## Conventions

### Markdown

- Plain markdown. No HTML unless absolutely necessary.
- Tables for reference data. SVGs for conceptual diagrams.
- Code blocks with language tags (`bash`, `apex`, `xml`, `json`, etc.).
- Sentence-case headings. No title case.

### File names

- Chapters use `NN-name-with-dashes.md` where NN is the chapter number.
- Diagrams use `descriptive-name.svg`, lowercase, dashes.
- Cookbook examples are folders with `NN-name/`.

### Diagrams

- SVG, hand-authored or generated from a tool that produces clean output.
- Embed via `![alt text](diagrams/name.svg)`.
- Use the existing diagrams as a style reference. Same fonts, same color palette, same arrow markers.
- No embedded fonts. Use system fonts (`system-ui, -apple-system, ...`).

### Code samples

- Apex samples should be deployable as-is, with placeholders clearly marked in `<ANGLE_BRACKETS>`.
- JSON Schemas should validate.
- Bash commands should work without modification on macOS or Linux. Note where Windows differs.
- `sf` commands should target the latest stable Salesforce CLI. Note any version-specific behaviour.

### Tone

Write like you are explaining to a colleague who is competent but unfamiliar with the specific topic. Direct, brief, no padding.

Compare:

- "It is generally recommended that practitioners consider the implications of activation before proceeding to deploy any changes."
- "Activate after every change. Trust the test, not the simulation."

The second version is the tone we are looking for.

## Adding a new case study

The structure is:

1. **Context.** What was the team building? What environment? What they wanted to achieve.
2. **Symptoms.** What did they see? Console errors, logs, traces.
3. **Initial hypothesis.** What did they think was wrong, before they investigated?
4. **What was actually wrong.** With diagnosis.
5. **What they did.** Step by step.
6. **What they learned.** Takeaways.

See [Case 01](./case-studies/01-action-name-not-found.md) for the format.

Anonymise everything. No real org names, customer names, employee names, project codenames. Use "the team", "the org", "the agent", or invented names like "AcmeBot".

## Reviewing pull requests

The maintainers will review:

1. **Accuracy.** Is what you wrote technically correct?
2. **Tone.** Does it match the rest of the guide?
3. **Scope.** Is it generic enough to be useful to other readers?
4. **Sources.** If you reference a Salesforce feature, can someone find it in the official docs?

We will leave comments rather than closing PRs without explanation. If you do not hear back in a week, ping the issue or PR.

## Code of Conduct

Be kind. Disagree about content, not about people. Follow the [Code of Conduct](./CODE_OF_CONDUCT.md).

## Licensing

By contributing, you agree that your contributions will be licensed under the same Creative Commons Attribution 4.0 license as the rest of the guide.

## Getting help

- Open an issue for questions.
- Tag a maintainer in a PR if you need a faster response.
- For broader Agentforce questions, the Salesforce Trailblazer Community is a better venue.

Thanks again for considering a contribution. This guide gets better with every pair of fresh eyes.
