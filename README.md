# RevSure Sentinel AI
### Revenue Leakage & Contract Compliance Case Manager
**UiPath AgentHack 2026 — Track 1: UiPath Maestro Case**

---

## 🎯 The Problem

Companies sign contracts with customers that include critical commercial clauses:
- Annual price escalations (e.g. 10% per year)
- Time-limited discounts (e.g. first-year only)
- Auto-renewal terms with notice periods
- Monthly usage billing obligations

In practice, billing systems don't enforce these clauses automatically. Discounts run past their expiry dates, escalations never get applied, renewals get missed. The result: **silent revenue leakage** — the company loses money without knowing it.

**Example:** A contract says INR 1,00,000/month with a 10% first-year discount (expires Dec 31, 2025) and a 10% annual escalation from Jan 1, 2026. The billing system still charges INR 90,000 in February 2026 — the discounted rate — missing both the discount expiry and the escalation. That's **INR 20,000/month in leakage = INR 2,40,000 annually**, per contract.

---

## 💡 The Solution

**RevSure Sentinel AI** is an agentic case management system built on UiPath Maestro Case. Each contract becomes a live case that moves through intelligent stages, with AI agents detecting compliance violations and routing exceptions to human decision-makers.

Every contract uploaded to the system is automatically analyzed, compared against actual billing data, and — if a violation is found — escalated to a Finance or Sales Manager with a clear recommended corrective action.

---

## 🏗️ Architecture

```
Contract Upload (Manual Trigger)
        ↓
┌─────────────────────────────────────────────────────┐
│                  MAESTRO CASE                        │
│                                                      │
│  Stage 1: INTAKE                                     │
│  └── Agent 1: Contract Analysis Agent               │
│      • Reads contract text                           │
│      • Extracts: price, escalation, discounts,      │
│        renewal terms, party names                    │
│      • Returns structured JSON + confidence scores  │
│                                                      │
│  Stage 2: COMPLIANCE CHECK                           │
│  └── Agent 2: Pricing Compliance Agent              │
│      • Compares contract terms vs invoice data      │
│      • Checks: discount expiry, escalation dates    │
│      • Calculates leakage amount                    │
│      • Returns: isCompliant, leakageAmount,         │
│        complianceIssues                             │
│                                                      │
│  Stage 3: HUMAN REVIEW ← Human-in-the-loop         │
│  ├── Agent 3: Recommendation Agent                  │
│  │   • Generates corrective action report           │
│  │   • Urgency level, annual impact, action steps  │
│  └── Human Action: Finance Manager reviews          │
│      and approves corrective actions                │
│                                                      │
│  Stage 4: RESOLUTION                                 │
│  └── Case closed with outcome logged                │
└─────────────────────────────────────────────────────┘
```

**Case Manager (AI Orchestrator):** Governs the entire case lifecycle, enforcing stage transitions, SLA tracking, and escalation rules.

---

## 🤖 Agents

### Agent 1: Contract Analysis Agent
- **Type:** Autonomous Agent (UiPath Agent Builder)
- **Model:** GPT-5.4
- **Job:** Reads contract text and extracts all key commercial terms as structured JSON
- **Output fields:** contractReferenceId, contractStartDate, buyerName, sellerName, contractedPrice, billingFrequency, priceEscalation (rate + appliesFrom), renewalTerms (autoRenewal + renewalDate + noticePeriod), discounts (value + expiryDate), summary, confidenceScores (per-field), overallConfidence, flaggedForReview
- **Anti-hallucination:** Returns empty string/array/false for missing fields; never invents values

### Agent 2: Pricing Compliance Agent  
- **Type:** Autonomous Agent (UiPath Agent Builder)
- **Model:** Claude Sonnet 4.6 (anthropic.claude-sonnet-4-6)
- **Job:** Compares contract terms against invoice data; determines correct expected amount; detects leakage
- **Logic:** Checks discount expiry dates vs invoice date; checks escalation appliesFrom vs invoice date; calculates expected amount step by step
- **Output fields:** invoiceDate, expectedAmount, actualBilledAmount, currency, leakageAmount, isCompliant, complianceIssues, explanation, confidenceScore

### Agent 3: Recommendation Agent
- **Type:** Autonomous Agent (UiPath Agent Builder)  
- **Model:** GPT-5.4
- **Job:** Takes compliance violation data and generates actionable corrective action report for Finance Manager
- **Output fields:** recommendationSummary, correctiveActions (array), urgencyLevel (High/Medium/Low), estimatedAnnualImpact, approvalRequired

---

## 🔧 UiPath Components Used

| Component | Usage |
|-----------|-------|
| **UiPath Maestro Case** | Core case management — stages, triggers, case manager AI orchestrator, SLA tracking |
| **UiPath Agent Builder** | Built all 3 autonomous agents with custom system prompts and output schemas |
| **UiPath Studio Web** | Built and tested the entire solution in-browser on UiPath Automation Cloud |
| **UiPath Automation Cloud** | Hosted sandbox environment (UiPath Labs, hackathon tenant) |

