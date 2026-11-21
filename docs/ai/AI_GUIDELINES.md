# AI GUIDELINES
AI are an assistant, a tool, not a replacement for engineering judgment.

### What AI can and cannot do

Allowed:
- Generate, explain, refactor and review code.
- Create tests and technical documentation.
- Summarize public provider, Salesforce and library documentation.
- Suggest designs and identify potential edge cases.
    Use synthetic data for examples and testing.

Never share with AI:
- Personal data (names, RUTs, emails, phone numbers).
Real policy numbers, transaction IDs or payment identifiers.
- Secrets, tokens, credentials, .env files or real configuration values.
- Production logs, webhook payloads, database records or incident data.
- Confidential contracts, unresolved security findings or unauthorized internal code.

### Code quality and validation
AI-generated code is not trusted by default.

Before merging:

- Understand and be able to explain every line of generated code.
- Verify technical claims against official provider, Salesforce and library documentation.
- Define tests from requirements and expected failure scenarios, not from the generated implementation.
- Review assertions carefully. Green tests do not necessarily mean correct behavior.
- Challenge tests by introducing intentional failures and verifying that they catch them.
- Run all existing quality gates: build, tests, security scans, dependency checks and static analysis.
- Verify the source and license of new dependencies or copied code.

### AI Context & Token Optimization

Create and maintain specific AI guidelines in AGENTS.md to control context loading, minimize unnecessary information, and optimize token consumption. Ensure AI agents load only the relevant instructions and context required for each task.




