# Organizational Process and Technical Governance

## Overview

DirectFile grew from a small pilot team to a 70+ member product organization. As the team scaled, they evolved their technical decision-making processes. Understanding these processes provides context for WHY certain decisions were made and how a replacement project might structure its own governance.

## RFC vs ADR: Two Complementary Processes

### Architecture Decision Records (ADRs)
- **Purpose**: Static, post-decisional artifacts that capture the rationale of a design choice
- **When to write**: After a decision has been made
- **Content**: Title, status, context, decision, consequences
- **Location**: `docs/adr/` in the codebase
- **Audience**: Engineers who need to understand why the system is configured as it is
- **Review process**: Minimal — simply codifies an already-approved decision

### Requests for Comments (RFCs)
- **Purpose**: Live documents for proposing and iterating on technical specifications
- **When to write**: Before implementation, when a significant technical decision is needed
- **Content**: Problem statement, requirements, proposed implementation, alternatives analysis
- **Location**: `docs/rfc/` in the codebase
- **Audience**: All engineers, IRS IT SMEs, leadership
- **Review process**: Async review first, synchronous meeting if unresolved questions remain

### When to Use Which

| Scenario | Process |
|----------|---------|
| System-wide change (new infrastructure, auth changes, new APIs) | RFC + ADR + IRS IT SME review |
| Cross-team feature requiring alignment | RFC (ADR optional) |
| Single-team feature, cross-cutting within team | RFC optional |
| Single-team, well-defined requirements | PR/ticket sufficient |
| Modifying pre-existing logic | PR/ticket sufficient |

### Examples Requiring IRS IT SME Involvement
- Provisioning new cloud infrastructure
- Changing network boundaries
- Adding/modifying API endpoints
- New vulnerability scan findings
- Changing authentication/authorization
- Storing PII/FTI in new locations
- Integrating with other IRS systems
- Major version upgrades of framework components
- Standing up new deployed services

## Decision-Making Flow

```
Requirements identified
        │
        ▼
Engineer drafts RFC
        │
        ▼
Async review (GitHub PR)
        │
        ├─── All questions resolved? ──► Approved ──► Implement
        │                                    │
        ▼                                    ▼
Synchronous RFC Review meeting          Write ADR (if needed)
        │
        ▼
Outstanding questions resolved
        │
        ▼
RFC approved ──► Implement ──► Write ADR
```

## The Directly Responsible Engineer (DRE)

For each significant feature or technical decision, a **Directly Responsible Engineer (DRE)** is identified. The DRE:
1. Provides initial scope and sizing estimates
2. Determines what documentation is needed (RFC, ADR, or just PR)
3. Drafts the RFC if needed
4. Coordinates with IRS IT SMEs when required
5. Owns the implementation end-to-end
6. Updates the RFC based on review feedback

This model was adopted to empower individual engineers to own features completely, rather than relying on a small group of senior architects.

## Why This Process Evolved

During the pilot phase, the team operated with minimal process. As the team grew to 70+, problems emerged:
- **Stakeholders weren't in the room**: Decisions were made without key people knowing, leading to delivery delays
- **Decisions happened across too many surfaces**: Slack threads, GitLab tickets, markdown files, and meetings — hard to track the outcome
- **Knowledge silos**: Not all engineers understood what was happening across teams
- **IRS IT engagement was too late**: Compliance and infrastructure requirements weren't identified early enough, creating last-mile delivery delays

The RFC process was specifically designed to:
1. Force engineers to substantiate their reasoning in a single document
2. Ensure IRS IT counterparts are engaged early
3. Provide a standing forum for cross-team discussion
4. Reduce knowledge silos by making all proposals visible

## Key Insight for a Replacement Project

Even for a smaller team building a new system, establishing lightweight decision documentation early is valuable:

1. **Write down WHY, not just WHAT**: Future developers (and AI agents) need to understand the rationale behind decisions, not just the outcome.

2. **Keep decisions in the codebase**: ADRs and RFCs in `docs/` are version-controlled, searchable, and always in context. Don't scatter decisions across Slack, email, and wikis.

3. **Distinguish proposals from decisions**: Proposals (RFCs) are for discussion and iteration. Decisions (ADRs) are for record-keeping. Don't conflate them.

4. **Engage compliance/security early**: For any government or regulated system, security and compliance stakeholders should review proposals before implementation, not after.

5. **Scale the process to team size**: A 3-person team doesn't need standing RFC review meetings. But even a small team benefits from writing down "we chose X because Y."

## Tax Expert Involvement

A critical process element in DirectFile is the involvement of **tax subject matter experts ("taxperts")**:

- Taxperts review tax logic changes for correctness
- Taxperts verify that knockout criteria match IRS rules
- Taxperts review content for plain-language accuracy
- Taxperts create and validate test scenarios
- The fact dictionary testing approach was specifically designed to produce artifacts (spreadsheets) that non-programmer taxperts could review

For a replacement system, having access to tax expertise (even if not full-time team members) is essential for validating that the system correctly implements tax law.

## Content Review Process

All user-facing content in DirectFile goes through a review pipeline:

1. **Engineering writes initial content** in English
2. **Content team reviews** for plain language, readability, and tone
3. **IRS General Counsel (GC) reviews** for legal accuracy
4. **After GC approval**, content is sent to the translation team
5. **Translation team** produces Spanish versions
6. **Translated content is imported** back into the codebase
7. **QA review** of translated content in context

This multi-step review process means content changes are slow and expensive. The standardized error message pattern system was designed specifically to minimize the number of unique strings that need this full review cycle.

## IRS-Specific References

DirectFile documentation references several IRS-specific resources that informed the design:

| Resource | Purpose |
|----------|---------|
| **VITA Volunteer Assistors test (Pub 6744)** | Source of test scenarios for tax calculations |
| **VITA Intake Form (Form 13614-C)** | Example of asking direct, situational tax questions |
| **Publication 4491 (VITA Training Guide)** | Comprehensive tax logic reference |
| **Publication 4012 (VITA Volunteer Resource Guide)** | Quick-reference for tax rules |
| **Publication 17 (Your Federal Income Tax)** | "Tax code manual" explaining laws and regulations |
| **IRS Pub 4801 (Line Item Estimates)** | Data about how many filers use each line — useful for prioritizing features |
| **IRS Direct File After Action Report (Pub 5969)** | Lessons learned from the pilot filing season |

These resources are invaluable for anyone building tax software and should be consulted during design and testing of a replacement system.