**Agent type:** Low-code agents built with UiPath Agent Builder (Autonomous Agent type), with system prompts and I/O schemas defined through Agent Builder's Autopilot-assisted configuration. No external agent frameworks used — all agents are UiPath-native.

---

## 🤝 Coding Agent Usage (Bonus Points)

This solution was built entirely in collaboration with **Claude (Anthropic)** as a coding agent, used throughout the full development lifecycle:

- **Platform navigation:** Claude interpreted screenshots of the brand-new Maestro Case UI (launched days before the hackathon) and provided step-by-step guidance for every configuration decision
- **Agent prompt engineering:** Claude authored all system prompts, user prompts, and I/O schema designs for all 3 agents
- **Architecture decisions:** Claude designed the case stage structure, data flow between agents, and human-in-the-loop checkpoint placement
- **Test data generation:** Claude generated realistic sample contract PDF and invoice PDF (Python/reportlab) with a deliberate pricing discrepancy for end-to-end testing
- **Debugging:** Claude diagnosed and resolved multiple runtime errors across agent configuration, input schema validation, and Maestro Case variable mapping
- **Documentation:** Claude wrote this README and the project description

Evidence: The full session log, build log (BUILD_LOG.md), and agent schemas are included in this repository.

---

## 📋 Demo Scenario

**Contract:** Brightline Solutions Pvt. Ltd. (seller) ↔ Northwind Retail Holdings Pvt. Ltd. (buyer)
- Reference: BLS-2025-0142
- Base price: INR 1,00,000/month
- First-year discount: 10% (expires December 31, 2025 → effective price INR 90,000/month in Year 1)
- Annual escalation: 10% from January 1, 2026
- Term: 12 months, auto-renewing

**Invoice:** INV-2026-0031, dated February 1, 2026, billed at INR 90,000

**What the system detects:**
1. Discount expired Dec 31 2025 — still being applied in Feb 2026 ❌
2. 10% escalation due from Jan 1 2026 — not applied ❌
3. Expected correct amount: INR 1,10,000
4. Actual billed: INR 90,000
5. **Leakage: INR 20,000/month = INR 2,40,000 annually**

---

## 🚀 Setup Instructions

### Prerequisites
- UiPath Automation Cloud account (UiPath Labs sandbox or standard tenant)
- Access to UiPath Studio Web
- Maestro Case enabled on your tenant

### Steps to run
1. Clone this repository
2. Log into your UiPath Automation Cloud tenant
3. Open UiPath Studio Web
4. Import the solution: Studio → Cloud Workspace → Import existing → select the solution folder
5. Open the Maestro Case project → Case plan
6. Click "Debug on cloud" to run a test case
7. View results in the Execution Trail

### Sample test data
Sample contract and invoice PDFs are included in the `/sample-data` folder:
- `Sample_Contract_Brightline_Solutions.pdf` — the contract with escalation and discount clauses
- `Sample_Invoice_INV-2026-0031.pdf` — the February 2026 invoice showing the pricing discrepancy

---

## 📁 Repository Structure

```
revSure-sentinel-ai/
├── README.md
├── BUILD_LOG.md                          # Full build log with Claude sessions
├── sample-data/
│   ├── Sample_Contract_Brightline_Solutions.pdf
│   └── Sample_Invoice_INV-2026-0031.pdf
├── schemas/
│   └── 01_contract_analysis_agent_schema.md
└── screenshots/
    └── (case plan, agent outputs, execution trail)
```

---

## 📊 Business Impact

- **Problem scale:** Enterprise companies typically have hundreds to thousands of active customer contracts
- **Leakage per contract:** INR 20,000/month in our demo scenario = INR 2,40,000/year
- **At scale:** 100 contracts with similar leakage = INR 2.4 crore annual revenue recovery
- **Time to detect:** Manual contract audits take days/weeks; RevSure Sentinel AI detects violations in seconds
- **Human effort:** Reduced to final approval only — all analysis is automated

---

## 🏆 Track Alignment

**Track 1: UiPath Maestro Case** — This solution is purpose-built for Maestro Case:
- ✅ Dynamic case management (each contract = one case)
- ✅ Exception-heavy workflow (only violations create active cases)
- ✅ Multi-stage progression (Intake → Compliance Check → Human Review → Resolution)
- ✅ Agent/human handoffs (3 AI agents + Finance Manager approval)
- ✅ Human in charge at key decision point (Human Review stage)
- ✅ Real business value (revenue recovery, contract enforcement)

---

## 📄 License

MIT License — see LICENSE file for details.

---

*Built for UiPath AgentHack 2026 | Track 1: UiPath Maestro Case*
*Powered by UiPath Agent Builder + Claude (Anthropic)*
