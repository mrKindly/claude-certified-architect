# Claude Certified Architect — Foundation Exam

## Study Guide: 4 Exam Scenario Groups

> **60 questions** across **4 real-world scenario groups** — exactly 15 questions each.  
> Correct answers are marked ✅ with full explanations.  
> Original exam question numbers are preserved where possible, but re-indexed for clarity. Generated questions are integrated into the sequence.

---

## Table of Contents

1. [Scenario 1 — Customer Support Agent](#scenario-1) _(Questions 1–15)_
2. [Scenario 2 — Multi-Agent Research System](#scenario-2) _(Questions 16–30)_
3. [Scenario 3 — Code Generation with Claude Code](#scenario-3) _(Questions 31–45)_
4. [Scenario 4 — Structured Data Extraction](#scenario-4) _(Questions 46–60)_

---

<a name="scenario-1"></a>

## 🎧 Scenario 1: Customer Support Agent

**Context:** You are building a customer support agent using the Claude Agent SDK. The agent handles complex, ambiguous tasks — processing returns, resolving billing disputes, and managing escalations to human agents. Core tools include `get_customer`, `lookup_order`, `process_refund`, and `escalate_to_human`. The agent must operate reliably at scale, enforce business rules deterministically, and know when to hand off to a human.

---

### Question 1 of 60

**Your expense reimbursement agent processes employee requests using a `process_reimbursement` tool. Company policy requires that reimbursements above $500 must be approved by a manager before funds are disbursed. The agent handles hundreds of requests daily, and you need the threshold enforcement to be tamper-proof regardless of how the agent is prompted. Which design ensures the $500 approval threshold cannot be bypassed?**

- A: Provide two tools: `auto_reimburse` (hard-coded $500 limit) and `request_manager_approval`. Include detailed system prompt instructions telling the agent to check the amount and call the appropriate tool. Add a PostToolUse hook for auditing.
- B: Implement the threshold check in a PreToolUse hook that inspects the amount parameter before `process_reimbursement` executes. If the amount exceeds $500, the hook modifies the tool call to add a `requires_approval: true` flag, which the tool checks before disbursing.
- C: The `process_reimbursement` tool accepts an `approved_by_manager: boolean` parameter. The system prompt instructs the agent to only set this to `true` after confirming manager approval. A nightly audit script reviews all approvals.
- D: ✅ The `process_reimbursement` tool accepts amount and details, and internally enforces the threshold: amounts ≤$500 are auto-disbursed; amounts >$500 cause the tool to create a pending approval request and return a status indicating manager review is pending.

**✅ Correct Answer: D**

> **Explanation:** Critical business rules that should not be bypassable by prompting alone belong in trusted server-side code — not only in instructions the model may ignore. Moving the threshold check _inside_ the tool means it executes as part of disbursement logic, regardless of how the model is prompted or which parameters it passes. Option A relies on the model choosing the correct tool. Option B via a hook is closer, but only adds a _flag_ that the tool still checks — a parameter the model could still manipulate. Option C is the weakest: it puts a safety-critical boolean directly under model control. Tool-internal enforcement is the strongest tamper-resistant pattern here because policy runs in trusted code paths independent of model-controlled approval bits.

---

### Question 2 of 60

**Compliance requires that refunds exceeding $500 must automatically escalate to a human agent — this rule cannot be left to model discretion. Despite emphatic system prompt instructions, production logs show the agent occasionally processes high-value refunds directly (3% failure rate). How should you achieve guaranteed compliance?**

- A: Strengthen the system prompt with emphatic language: "CRITICAL POLICY: Refunds over $500 MUST trigger human escalation. NEVER process these directly."
- B: Add few-shot examples to the prompt showing correct escalation behavior at various refund amounts ($400, $500, $600).
- C: ✅ Implement a hook to intercept tool calls; when the refund amount exceeds $500, block it and invoke human escalation.
- D: Modify the refund tool to return an error with message "Amount exceeds policy limit — please escalate" when threshold is exceeded.

**✅ Correct Answer: C**

> **Explanation:** A 3% failure rate on a safety-critical rule means the model is non-deterministically ignoring system prompt instructions. The approach that best matches *guaranteed* compliance here is enforcement at the infrastructure layer via a hook — code that runs regardless of model behavior and blocks the non-compliant tool call before it executes. Options A and B iterate on the failing approach (prompting). Option D is better than prompting but still relies on the model to then escalate after seeing the error; a hook eliminates that dependency entirely.

---

### Question 3 of 60

**Your agent is handling a billing dispute. After calling `get_customer` and `lookup_order`, it identifies that the dispute involves a promotional pricing error requiring manager approval — beyond the agent's authorization level. How should the workflow handle this mid-process escalation?**

- A: Persist the complete conversation and tool response history to a database, then call `escalate_to_human` with a reference ID.
- B: Call `escalate_to_human` passing only the customer's original message.
- C: Attempt the refund with `process_refund` anyway, escalating only if the system rejects the transaction.
- D: ✅ Compile a structured handoff with customer details, order info, and the identified issue before calling `escalate_to_human`.

**✅ Correct Answer: D**

> **Explanation:** A human agent taking over needs to understand what happened without re-investigating from scratch. A structured handoff — customer details, order info, what was found, why escalation is needed — enables immediate informed action. Option A (full transcript + reference ID) is impractical if the human doesn't have tool access to retrieve the context. Option B (original message only) discards all the investigative work done. Option C is wrong in principle: attempting unauthorized actions and relying on the system to reject them is not a valid safety approach.

---

### Question 4 of 60

**A customer returns 4 hours after their initial session about the same billing dispute. The previous 32-turn session contains `lookup_order` results showing "Status: PENDING, Expected resolution: 24-48 hours." The agent often references this outdated data in responses even after subsequent fresh tool calls return different information. What approach most reliably handles returning customers?**

- A: Resume with full history but filter out previous `tool_result` messages, keeping only the human/assistant turns so the agent must re-fetch needed data.
- B: Resume with full history and add a system prompt instruction telling the agent to always prefer the most recent tool results when multiple calls to the same tool exist in context.
- C: ✅ Start a new session, inject a structured summary of the previous interaction (issue type, actions taken, resolution status), then make fresh tool calls before engaging.
- D: Resume with full history and configure the agent to automatically re-call all previously-used tools at session start to ensure data freshness.

**✅ Correct Answer: C**

> **Explanation:** After a 4-hour gap, all tool data is stale and the 32-turn history creates noise that can confuse the model. Starting fresh — with a compact structured summary of _what happened_ and fresh tool calls — gives the model current, accurate data in a clean context. Option A (removing tool results but keeping conversation) still leaves 32 turns of potentially misleading context. Option B is fragile — "prefer most recent" can be misapplied when multiple tool calls of the same type exist. Option D (re-calling all tools) is expensive and may be unnecessary.

---

### Question 5 of 60

**Your `process_refund` tool returns two types of errors: technical errors ("503 Service Unavailable", "Connection timeout") that are transient (5% of calls), and business errors ("Order exceeds 30-day return window", "Item already refunded") that are permanent (12% of calls). The agent wastes 3-4 turns retrying business errors. Currently, both error types return only a plain text message. What's the most effective way to reduce wasted retries while improving customer-facing response quality?**

- A: Implement automatic retry logic at the tool level for technical errors only, passing business errors to Claude without retries.
- B: ✅ Return structured error responses with `retriable: false` for business errors and a customer-friendly explanation for Claude to use.
- C: Add a `check_refund_eligibility` tool that must be called before `process_refund` to prevent business rule violations.
- D: Add few-shot examples showing how to distinguish retriable from non-retriable errors by parsing error message text.

**✅ Correct Answer: B**

> **Explanation:** The root problem is that the model cannot distinguish permanent failures from transient ones when both return plain text. Adding a `retriable: false` flag gives the model an unambiguous signal to stop retrying and communicate the policy reason to the customer. Including a customer-friendly explanation means the model can immediately use that text in its response. Option C adds a mandatory pre-flight call, increasing latency for every refund including the 83% that are valid. Option A only half-solves the problem — it handles retries on the tool side but doesn't give the model actionable context for permanent business failures.

---

### Question 6 of 60

**A customer sends: "This is frustrating. I've explained my issue twice and nothing is being resolved. I want to talk to a real person NOW." The agent has not yet called any tools. What should the agent do?**

- A: ✅ Immediately call `escalate_to_human` with the conversation history.
- B: Acknowledge the frustration and ask one targeted question to understand the specific issue before escalating.
- C: Briefly explain what the agent can help with and offer to resolve the issue quickly, escalating only if the customer repeats their request.
- D: First call `get_customer` and `lookup_order` to gather account context, then escalate to a human agent.

**✅ Correct Answer: A**

> **Explanation:** The customer has made an unambiguous, explicit request for a human agent — stated in the imperative ("I want to talk to a real person NOW") after expressing sustained frustration. Continuing with clarifying questions (B) or attempting to retain the customer in the automated system (C) dismisses their stated preference and deepens frustration. Pausing to run tool lookups (D) before escalating delays honoring an explicit request. When a customer explicitly and emphatically requests a human, that request should be honored immediately.

---

### Question 7 of 60

**Your agent has called `lookup_order` multiple times while investigating a customer's return requests. Each response includes 40+ fields (items, shipping details, payment info, status history). Tool outputs now represent the majority of the conversation's context. The customer mentions two more orders they want to discuss. What's the most effective approach before making additional lookups?**

- A: ✅ Extract only return-relevant fields (items, purchase date, return window, status) from each existing order response, removing verbose details.
- B: Move all tool responses to a vector database with semantic indexing, retrieving relevant portions as the conversation continues.
- C: Proceed with additional lookups without modifying the existing tool output context.
- D: Have the model generate a natural language summary of each order's key details, replacing structured responses with prose descriptions.

**✅ Correct Answer: A**

> **Explanation:** Before adding more context, prune existing context to only what's needed for the current task. For a return request, only a handful of the 40+ fields matter: items, purchase date, return window, status. Stripping irrelevant fields frees significant context budget for the upcoming lookups. Option B (vector DB) is architectural overkill for a within-session problem. Option C makes the problem worse. Option D (prose summaries) loses the structure needed for downstream tool calls like `process_refund`.

---

### Question 8 of 60

**Production logs reveal inconsistent error handling: when `lookup_order` fails, the agent sometimes retries 5+ times, sometimes escalates immediately, sometimes asks users for clarification. Your MCP tool returns uniform error responses: `{"isError": true, "content": [{"type": "text", "text": "Operation failed"}]}`. The agent cannot distinguish between error types. What's the most effective improvement?**

- A: ✅ Enhance error responses with structured metadata: include `errorCategory` (transient/validation/permission), `isRetryable` boolean, and a description of what caused the failure.
- B: Create an `analyze_error` MCP tool the agent calls after any failure to determine the error category and recommended action.
- C: Implement retry logic with exponential backoff in your MCP server for all errors, returning to the agent only after retries are exhausted.
- D: Add few-shot examples to the system prompt demonstrating how to interpret error message patterns and select appropriate responses.

**✅ Correct Answer: A**

> **Explanation:** The inconsistency is caused by identical error signals for qualitatively different situations. Structured metadata — `errorCategory`, `isRetryable`, and a cause description — gives the model the information it needs to select the right response deterministically: transient errors → retry, validation errors → fix or ask user, permission errors → escalate. An `analyze_error` tool (B) adds an extra round-trip for every failure. Server-side retries for all errors (C) can mask problems and adds unnecessary latency for non-transient failures.

---

### Question 9 of 60

**After investigating a billing dispute over 25+ turns, you've identified that duplicate charges occurred due to a payment gateway timeout. The required refund ($847) exceeds your $500 authorization limit. You need to call `escalate_to_human`, and the human agent won't have access to your conversation transcript. What context should you pass?**

- A: The complete conversation transcript with all tool results.
- B: ✅ A structured summary: customer ID, root cause, refund amount, and recommended action.
- C: The customer's original complaint verbatim plus tool result excerpts showing duplicate transactions.
- D: Your diagnosis and the refund amount only.

**✅ Correct Answer: B**

> **Explanation:** Effective escalation gives the human agent exactly what they need to act — no more, no less. A structured summary (customer ID, root cause, amount, recommended action) is scannable, actionable, and complete. A full 25+ turn transcript (A) buries the signal in noise. Raw tool excerpts (C) require the human to redo the diagnostic work already performed. Just diagnosis + amount (D) omits the customer ID needed to pull the account. Structured summaries are the gold standard for agent-to-human handoffs.

---

### Question 10 of 60

**When the agent calls `lookup_order` and receives order details showing the item was purchased 45 days ago, how does the agentic loop determine whether to call `process_refund` or `escalate_to_human` next?**

- A: The orchestration layer automatically routes to the next tool based on the order's status field.
- B: The agent executes the remaining steps in a tool sequence planned at the start of the request.
- C: The agent follows a pre-configured decision tree mapping order attributes to specific tool calls.
- D: ✅ The order details are added to the conversation and the model reasons about which action to take.

**✅ Correct Answer: D**

> **Explanation:** This describes how agentic loops fundamentally work. Tool results are returned as messages in the conversation; the model reads them and uses its reasoning to select the next action. There is no separate orchestration layer, decision tree, or pre-planned sequence — the model dynamically determines next steps based on what it observes. A 45-day-old purchase may be outside the return window, which the model can reason about and decide to escalate rather than process the refund. This dynamic, model-driven reasoning is what distinguishes agents from scripted workflows.

---

### Question 11 of 60

**A customer raises three separate issues during one session: a refund inquiry (turns 1-15), a subscription question (turns 16-30), and a payment method update (turns 31-45). At turn 48, the customer asks "What happened with my refund?" The conversation is approaching context limits. What strategy best maintains the agent's ability to address all issues throughout the session?**

- A: Implement sliding window context that retains the most recent 30 turns.
- B: Rely on MCP tools to re-fetch relevant information on demand when the customer references earlier issues.
- C: Summarize earlier turns into a narrative description, preserving full message history only for the active issue.
- D: ✅ Extract and persist structured issue data (order IDs, amounts, statuses) into a separate context layer.

**✅ Correct Answer: D**

> **Explanation:** Three parallel issues require tracking specific data points (IDs, amounts, statuses, resolutions) — not conversation prose. A separate structured context layer acts as a persistent "working memory" that's compact and not subject to window truncation. Sliding window (A) would have dropped the refund discussion from turns 1-15 by turn 48. Narrative summaries (C) lose the precise data needed to answer "What happened with my refund?" Tool re-fetching (B) retrieves current state but cannot recover the history of actions taken during the session.

---

### Question 12 of 60

**When implementing your `lookup_order` MCP tool, the backend sometimes returns errors (e.g., "Order not found" or temporary database failures). What is the correct pattern for communicating these errors back to the agent?**

- A: ✅ Return the error message in the tool result content with the `isError` flag set to `true`.
- B: Log the error server-side and return an empty result to avoid confusing the model.
- C: Return a success response with a "status" field indicating the error type.
- D: Throw an exception from the tool handler so the agent framework can catch and log it.

**✅ Correct Answer: A**

> **Explanation:** MCP defines `isError: true` in the tool result as the correct mechanism for communicating tool-level failures. This keeps the error in the model's reasoning loop so it can respond appropriately. Returning an empty result (B) misrepresents the situation — the model can't distinguish "order not found" from "found nothing." A "status" field on a success response (C) conflates success and failure semantics. Throwing exceptions (D) is a protocol-level error and removes the failure from the model's awareness entirely.

---

### Question 13 of 60

**The agent verifies customer identity through a multi-step process before resetting passwords. After the customer answers the third verification question, the agent asks them to provide their name again, as if the earlier exchange never happened. What's the most likely cause?**

- A: The prompt lacks instructions telling Claude to remember information across multiple exchanges.
- B: Claude's memory retention is limited to two conversational turns by default, requiring explicit configuration to extend it.
- C: The verification tool is clearing the agent's internal state after each successful validation step.
- D: ✅ The conversation history isn't being passed in subsequent API requests.

**✅ Correct Answer: D**

> **Explanation:** The Claude API is stateless — there are no sessions, no persistent memory, and no in-model state between calls. Each API request must include the full conversation history (all prior messages) for the model to have context of what was previously said. There is no "default memory limit" of two turns (B). The verification tool (C) cannot clear model "state" because there is no such state. Instructions (A) can't help if history isn't being passed.

---

### Question 14 of 60

**A customer writes: "I've been going back and forth on this return for days. I just want to speak to someone who can actually help me." The agent has confirmed via `lookup_order` that the return is within policy and eligible for immediate processing. What should the agent do?**

- A: Ask what specifically hasn't worked in previous attempts before deciding whether to escalate or resolve automatically.
- B: Call `escalate_to_human` immediately to honor the customer's request.
- C: ✅ Acknowledge frustration, inform them this is resolvable now, and offer to complete it or escalate.
- D: Process the refund via `process_refund` to resolve the underlying issue, then inform them it's complete.

**✅ Correct Answer: C**

> **Explanation:** Unlike Question 6 (an unambiguous, emphatic demand), this customer is expressing frustration and a preference — not an absolute command. They want the _problem solved_, and the agent _can_ solve it right now. The right response is to acknowledge their frustration, provide new information ("this is actually resolvable right now"), and give them the choice. Processing without asking (D) ignores their stated preference. Immediate escalation (B) ignores that the agent can fully resolve the issue. More questions (A) adds the very friction they're complaining about.

---

### Question 15 of 60

**During a billing dispute resolution, your agent successfully retrieves customer info via `get_customer` and order details via `lookup_order`, but `process_refund` returns a timeout error. The agent can explain the charges and verify refund eligibility but cannot process the refund. What approach best balances first-contact resolution with appropriate error handling?**

- A: Implement automatic retries with exponential backoff for `process_refund`, keeping the conversation open until the refund is successfully processed.
- B: Confirm the refund will be processed and close the conversation, since the system has all necessary information to complete it automatically.
- C: ✅ Explain the billing, confirm refund eligibility, acknowledge the system issue preventing immediate processing, and offer escalation or retry later.
- D: Escalate immediately to a human agent since the refund action cannot be completed.

**✅ Correct Answer: C**

> **Explanation:** The agent should deliver maximum value with the information it _does_ have (billing explanation, eligibility confirmation) while being transparent about what it _can't_ do (process now due to system issue). False confirmation (B) is harmful — the refund won't happen automatically. Immediate escalation (D) throws away the valuable diagnostic work completed. Endless retries (A) create a poor user experience during a transient backend failure. Transparency + partial resolution + clear next steps is the right pattern for graceful degradation.

---

<a name="scenario-2"></a>

## 🔬 Scenario 2: Multi-Agent Research System

**Context:** You are designing an orchestrator-worker architecture for automated research and document analysis. An orchestrator agent delegates tasks to specialized worker agents equipped with MCP tools for structured data retrieval, document analysis, and report generation. Key concerns include reliable tool interface design, safe parameter handling, error taxonomy, tool composition efficiency, parallel execution, and trust boundaries when integrating third-party MCP servers.

---

### Question 16 of 60

**Your MCP server implements a `check_availability` tool that queries an external calendar API. During testing, you encounter three error conditions: (1) the tool is called with a malformed request missing the required `user_email` parameter, (2) the calendar API returns a 404 because the specified user doesn't exist, and (3) the calendar API returns a 503 because the service is temporarily unavailable. How should each error be reported according to MCP's error handling design?**

- A: ✅ Report error 1 as a JSON-RPC protocol error; report errors 2 and 3 as tool results with `isError: true`.
- B: Report all three as tool results with `isError: true`.
- C: Report errors 1 and 2 as JSON-RPC protocol errors; report error 3 as a tool result with `isError: true`.
- D: Report all three as JSON-RPC protocol errors.

**✅ Correct Answer: A**

> **Explanation:** MCP distinguishes between two error layers. JSON-RPC protocol errors represent failures _before_ the tool executes — a missing required parameter means the call itself is malformed and should never reach tool logic. Errors 2 (user not found) and 3 (service unavailable) occur _during_ tool execution and are semantically meaningful outcomes the orchestrator needs to reason about. Returning them as `isError: true` tool results gives the orchestrator the context needed to decide next steps (retry, route to a different worker, adjust the report).

---

### Question 17 of 60

**Your `update_user_profile` tool accepts a `user_id` (required) and an optional `fields_to_update` object. In testing, Claude frequently omits `user_id` or passes incorrectly structured data. What is most critical for helping Claude understand what parameter values to provide?**

- A: Detailed error responses explaining why invalid parameter values were rejected.
- B: Strict JSON Schema type constraints marking `user_id` as required and defining `fields_to_update` as an object type.
- C: Verbose parameter names encoding format hints, such as `user_id_string_uuid_format`.
- D: ✅ Clear parameter descriptions explaining expected format, such as "user_id: UUID of the user to update (required)".

**✅ Correct Answer: D**

> **Explanation:** Claude primarily uses natural language descriptions to understand what values to provide. Clear, explicit descriptions like "UUID of the user to update (required)" give the model both the format expectation and the required/optional status in a form it can directly act on. JSON Schema constraints (B) help with validation _after_ the fact but don't proactively inform the model what to pass. Verbose parameter names (C) are an antipattern — they clutter the interface. Error responses (A) only help after a failure, not before.

---

### Question 18 of 60

**Your shipment tracking tool queries multiple carriers (FedEx, UPS, DHL) that each return status information in different formats — FedEx uses numeric codes, UPS uses descriptive phrases, and DHL returns timestamped event arrays. The agent uses results to determine delivery status and escalate delayed shipments. How should you structure the tool's return value to best enable the agent's reasoning?**

- A: ✅ Return a normalized schema with `status`, `estimated_delivery`, `delay_reason`, and `requires_action` fields, converting carrier formats internally.
- B: Return raw carrier responses with source metadata, encoding carrier-specific interpretation logic in the system prompt.
- C: Return both a normalized summary and the complete raw carrier response in each tool call.
- D: Design separate tool endpoints for each carrier (`track_fedex`, `track_ups`, `track_dhl`) with carrier-specific response schemas.

**✅ Correct Answer: A**

> **Explanation:** In a multi-agent system, the tool layer should absorb complexity so the orchestrator can reason at a semantic level. Normalizing to a common schema means the orchestrator always works with consistent concepts regardless of which data source was used. Raw responses (B) force the orchestrator to parse source-specific formats, increasing error risk. Separate tools per carrier (D) is semantically odd — "track a shipment" is one concept that should have one interface. Both summary and raw (C) doubles token usage without adding reasoning value.

---

### Question 19 of 60

**Your `post_content` tool requires user confirmation before publishing. The current workflow displays "Ready to post to social media. Confirm?" and analytics show users approve 98% of requests within 2 seconds. Post-mortems reveal incidents where posts went to wrong accounts, were scheduled for wrong times, or contained errors — all confirmed by users without catching the mistakes. How should you redesign the confirmation workflow?**

- A: Add a mandatory waiting period before the confirm option becomes available.
- B: ✅ Include the complete post text, target account, scheduled time, and platform in the confirmation request.
- C: Require users to type a confirmation phrase instead of clicking a button.
- D: Auto-approve routine posts and only require explicit confirmation for unusual patterns like posting to new accounts or large audiences.

**✅ Correct Answer: B**

> **Explanation:** The 98% quick approval rate signals users are approving _without reading_ because the confirmation lacks actionable detail. The fix is to present all information that could go wrong — post text, account, time, platform — so users have something meaningful to verify. In a multi-agent research system this maps directly to any irreversible publish action: a confirmation that shows only "Ready to publish report. Confirm?" fails the same way. Users must be shown the full content and destination before confirming. Waiting periods (A) add friction without adding information. Auto-approval (D) moves in the wrong direction for irreversible public actions.

---

### Question 20 of 60

**Your content curation agent discovers articles, analyzes each for relevance, then adds selected articles to themed collections. With separate `discover_articles(topic)`, `analyze_article(id)`, and `add_to_collection(article_id, collection_id)` tools, you observe 18+ sequential tool calls per request. The agent must make editorial judgments requiring seeing all candidates with their analysis scores simultaneously. What tool composition best addresses efficiency while preserving editorial judgment?**

- A: Add a `preview_curation(topic, collection_id)` tool that shows what would be added based on predefined rules, with an `approve_curation()` tool to confirm.
- B: Create a `curate_collection(topic, collection_id)` tool that handles discovery, analysis, and selection internally using configurable quality thresholds.
- C: ✅ Create a `discover_and_analyze(topic)` composite tool that returns all candidates with their analysis scores, keeping `add_to_collection` separate for selective calls.
- D: Keep all tools separate but implement response caching for `analyze_article` calls.

**✅ Correct Answer: C**

> **Explanation:** This is the core orchestrator-worker pattern: a worker tool handles the expensive data collection phase (`discover_and_analyze`), while the orchestrator retains editorial judgment over which articles to add. The composite tool eliminates 17+ sequential worker calls while returning all information the orchestrator needs to reason across candidates simultaneously. Option B removes the orchestrator's editorial agency by making selection happen inside the worker with "configurable thresholds." Option A adds unnecessary confirmation overhead. Caching (D) reduces some latency but doesn't address the sequential call count.

---

### Question 21 of 60

**Your `search_flights` tool calls an external airline API that occasionally returns a 503 Service Unavailable error. What is the most effective way to handle this error in your tool implementation?**

- A: Automatically retry the request up to five times with exponential backoff before returning results to the agent.
- B: Return an empty flight list as if the search succeeded but found no matching flights.
- C: ✅ Return an error message in the tool result explaining the service is temporarily unavailable.
- D: Log the error internally and return an empty response, letting the model continue without the flight data.

**✅ Correct Answer: C**

> **Explanation:** Worker tools must provide accurate signal about what happened. Returning an empty result (B) is a lie — the orchestrator will incorrectly conclude there are no flights, not that the search failed. Silently swallowing the error (D) hides problems from the orchestrator entirely. Retrying 5 times (A) creates unacceptable latency — a better design lets the orchestrator decide retry strategy based on context. Returning a clear, honest error result lets the orchestrator inform downstream steps, try alternative sources, or notify the user.

---

### Question 22 of 60

**Your `search_documents(query)` tool returns results as plain text: "Found 3 documents: 'Q1 Budget Proposal', 'Q2 Budget Forecast', 'Annual Review'". You want the agent to chain this with `share_document(document_id, email)` and `move_document(document_id, folder)`. What return format would best enable these multi-step workflows?**

- A: More detailed human-readable descriptions including file sizes and authors.
- B: URLs that users can click to access documents in their browser.
- C: A JSON array of document titles extracted from the search results.
- D: ✅ Structured data containing document IDs and metadata for each result.

**✅ Correct Answer: D**

> **Explanation:** In a multi-agent workflow, tool outputs must be structured to serve as inputs to downstream tools. Action tools like `share_document` and `move_document` require `document_id` — they cannot work with titles or prose descriptions. This is fundamental to tool chaining design: a worker agent's output format must match the parameter requirements of subsequent worker tools. Human-readable strings (A, C) would require the orchestrator to "guess" IDs that were never returned, making chaining unreliable and error-prone.

---

### Question 23 of 60

**Your marketing agent connects to MCP servers from three different vendors: an email service, a social media platform, and an analytics dashboard. Each server provides tool annotations — some tools marked with `readOnlyHint: true`, others with `destructiveHint: true`. Your team proposes automatically bypassing confirmation prompts for tools annotated as read-only. What should guide this decision?**

- A: ✅ Treat tool annotations as untrusted metadata unless the server itself is trusted. Base confirmation requirements on your assessment of each vendor's trustworthiness, not on their self-reported annotations.
- B: MCP servers run as local processes, so tools from any properly initialized connection inherit your application's security context and can be trusted regardless of vendor.
- C: Read-only annotations indicate intended behavior and are reliable for tools that perform GET requests to external APIs. Skip confirmation for data-fetching tools from properly connected servers.
- D: Implement a verification layer that tests each tool's actual behavior before adding it to an allowlist. Skip confirmation only for tools validated through behavioral testing.

**✅ Correct Answer: A**

> **Explanation:** In a multi-agent system using third-party MCP servers, tool annotations are self-reported metadata — a malicious or poorly implemented server could annotate a destructive tool as `readOnlyHint: true`. Trust must be based on your independent assessment of the server's trustworthiness, not on claims the server makes about itself. This is a core security principle in multi-agent architectures: external agents and tools should be granted only the trust warranted by their actual provenance, not their self-declared capabilities.

---

### Question 24 of 60

**Your order management system requires tools for three distinct operations: issuing refunds (requires amount and reason), canceling orders (requires reason), and requesting reshipments (requires shipping address). Each shares an `order_id` parameter but has different additional requirements. The agent frequently omits required parameters or includes irrelevant ones. What design change will most effectively improve parameter accuracy?**

- A: ✅ Split into three separate tools (`issue_refund`, `cancel_order`, `request_reshipment`), each defining only the parameters required for that specific operation.
- B: Keep one unified tool but add JSON Schema `if-then-else` conditionals to enforce that parameters like `amount` are required only when the operation type is "refund".
- C: Keep one unified tool with a nested `operation_details` object parameter whose internal structure varies by operation type, documented in the tool description.
- D: Keep one unified tool with all parameters marked optional, but add detailed few-shot examples in the system prompt showing correct parameter combinations.

**✅ Correct Answer: A**

> **Explanation:** Each operation has a distinct parameter set. When sharing a single tool, the model must infer which parameters apply based on the operation type — a cognitively complex task prone to errors. Splitting into separate tools makes each interface unambiguous: the model selects the correct tool based on intent, and that tool's parameters are always relevant. This is the "one tool, one job" principle. JSON Schema conditionals (B) are complex to author and maintain; few-shot examples (D) add prompting overhead without fixing the structural problem.

---

### Question 25 of 60

**Your product search tool queries an external catalog API and returns matching items. The agent frequently retries searches immediately after receiving zero results, treating "no matches found" as a failure. The external API returns HTTP 200 with an empty results array — a valid response. How should you restructure the tool's result?**

- A: Return a natural language string describing the outcome, allowing the agent to interpret the result contextually.
- B: Return a result object with `isError: true` and a message explaining no products matched.
- C: ✅ Return a structured result with a `success` boolean and `results` array, reserving `isError: true` for actual execution failures only.
- D: Add a `suggestions` field containing alternative search strategies when results are empty.

**✅ Correct Answer: C**

> **Explanation:** `isError: true` should be reserved for _execution failures_ — situations where the tool could not complete its task. An empty result set is a successful execution that returned no data. Setting `isError: true` for empty results (B) corrupts the signal: the orchestrator can't distinguish "API broke" from "no results found." A structured response with `success: true` and an empty `results: []` accurately communicates what happened. Option D is a nice enhancement but doesn't fix the root miscommunication causing wasteful retries.

---

### Question 26 of 60

**Your `track_shipment(tracking_id)` tool raises a Python exception when errors occur. Users report the agent gives unhelpful responses like "I'm having trouble with that request" instead of suggesting alternatives. How should you handle errors in tool results?**

- A: Return a generic error response (`{"success": false, "error": "lookup_failed"}`) for all failure cases to maintain a consistent schema.
- B: ✅ Return structured error information as normal tool output including error type, recoverability status, and actionable context for the user.
- C: Implement retry logic with exponential backoff inside the tool so transient errors are automatically handled.
- D: Create dedicated error-recovery tools (`retry_tracking_lookup`, `search_by_order_number`) that the model can invoke after the primary tool returns a failure indicator.

**✅ Correct Answer: B**

> **Explanation:** The model needs rich, differentiated error context to generate helpful responses. "API unavailable" (transient, retry suggested), "malformed tracking ID" (validation error, ask user to check), and "shipment not found" (no result, suggest order number lookup) each warrant different user-facing messages. Generic errors (A) deprive the model of actionable signal. Unhandled Python exceptions represent the current failing state. Structured error output — including error type and recoverability — gives the model everything it needs to respond helpfully.

---

### Question 27 of 60

**Your `search_documents` tool needs a parameter to specify which database to search. Your organization has three document databases: "research_papers", "internal_reports", and "technical_specs". Users express this naturally ("search the research database", "check technical documents"). How should you design the database selection parameter?**

- A: ✅ An enum parameter with values `["research_papers", "internal_reports", "technical_specs"]`, requiring the model to map natural language to the appropriate value.
- B: A freeform string parameter with runtime validation that returns an error if the value doesn't match a known database.
- C: A freeform string parameter where the backend uses semantic matching to determine which database(s) to search.
- D: No explicit parameter — search all three databases by default, then have the model filter results by source.

**✅ Correct Answer: A**

> **Explanation:** Enums are ideal when the valid set of values is small, fixed, and known at design time. They constrain the model to valid inputs, prevent typos, and make the interface self-documenting. Claude handles the semantic mapping from "research database" → `"research_papers"` naturally. Freeform strings (B, C) introduce unnecessary validation complexity. Searching all databases (D) is wasteful and may flood the orchestrator with irrelevant noise on each request.

---

### Question 28 of 60

**Your agent includes an `update_game_score` tool that accepts `game_date` (string), `home_team` (string), and `away_team` (string). Production logs reveal recurring issues: team nicknames instead of official names, inconsistent date formats, and the wrong game selected when teams have rematches in the same season. What tool interface change would most effectively prevent these errors?**

- A: ✅ Replace the three parameters with a single `game_id` parameter and a separate `search_games` lookup tool that returns matching game IDs.
- B: Add a `season` parameter to disambiguate rematches, and add a `confirm_before_update` flag.
- C: Add detailed examples to the tool description showing the required date format and complete list of official team names.
- D: Add enum constraints listing valid team names for both team parameters, and a regex pattern enforcing ISO 8601 format for the date parameter.

**✅ Correct Answer: A**

> **Explanation:** All three error modes stem from asking the model to construct a unique identifier for a write operation from ambiguous human-readable inputs. The fix is to separate the lookup from the mutation: a `search_games` tool returns actual `game_id` values; the model then passes the verified ID to `update_game_score`. This "lookup then mutate" pattern eliminates the entire class of ambiguity errors. Constraints (D) help with format but don't solve the rematch disambiguation problem. Examples (C) are the weakest fix for a structural interface problem.

---

### Question 29 of 60

**Your document extraction tool uses ML models to extract invoice fields (vendor, amount, date). The models return confidence scores (0.0-1.0) per field. In production: (1) the agent proceeds with low-confidence extractions that are incorrect 23% of the time, and (2) the agent requests unnecessary human review for 31% of extractions that were actually correct. How should you restructure the tool's output?**

- A: Return fields organized into `verified` and `needs_verification` objects based on confidence thresholds.
- B: ✅ Return fields with confidence scores, plus a `requires_review` boolean computed using your tested confidence thresholds, along with a `review_reasons` array explaining which fields triggered review.
- C: Compute an aggregate `extraction_quality` score across all fields and return it alongside the extracted values.
- D: Return fields with their raw confidence scores and add detailed few-shot examples to your system prompt demonstrating how to interpret confidence ranges.

**✅ Correct Answer: B**

> **Explanation:** The orchestrator is incorrectly calibrating when to escalate because it's making its own threshold decisions from raw numbers. Pre-computing `requires_review` using your _tested_ thresholds moves that calibrated decision into code where you control it. The `review_reasons` array gives the orchestrator actionable context: it can communicate which specific fields need human verification. Option A restructures but doesn't explain _why_. Option D leaves threshold interpretation to the model, recreating the original problem.

---

### Question 30 of 60

**Your research orchestrator dispatches five worker agents in parallel to retrieve data from different sources. Three workers return successfully within 8 seconds, but two are still pending at the 30-second mark with no response. The orchestrator must decide whether to wait or proceed. Which strategy best balances result completeness with system reliability?**

- A: Always wait for all workers to complete before proceeding, regardless of elapsed time, to ensure the report has complete source coverage.
- B: ✅ Set a per-worker timeout; if a worker exceeds it, mark that source as unavailable in the report and proceed with results from completed workers, flagging the incomplete coverage explicitly.
- C: Cancel all pending workers and restart the entire parallel dispatch with a fresh set of worker agents.
- D: Promote one of the slow workers to a "blocking" dependency and apply timeouts only to the remaining pending worker.

**✅ Correct Answer: B**

> **Explanation:** In a multi-agent system, individual worker failures or slowdowns should not block the entire orchestration. Setting per-worker timeouts, proceeding with available results, and transparently flagging incomplete coverage lets the system deliver value while honestly communicating its limitations. Waiting indefinitely (A) creates unbounded latency risk — a single hung worker blocks the entire report. Full restart (C) wastes all completed work and may reproduce the same timeout. Selectively promoting one worker to "blocking" (D) is ad hoc and doesn't resolve the fundamental timeout problem. Partial results with transparent coverage flags are almost always more useful than no result at all.

---

<a name="scenario-3"></a>

## 💻 Scenario 3: Code Generation with Claude Code

**Context:** You are building a developer assistant powered by Claude that helps with large-scale refactoring, code generation in complex existing projects, and extended conversational coding sessions. Key challenges include managing persistent context across sessions (analogous to `CLAUDE.md`), handling behavioral drift in long conversations, preserving critical technical constraints through context compression, and building stateful multi-turn interactions on a stateless API.

---

### Question 31 of 60

**You're implementing a feature where users refine their playlist preferences through multiple conversation turns. After deploying, you notice Claude's responses don't reflect what users said earlier in the same conversation — a user says they love jazz, but two messages later Claude asks what genres they enjoy. What is the most likely cause?**

- A: The model's context window has been exceeded by the conversation length.
- B: The Claude API requires a `session_id` parameter that you haven't configured.
- C: ✅ Your application isn't including prior messages in the `messages` array.
- D: Claude requires a vector database connection to maintain conversation memory.

**✅ Correct Answer: C**

> **Explanation:** The Claude API is stateless — there are no sessions, no persistent memory, and no vector database built in. Each API call is completely independent. To maintain conversation context, your application must include all prior turns in the `messages` array with each request. The conversation history is not stored on Anthropic's side between calls. This is the most common integration mistake when building conversational applications on Claude.

---

### Question 32 of 60

**During initial testing, you notice Claude doesn't remember vocabulary words from earlier in the conversation. When a student asks "Can you quiz me on those words?", Claude responds as if no words have been discussed. What is the most likely explanation?**

- A: The model's context window has filled up, causing earlier content to be dropped.
- B: You need to enable conversation persistence by passing a `session ID` parameter with each API call.
- C: Your system prompt needs explicit instructions telling Claude to remember information from earlier turns.
- D: ✅ You're not including prior messages in each API request — the stateless API doesn't retain conversation history.

**✅ Correct Answer: D**

> **Explanation:** The Claude API is completely stateless. There is no session ID parameter (B), no built-in memory mechanism (C doesn't fix missing history), and the context window (A) is unlikely to fill with vocabulary words. The application is responsible for building and passing the full `messages` array in every API call. This is foundational knowledge for any developer building on the Claude API.

---

### Question 33 of 60

**A new user's first message is "Set up my focus music." This could mean configure preferences, create a playlist, or play music immediately. Your system supports all three actions. What's the most effective approach?**

- A: Create a new "Focus" playlist with curated tracks and notify the user it's ready.
- B: ✅ Ask one clarifying question about action type: play now or configure for later.
- C: Play popular focus tracks immediately and let the user redirect if needed.
- D: Start preference configuration by asking about genres, tempo, and artists they prefer for focus.

**✅ Correct Answer: B**

> **Explanation:** The request is genuinely ambiguous at the action level (immediate playback vs. configuration), and the three interpretations lead to very different outcomes. One targeted clarifying question resolves the ambiguity with minimal friction. Starting configuration (D) assumes the user doesn't want immediate music. Auto-playing (C) makes an unconfirmed assumption. Creating a playlist (A) assumes they need a new list rather than using an existing one. Crucially, B asks _one_ question — not a multi-part interrogation — which respects the user's time and minimizes abandonment risk.

---

### Question 34 of 60

**Your fitness coaching assistant correctly adapts to explicit expertise declarations but struggles with implicit signals — defaulting to over-detailed responses even when users use advanced terminology. Which system prompt change most directly addresses this?**

- A: Add an explicit instruction to ask a clarifying question whenever expertise isn't immediately clear from the first message.
- B: ✅ Replace most conditionals with a general principle: "Adapt explanation depth to match user expertise, mirroring their terminology." Keep only the safety-critical conditional about injury consultations.
- C: Implement a pre-conversation intake asking users to rate their experience level.
- D: Add more conditional branches to cover additional expertise signals, such as "If user mentions specific rep ranges or asks about periodization, treat as advanced."

**✅ Correct Answer: B**

> **Explanation:** The problem is over-reliance on explicit conditional rules that require the user to _declare_ their level. A general principle ("adapt to expertise, mirror terminology") activates the model's contextual reasoning across any signal, including implicit ones like vocabulary choices. This is more flexible than adding more conditionals (D), which still requires enumerated triggers. Pre-conversation intake (C) adds friction. Asking clarifying questions (A) is annoying for advanced users who've already demonstrated expertise through their language.

---

### Question 35 of 60

**Your music discovery assistant should consistently maintain an enthusiastic tone, explain its reasoning for each recommendation, and ask clarifying questions to better understand user preferences. You want this behavior to persist reliably across all user interactions. Where should you define these behavioral guidelines?**

- A: In the first assistant message, instructing Claude to follow these guidelines going forward.
- B: In environment variables that your application passes to the API client.
- C: Prepended to each user message before sending to the API.
- D: ✅ In the system prompt.

**✅ Correct Answer: D**

> **Explanation:** The system prompt is the designated location for persistent operator-level instructions that apply across all turns — analogous to `CLAUDE.md` in Claude Code, which defines project-wide context and behavior. First assistant messages (A) don't carry system-level authority and can be overridden by user instructions. Environmental variables (B) are not a Claude API concept. Prepending to each user message (C) is fragile, clutters the user turn, and conflates operator and user instructions.

---

### Question 36 of 60

**Performance analysis reveals your context is composed of accumulated RAG results from all previous queries, which is crowding out conversation history and causing coherence degradation after 15+ turns. Which approach best addresses this issue?**

- A: Compress all RAG results into a consolidated summary document that updates incrementally after each retrieval.
- B: ✅ Implement a sliding window for RAG results from the last 2-3 queries while preserving conversation history.
- C: Shift context budget to favor RAG results while reducing conversation history allocation.
- D: Implement semantic deduplication to identify and remove redundant information across the accumulated RAG results and conversation turns.

**✅ Correct Answer: B**

> **Explanation:** The cause is identified: accumulated RAG results are crowding out conversation history. The targeted fix is to keep only recent, relevant RAG context (last 2-3 queries) while protecting conversation history that drives coherence. RAG results from 10 turns ago are likely irrelevant to the current query. Option C trades one problem (RAG bloat) for another (conversation coherence). Option A compresses but doesn't prune stale results. Option D is computationally expensive and doesn't address the recency problem.

---

### Question 37 of 60

**After three months of weekly sessions, your conversation history has grown to 85,000 tokens. When users ask "What did we conclude about the theme of isolation?", the assistant provides generic analysis rather than referencing the group's specific insights from earlier sessions. Discussions build on previous meetings' conclusions. What's the most effective approach?**

- A: ✅ Implement progressive summarization where older conversation blocks are replaced with concise summaries that explicitly extract key conclusions, decisions, and recurring themes, while keeping recent exchanges verbatim.
- B: Add structured XML tags to mark significant discussion conclusions throughout the conversation history.
- C: Use semantic embedding to index the full conversation history and retrieve only relevant past exchanges for each user query, replacing the linear conversation format with retrieved chunks.
- D: Implement rolling window truncation to keep only the most recent 25,000 tokens.

**✅ Correct Answer: A**

> **Explanation:** Progressive summarization balances two requirements: preserving the _substance_ of earlier sessions (specific conclusions, decisions, themes) while managing token count. Summaries are structured to _explicitly extract_ key conclusions — not lose them in compression. Rolling window (D) would discard the earlier conclusions users are asking about. XML tags (B) don't reduce token count. Semantic retrieval (C) fragments the narrative thread important for building on past discussions.

---

### Question 38 of 60

**After deploying an updated system prompt that improves response quality, users with multi-session conversations spanning several weeks report the assistant contradicts its earlier statements and has a noticeably different communication style. New users don't experience these issues. What's the best approach to resolve this?**

- A: Add a transition message when sessions resume explaining that the assistant has been updated.
- B: ✅ Version system prompts and associate each conversation with the prompt version under which it started, applying updates only to new conversations.
- C: Add instructions to the new system prompt directing the assistant to maintain consistency with any prior statements in the conversation history.
- D: Regenerate summaries of existing conversations using the new prompt and replace the stored histories to align past context with current behavior.

**✅ Correct Answer: B**

> **Explanation:** The problem is a mismatch between existing conversation context (shaped by the old prompt) and the new prompt's behavior — equivalent to changing `CLAUDE.md` mid-project. The cleanest solution is prompt versioning: conversations continue under the prompt version that shaped them; only new conversations get the new behavior. Option C (consistency instructions) can't override fundamental behavioral changes in the new prompt. Option D (regenerating summaries) risks introducing inconsistencies and is expensive.

---

### Question 39 of 60

**Your research assistant helps users analyze academic papers over extended conversations. After 60K+ tokens, users ask follow-up questions requiring precise numerical details from papers discussed earlier — sample sizes, exact p-values, inclusion criteria. Your current approach summarizes paper discussions after 8 turns. Responses are often hedged or inaccurate. What's the most effective architectural change?**

- A: ✅ Maintain a structured database of key facts extracted from each paper (sample sizes, statistics, methods) and retrieve relevant entries into context when precision-dependent questions are detected.
- B: Implement retrieval that re-injects relevant paper sections when the user's question suggests specific numerical details are needed.
- C: Keep source text from methodology and results sections in context permanently, while summarizing only the conversational discussion portions.
- D: Use a separate Claude call with explicit instructions to generate higher-fidelity summaries that preserve all numerical details.

**✅ Correct Answer: A**

> **Explanation:** The problem is that free-text summarization loses specific numerical values. The solution is to preserve those values in a structured, queryable format — separate from the conversational context. When precision-dependent questions arrive, retrieve the relevant structured facts into context. Option B (re-injecting full paper sections) is less precise and token-efficient. Option C keeps too much in context permanently. Option D still relies on a summarization step that may lose numerical precision.

---

### Question 40 of 60

**Users report the AI loses track of specific topics, examples, and preferences from earlier in the session. Your current implementation uses a sliding window that keeps only the most recent 25 message pairs. What's the most effective approach to maintain awareness of earlier conversation content while managing context size?**

- A: Increase the window size to 50 message pairs to retain more conversation history before truncation.
- B: Add a separate API call each turn to summarize messages being dropped, prepending this running summary to the conversation.
- C: ✅ Replace the sliding window with a hybrid approach: summarize older messages while keeping recent messages verbatim.
- D: Implement vector similarity search over the full conversation history, retrieving relevant past messages for each user query.

**✅ Correct Answer: C**

> **Explanation:** A hybrid approach preserves the best of both worlds: recent messages stay verbatim (full fidelity for active context) while older messages are compressed into summaries (important themes and decisions retained at lower token cost). Simply expanding the window (A) delays the problem without solving it. Separate summarization API calls (B) add latency and cost per turn. Vector retrieval (D) is powerful but can miss relevant context that doesn't semantically match the current query.

---

### Question 41 of 60

**Users report that responses feel repetitive across turns — each message begins with phrases like "Certainly!" or "I'd be happy to help!" even deep into conversations. What's the most effective approach?**

- A: Implement post-processing to detect and strip common greeting phrases from response beginnings.
- B: ✅ Add system prompt instructions specifying phrases to avoid, such as "Never begin responses with 'Certainly' or similar affirmations."
- C: Append a partial assistant message with a direct response opening that the model will continue from.
- D: Lower the temperature parameter to make response openings more deterministic and less variable.

**✅ Correct Answer: B**

> **Explanation:** Direct system prompt instructions specifying prohibited phrases are the standard, most maintainable approach. They address the problem at the source (model behavior) rather than after the fact (A). Temperature (D) affects randomness but doesn't target specific phrases. Partial assistant turn injection (C) is a valid technique but creates rigid response structure and needs maintenance as conversation styles evolve.

---

### Question 42 of 60

**During QA testing, Claude follows your system prompt guidelines consistently for the first 10-15 turns, but by turn 25-30, responses begin deviating — informal tone, skipped formatting, restricted information appearing. Context is well within limits. What's the most effective approach to maintain consistent behavior?**

- A: Automatically start a new conversation after 20 turns, passing a summary of the prior context.
- B: ✅ Insert user-role messages that reinforce critical guidelines at natural conversation breakpoints, especially before complex requests.
- C: Move behavioral guidelines from the system prompt into the first user message.
- D: Implement post-response validation that regenerates each response until it conforms to the specified guidelines.

**✅ Correct Answer: B**

> **Explanation:** "Instruction following drift" in long conversations is a known phenomenon where the model's attention to early instructions weakens as more context accumulates. Periodically reinforcing critical guidelines at natural breakpoints re-anchors the model's behavior. Option A (restart at 20 turns) disrupts user experience. Moving instructions to the user turn (C) gives them less authority than the system prompt. Auto-regeneration (D) is expensive and creates unpredictable latency.

---

### Question 43 of 60

**After a 40-minute session, the conversation has grown to 78,000 tokens. The history includes: (1) a severe shellfish allergy, (2) specific measurements scaled to 8 servings, (3) a user-defined term ("room temperature" = 68°F), and (4) general back-and-forth about timing and presentation. You need to implement context management. What approach best balances preservation with token reduction?**

- A: Summarize the entire conversation history into a concise summary, then append new messages going forward.
- B: Store the full conversation externally and use semantic search to retrieve relevant portions for each turn.
- C: ✅ Extract critical structured data (allergies, serving counts, user-defined terms) into a compact reference section, summarize general discussion, and retain recent exchanges verbatim.
- D: Implement a sliding window retaining only the most recent 20,000 tokens, relying on users to re-state important information when relevant.

**✅ Correct Answer: C**

> **Explanation:** Different information requires different preservation strategies. Critical facts that must never be lost — safety constraints, version pins, user-defined terms — belong in a structured reference section. General discussion about approach can be summarized. Recent exchanges stay verbatim for continuity. A single summary (A) risks losing safety-critical data mentioned early. Semantic search (B) is risky for safety-critical information that might not match current query semantics. Sliding window (D) drops critical data if it occurred early in the session.

---

### Question 44 of 60

**Users report that API latency increases noticeably and costs rise as practice conversations extend beyond 50+ turns. What is the PRIMARY cause of this behavior?**

- A: ✅ The entire conversation history is included with each API request, so input tokens grow with every turn.
- B: Database operations for retrieving and storing conversation history slow down as the table grows larger.
- C: The model builds an internal profile of the user's conversation patterns, requiring more processing as the profile grows.
- D: The model generates progressively longer responses as it accumulates more context to reference.

**✅ Correct Answer: A**

> **Explanation:** The Claude API is stateless — every request includes the full conversation history. As the conversation grows, each request sends more input tokens. More input tokens = higher cost and higher latency, since processing time scales with input size. There is no database latency in the standard API (B). There is no persistent user profile being built across calls (C). Response length is not inherently tied to context length (D).

---

### Question 45 of 60

**Your conversational assistant frequently generates multiple clarifying questions when users make ambiguous requests. "Can you help me with the report?" triggers a 3-question response. User analytics show a 40% conversation abandonment rate. What's the most effective fix?**

- A: Limit the assistant to one clarifying question per turn, accumulating answers over multiple exchanges.
- B: Create a lookup table of common request patterns with predefined default interpretations, responding without stating the assumptions made.
- C: Add a preprocessing step using a smaller model to classify request ambiguity, routing high-ambiguity requests to a clarification dialog.
- D: ✅ Modify the system prompt to instruct the assistant to make reasonable assumptions from available context, state those assumptions explicitly, and offer to adjust if the interpretation is wrong.

**✅ Correct Answer: D**

> **Explanation:** The best response to most ambiguous requests is to make a reasonable interpretation, act on it, and offer to adjust — not interrogate the user. "I'll assume you need help drafting the report — let me start with an outline. Let me know if you meant something else." This gets the conversation moving. Option A (one question per turn) still delays. Option B silently assumes, creating worse surprises when the assumption is wrong. Option C adds infrastructure complexity for a problem better solved in the prompt.

---

<a name="scenario-4"></a>

## 📊 Scenario 4: Structured Data Extraction

**Context:** You are implementing a high-reliability data parsing pipeline that extracts structured information from unstructured documents — menus, contracts, product listings, resumes, and reviews. The pipeline must produce valid JSON conforming to predefined schemas, handle ambiguous or conflicting source data, and route uncertain extractions for human review — all while managing API costs at scale using the Message Batches API.

---

### Question 46 of 60

**Your extraction pipeline processes restaurant menus and must output structured JSON with fields for item names, descriptions, prices, and dietary tags. Some menus use inconsistent formatting — prices as "$12" vs "12.00", dietary info as icons vs text. What's the most reliable approach?**

- A: ✅ Define a strict output schema and include format normalization rules in your prompt.
- B: Extract data as-is and normalize formats in post-processing code after Claude returns.
- C: Use separate extraction calls for each field to ensure consistent handling of each type.
- D: Request multiple extraction attempts per document and select the most common format.

**✅ Correct Answer: A**

> **Explanation:** Combining a strict output schema with prompt-level normalization rules produces consistent output in one pass. The model handles both extraction _and_ normalization simultaneously. Option B (post-processing normalization) works but adds pipeline complexity and an additional failure point. Separate calls per field (C) multiplies API costs and latency proportionally. Multiple attempts and voting (D) is expensive and doesn't address the root consistency issue.

---

### Question 47 of 60

**Your extraction system processes two document types: standard monthly reports (archived after processing) and urgent exception reports (must trigger business alerts within 30 minutes). Both use the same JSON schema. You want to minimize API costs while meeting latency requirements. How should you architect the pipeline?**

- A: Queue all documents and submit hourly batches, flagging urgent documents for expedited handling when batch results return.
- B: Submit all documents to the Batch API with `custom_ids` for tracking. When results arrive, immediately process urgent documents.
- C: ✅ Route standard reports to the Batch API for 50% cost savings, and route urgent exception reports to the real-time Messages API.
- D: Submit all documents to the real-time Messages API to ensure consistent processing latency across document types.

**✅ Correct Answer: C**

> **Explanation:** The Message Batches API offers 50% cost savings but has up to a 24-hour processing window — incompatible with a 30-minute alert requirement. The correct architecture routes based on SLA: standard reports get the cost savings of batch processing; urgent reports get the latency guarantees of real-time processing. Option B cannot guarantee 30-minute turnaround since batch results can take up to 24 hours. Option D forfeits all cost savings unnecessarily.

---

### Question 48 of 60

**After your daily batch of 10,000 documents completes, 300 documents (3%) failed with "context_length_exceeded" errors. The results file identifies each failure by `custom_id`. What's the most cost-effective approach to process these failures?**

- A: Increase the `max_tokens` parameter for the 300 failed documents and resubmit them in a new batch.
- B: Resubmit the entire 10,000 document batch using a model tier with a larger context window.
- C: ✅ Resubmit only the 300 failed documents after chunking them into smaller pieces, then combine the partial extractions.
- D: Reprocess the entire batch with prompt caching enabled to reduce the cost of retrying requests with identical system prompts.

**✅ Correct Answer: C**

> **Explanation:** `context_length_exceeded` means the documents are _too large to process as-is_ — increasing `max_tokens` (A) won't help since `max_tokens` controls _output_ length, not input capacity. Resubmitting all 10,000 documents (B, D) wastes the cost of the 9,700 successful ones. The correct approach is to resubmit only the 300 failures, chunked into pieces that fit within the context window, then combine the partial extractions.

---

### Question 49 of 60

**Your extraction system parses e-commerce product descriptions to extract specifications into JSON. Despite having a well-defined schema, the model inconsistently extracts the "materials" field — different formats, occasional omissions when material info is clearly present. What's the most effective improvement?**

- A: Make the "materials" field required instead of optional in the schema to force the model to always extract a value.
- B: Switch to a more capable model tier since inconsistent extraction indicates insufficient model capability.
- C: Set temperature to 0 to eliminate randomness and ensure deterministic outputs.
- D: ✅ Add few-shot examples showing 2-3 complete input-output pairs with standardized material description formats.

**✅ Correct Answer: D**

> **Explanation:** The problem is inconsistent _format_ and occasional omissions — the model can extract materials but doesn't know which format to use. Few-shot examples provide concrete demonstrations of the desired input→output mapping, including format normalization. Temperature=0 (C) reduces randomness but doesn't teach the desired format. Making the field required (A) prevents omissions but doesn't fix format inconsistency. A more capable model (B) is an expensive guess when the issue is a clear training signal gap.

---

### Question 50 of 60

**Your extraction pipeline processes contracts that frequently include amendments. When a contract contains both original terms ("30-day payment") and amendments ("45 days"), the model inconsistently extracts one value or the other with no indication of which applies. What's the most effective approach?**

- A: ✅ Redesign the schema so amended fields capture multiple values, each with source location and effective date.
- B: Implement post-extraction validation using pattern matching to detect amendments and flag those extractions for manual review.
- C: Preprocess documents with a classifier that identifies and removes superseded sections before the main extraction step.
- D: Add prompt instructions to always extract the most recent amendment value and ignore superseded original terms.

**✅ Correct Answer: A**

> **Explanation:** The schema should represent the reality of the document. Contracts with amendments genuinely contain multiple versions of a term; a single-value field can't faithfully represent that. Capturing each value with its source location and effective date enables downstream systems to apply the correct precedence rules with full traceability. Option D destroys original terms that may be legally relevant. Option C risks removing content that still matters. Option B flags rather than solves the fundamental schema mismatch.

---

### Question 51 of 60

**Your extraction uses tool use with a JSON schema where `property_type` is defined as an enum: `['house', 'apartment', 'condo', 'townhouse']`. After deployment, 8% of extractions fail schema validation. Investigation reveals listings mention many uncommon types — "studio", "loft", "duplex", "mobile home" — and new types continue appearing regularly. What's the most effective long-term solution?**

- A: Change `property_type` from an enum to a free-form string and implement a normalization step in post-processing.
- B: ✅ Add an `"other"` value to your enum with a separate `property_type_detail` string field for specifics when `"other"` is selected.
- C: Continuously expand the enum to include newly observed property types and add monitoring for additional edge cases.
- D: Add few-shot examples to your prompt demonstrating how to map unexpected property types to the closest existing enum value.

**✅ Correct Answer: B**

> **Explanation:** Adding `"other"` with a detail field handles the open-ended nature of property types while preserving enum structure for the common cases. The system remains compatible with consumers that rely on the enum, while new/rare types are captured in `property_type_detail`. Free-form strings (A) lose the categorical structure entirely. Continuously expanding the enum (C) is a maintenance treadmill that never ends. Few-shot mapping (D) forces imprecise categorization — a "studio" mapped to "apartment" loses meaningful information.

---

### Question 52 of 60

**Your system extracts event metadata (date, location, organizer, `attendee_count`) from news articles using a JSON schema with all nullable fields. The model frequently generates plausible but incorrect values for fields not mentioned in the article — for example, outputting "500" for `attendee_count` when no attendance data exists. What's the most effective way to reduce these false extractions?**

- A: Make all schema fields required (non-nullable) with strict validation rules to ensure the model only outputs verifiable data.
- B: ✅ Add prompt instructions to return `null` for any field where information is not directly stated in the source.
- C: Upgrade to a more capable model tier with improved instruction-following to reduce hallucination tendencies.
- D: Add a post-processing step using a second LLM call to verify each extracted value exists in the source document.

**✅ Correct Answer: B**

> **Explanation:** The model is hallucinating values to fill nullable fields, likely because the extraction prompt implicitly encourages filling in plausible values. An explicit instruction to return `null` when information is _not directly stated_ reframes the task as "extract what's there" rather than "fill in the form." Making fields required (A) would _increase_ hallucination — with no null option, the model must invent values for every field. A verification LLM call (D) adds significant cost for what is a prompting problem.

---

### Question 53 of 60

**Your team is extracting structured data from 50,000 legacy legal contracts under a two-week deadline. Initial testing on 500 samples shows 82% pass JSON schema validation on first attempt; the remaining 18% fail from diverse issues requiring 2-3 prompt refinements. Which batch processing strategy is the most cost-efficient while meeting the deadline?**

- A: Submit all 50,000 via batch API, then submit failed extractions in successive batches — refining prompts between each batch.
- B: Use the real-time API for all 50,000 documents since the batch API's 24-hour window creates unacceptable deadline risk.
- C: ✅ Process 2,000 sample documents via real-time API to identify failure patterns and refine prompts, then batch process all 50,000 with the optimized prompts.
- D: Split into 10 sequential batches of 5,000 each, analyzing results and refining prompts between batches.

**✅ Correct Answer: C**

> **Explanation:** The key insight is to invest upfront in prompt optimization using a small real-time sample, then batch process the full corpus once with optimized prompts. This front-loads the learning, minimizes the volume processed with suboptimal prompts, and uses batch processing for the bulk work (50% cost savings). Option A submits 50,000 with an 18% failure rate and then iterates expensively. Option B doubles cost unnecessarily. Option D processes ~35,000 documents before fully learning from failure patterns.

---

### Question 54 of 60

**The system extracts candidate information (name, contact details, skills, work experience, education) from uploaded resumes. The extracted data must strictly conform to a predefined JSON schema, as missing required fields or incorrect data types will cause downstream validation failures. What is the most reliable approach?**

- A: Parse Claude's text response with regex patterns to extract JSON objects, using retry logic for malformed responses.
- B: ✅ Define a tool with an input schema matching your required JSON structure and extract the data from Claude's `tool_use` response.
- C: Make two separate API calls — first extracting information as text, then asking Claude to format that text as JSON.
- D: Include detailed JSON formatting instructions and a template example in the system prompt, asking Claude to output only valid JSON.

**✅ Correct Answer: B**

> **Explanation:** Tool use (function calling) is the most reliable mechanism for structured output. When you define a tool with an input schema, Claude is constrained to produce output matching that schema as part of the `tool_use` response — not as free text that might contain markdown or prose artifacts. Regex parsing (A) is brittle and fails on edge cases. Two-pass approaches (C) double cost and latency without adding reliability. Prompt-based JSON instructions (D) work most of the time but lack the structural guarantee that tool use provides — critical when downstream systems depend on strict schema compliance.

---

### Question 55 of 60

**Your schema includes a `skills: string[]` field. Production monitoring reveals three consistency issues: (1) compound phrases like "Python and SQL" are sometimes kept as one entry, sometimes split; (2) implied but unstated skills occasionally appear; (3) similar documents produce wildly different array lengths (5-10 vs 40+ entries). Your prompt says "Extract all skills mentioned." What's the most effective improvement?**

- A: Add post-extraction normalization that maps skills to a canonical taxonomy and deduplicates similar entries.
- B: ✅ Add few-shot examples demonstrating compound phrase handling, explicit mention criteria, and appropriate entry granularity.
- C: Add constraints: "Extract 10-20 skills maximum, one skill per entry, only explicitly named skills."
- D: Enrich the schema to `{skill: string, confidence: float, source_quote: string}[]` to capture extraction metadata.

**✅ Correct Answer: B**

> **Explanation:** All three issues stem from underspecified extraction criteria. Few-shot examples teach the model _how_ to apply the rules through demonstration: showing a compound phrase being split correctly, an implied skill being excluded, and what an appropriate-length skills list looks like. Option C adds hard constraints that address some issues but a hard cap of 20 may truncate genuinely skill-rich resumes. Few-shot examples provide nuanced guidance that blunt constraints alone can't convey. Post-processing normalization (A) helps but doesn't prevent root inconsistency.

---

### Question 56 of 60

**Specifications sometimes conflict within source documents. A summary section states "Battery: 4000mAh" while the detailed specs table shows "Battery: 4200mAh." Your current schema uses a single `battery_capacity_mah` field. This inconsistency occurs in ~15% of documents; the detailed specs table is more accurate 90% of the time. What's the most effective approach?**

- A: Change to an array field that captures all values found with their source locations, letting downstream systems apply precedence rules.
- B: Add a `conflict_detected` boolean field that flags inconsistencies, triggering manual review for affected documents.
- C: Implement schema validation that rejects results containing conflicting values, requiring source document correction before processing.
- D: ✅ Include extraction instructions specifying to prefer values from the detailed specs table when multiple values exist, keeping the single-value schema.

**✅ Correct Answer: D**

> **Explanation:** When you have a known source priority (specs table is more accurate 90% of the time), encode that preference as an extraction instruction. This is a simple, cost-effective fix that maintains a clean single-value schema. Option A adds schema complexity for downstream consumers who just want the right value. Option B triggers manual review but doesn't resolve the conflict. Option C requires manual correction of 15% of documents. Always use domain knowledge in extraction instructions before complicating the schema.

---

### Question 57 of 60

**The system processes product reviews using tool use with a defined schema: `rating` (integer 1-5), `pros` (string array), `cons` (string array), and `overall_sentiment` (enum: positive, negative, mixed). Two issues emerge with brief or ambiguous reviews (~20% of dataset): (1) for reviews like "Great product!", Claude fabricates specific pros and cons, and (2) for sarcastic reviews like "Well that was... interesting", Claude picks sentiment arbitrarily since there's no option for ambiguous cases. What schema modification best addresses both issues?**

- A: ✅ Allow `null` values for pros/cons, and add `"unclear"` to the sentiment enum.
- B: Make pros and cons optional fields, and add `"neutral"` and `"unclear"` to the sentiment enum.
- C: Add an `extraction_confidence` field (0.0-1.0) for each value, and filter outputs where any confidence falls below a threshold.
- D: Allow empty arrays for pros/cons as valid output, and add `"unclear"` to the sentiment enum.

**✅ Correct Answer: A**

> **Explanation:** Null values for pros/cons signal "this information was not present in the source" — semantically distinct from an empty array which could mean "the reviewer had no specific pros/cons to mention." Null explicitly communicates _absence of information_, preventing fabrication. Adding `"unclear"` to the sentiment enum gives the model a valid option for genuinely ambiguous or sarcastic content. Empty arrays (D) are semantically ambiguous. Optional fields (B) don't signal the difference between "not mentioned" and "mentioned but empty." Confidence scores (C) flag the problem but don't fix the schema gap causing fabrication.

---

### Question 58 of 60

**Your extraction pipeline validates outputs against JSON schemas, but you need to implement human review given limited reviewer capacity (approximately 5% of total volume). What's the most effective basis for selecting which extractions to route for human review?**

- A: ✅ Route extractions where the model indicates low confidence or where source documents contain ambiguous or contradictory information.
- B: Randomly sample 5% of extractions for review.
- C: Route extractions for review only when downstream systems report data quality issues or processing failures.
- D: Route extractions containing specific high-priority entity types (e.g., financial figures, dates) for human review, regardless of extraction confidence.

**✅ Correct Answer: A**

> **Explanation:** Risk-based routing — directing review capacity toward extractions most likely to contain errors — maximizes the value of limited reviewer capacity. Low model confidence and source ambiguity/contradiction are strong predictors of extraction errors. Random sampling (B) wastes capacity on high-quality extractions. Downstream failure routing (C) is reactive — errors have already propagated. Entity-based routing (D) over-routes potentially high-quality extractions of critical fields while missing low-confidence extractions of less critical fields.

---

### Question 59 of 60

**Documents arrive continuously throughout business hours and need structured data extracted. To reduce costs, you want to use the Message Batches API (50% discount, up-to-24-hour processing window). Your SLA specifies that extraction results must be available within 30 hours of document arrival with 99.9% reliability. Which batching strategy is most appropriate?**

- A: Use the real-time API for all documents instead of batch processing.
- B: Submit a single batch at end of day containing all documents from that day.
- C: ✅ Submit batches every 4 hours containing documents from that window.
- D: Submit batches every 6 hours containing documents from that window.

**✅ Correct Answer: C**

> **Explanation:** Work backward from the SLA: 30-hour deadline = batch wait time + processing time. With up to 24 hours of processing, you have at most 6 hours of wait time budget. With 6-hour batches (D): max wait = 6h + 24h = **30h exactly** — zero buffer, making 99.9% reliability impossible. With 4-hour batches (C): max wait = 4h + 24h = **28h** — a 2-hour buffer that supports 99.9% reliability. End-of-day batching (B) violates the SLA for early-day documents. Real-time API (A) forfeits the 50% cost savings.

---

### Question 60 of 60

**Your pipeline processes 8,000 product listings daily using the Message Batches API. Each request uses the same 1,800-token system prompt (JSON schema definition + extraction instructions + normalization rules) alongside variable product description content. Without optimization, input token costs dominate your monthly bill. Which approach most effectively reduces input token costs for this workload?**

- A: Switch to a model with a larger context window to process multiple product listings in a single API call, reducing the number of requests.
- B: ✅ Enable prompt caching on the system prompt so its tokens are billed at the reduced cache-read rate on repeated requests within the same batch.
- C: Shorten the system prompt by removing normalization rules and schema documentation, relying on the model's base instruction-following capability.
- D: Submit documents in fewer, larger batches to amortize the per-batch fixed overhead across more documents.

**✅ Correct Answer: B**

> **Explanation:** Prompt caching is designed for this use case: a fixed system prompt repeated across many requests. Cached tokens are billed at significantly reduced rates compared to uncached input tokens when cache reads apply. Message Batches support prompt caching, but because batch requests are processed asynchronously and concurrently, cache hits are best-effort — include identical `cache_control` blocks and follow Anthropic's batch + caching guidance to maximize hit rates. With a 1,800-token system prompt across 8,000 daily requests, successful cache reuse can greatly reduce full-price input-token spend versus no caching. Option A (multi-document batching per call) reduces request count but complicates response parsing and doesn't reduce token cost per document. Option C (shorter prompt) risks degrading extraction quality — the normalization rules and schema documentation are doing real work. Option D addresses per-batch API overhead, which is not the dominant cost in this scenario; token cost is.

---

## Exam access and registration

Registration runs through the Anthropic Partner Academy and requires affiliation with a Claude Partner Network organization; there is no individual sign-up. Exams are delivered by [Pearson VUE](https://www.pearsonvue.com/us/en/anthropic.html) (online proctored or at a test center), the fee is $125 per attempt as of the mid-2026 exam guide, and credentials are valid for 12 months with a free on-time renewal.

No partner organization? The routes in are an employer that joins the [Claude Partner Network](https://claude.com/partners), a qualifying company of your own, or membership in an existing partner firm ([how the partner-firm route works](https://youraidept.com/network/claude-certification) — disclosure: that guide is maintained by YAID, a partner firm).
