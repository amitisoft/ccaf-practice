## Exercise 1: Build a Multi-Tool Agent with Escalation Logic

**Objective:** Practice designing an agentic loop with tool integration, structured error handling, and escalation patterns.

### Scenario

You are building a customer support resolution agent for an e-commerce company. The agent can resolve common issues such as order lookups, return eligibility checks, refund requests, and account verification. Some actions must be blocked or escalated when they exceed policy limits.

### Steps

1. **Define 3–4 MCP tools with differentiated descriptions.**

   Create tool definitions for:

   * `get_customer`
   * `lookup_order`
   * `check_return_eligibility`
   * `process_refund`

   Include detailed descriptions for each tool that specify:

   * Purpose
   * Expected inputs
   * Output shape
   * When to use the tool
   * When not to use the tool
   * Boundary conditions and edge cases

   Make `get_customer` and `lookup_order` intentionally similar enough that careless descriptions could cause confusion, then rewrite their descriptions to clearly distinguish them.

2. **Implement an agentic loop using `stop_reason`.**

   Build a loop that:

   * Sends the current conversation to Claude
   * Checks whether `stop_reason` is `"tool_use"` or `"end_turn"`
   * Executes tools when tool use is requested
   * Appends tool results back into conversation history
   * Ends only when Claude returns `"end_turn"`

   Avoid relying on assistant text like “I’m done” as a stopping signal.

3. **Add structured error responses to tools.**

   Each tool should return structured errors with:

   ```json
   {
     "isError": true,
     "errorCategory": "transient | validation | permission | business",
     "isRetryable": true,
     "description": "Human-readable explanation"
   }
   ```

   Test that the agent:

   * Retries transient errors
   * Does not retry validation errors
   * Explains business-rule failures to the user
   * Escalates permission errors when appropriate

4. **Implement a programmatic escalation hook.**

   Add a hook that intercepts outgoing `process_refund` calls.

   Business rule:

   * Refunds up to `$500` may proceed automatically.
   * Refunds above `$500` must be blocked and redirected to `escalate_to_human`.

   The hook should enforce the rule deterministically, not merely rely on the system prompt.

5. **Test multi-concern messages.**

   Test the agent with requests such as:

   > “I need a refund for order 123 because it arrived damaged, and also I can’t log into my account.”

   Verify that the agent:

   * Separates the refund issue from the account issue
   * Uses the right tools for each concern
   * Preserves key case facts such as customer ID, order ID, refund amount, and issue type
   * Produces one unified final response

### Success Criteria

* The agent uses tools based on clear descriptions, not keyword guessing.
* The loop continues on `"tool_use"` and stops on `"end_turn"`.
* Tool errors are handled differently based on category and retryability.
* High-value refunds are blocked by a deterministic hook.
* Multi-issue requests are decomposed and resolved coherently.

### Domains Reinforced

* Domain 1: Agentic Architecture & Orchestration
* Domain 2: Tool Design & MCP Integration
* Domain 5: Context Management & Reliability
