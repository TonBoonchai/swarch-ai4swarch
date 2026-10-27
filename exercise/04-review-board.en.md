**Language:** [ภาษาไทย](04-review-board.md) | English

# Part B: Architecture review board (approximately 25 min, groups of 4)

> **Lecture link:** AICH4 (reviews need domain expertise; *plausible ≠ appropriate*), Pillar 5 (AI as Chief Analyst, humans as Chief Strategist), JSS (85% of studies keep a human in the loop).
> **Question:** what does a human review board catch that the AI's review missed, and why?

The company's architecture review board meets to approve the R-12 design before the 11.11 sale. One ADR goes to the board: the **architect** presents it, the **AI analyst** presents the AI's review, and two stakeholders bring knowledge the AI never saw.

## 1. Form the board (2 min)

In groups of 4, choose **one** member's ADR to review (`docs/adr/0002-*.md`, plus their `exercise/outputs/3-review.md`). Then assign roles:

| Role | Who | Brings |
|---|---|---|
| **Architect** | Ratsaporn (the ADR's author) | defends the ADR |
| **AI analyst** | Weeraphat | reads out the AI's review only, and may not add their own opinions |
| **Product owner** | Poonyapat | role card P (business side) |
| **Site reliability engineer (SRE)** | Boonchai | role card S (operations side) |

Groups of 5 add a **security & privacy officer** (role card X). In groups of 3, the architect also plays the AI analyst.

**Role cards** are in [`docs/review-board/`](../docs/review-board/README.en.md). Each stakeholder reads **only their own card** and keeps it secret until the meeting.

## 2. Meeting (15 min)

| Min | Step |
|---|---|
| 3 | **Architect** presents the decision, the options considered and the main consequences |
| 3 | **AI analyst** reads the AI review's findings and verdict, word for word |
| 7 | **Open discussion.** Each stakeholder raises the concerns from their card, in their own words. The architect responds; anyone may ask the AI analyst "did the AI see this?" |
| 2 | **Decision.** The board votes: accept / accept with changes / reject |

## 3. Record the decision (8 min, together)

Security & privacy officer (role card X, group of 5): Rachata

**ADR reviewed:** `docs/adr/0002-myadr.md` by Rachata  
**Board members and roles:** security & privacy officer

**Decision:** accept with changes

**Required changes:**

1. Define separate message schemas for email, loyalty, and analytics. Document personal data in each message and avoid distributing customer details to every consumer.
2. Exclude email addresses and full names from analytics. Assess whether order IDs or other identifiers could identify customers through linkage.
3. Specify access permissions for each queue, including retry and dead-letter queues. Grant services and operators only the access they need.

**ADR reviewed:** `docs/adr/0002-myadr.md` by Boonchai  
**Board members and roles:** SRE

**Decision:** accept with changes

**Required changes:**

1. Correct the experience assumption: the two developers used RabbitMQ only on laptops. Production broker operations are new to this team, with only two people on call.
2. Make operational readiness a release gate: require monitoring dashboards, alerting, backups, and a written runbook before introducing new stateful components. Allocate approximately one week for this work.
3. Assign incident ownership: name who receives alerts for broker, relay, consumer, and storage failures, with escalation and recovery procedures.

**ADR reviewed:** `docs/adr/0002-myadr.md` by Poonyapat  
**Board members and roles:** Product Owner

**Decision:** accept with changes

**Required changes:**

1. Points: update the design and acceptance criteria so the confirmation page displays the actual, saved points total.
2. "Points pending" or "within one minute" does not meet the campaign promise.
3. Define what happens when loyalty is unavailable, including the conflict with keeping checkout available.

**ADR reviewed:** `docs/adr/0002-myadr.md` by Weeraphat  
**Board members and roles:** AI Analyst

**Decision:** accept

**ADR reviewed:** `docs/adr/0002-myadr.md` by Ratsaporn  
**Board members and roles:** Architect

**Decision:** accept

**Who raised what?** List every concern the board discussed.

| # | Concern | Raised by (AI review / architect / PO / SRE / security) | Was it in the AI's review? | Could the AI have known? Why or why not? |
|---|---|---|---|---|
| 1 | Duplicate points cause financial loss; missing points cause complaints and break the marketing promise. Adding storage, outbox, RabbitMQ, and a loyalty ledger creates delivery risk for a four-person team. The ADR contains a testing plan, but no results demonstrating readiness. | Product Owner | Yes | AI could have known it since fincancial loss is substantial |
| 2 | On-call load: broker, outbox, and additional storage create substantial new responsibilities for a two-person rotation. | SRE | Yes | Ai sholud know this because it thought this problem could be impact more on our website |
| 3 | Unnecessary exposure: A shared event could distribute personal data to consumers that do not need it. | Security | Yes | unnecessary data Ai should know that should not be show in website |
| 4 | Payment can succeed before the order is durably recorded. | AI Analyst | Yes | This should be the workflow problem |
| 5 | Delivery guarantees need a clear scope. | Architect | Yes | If the scope is not clear AI can not aim for the goal of the problem |

**The AI's best finding** (one the humans had missed): Payment and shipping happen before the durable order commit, creating a failure window where a customer could be charged without a recorded order. The outbox does not protect this gap.  
**The AI's weakest finding** (generic, wrong or irrelevant): The email capacity mismatch. It is valid, but the ADR already explicitly calculates the backlog and identifies the required follow-up, so the review added little new insight.

## 4. Individual reflection (≤ 150 words, each student, after class is fine)

Write in [`05-reflection.md`](05-reflection.en.md): after Part A and Part B, which parts of an architect's work would you trust AI with, and which would you keep for humans? Use one concrete example from your board meeting.

> Each group member commits this record to their own repository (copy the same text), so everyone submits a complete repo.
