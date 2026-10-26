**Language:** [ภาษาไทย](04-review-board.md) | English

Google Link : https://docs.google.com/document/d/190EcYL-WkAn7ErngxUUHo4imKY1ttGpcpybmkp6Dla8/edit?usp=sharing

# Part B: Architecture review board (approximately 25 min, groups of 4)

> **Lecture link:** AICH4 (reviews need domain expertise; _plausible ≠ appropriate_), Pillar 5 (AI as Chief Analyst, humans as Chief Strategist), JSS (85% of studies keep a human in the loop).
> **Question:** what does a human review board catch that the AI's review missed, and why?

The company's architecture review board meets to approve the R-12 design before the 11.11 sale. One ADR goes to the board: the **architect** presents it, the **AI analyst** presents the AI's review, and two stakeholders bring knowledge the AI never saw.

## 1. Form the board (2 min)

In groups of 4, choose **one** member's ADR to review (`docs/adr/0002-*.md`, plus their `exercise/outputs/3-review.md`). Then assign roles:

| Role                                | Who                              | Brings                                                             |
| ----------------------------------- | -------------------------------- | ------------------------------------------------------------------ |
| **Architect**                       | Naphat Serirak (6632061321)      | defends the ADR                                                    |
| **AI analyst**                      | Chanatda Konchom (6631305321)    | reads out the AI's review only, and may not add their own opinions |
| **Product owner**                   | Ittichet Thongsang (6631363721)  | role card P (business side)                                        |
| **Site reliability engineer (SRE)** | Chayut Archamongkol (6631307621) | role card S (operations side)                                      |
| **Security & privacy officer**      | Napat Srisamut (6631314021)      | role card X (security & privacy)                                   |

Groups of 5 add a **security & privacy officer** (role card X). In groups of 3, the architect also plays the AI analyst.

**Role cards** are in [`docs/review-board/`](../docs/review-board/README.en.md). Each stakeholder reads **only their own card** and keeps it secret until the meeting.

## 2. Meeting (15 min)

| Min | Step                                                                                                                                                                        |
| --- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 3   | **Architect** presents the decision, the options considered and the main consequences                                                                                       |
| 3   | **AI analyst** reads the AI review's findings and verdict, word for word                                                                                                    |
| 7   | **Open discussion.** Each stakeholder raises the concerns from their card, in their own words. The architect responds; anyone may ask the AI analyst "did the AI see this?" |
| 2   | **Decision.** The board votes: accept / accept with changes / reject                                                                                                        |

## 3. Record the decision (8 min, together)

