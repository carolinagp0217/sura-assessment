# AI Usage

## Tools used
- Claude (Anthropic), chat assistant.

## What I used it for
I used AI to organize my ideas, order the wording of the solution, and format the `.md` files with Markdown. Once I had defined the solution myself, I used AI to help produce the context and sequence diagrams, with this prompt:

> **Prompt (translated from Spanish):**
> Based on the proposed solution and the summary provided, produce the context and sequence diagrams, taking into account the components involved, the defined integrations and the documented ADRs, ensuring the diagrams reflect the architectural decisions made. Additionally, help me review, organize and improve the wording of the Markdown documentation, aiming for a clear, coherent and professional structure without changing the meaning of the original proposal. Use Mermaid for the diagrams and, if you find inconsistencies or missing information, flag them without assuming decisions that are not documented.

## How I validated the output
- I read the generated text in full and compared it against my own solution, checking that every component, integration and decision in my version appeared and that nothing I had not decided was added.
- I checked the diagrams against the documented ADRs
- I rendered the Mermaid diagrams to confirm they were valid

## What I discarded or changed
- In the ADRs, the assistant proposed more alternatives than I had considered. I removed those that did not fit the problem and kept a smaller set, so each ADR compares only options I considered viable.
- I also removed the requirement references the assistant had added.

- I rejected architectural suggestion, because the assistant does not have the full context of the problem.