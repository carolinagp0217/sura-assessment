# AI usage disclosure

> **Mandatory.** AI assistants are allowed. What we value is how you governed their use.
> If you did not use any AI tool, tick the box below and leave the rest empty.

- [ ] I did not use any AI tool in this assessment.

## Tools

| Tool (name and version, if known) | Where you used it (part and files) | What you used it for |
|---|---|---|
| Claude (Anthropic), Claude Sonnet 5.5 | Part A: `docs/design/SOLUTION_DESIGN.md` and `docs/adr/*.md` | Organizing my ideas, ordering the wording of the solution, and formatting the `.md` files with Markdown. The solution itself was defined by me beforehand |
| Claude (Anthropic), Claude Sonnet 5.5 | Part A: context and sequence diagrams in `docs/design/SOLUTION_DESIGN.md` | Producing the diagrams in Mermaid from my solution, with the prompt shown below |

**Prompt used for the diagrams and the wording review (translated from Spanish):**

> Based on the proposed solution and the summary provided, produce the context and sequence diagrams, taking into account the components involved, the defined integrations and the documented ADRs, ensuring the diagrams reflect the architectural decisions made. Additionally, help me review, organize and improve the wording of the Markdown documentation, aiming for a clear, coherent and professional structure without changing the meaning of the original proposal. Use Mermaid for the diagrams and, if you find inconsistencies or missing information, flag them without assuming decisions that are not documented.

## Validation

<!-- For each significant use: how did you check the output? (tests you wrote or ran, documentation you checked, reasoning you redid, reviews you did line by line...). -->

- I read the generated documents carefully and checked that every idea of my solution was included and that nothing I had not decided was added.
- I compared the diagrams with the documented ADRs to confirm they reflect the decisions made.
- I rendered the Mermaid diagrams to confirm they were valid.

## What I discarded or corrected

<!-- Suggestions you rejected or fixed, and why. Concrete examples are more useful than general statements. -->

- **ADR alternatives.** In the ADRs, the AI proposed more options than I had initially considered, including solutions I had not taken into account and that, in my judgment, did not fit the problem. I removed them and kept only the options I considered viable. 
- **Architectural suggestions.** I rejected some architectural suggestions because the AI does not have the full context of the problem. I did this on purpose, so that the AI would not influence my engineering judgment during the test. 

## Data I shared with AI tools

<!-- What did you paste or upload into AI tools? Confirm that no secrets or personal data were shared. -->
- No employer data, customer data, secrets or personal data were shared.