**ADR reviewed:** `docs/adr/0002-use-rabbitmq-for-post-order-processing.md` by Naphat Serirak (6632061321) ([GitHub Link](https://github.com/neennera/naphat-salab-aiagent/blob/main/docs/adr/0002-use-rabbitmq-for-post-order-processing.md))
**Board members and roles:**

- Architect: Naphat Serirak (6632061321)
- AI analyst: Chanatda Konchom (6631305321)
- Product owner: Ittichet Thongsang (6631363721)
- Site reliability engineer (SRE): Chayut Archamongkol (6631307621)
- Security & privacy officer: Napat Srisamut (6631314021)

**Decision:** accept with changes

**Required changes:**

1. Mitigate Dual-Write Risk (from AI review): Implement a Transactional Outbox pattern or local persistent write in checkoutservice to guarantee that paid orders are never lost if the pod crashes before publishing to RabbitMQ.
2. Immediate Loyalty Points (PO): Ensure customers see their updated loyalty points on the order confirmation page, instead of waiting up to 1 minute.
3. Fraud Scoring Before Confirmation (PO): Move fraud scoring before order confirmation so suspicious orders can be blocked or cancelled.
4. Meet the 3-Week Deadline (PO): Create a realistic implementation and testing plan to ensure the system is ready and load-tested before 11.11.
5. Handle Cancellations and Refunds (PO): Add a mechanism to reverse or adjust loyalty points when orders are cancelled or refunded.
6. PDPA (Security): get legal approval for MailFast, or keep the current emailservice until it’s approved.
7. Data minimisation (Security): send only IDs to analytics, encrypt the queues at rest, and set a 30-day TTL on the queues and the DLQ.
8. Operational readiness (SRE): prepare RabbitMQ monitoring dashboards, alerts, backups and a runbook before it goes to production.
9. End-to-end traceability (SRE): include order_id and a correlation ID in every event so an order can be traced across services, and alert when MailFast is down or the backlog grows abnormally. Define who is paged when the broker or a consumer fails.
10. Timeline fallback (SRE): the 1-week ops preparation and the 2-week HA budget approval conflict with the 3-week deadline. If HA is not approved before 11.11, define a fallback (e.g. a single-node broker with a documented risk) and agree on it with the product owner.

**Who raised what?** List every concern the board discussed.

| #   | Concern                                                                                                                                                                                                            | Raised by (AI review / architect / PO / SRE / security) | Was it in the AI's review?                                                                                  | Could the AI have known? Why or why not?                                                                                                           |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | Dual-Write / Crash-after-Payment Risk: If pod restarts right after credit card charge but before publishing to RabbitMQ, message is lost.                                                                          | AI review                                               | Yes                                                                                                         | Yes; the AI analyzed checkoutservice/main.go and recognized two decoupled sequential calls without transactional guarantees.                       |
| 2   | Immediate Loyalty Point: We have marketed it “see your loyalty point the moment you past your order”, but AI allows for up to 1 minute                                                                             | PO                                                      | No                                                                                                          | Partially. AI recognized eventual consistency but it had no idea about the marketing promise                                                       |
| 3   | Fraud Scoring Before Confirmation: The ADR have fraud scoring as post-order but it should have been before order confirmation                                                                                      | PO                                                      | No                                                                                                          | Partially. AI could recognize that post-order processing cannot block confirmation, but it did not know that we want it before order confirmation. |
| 4   | Three-week deadline implementation: We have to implement everything before 11.11 (in 3 weeks)                                                                                                                      | PO                                                      | No                                                                                                          | No, AI could recognize the how difficult the implementation is but it did not know the time limit as it was not in context.                        |
| 5   | Cancellation and Refund Handling: no explanation of how loyalty point is dealt with if there is cancellation or refund                                                                                             | PO                                                      | No                                                                                                          | Yes, the AI could have identified this missing workflow.                                                                                           |
| 6   | PDPA approval is pending and there’s no fallback                                                                                                                                                                   | Security                                                | No                                                                                                          | No, the legal status is only on card X                                                                                                             |
| 7   | The fanout event sends email and name to analytics, against company policy                                                                                                                                         | Security                                                | No                                                                                                          | Partially: it saw the fanout but not the policy                                                                                                    |
| 8   | Queues and the DLQ hold personal data without encryption or a 30-day limit                                                                                                                                         | Security                                                | No                                                                                                          | Partially: the 30-day rule is internal policy                                                                                                      |
| 9   | No one can run a broker in production, and on-call is only 2 people: the developers who "have used RabbitMQ" only used it on their own laptops; nobody in the company has operated a message broker in production. | SRE                                                     | No                                                                                                          | No; this is internal team knowledge that was not in the AI's context.                                                                              |
| 10  | Operational readiness rule: a new stateful component cannot go to production without monitoring dashboards, alerts, backups and a written runbook, which takes about 1 week to prepare.                            | SRE                                                     | No                                                                                                          | No; this is an internal ops policy only on card S.                                                                                                 |
| 11  | Cluster is near its node limit: a 3-node high-availability broker needs budget approval, which takes 2 weeks.                                                                                                      | SRE                                                     | No                                                                                                          | No; the AI had no information about cluster capacity or budget.                                                                                    |
| 12  | Email outages go undetected: last year's 11.11 email provider outage lasted 20 minutes and nobody noticed until customers complained on social media.                                                              | SRE                                                     | Partially: the review covers MailFast failure, the DLQ and backlog alerting, but not detection of an outage | Partially: the AI saw the risk but did not know about last year's incident.                                                                        |
| 13  | Cannot trace an order across async boundaries: if a customer says "I never got the email", there is no correlation ID or tracing to follow one order.                                                              | SRE                                                     | No                                                                                                          | Yes; the AI could have noticed the missing observability from the ADR itself.                                                                      |

**The AI's best finding** (one the humans had missed): The identification of the Dual-Write risk (crash-after-payment) in checkoutservice, exposing the critical flaw where customer cards are charged but RabbitMQ publication could fail silently without an Outbox pattern.
**The AI's weakest finding** (generic, wrong or irrelevant): The generic warning about Eventual Consistency impacting UX, which provided standard textbook commentary without offering concrete business mitigations or operational thresholds.

## 4. Individual reflection (≤ 150 words, each student, after class is fine)

Write in [`05-reflection.md`](05-reflection.en.md): after Part A and Part B, which parts of an architect's work would you trust AI with, and which would you keep for humans? Use one concrete example from your board meeting.

> Each group member commits this record to their own repository (copy the same text), so everyone submits a complete repo.
