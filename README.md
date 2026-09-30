# Supplier Evaluation & Operations Agent

A multi-agent AI workflow built in **n8n** that evaluates laptop supplier quotations for a fictional company, **Everest Manufacturing Pvt. Ltd.**, and sends a management-ready report by email for human approval.

Built as a university capstone project.

> The system supports the decision. It never approves a purchase. The Operations Manager gives the final approval.

## The business problem

Everest Manufacturing needs 25 laptops (budget NPR 2,500,000). Three suppliers sent quotations with different prices, specifications, delivery times, warranties and payment terms, and each has a different track record. The Operations team compares them by hand, which is slow, inconsistent between reviewers, and can miss risks such as late deliveries or missing warranty information.

## Design idea

**AI does the reading, comparing and explaining. Fixed code applies the rules. A human makes the decision.**

Policy thresholds (budget, minimum warranty, rating, late deliveries) are exact, so they are checked in code and give the same result on every run. Language-model agents handle interpretation, risk explanation, trade-offs and report writing.

## Workflow

```
Chat Trigger
   -> Load Business Data
   -> Agent 1  Intake & Validation
   -> Rule Engine (code)
   -> Agent 2  Compliance & Risk
   -> Agent 3  Comparison & Recommendation
   -> Agent 4  QA Critic
   -> Agent 5  Report Writer
   -> Prepare Email
   -> Gmail: Send and Wait for Approval
   -> Approved? -> record decision and finish
```

| Component | Responsibility |
|---|---|
| Agent 1 - Intake & Validation | Structures the data and applies changes typed in chat. Unknown values become `null`, never guessed. |
| Rule Engine (code) | Checks every supplier against mandatory requirements, preferences and policy flags. |
| Agent 2 - Compliance & Risk | Turns rule results into a risk register with evidence and clarification questions. It cannot change any check result. |
| Agent 3 - Comparison & Recommendation | Compares suppliers in tiers (mandatory gate, missing information, policy flags, preferences, combined comparison). Never picks on price alone. |
| Agent 4 - QA Critic | Audits the recommendation for invented facts, ignored rules and price-only logic. |
| Agent 5 - Report Writer | Writes the HTML report. It may not add facts or re-evaluate. |
| Gmail Send and Wait | Emails the report with Approve and Decline buttons and pauses until the manager responds. |

### How requirements are handled

| Category | Examples | Behaviour |
|---|---|---|
| Mandatory | Budget, quantity, RAM, storage, OS, processor, warranty | A failure makes the supplier non-compliant. |
| Preference | Delivery within 14 days, payment 30 days after delivery | Counted as a trade-off, never disqualifies alone. |
| Policy flag | Rating below 3, more than 2 late deliveries, missing tax, warranty, delivery or payment information | Flagged for management review. |
| Unclear | Delivery is "Required" in the purchase request but "Preferred" in the policy | Treated as a preference and flagged for the manager to interpret. |

## Tech stack

- [n8n](https://n8n.io) (self-hosted, Docker)
- OpenRouter with an OpenAI nano model (default `openai/gpt-5-nano`, configurable per agent)
- Gmail (Send and Wait for Response)
- JavaScript in n8n Code nodes

## Setup

1. Run n8n (for example with Docker) and open it in the browser.
2. Import `workflow/Everest_Supplier_Evaluation_Workflow.json` (Workflows, then Import from File).
3. Add credentials:
   - **OpenRouter API** on the five model nodes.
   - **Gmail OAuth2** on the Gmail node.
4. In the **Send Report & Wait for Approval** node, replace the `sendTo` address with the Operations Manager's email.
5. Click **Chat** on the canvas and send a message, for example:
   `Run the evaluation with the provided dataset.`

Credentials are not included in this repository. Never commit API keys.

## Test scenarios

Changes to the evidence are typed into the chat message, so no node needs editing.

| # | Chat message | What it tests |
|---|---|---|
| 1 | `Run the evaluation with the provided dataset.` | Normal operation |
| 2 | `Run the evaluation. Digital Hub's warranty is not stated in their quotation.` | Missing information |
| 3 | `Run the evaluation. TechNova's unit price is now NPR 105,000.` | Policy conflict (budget) |
| 4 | `Run the evaluation. Apex confirmed a 3-year warranty.` | Changed evidence |

### Results observed

| # | Outcome |
|---|---|
| 1 | Ran end to end. TechNova recommended. Digital Hub compliant with flags (21-day delivery, advance payment, 3 late deliveries). Apex non-compliant (1-year warranty). |
| 2 | Warranty stayed `null` and shown as "cannot verify". Digital Hub marked as needing clarification. No value invented. |
| 3 | Total recalculated to NPR 2,625,000. TechNova failed the budget check and was not recommended. Overrun stated correctly. |
| 4 | Run completed, but the report showed only one supplier and did not reflect the change (see known issues). |

## Known issues and limitations

Being explicit about these is part of the project.

- **Report writer variability.** In some runs the report omitted suppliers or added statements not present in the data. Planned fix: build the comparison table and requirements check in code from the Rule Engine output, and let the agent write only the narrative.
- **Agent 3 can be over-cautious.** In one run it returned no recommendation although one supplier passed all mandatory checks.
- **QA can raise false alarms.** In one run it reported a failure for a correctly flagged input problem.
- **The QA agent is also a language model.** It reduces errors but does not guarantee correctness.
- **Data is supplied by hand.** There is no live supplier or procurement system, and the system cannot verify ratings or history.
- **Nano models vary between runs.** A stronger model for Agents 3 and 4 may improve reliability.
- **No audit log or access control** in this prototype.

## Human oversight

- The Gmail approval step is the only place a purchase decision is made.
- If the manager does not respond within 48 hours, the outcome is recorded as declined or no response.
- The system never approves a purchase, raises a purchase order, or makes a payment.

## Future improvements

1. Connect real data sources and validate incoming data.
2. Store every run, its outputs and the manager's decision in an audit log.
3. Generate tables in code and restrict agents to narrative sections.
4. Add a reviewed step for asking suppliers to clarify missing information.
5. Test with more real quotations and compare against manual evaluations.
6. Confirm ambiguous policy wording (such as required vs preferred delivery) with management.

## Repository structure

```
.
├── README.md
├── .gitignore
├── workflow/
│   └── Everest_Supplier_Evaluation_Workflow.json
└── docs/
    └── Everest_Capstone_Report.docx
```

## Disclaimer

Everest Manufacturing, the suppliers and all figures are fictional and come from a course assignment. This is a prototype for learning, not production software.
