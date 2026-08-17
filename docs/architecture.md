# Architecture and Reliability Notes

## Design goal

The automation has a simple business purpose: turn one week of GitHub engineering activity into a concise narrative report and deliver it to a team channel.

The more important engineering question is what happens when the inputs or external services are imperfect.

The hardened implementation therefore treats the workflow as a set of contracts rather than a chain of nodes that are assumed to succeed.

## Processing stages

### 1. Schedule and configuration

The workflow starts on a weekly schedule and loads configuration such as repository, reporting language, reporting window, and Discord destination.

The Bulletproof Edition performs a preflight check before spending API calls. Invalid repository format, unsupported language settings, invalid lookback periods, or missing delivery configuration cause an explicit stop.

### 2. GitHub collection

Three GitHub data sets are collected:

- commits
- closed issues
- merged pull requests

External requests use bounded timeouts and retries for transient service failures.

### 3. Contract validation

Retries do not protect against a successful HTTP response containing the wrong data shape.

Before the AI layer runs, the hardened workflow checks that the GitHub responses match the structures the downstream code expects. Unexpected response shapes fail explicitly instead of being silently converted into an incomplete report.

### 4. Structured brief

The Code node normalizes the GitHub responses and builds a compact JSON payload containing the reporting period, counts, commit metadata, issues, and pull requests.

This node also constructs the report-generation prompt.

### 5. AI reporting

The n8n AI Agent is instructed to report only what the supplied repository data supports.

The system prompt deliberately prohibits unsupported statements about:

- developer intent
- why a change was made
- whether activity was automated
- project health or stability
- business impact
- system architecture
- future plans

Visible patterns may be described, but the model is not allowed to convert those patterns into unsupported causal conclusions.

### 6. Output validation

The hardened workflow checks the model response before delivery.

A report is rejected when it is missing, implausibly short, or missing required sections such as Development Activity, Issues and Pull Requests, Notable Patterns, or What to Watch Next.

This creates an important boundary: a successful LLM API call is not automatically treated as a successful business output.

### 7. Delivery

Validated reports can be posted to Discord. Success delivery can be disabled through configuration without removing nodes or rewriting the main workflow.

Discord calls also use retries for temporary provider failures.

## Error handling strategy

The project distinguishes three classes of problems.

### Transient provider failures

Examples: temporary GitHub timeout, Discord service interruption, network error.

Response: retry a limited number of times and then fail visibly.

### Contract or validation failures

Examples: malformed GitHub response, missing configuration, incomplete AI response.

Response: do not retry blindly. Stop the workflow and expose the invalid state for review.

### Operational notification

A separate n8n Error Trigger workflow can capture failed executions and notify an operator with useful context such as workflow name, execution ID, last node, and error message.

Keeping this notification workflow separate prevents alerting logic from making the primary business workflow harder to understand.

## Why OpenRouter and Anthropic variants exist

The live-tested implementation uses Claude Sonnet 4 through OpenRouter. This keeps the model-provider layer replaceable without changing GitHub collection, prompt construction, validation, or delivery logic.

A second reference workflow uses n8n's native Anthropic Chat Model configured for `claude-sonnet-4-20250514`. It demonstrates the same workflow architecture using direct Anthropic integration.

## Principle demonstrated

A production automation should not be judged only by whether every node turns green once.

It should make invalid assumptions visible, distinguish retryable failures from bad data, validate important handoffs, and provide a controlled path for human intervention when it cannot safely continue.