# Work Breakdown Structure (WBS)

## PL/SQL → .NET REST Services Modernization Program — Frontdoor

This WBS converts **122 confirmed in-scope Oracle PL/SQL packages (~311k source LOC, ~156k effective LOC after 50% dead code factor)** into **18 domain-aligned .NET REST services + 6 shared libraries** running on AWS, delivered across **three decommission phases** using **66Way (hybrid waterfall + agile)**. Effort assumes **100% AI-driven code generation** (Cursor + LLM agents) with **HITL** validation — translation, test generation, OpenAPI synthesis, sequence diagram drafting, manifest and mesh configuration, doc generation — calibrated against _Global Delivery Staffing Standards_ (Software Engineering: Migration template) and priced against the _66degrees Rate Card 2023_. The plan targets **~6.0 months (~26 weeks)** wall-calendar delivery using **parallel Build pods**, **earlier phase overlap**, **parallel UAT lanes** where Frontdoor can supply SMEs, and **~2,100 hours** of acceleration capacity (cross-pod integration, extra QA execution, DM/UAT coordination, incremental architect HITL) — **without** reducing SOW scope, **Verify** calendar weeks (**3 / 2 / 2** per phase), or behavioral-parity / contract-test obligations. **No contingency is applied.**

**Target architecture: service mesh-mediated REST with strangler fig traffic routing.** Frontdoor's **existing Istio on EKS** provides mTLS-secured inter-service communication and traffic management. This program adds **project-specific** VirtualServices, DestinationRules, and AuthorizationPolicies for the **strangler fig pattern**: per-service traffic splitting between legacy PL/SQL logic exposed as REST from Oracle and modernized .NET microservices — **no Istio control-plane installation**. This enables progressive, reversible migration without big-bang cutovers. **The 18 microservices communicate extensively through the mesh** — the service dependency analysis identifies 11 critical, 17 strong, and 28 moderate inter-service couplings. Calling applications route through the mesh; project routing rules control whether a given request reaches the legacy Oracle REST endpoint or the new .NET service. AHS_QUEUE_FUNCTIONS is preserved as an in-process shared library (NuGet) because it is a CRUD layer over the AHS_QUEUE / AHS_QUEUE_DETAILS tables consumed by Cancellation, Sales Transaction, Work Order & Fulfillment, and Customer services.

**Excluded from this WBS:** Business Analyst activities; Subject Matter Expert activities; Hypercare; Phase 2 & Phase 3 Prepare stages; Async Messaging Service wrapper (only the 3 inter-service packages: ENQUEUE_MESSAGE, DEQUEUE_MESSAGE, PROC_INCREMENT_USAGE_COUNT — source now in src/additional_code/ but behavior remains Frontdoor-owned / out of 66degrees build scope); **new database provisioning and data migration** (data remains in Oracle); **Terraform / IaC infrastructure provisioning** (deploy to existing K8s cluster via manifests + Istio routing config only); **Istio control plane installation, upgrade, and cluster-wide mesh baseline** (Frontdoor-operated existing mesh).

**Supplemental source (src/additional_code/):** Resolves **17 / 19** objects that lacked eponymous package files (all Phase 1 gap procedures, most Phase 3 standalone functions/procedures). Still missing: **STAR_CLOUD**, **COMMON_FUNC_EMAIL_ROB3**. **PROC_INCREMENT_USAGE_COUNT** is a discontinued stub. See scope_source_mapping.md.

## 1\. Executive summary

| **Metric** | **Value** |
| --- | --- |
| In-scope PL/SQL objects | **122 confirmed** (4 deduplicated into parent packages, 3 inter-service queue objects removed; **2** objects without any source in package or src/additional_code/ export; **1** discontinued stub) |
| --- | --- |
| Source LOC | ~311,855 (43,863 spec + 267,992 body); **effective LOC ~156k** (50% dead code factor) |
| --- | --- |
| Target services | **18 microservices + 6 shared libraries** |
| --- | --- |
| Target architecture | Existing Istio on EKS + project VirtualServices/DestinationRules; strangler fig traffic routing; mTLS via Frontdoor-operated mesh |
| --- | --- |
| Program structure | Phase 0 Foundation + Phase 1 + Phase 2 + Phase 3 (with overlap) |
| --- | --- |
| Calendar duration | **~6.0 months (~26 wks)** (Phase 0 = **2.5 wks**; P1/P2/P3 **Build** **9 / 8 / 7 wks**; **Verify** **3 / 2 / 2 wks**; **P2 starts P1 week 6**, **P3 starts P2 week 4**) |
| --- | --- |
| Total effort | **~17,120 hours** (gross **~20,750 h** before overlap credit; **~15,020 h** after **5,730 h** overlap savings; **+2,100 h** acceleration premium — §9) |
| --- | --- |
| Total program cost | **~$1.95M (USD)** at effective blend ~~$114/hr (planning band \*\*~~$1.85M–$2.05M\*\*); overlap-adjusted labor **~15,020 h** ≈ **~$1.65M** before acceleration premium |
| --- | --- |
| Onshore / Offshore mix | **~29% / ~71%** by hours (role table §10) |
| --- | --- |
| AI productivity uplift assumed | **100% AI-driven code generation with HITL validation** — humans review/validate/iterate, not author |
| --- | --- |
| Dead code factor | **50%** — only half of source LOC assumed to require conversion |
| --- | --- |
| Data layer | Data remains in Oracle — no new database provisioning or data migration |
| --- | --- |
| Contingency | **None applied** — dead code factor and AI uplift absorb estimation variance |
| --- | --- |
| Out of scope | Decommission execution; security/compliance testing; business process analysis; **business analyst work**; **subject-matter-expert services**; **hypercare**; **Phase 2 & Phase 3 Prepare stages**; **Async Messaging Service** (excludes ENQUEUE_MESSAGE, DEQUEUE_MESSAGE, PROC_INCREMENT_USAGE_COUNT only); **new database implementation / data migration** |
| --- | --- |

**Cost view:** Phase 0 Foundation ≈ $0.11M · Phase 1 ≈ $0.76M · Phase 2 ≈ $0.57M · Phase 3 ≈ $0.38M · Cross-phase governance ≈ $0.02M · **Acceleration premium** ≈ **$0.23M** (~2,100 h). Less overlap savings ≈ ($0.13M). **Net ≈ $1.95M.** No contingency. Detail in §9–§10.

**Cost-optimization pass (this revision):** Trimmed **~~1,500 h (~~$0.10M, ~5%)** of genuinely overspecified effort that contradicted the 100% AI-generation assumption — Associate Software Engineer role reduced to residual test-data/fixture support (authoring is AI-absorbed), offshore Define "triage" validation right-sized, P2/P3 Cloud Engineering tapered to copy-modify against proven templates, and the double-counted PMO line removed. **No reduction to Verify weeks, behavioral/performance parity, financial/PCI HITL review, core build hours, or the ~6.0-month calendar.**

## 2\. Estimating assumptions

### 2.1 AI tooling productivity uplift

| **Activity** | **Traditional** | **With 100% AI + HITL** | **Uplift used** |
| --- | --- | --- | --- |
| PL/SQL → C# translation (LOC/dev-day) | 80–150 | 400–800 | **~5×** |
| --- | --- | --- | --- |
| Unit + integration test authoring | 40 hrs / svc | 10 hrs / svc | **~75%** |
| --- | --- | --- | --- |
| OpenAPI spec synthesis | 16 hrs / svc | 2 hrs / svc | **~88%** |
| --- | --- | --- | --- |
| Sequence diagram drafting | 12 hrs / svc | 2 hrs / svc | **~83%** |
| --- | --- | --- | --- |
| K8s manifest generation | 12 hrs / svc | 2 hrs / svc | **~83%** |
| --- | --- | --- | --- |
| PL/SQL spec extraction / call-graph analysis | 24 hrs / svc | 8 hrs / svc | **~67%** |
| --- | --- | --- | --- |
| Code review / QA triage | n/a | 35% slower (AI-generated code requires careful HITL review) | **(headwind)** |
| --- | --- | --- | --- |
| Dead code factor | 100% LOC assumed live | 50% LOC requires conversion | **50% scope reduction** |
| --- | --- | --- | --- |
| Net effect on engineering build hours | baseline | ~60% AI reduction applied to 50% effective LOC | **~80% vs. naïve baseline** |
| --- | --- | --- | --- |

### 2.2 Staffing model (per _Software Engineering: Migration_, Global Delivery Staffing Standards)

- **Global mix:** **~28%** Onshore (NorthAM) / **~72%** Offshore (India) by hours (§10). No nearshore tier — all offshore work is performed from India.
- **Architect model:** 1 Onshore lead + 1 Offshore architect per active phase. AI-assisted OpenAPI synthesis and sequence diagram drafting reduce architectural staffing needs.
- **Ratios:** Architect:Engineer **~1:6** at peak; Associates retained only at a residual level (~10–15% of standard pairing) for test-data preparation, fixtures, and manual smoke checks — AI fully absorbs the traditional associate authoring role and QA Engineers cover coverage validation.
- **Delivery management:** 25 hrs/wk full-lifecycle baseline. IST shadow DM at 50% allocation in Phase 1, tapering to 25% in P2/P3 Build (on-call for blockers, not continuous follow-the-sun). DM also owns UAT coordination, sponsor relationship management, and commercial governance. **No Engagement Manager** — fixed-scope behavioral translation does not require dedicated commercial governance. **No Solution Architect** — architecture is frozen after Phase 0; App Dev Architect provides day-to-day leadership; Cloud Architect co-owns platform decisions.
- **Quality:** QA Lead (Off) + QA Engineer (Off). No Associate QA Engineer role — AI-generated tests and 1:1 behavioral parity reduce the need for paired QA testing. QA Engineers shift from test authoring to coverage validation under 100% AI test generation. Add **~2 FTE-equivalent QA Engineer** capacity (offshore) during Build + Verify peaks and **parallel UAT lanes** (see §2.5) so parity / contract / performance execution keeps pace with dual Build pods.
- **No UX/UI:** Backend REST services only; XD staff not required.
- **No BA / no SME / no Hypercare / no EM / no SA:** These are explicitly excluded. Frontdoor provides business-analyst capacity, domain SMEs for requirements & UAT, and post-go-live hypercare. 66degrees engages with Frontdoor's BA/SME pool through the Delivery Manager and App Dev Architect.
- **Existing Istio + project routing in scope:** Each service is a .NET REST API deployed to the existing Kubernetes cluster, fronted by Amazon Cognito / API Gateway for external caller authentication. **Frontdoor's existing Istio on EKS** provides mTLS and mesh infrastructure; this program configures **project-specific** VirtualServices, DestinationRules, and AuthorizationPolicies for **strangler fig traffic routing** — per-service traffic splitting between legacy Oracle REST endpoints and new .NET microservices. **No Istio installation or control-plane work.** Services communicate extensively through the mesh — 11 critical, 17 strong, and 28 moderate inter-service couplings. No inter-service SNS/SQS. No Terraform/IaC provisioning — K8s manifests + Istio routing configuration only. Routing templates are proven in Phase 0 POC and reused from Phase 1 onward.
- **Data remains in Oracle:** Services connect to the existing Oracle database via Oracle EF Core provider / Dapper with Oracle.ManagedDataAccess. No Amazon RDS provisioning, no new schema design, no data migration. Oracle HA / backup / tuning remain Frontdoor's operational responsibility.
- **Standard week:** 45 hours/week per current staffing standards.

### 2.3 Rate card (USD/hr, applied per 66degrees 2023 rate card)

Only roles consumed in this engagement are listed.

| **Role** | **Onshore (NorthAM)** | **Offshore (India)** |
| --- | --- | --- |
| App Dev Architect | 275 | 95  |
| --- | --- | --- |
| Cloud Architect | 300 | 95  |
| --- | --- | --- |
| Security Architect | 325 | —   |
| --- | --- | --- |
| Back End Engineer | 200 | 85  |
| --- | --- | --- |
| Cloud Engineer | 225 | 85  |
| --- | --- | --- |
| Security Engineer | 300 | 95  |
| --- | --- | --- |
| Associate Software Engineer | 125 | 55  |
| --- | --- | --- |
| QA Lead | —   | 70  |
| --- | --- | --- |
| QA Engineer | —   | 65  |
| --- | --- | --- |
| Delivery Manager | 225 | 85  |
| --- | --- | --- |

### 2.4 Methodology mapping (66Way)

Each program phase (P1, P2, P3) executes a tailored 66Way lifecycle. Phase 2 and Phase 3 omit Prepare entirely (team continuity from P1) and run a 1-week Define and 1-week Plan stage. Operate is intentionally lighter without hypercare — it covers SRE handover, knowledge transfer, and close-out only.

| **66Way stage** | **Phase 1 (wks)** | **Phase 2 (wks)** | **Phase 3 (wks)** | **Notes** |
| --- | --- | --- | --- | --- |
| Prepare | 1   | —   | —   | Only Phase 1; team continuous from P0 |
| --- | --- | --- | --- | --- |
| Define | 2   | 1   | 1   | Architect + Sr Eng-led requirements (PL/SQL spec extraction with AI) |
| --- | --- | --- | --- | --- |
| Design | 2.5 | 1   | **2** | Sequence diagrams, OpenAPI, IaC patterns per service (**P3** includes Card Security PCI extension design — **2 wks** wall-clock) |
| --- | --- | --- | --- | --- |
| Plan | 1   | 1   | 1   | Sprint plan, test strategy, value checkpoint prep |
| --- | --- | --- | --- | --- |
| **Value Checkpoint** | <1  | <1  | <1  | Sponsor sign-off prior to Build |
| --- | --- | --- | --- | --- |
| Build | **9** | **8** | **7** | Dual Pod A/B sustained HITL; §3 build-hour inventory unchanged; wall-clock shortened via overlap and parallel capacity (see §2.5) |
| --- | --- | --- | --- | --- |
| Verify | **3** | **2** | **2** | QA + UAT support (Frontdoor-led UAT); **QA/DM hours** in §10 fund parallel lanes, not shorter calendar gates |
| --- | --- | --- | --- | --- |
| Operate | 2   | 1   | 1   | SRE handover + KT + close-out only (no hypercare) |
| --- | --- | --- | --- | --- |
| **Total active** | **~21** | **~14.5** | **~14.5** | **P2 starts P1 week 6; P3 starts P2 week 4** — phase overlap + dual Build pods → **~6.0 mo** program wall-clock |
| --- | --- | --- | --- | --- |

### 2.5 Program delivery model and calendar (~6.0 months)

**Integrated AI + HITL** implements the SOW without reducing scope (18 microservices + 6 shared libraries, behavioral parity, strangler fig, Verify gates):

- **Phase 0 accounting:** WBS §4 line items sum to the Phase 0 figure in §9 (no double-count in program roll-up).
- **Discovery:** One AI-assisted extraction pipeline with HITL on risk-flagged packages (financial, PCI, autonomous transactions, prod.\*) and weekly Frontdoor BA validation gates.
- **Design after Phase 0:** Delta-first — full OpenAPI/diagram coverage where SOW requires it; extensions emphasize diffs against frozen Phase 0 templates.
- **Build — architects:** Spike/office-hours model — exception-based reviews emphasizing complex business paths (~15%); tech leads own pattern adherence with ADR backlog.
- **Build — associates:** HITL support (fixtures, smoke checks, evidence packs), not parallel implementation of shared libraries.
- **Build — Istio routing:** VirtualService/DestinationRule work templated and batch-generated after Phase 0 POC on existing mesh; Cloud Engineer effort on pilots and spot-check.
- **QA foundation:** Risk-based mutation testing on high-risk domains (Payment, Card Security, Commission/Pricing, Cancellation, Vendor Invoice & write-off, Finance Integration); contract and behavioral-parity coverage portfolio-wide per SOW.
- **Governance:** Sponsor steering on monthly cadence after mobilization; IST shadow DM remains blocker-based; platform, project Istio routing, and security ops sized to release-window cadence.
- **Verify / Operate:** Verify execution and parity/UAT coordination preserved as quality gates; Operate uses AI-first artifacts with lean human publish cycles.

**Wall-calendar target (~6.0 months / ~26 weeks):** Parallel engineering capacity and Frontdoor-aligned UAT throughput — not fewer packages, shorter Verify weeks, or relaxed parity, performance, or contract-test obligations.

| **Mechanism** | **What changes** | **What does not change** |
| --- | --- | --- |
| **Dual Build pods** | Two domain-aligned pods (e.g., **Pod A** — Customer, Contract, Dispatch, Work Order & Fulfillment, Service Order, Web Portal & User; **Pod B** — Pricing, Invoicing, Payment, Vendor + AP, Commission, Finance, Card Security, Reporting, Renewal, Campaign, Notification, Cancellation) run the same AI+HITL pipeline on disjoint package sets; shared App Dev Architect (onshore lead + offshore), Cloud/Security spikes, mesh templates, **weekly cross-pod integration / releasability** | Per-service **§3 / §11** build-hour inventory; ADR + mesh patterns; PCI/financial pair-review rules |
| --- | --- | --- |
| **Phase overlap** | **Phase 2 starts Phase 1 week 6**; **Phase 3 starts Phase 2 week 4** — Pod B stable by mid–Phase 1 Build; net-new Phase 2 services onboard via Phase 1 Operate + Phase 2 Define | Phase 2/3 Prepare out of scope; Value Checkpoint before each Build |
| --- | --- | --- |
| **Phase 0** | **2.5-week** wall-clock via parallel Sr/Cloud staffing on Istio routing integration and shared libraries | PCI scope analysis, strangler POC on existing mesh, golden-example pipeline, six shared libraries complete before Phase 1 Build |
| --- | --- | --- |
| **UAT cadence** | DM books **two parallel UAT tracks** where Frontdoor supplies SMEs; staggered environment smoke windows to reduce queue risk (**R6**) | Frontdoor owns UAT execution; severity gates unchanged |
| --- | --- | --- |
| **Acceleration premium** | **~2,100 h** for cross-pod integration, extra QA, DM/UAT coordination, incremental architect HITL on shared contracts/mesh | **5,730 h** overlap savings methodology; **no** contingency |
| --- | --- | --- |

**Critical dependency:** Frontdoor confirms **≥2 parallel UAT cohorts** (or equivalent SME hours) during each Verify window; limited acceptance capacity caps how much wall-calendar benefit engineering parallelism can deliver.

## 3\. Service-level effort sizing

**Note — upper-bound sizing reference (not the costed model).** §3 and §11 are a **top-down tier-based inventory** retained for traceability and per-service Microsoft Project / Smartsheet import. They are an **upper bound** and are **not** reconciled line-for-line into the costed program total: the authoritative bottom-up cost model is the per-stage WBS tables in **§4–§8**, rolled up in **§9–§10**. Where the tier inventory (e.g., **§3.3 / §11**, ~29,450 build hrs) exceeds the costed engineering labor in the stage tables, the **stage tables govern** for effort and cost.

Build hours include design refinement, code translation, unit + integration tests, OpenAPI spec, Dockerfile, K8s manifests, code review, and bug fixes. They **exclude** Define/Design/Plan/Verify/Operate phase activities (estimated separately) and exclude DM roles.

### 3.1 Sizing tiers (AI-assisted, all-in build hours per service)

| **Tier** | **LOC range** | **Build hours / service** |
| --- | --- | --- |
| XS  | <500 | 200 |
| --- | --- | --- |
| S   | 500–2,000 | 400 |
| --- | --- | --- |
| M   | 2,000–5,000 | 900 |
| --- | --- | --- |
| L   | 5,000–15,000 | 1,800 |
| --- | --- | --- |
| XL  | 15,000–25,000 | 2,800 |
| --- | --- | --- |
| XXL | \>25,000 | 4,000 |
| --- | --- | --- |

### 3.2 Per-service sizing

**Dead code factor: 50%.** Only 50% of source LOC is assumed to require conversion. "Effective LOC" drives tier assignment and build hours. Source LOC is retained for traceability.

| **#** | **Service** | **Domain** | **Source LOC** | **Effective LOC (50%)** | **Tier** | **Build Hrs** | **Primary phase** | **Touched phases** |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 1   | Customer Service | Customer & Property | 19,113 | 9,557 | L   | 1,800 | P1  | P1·P2·P3 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 2   | Contract Service | Contracts | 20,079 | 10,040 | L   | 1,800 | P1  | P1·P3 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 3   | Pricing & Revenue Service | Contracts | 5,815 | 2,908 | M   | 900 | P1  | P1·P2 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 4   | Renewal Service | Contracts | 15,553 | 7,777 | L   | 1,800 | P2  | P2·P3 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 5   | Cancellation Service | Contracts | 8,356 | 4,178 | M   | 900 | P3  | P3  |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 6   | Dispatch Service | Service Ops | 33,666 | 16,833 | XL  | 2,800 | P1  | P1·P2·P3 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 7   | Service Order Service | Service Ops | 23,079 | 11,540 | L   | 1,900 | P3  | P1·P2·P3 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 8   | Work Order & Fulfillment Service | Service Ops | 30,274 | 15,137 | XL  | 2,800 | P2  | P1·P2·P3 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 9   | Invoicing Service | Billing & Receivables | 22,549 | 11,275 | L   | 1,800 | P1  | P1·P2·P3 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 10  | Payment Service | Billing & Receivables | 10,182 | 5,091 | M+  | 1,350 | P1  | P1·P2·P3 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 11  | Vendor Service | Vendors & Payables | 14,240 | 7,120 | L   | 1,800 | P2  | P1·P2·P3 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 12  | Vendor Invoice & Writeoff Service | Vendors & Payables | 16,469 | 8,235 | L   | 1,800 | P1  | P1·P2·P3 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 13  | Sales Transaction Service | Sales & Marketing | 7,711 | 3,856 | M   | 900 | P3  | P2·P3 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 14  | Commission & Territory Service | Sales & Marketing | 13,767 | 6,884 | M+  | 1,500 | P1  | P1·P2 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 15  | Campaign & Lead Service | Sales & Marketing | 6,629 | 3,315 | M   | 900 | P2  | P2  |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 16  | Finance Integration Service | Finance Integration | 7,113 | 3,557 | M   | 900 | P1  | P1·P2·P3 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 17  | Card Security Service (PCI premium +50%) | Platform | 1,456 | 728 | S+  | 300 | P1  | P1·P3 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 18  | Notification Service | Platform | 3,835 | 1,918 | S   | 400 | P2  | P2·P3 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 19  | Reporting Service | Platform | 6,937 | 3,469 | M   | 900 | P1  | P1·P2·P3 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 20  | Web Portal & User Service | Platform | 8,200 | 4,100 | M   | 900 | P3  | P2·P3 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
|     | **Microservices subtotal** |     | **~248k** | **~124k** |     | **27,750** |     |     |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| L1  | Core Utilities (shared lib) | Platform | 27,559 | 13,780 | M   | 700 | Phase 0 | All |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| L2  | Error Handling (shared lib) | Platform | 628 | 314 | XS  | 100 | Phase 0 | All |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| L3  | Feature Flags & Config (shared lib) | Platform | 117 | 59  | XS  | 60  | Phase 0 | All |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| L4  | File & S3 Storage (shared lib) | Platform | 2,038 | 1,019 | S   | 400 | Phase 0 | All |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| L5  | Crypto Primitives (shared lib, PCI +50%) | Platform | 697 | 349 | XS+ | 220 | Phase 0 | All |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| L6  | AHS Queue Persistence (shared lib) | Platform | 668 | 334 | XS  | 120 | Phase 0 | P3  |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
|     | **Shared libs subtotal** |     | **31,707** | **~15,855** |     | **1,700** |     |     |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
|     | **TOTAL** |     | **~311k** | **~156k** |     | **~29,450** |     |     |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |

Removed PL/SQL packages (out of scope — inter-process messaging plumbing, not service mesh traffic): ENQUEUE_MESSAGE · DEQUEUE_MESSAGE · PROC_INCREMENT_USAGE_COUNT. These are Oracle DBMS_AQ wrappers and usage counters with no in-scope callers. **AHS_QUEUE_FUNCTIONS is retained** as the shared library AHS Queue Persistence (L6) because it is a database-table CRUD layer over AHS_QUEUE / AHS_QUEUE_DETAILS consumed in-process (NuGet) by Cancellation, Sales Transaction, Work Order & Fulfillment, and Customer services.

**Dead code assumption:** 50% of source LOC is assumed to be dead/unreachable code that does not require conversion. This factor is applied uniformly across all services and shared libraries. Dead code identification occurs during Define (AI-assisted PL/SQL call-graph analysis). If the actual live-code ratio differs significantly from 50%, build hours should be re-baselined at the Phase 1 Value Checkpoint.

### 3.3 Phase allocation of engineering build hours

Build hours distributed by share of effective LOC (50% dead code factor) delivered in each phase, with shared-library work front-loaded into Phase 0.

| **Phase** | **Effective LOC delivered** | **Share** | **Engineering build hours** |
| --- | --- | --- | --- |
| Phase 0 (shared libs + foundation) | 15,855 | —   | **1,700** |
| --- | --- | --- | --- |
| Phase 1 | 63,350 | 37.4% | **9,935** |
| --- | --- | --- | --- |
| Phase 2 | 57,112 | 33.7% | **11,340** |
| --- | --- | --- | --- |
| Phase 3 | 35,132 | 20.7% | **6,475** |
| --- | --- | --- | --- |
| **Total** |     |     | **29,450** |
| --- | --- | --- | --- |

## 4\. Phase 0 — Program initiation & foundation (**2.5 weeks**)

Establish the platform, AI tooling, shared libraries, AWS landing zone, and CI/CD ahead of Phase 1 build.

| **WBS** | **Workstream** | **Task** | **Role** | **Loc.** | **Hrs** |
| --- | --- | --- | --- | --- | --- |
| 0.01 | Governance | Mobilize program; charter, RACI, comms plan | Delivery Manager | On  | 9   |
| --- | --- | --- | --- | --- | --- |
| 0.02 | Governance | Steering committee mobilization; **monthly cadence** after stand-up (charter, RAID hooks to sprint governance) | Delivery Manager | On  | 34  |
| --- | --- | --- | --- | --- | --- |
| 0.03 | Governance | Establish IST follow-the-sun coverage (50% allocation) | Delivery Manager | Off | 17  |
| --- | --- | --- | --- | --- | --- |
| 0.04a | Architecture | Reference architecture — platform (EKS, existing Istio integration, AWS KMS, strangler routing) | Cloud Architect | On  | 15  |
| --- | --- | --- | --- | --- | --- |
| 0.04b | Architecture | Reference architecture — application (.NET 8, Oracle ODP.NET, API standards) | App Dev Architect | On  | 11  |
| --- | --- | --- | --- | --- | --- |
| 0.05 | Architecture | API standards (REST/OpenAPI 3.1, error model, external-caller auth via Amazon Cognito / API Gateway) | App Dev Architect | On  | 22  |
| --- | --- | --- | --- | --- | --- |
| 0.06 | Architecture | Sequence diagram template & doc-as-code toolchain (Mermaid + ADRs) — caller→service flows only | App Dev Architect | Off | 22  |
| --- | --- | --- | --- | --- | --- |
| 0.07 | Architecture | Service template (.NET solution, Polly for downstream HTTP resilience, Serilog, OpenTelemetry, AutoMapper, MediatR in-process) | App Dev Architect | Off | 22  |
| --- | --- | --- | --- | --- | --- |
| 0.08 | Cloud Platform | K8s namespace setup, RBAC policies, AWS Secrets Manager integration (existing cluster) | Cloud Architect | On  | 19  |
| --- | --- | --- | --- | --- | --- |
| 0.09 | Cloud Platform | K8s manifest templates (Deployment, Service, Ingress, ConfigMap, Secrets, HPA) + Kustomize overlays | Cloud Engineer | Off | 23  |
| --- | --- | --- | --- | --- | --- |
| 0.10 | Cloud Platform | CI/CD pipelines (AWS CodeBuild + GitHub Actions; build/test/scan/kubectl apply promotion) | Cloud Engineer | Off | 30  |
| --- | --- | --- | --- | --- | --- |
| 0.11 | Cloud Platform | Observability baseline (Amazon CloudWatch / AWS X-Ray dashboards, SLOs) | Cloud Engineer | On  | 15  |
| --- | --- | --- | --- | --- | --- |
| 0.12 | Security | PCI scope analysis for Card Security & Crypto libs; AWS KMS / AWS CloudHSM design | Security Architect | On  | 23  |
| --- | --- | --- | --- | --- | --- |
| 0.13 | Security | Identity model — external caller authentication (Amazon Cognito / API Gateway), per-service IAM roles for AWS resource access | Security Engineer | Off | 23  |
| --- | --- | --- | --- | --- | --- |
| 0.14 | AI Enablement | Cursor / LLM agent setup; per-domain prompt templates; translation rules | App Dev Architect | On  | 15  |
| --- | --- | --- | --- | --- | --- |
| 0.15 | AI Enablement | PL/SQL parser + spec extractor agent (CallGraph, side-effect tags) | Sr Back End Engineer | On  | 15  |
| --- | --- | --- | --- | --- | --- |
| 0.16 | AI Enablement | Translation cookbook + golden examples (1 small package end-to-end) | Sr Back End Engineer | Off | 45  |
| --- | --- | --- | --- | --- | --- |
| 0.17 | Shared Libs | Build **Core Utilities** library (COMMON_FUNC, COMMON_FUNC_SQL, PKG_COMMON, P_LIBRARIES, STRINGOBJ, ASSERT, VALIDATION_FUNCTIONS, ETL_COMMON, CONTEXT_API) | Sr Back End Engineer | Off | 118 |
| --- | --- | --- | --- | --- | --- |
| 0.18 | Shared Libs | Build **Core Utilities** library — onshore lead/review | Sr Back End Engineer | On  | 26  |
| --- | --- | --- | --- | --- | --- |
| 0.20 | Shared Libs | Build **Error Handling** library (COMMON_FUNC_ERROR_HANDLER) + structured logging | Sr Back End Engineer | Off | 23  |
| --- | --- | --- | --- | --- | --- |
| 0.22 | Shared Libs | Build **Feature Flags & Config** library (SSI_FEATURES + FLAG_CS1 from additional_code/functions/flag_cs1.sql) | Sr Back End Engineer | Off | 23  |
| --- | --- | --- | --- | --- | --- |
| 0.23 | Shared Libs | Build **File & S3 Storage** lib (FILEOP + S3 adapter for prod.star_cloud / clob_to_file from additional_code/procedures/clob_to_file.sql) | Sr Back End Engineer | Off | 120 |
| --- | --- | --- | --- | --- | --- |
| 0.25 | Shared Libs | Build **Crypto Primitives** library (STARCRYPTO) — AWS KMS-backed | Security Engineer | Off | 49  |
| --- | --- | --- | --- | --- | --- |
| 0.26 | Shared Libs | Crypto Primitives — onshore security review | Security Architect | On  | 17  |
| --- | --- | --- | --- | --- | --- |
| 0.27 | Shared Libs | Crypto Primitives unit tests + FIPS validation | QA Engineer | Off | 17  |
| --- | --- | --- | --- | --- | --- |
| 0.28 | Shared Libs | Build **AHS Queue Persistence** library (AHS_QUEUE_FUNCTIONS) — CRUD over AHS_QUEUE / AHS_QUEUE_DETAILS | Sr Back End Engineer | Off | 30  |
| --- | --- | --- | --- | --- | --- |
| 0.30 | Shared Libs | Shared libraries — architecture review & sign-off | App Dev Architect | On  | 23  |
| --- | --- | --- | --- | --- | --- |
| 0.31 | Shared Libs | NuGet package publishing pipeline + versioning policy | Cloud Engineer | Off | 30  |
| --- | --- | --- | --- | --- | --- |
| 0.32 | QA Foundation | QA strategy: contract tests, **risk-based mutation tests** (financial/PCI/payment/cancellation/vendor-AP/finance-integration paths), performance baselines | QA Lead | Off | 38  |
| --- | --- | --- | --- | --- | --- |
| 0.33 | Knowledge Base | Decompose prod.star_cloud (no local source); validate adapters against additional_code exports for clob_to_file and cost_center2department | Sr Back End Engineer | Off | 30  |
| --- | --- | --- | --- | --- | --- |
| 0.34 | Communication | Stakeholder kickoff workshops & comms plan | Delivery Manager | On  | 7   |
| --- | --- | --- | --- | --- | --- |
| 0.35 | Istio Routing | Istio routing architecture on **existing** mesh — strangler fig strategy, VirtualService / DestinationRule patterns, alignment with Frontdoor mesh conventions | Cloud Architect | On  | 22  |
| --- | --- | --- | --- | --- | --- |
| 0.36 | Istio Routing | Validate existing Istio integration (namespaces, sidecar injection, gateways); author **project** VirtualService / DestinationRule templates | Cloud Engineer | Off | 18  |
| --- | --- | --- | --- | --- | --- |
| 0.37 | Istio Routing | Project-scoped mesh security — K8s service accounts for new microservices; AuthorizationPolicy (PeerAuthentication overrides only if required) | Security Engineer | Off | 8   |
| --- | --- | --- | --- | --- | --- |
| 0.38 | Istio Routing | Strangler fig routing POC on live mesh — dual-route 1 pilot service (legacy Oracle REST ↔ new .NET), weighted traffic splitting, canary template, rollback procedure | Cloud Engineer | Off | 24  |
| --- | --- | --- | --- | --- | --- |
|     | **Phase 0 subtotal** |     |     |     | **~983** |
| --- | --- | --- | --- | --- | --- |

Phase **2.5 weeks** through parallel Sr/Cloud capacity on architecture, shared libraries, and Istio routing integration (full WBS in §4). All six shared libraries and strangler-routing templates on the **existing** mesh are **Phase 1 ready**. §4 line items sum to this subtotal. The **acceleration premium** (§2.5) can absorb marginal overtime or parallel shift within Phase 0 if actual burn exceeds plan.

## 5\. Phase 1 — Translate & decommission first 45 PL/SQL objects (**~21 weeks** active 66Way stack)

**Services touched in P1:** Customer · Contract · Pricing & Revenue · Dispatch · Work Order & Fulfillment (subset) · Invoicing · Payment · Vendor (subset) · Vendor Invoice & Writeoff · Commission & Territory · Finance Integration · Card Security · Reporting (subset) · Service Order (subset). **13 services.**

### 5.1 Phase 1 — Prepare (1 week)

| **WBS** | **Workstream** | **Task** | **Role** | **Loc.** | **Hrs** |
| --- | --- | --- | --- | --- | --- |
| 1.P.01 | Governance | Phase 1 kickoff; finalize phase plan, RAID log | Delivery Manager | On  | 16  |
| --- | --- | --- | --- | --- | --- |
| 1.P.02 | Governance | Phase 1 kickoff — IST DM (50%) | Delivery Manager | Off | 8   |
| --- | --- | --- | --- | --- | --- |
| 1.P.03 | Governance | Phase 1 sponsor briefing | Delivery Manager | On  | 3   |
| --- | --- | --- | --- | --- | --- |
| 1.P.04 | Architecture | Single facilitated session — 13-service scope + dependency map (feeds Define) | App Dev Architect | On  | 9   |
| --- | --- | --- | --- | --- | --- |
| 1.P.05 | Architecture | Offshore architect read-in + validation of AI-generated dependency graph | App Dev Architect | Off | 12  |
| --- | --- | --- | --- | --- | --- |
| 1.P.06 | Architecture | Platform tooling & cluster access checklist (shared with offshore read-in) | App Dev Architect | Off | 4   |
| --- | --- | --- | --- | --- | --- |
| 1.P.07 | Engineering | Engineering leads — joint scope read-in | Sr Back End Engineer | On  | 7   |
| --- | --- | --- | --- | --- | --- |
| 1.P.08 | Engineering | Offshore engineering leads — joint scope read-in | Sr Back End Engineer | Off | 11  |
| --- | --- | --- | --- | --- | --- |
| 1.P.09 | QA  | QA Lead read-in | QA Lead | Off | 8   |
| --- | --- | --- | --- | --- | --- |
|     | **Prepare subtotal** |     |     |     | **78** |
| --- | --- | --- | --- | --- | --- |

### 5.2 Phase 1 — Define (2 weeks) · Discovery & Requirements

Without a dedicated BA, requirements are extracted via AI-assisted PL/SQL specification mining (call graphs, side-effect tags, parameter contracts) led by App Dev Architects + Sr Engineers. Frontdoor provides a BA / SMEs separately to validate requirements.

| **WBS** | **Workstream** | **Task** | **Role** | **Loc.** | **Hrs** |
| --- | --- | --- | --- | --- | --- |
| 1.D.01 | Architecture | Service contract scoping per service (13 svcs) — **AI-executed pipeline**, onshore HITL on contracts + risk-flagged services | App Dev Architect | On  | 46  |
| --- | --- | --- | --- | --- | --- |
| 1.D.02 | Architecture | Service contract validation — offshore (**triage** of AI outputs, not second full pass) | App Dev Architect | Off | 69  |
| --- | --- | --- | --- | --- | --- |
| 1.D.03 | Engineering | Per-service PL/SQL validation (13 svcs) — AI-driven spec extraction; onshore HITL on financial/PCI/critical couplings | Sr Back End Engineer | On  | 44  |
| --- | --- | --- | --- | --- | --- |
| 1.D.04 | Engineering | PL/SQL extraction QA — offshore (sustained dual sr-eng capacity) | Sr Back End Engineer | Off | 159 |
| --- | --- | --- | --- | --- | --- |
| 1.D.05 | Architecture | Cross-service architecture reviews (critical + strong couplings only) | App Dev Architect | On  | 12  |
| --- | --- | --- | --- | --- | --- |
| 1.D.06 | Governance | Frontdoor-BA liaison, requirements validation gates | Delivery Manager | On  | 41  |
| --- | --- | --- | --- | --- | --- |
| 1.D.07 | Governance | Define-stage IST DM coverage (50%) | Delivery Manager | Off | 15  |
| --- | --- | --- | --- | --- | --- |
| 1.D.08 | Governance | Sponsor liaison / governance | Delivery Manager | On  | 8   |
| --- | --- | --- | --- | --- | --- |
|     | **Define subtotal** |     |     |     | **394** |
| --- | --- | --- | --- | --- | --- |

### 5.3 Phase 1 — Design (2.5 weeks) · Solution Design

| **WBS** | **Workstream** | **Task** | **Role** | **Loc.** | **Hrs** |
| --- | --- | --- | --- | --- | --- |
| 1.S.01 | Architecture | API design + sequence diagrams — onshore lead (**delta + critical flows**; AI-generated, human review) | App Dev Architect | On  | 28  |
| --- | --- | --- | --- | --- | --- |
| 1.S.02 | Architecture | API design + OpenAPI per service — offshore architect (13 svcs; AI gen + HITL) | App Dev Architect | Off | 56  |
| --- | --- | --- | --- | --- | --- |
| 1.S.03 | Engineering | Detailed design (DTO mapping, ports/adapters) — onshore (**exceptions / new surfaces**) | Sr Back End Engineer | On  | 16  |
| --- | --- | --- | --- | --- | --- |
| 1.S.04 | Engineering | Detailed design — offshore (dual sr-eng capacity; template-heavy services spot-checked) | Sr Back End Engineer | Off | 72  |
| --- | --- | --- | --- | --- | --- |
| 1.S.05 | Cloud | K8s manifest pattern per service (from Phase 0 templates) | Cloud Engineer | Off | 24  |
| --- | --- | --- | --- | --- | --- |
| 1.S.06 | Cloud | Cloud architect review of K8s patterns | Cloud Architect | On  | 6   |
| --- | --- | --- | --- | --- | --- |
| 1.S.07 | Security | Card Security service threat model (PCI DSS aligned) | Security Architect | On  | 20  |
| --- | --- | --- | --- | --- | --- |
| 1.S.08 | Security | Security control implementation design | Security Engineer | Off | 28  |
| --- | --- | --- | --- | --- | --- |
| 1.S.09 | QA  | Test strategy & data-set design per service | QA Lead | Off | 38  |
| --- | --- | --- | --- | --- | --- |
| 1.S.10 | Governance | Design-stage status & risk | Delivery Manager | On  | 16  |
| --- | --- | --- | --- | --- | --- |
| 1.S.11 | Governance | IST DM coverage (50%) | Delivery Manager | Off | 8   |
| --- | --- | --- | --- | --- | --- |
| 1.S.12 | Architecture | Cross-cutting architecture review | App Dev Architect | On  | 6   |
| --- | --- | --- | --- | --- | --- |
|     | **Design subtotal** |     |     |     | **318** |
| --- | --- | --- | --- | --- | --- |

### 5.4 Phase 1 — Plan (1 week) · Implementation Plan

| **WBS** | **Workstream** | **Task** | **Role** | **Loc.** | **Hrs** |
| --- | --- | --- | --- | --- | --- |
| 1.L.01 | Governance | Sprint plan, release plan, dependency graph (13 svcs across **9-week** dual-pod Build) | Delivery Manager | On  | 49  |
| --- | --- | --- | --- | --- | --- |
| 1.L.02 | Governance | Plan — IST DM (50%) | Delivery Manager | Off | 12  |
| --- | --- | --- | --- | --- | --- |
| 1.L.03 | Architecture | LOE refinement per service & cross-service sequencing | App Dev Architect | On  | 22  |
| --- | --- | --- | --- | --- | --- |
| 1.L.04 | Architecture | Plan refinement — offshore architect | App Dev Architect | Off | 42  |
| --- | --- | --- | --- | --- | --- |
| 1.L.05 | Engineering | Backlog grooming (translate-task creation per package, AI-augmented sizing) | Sr Back End Engineer | On  | 12  |
| --- | --- | --- | --- | --- | --- |
| 1.L.06 | Engineering | Backlog grooming — offshore | Sr Back End Engineer | Off | 38  |
| --- | --- | --- | --- | --- | --- |
| 1.L.07 | QA  | Test plan per service (functional + integration + perf parity) | QA Lead | Off | 38  |
| --- | --- | --- | --- | --- | --- |
| 1.L.08 | Cloud | Environment plan (namespace-per-env, Kustomize overlays, promotion gates) | Cloud Engineer | Off | 8   |
| --- | --- | --- | --- | --- | --- |
| 1.L.09 | Governance | UAT scheduling with Frontdoor (test users, environment slots) | Delivery Manager | On  | 16  |
| --- | --- | --- | --- | --- | --- |
| 1.L.10 | Architecture | Architecture plan review | App Dev Architect | On  | 8   |
| --- | --- | --- | --- | --- | --- |
|     | **Plan subtotal** |     |     |     | **245** |
| --- | --- | --- | --- | --- | --- |

### 5.5 Phase 1 — Value Checkpoint (<1 week)

| **WBS** | **Workstream** | **Task** | **Role** | **Loc.** | **Hrs** |
| --- | --- | --- | --- | --- | --- |
| 1.V.01 | Governance | Value Checkpoint deck + sponsor walkthrough | Delivery Manager | On  | 15  |
| --- | --- | --- | --- | --- | --- |
| 1.V.02 | Governance | Sponsor VC attendance | Delivery Manager | On  | 3   |
| --- | --- | --- | --- | --- | --- |
| 1.V.03 | Architecture | Architecture VC attendance | App Dev Architect | On  | 5   |
| --- | --- | --- | --- | --- | --- |
| 1.V.04 | Architecture | App Dev Architect VC walkthrough | App Dev Architect | On  | 10  |
| --- | --- | --- | --- | --- | --- |
|     | **Value Checkpoint subtotal** |     |     |     | **33** |
| --- | --- | --- | --- | --- | --- |

### 5.6 Phase 1 — Build (**9 weeks**, dual Pod A/B) · Translation, test, integration, deploy

Engineering build hours from **§3.3 (9,935 hrs for Phase 1 effective LOC)** are carried in Sr/Assoc/QA sprint labor below. Additional rows are **program overhead** (architecture spikes, cloud/security, DM, **templated** mesh routing) per §2.5 — not duplicate counting of §11 per-service build weights.

| **WBS** | **Workstream** | **Task** | **Role** | **Loc.** | **Hrs** |
| --- | --- | --- | --- | --- | --- |
| 1.B.01 | All svcs | Architectural leadership — onshore (**spike / office-hours model**) | App Dev Architect | On  | 175 |
| --- | --- | --- | --- | --- | --- |
| 1.B.02 | All svcs | Architectural leadership — offshore (**exception-based**; templates frozen) | App Dev Architect | Off | 340 |
| --- | --- | --- | --- | --- | --- |
| 1.B.03 | All svcs | Senior engineering — onshore | Sr Back End Engineer | On  | 385 |
| --- | --- | --- | --- | --- | --- |
| 1.B.05 | All svcs | Senior engineering — offshore | Sr Back End Engineer | Off | 2,849 |
| --- | --- | --- | --- | --- | --- |
| 1.B.06 | All svcs | Associate engineering — offshore (residual test-data prep / fixtures only; authoring absorbed by 100% AI generation) | Assoc Software Engineer | Off | 120 |
| --- | --- | --- | --- | --- | --- |
| 1.B.07 | All svcs | Cloud engineering — K8s manifests, CI/CD — onshore | Cloud Engineer | On  | 85  |
| --- | --- | --- | --- | --- | --- |
| 1.B.08 | All svcs | Cloud engineering — offshore | Cloud Engineer | Off | 300 |
| --- | --- | --- | --- | --- | --- |
| 1.B.09 | Card Security | Security engineering — PCI controls, AWS KMS integration | Security Engineer | Off | 170 |
| --- | --- | --- | --- | --- | --- |
| 1.B.10 | Card Security | Security architect review (PCI) | Security Architect | On  | 30  |
| --- | --- | --- | --- | --- | --- |
| 1.B.11 | Sprint coverage | QA Lead — sprint-by-sprint shift-left | QA Lead | Off | 375 |
| --- | --- | --- | --- | --- | --- |
| 1.B.12 | Sprint coverage | QA engineering (in-sprint test validation) — offshore (×3) | QA Engineer | Off | 835 |
| --- | --- | --- | --- | --- | --- |
| 1.B.14 | Sprint coverage | Architecture checkpoint reviews | App Dev Architect | On  | 22  |
| --- | --- | --- | --- | --- | --- |
| 1.B.15 | Governance | Delivery Manager — sprint cadence, RAID, financials | Delivery Manager | On  | 195 |
| --- | --- | --- | --- | --- | --- |
| 1.B.16 | Governance | DM — IST shadow (50%) | Delivery Manager | Off | 90  |
| --- | --- | --- | --- | --- | --- |
| 1.B.17 | Governance | Sponsor governance | Delivery Manager | On  | 12  |
| --- | --- | --- | --- | --- | --- |
| 1.B.18 | Istio Routing | Strangler fig VirtualService/DestinationRule — **templated generation** per P1 service + Cloud Eng HITL on pilots / spot-check | Cloud Engineer | Off | 45  |
| --- | --- | --- | --- | --- | --- |
|     | **Build subtotal** |     |     |     | **~6,028** |
| --- | --- | --- | --- | --- | --- |

### 5.7 Phase 1 — Verify (3 weeks)

| **WBS** | **Workstream** | **Task** | **Role** | **Loc.** | **Hrs** |
| --- | --- | --- | --- | --- | --- |
| 1.Y.01 | QA  | Test execution (functional, integration, contract, perf parity vs PL/SQL) | QA Lead | Off | 85  |
| --- | --- | --- | --- | --- | --- |
| 1.Y.02 | QA  | QA engineers (×4) execution | QA Engineer | Off | 395 |
| --- | --- | --- | --- | --- | --- |
| 1.Y.04 | Engineering | Defect remediation — sr eng onshore | Sr Back End Engineer | On  | 60  |
| --- | --- | --- | --- | --- | --- |
| 1.Y.05 | Engineering | Defect remediation — offshore (×3) | Sr Back End Engineer | Off | 250 |
| --- | --- | --- | --- | --- | --- |
| 1.Y.06 | Engineering | Associate eng remediation — offshore | Assoc Software Engineer | Off | 16  |
| --- | --- | --- | --- | --- | --- |
| 1.Y.07 | Architecture | App Dev Architect triage — onshore | App Dev Architect | On  | 30  |
| --- | --- | --- | --- | --- | --- |
| 1.Y.08 | Architecture | App Dev Architect triage — offshore | App Dev Architect | Off | 30  |
| --- | --- | --- | --- | --- | --- |
| 1.Y.09 | UAT | UAT coordination with Frontdoor (env, accounts, schedule, defect intake) | Delivery Manager | On  | 70  |
| --- | --- | --- | --- | --- | --- |
| 1.Y.10 | Cloud | Performance/load testing scripts & runs (API response time) | Cloud Engineer | Off | 45  |
| --- | --- | --- | --- | --- | --- |
| 1.Y.11 | Governance | Verify-stage DM | Delivery Manager | On  | 60  |
| --- | --- | --- | --- | --- | --- |
| 1.Y.12 | Governance | DM IST (50%) | Delivery Manager | Off | 30  |
| --- | --- | --- | --- | --- | --- |
|     | **Verify subtotal** |     |     |     | **1,071** |
| --- | --- | --- | --- | --- | --- |

### 5.8 Phase 1 — Operate (2 weeks) · Handover & close-out

No hypercare. Activities are limited to SRE handover, documentation finalization, knowledge transfer, and phase close-out.

| **WBS** | **Workstream** | **Task** | **Role** | **Loc.** | **Hrs** |
| --- | --- | --- | --- | --- | --- |
| 1.O.01 | Cloud | SRE handover — runbooks, dashboards, alerting handoff (K8s manifests) | Cloud Engineer | Off | 50  |
| --- | --- | --- | --- | --- | --- |
| 1.O.02 | KT  | Knowledge transfer to Frontdoor SRE/dev teams | Sr Back End Engineer | On  | 60  |
| --- | --- | --- | --- | --- | --- |
| 1.O.03 | KT  | KT — offshore co-leads | Sr Back End Engineer | Off | 90  |
| --- | --- | --- | --- | --- | --- |
| 1.O.04 | KT  | Documentation finalization — review & publish AI-generated artifacts | Sr Back End Engineer | Off | 45  |
| --- | --- | --- | --- | --- | --- |
| 1.O.05 | Architecture | Architecture handover packet | App Dev Architect | On  | 30  |
| --- | --- | --- | --- | --- | --- |
| 1.O.06 | Governance | Phase close-out, retrospective | Delivery Manager | On  | 50  |
| --- | --- | --- | --- | --- | --- |
| 1.O.07 | Governance | DM IST (50%) | Delivery Manager | Off | 15  |
| --- | --- | --- | --- | --- | --- |
| 1.O.08 | Governance | Sponsor close-out | Delivery Manager | On  | 10  |
| --- | --- | --- | --- | --- | --- |
|     | **Operate subtotal** |     |     |     | **350** |
| --- | --- | --- | --- | --- | --- |

### 5.9 Phase 1 — Subtotal

| **Stage** | **Hours** |
| --- | --- |
| Prepare | 78  |
| --- | --- |
| Define | 394 |
| --- | --- |
| Design | 318 |
| --- | --- |
| Plan | 245 |
| --- | --- |
| Value Checkpoint | 33  |
| --- | --- |
| Build | 6,028 |
| --- | --- |
| Verify | 1,071 |
| --- | --- |
| Operate | 350 |
| --- | --- |
| **Phase 1 total** | **~8,517** |
| --- | --- |

## 6\. Phase 2 — Translate & decommission next 45 PL/SQL objects (**~14.5 weeks** active)

**Services touched in P2:** Customer (extend) · Pricing (extend) · Renewal (new) · Dispatch (extend) · Service Order (extend) · Work Order & Fulfillment (extend) · Invoicing (extend) · Payment (extend) · Vendor (extend) · Vendor Invoice (extend) · Sales Transaction (start) · Commission & Territory (extend) · Campaign & Lead (new) · Finance Integration (extend) · Notification (new) · Reporting (extend) · Web Portal & User (start). **17 services touched.**

Phase 2 has **no dedicated Prepare stage** — the team and operating model carry over from Phase 1.

### 6.1 Phase 2 — Define (1 week)

| **WBS** | **Workstream** | **Task** | **Role** | **Loc.** | **Hrs** |
| --- | --- | --- | --- | --- | --- |
| 2.D.01 | Architecture | Service contracts (8 extensions + 3 net-new; AI-driven spec extraction) — onshore HITL | App Dev Architect | On  | 22  |
| --- | --- | --- | --- | --- | --- |
| 2.D.02 | Architecture | Service contracts — offshore triage | App Dev Architect | Off | 32  |
| --- | --- | --- | --- | --- | --- |
| 2.D.03 | Engineering | PL/SQL delta validation (45 P2 packages) — onshore | Sr Back End Engineer | On  | 18  |
| --- | --- | --- | --- | --- | --- |
| 2.D.04 | Engineering | PL/SQL delta validation — offshore | Sr Back End Engineer | Off | 71  |
| --- | --- | --- | --- | --- | --- |
| 2.D.06 | Governance | Frontdoor-BA liaison, requirements validation gates | Delivery Manager | On  | 10  |
| --- | --- | --- | --- | --- | --- |
| 2.D.07 | Governance | DM IST (50%) | Delivery Manager | Off | 5   |
| --- | --- | --- | --- | --- | --- |
|     | **Define subtotal** |     |     |     | **161** |
| --- | --- | --- | --- | --- | --- |

### 6.2 Phase 2 — Design (1 week)

| **WBS** | **Workstream** | **Task** | **Role** | **Loc.** | **Hrs** |
| --- | --- | --- | --- | --- | --- |
| 2.S.01 | Architecture | API design + OpenAPI + sequence diagrams (**delta-first**; 15 effective svcs) — onshore | App Dev Architect | On  | 28  |
| --- | --- | --- | --- | --- | --- |
| 2.S.02 | Architecture | Design — offshore | App Dev Architect | Off | 52  |
| --- | --- | --- | --- | --- | --- |
| 2.S.03 | Engineering | Detailed design (DTO mapping, Oracle query patterns; delta for extensions) — onshore | Sr Back End Engineer | On  | 18  |
| --- | --- | --- | --- | --- | --- |
| 2.S.04 | Engineering | Detailed design — offshore | Sr Back End Engineer | Off | 68  |
| --- | --- | --- | --- | --- | --- |
| 2.S.05 | Cloud | K8s manifest patterns (reuse P1 templates) | Cloud Engineer | Off | 18  |
| --- | --- | --- | --- | --- | --- |
| 2.S.06 | Cloud | Cloud architect review | Cloud Architect | On  | 3   |
| --- | --- | --- | --- | --- | --- |
| 2.S.07 | QA  | Test strategy / data sets | QA Lead | Off | 28  |
| --- | --- | --- | --- | --- | --- |
| 2.S.08 | Governance | DM  | Delivery Manager | On  | 12  |
| --- | --- | --- | --- | --- | --- |
| 2.S.09 | Governance | DM IST (50%) | Delivery Manager | Off | 6   |
| --- | --- | --- | --- | --- | --- |
|     | **Design subtotal** |     |     |     | **235** |
| --- | --- | --- | --- | --- | --- |

### 6.3 Phase 2 — Plan (1 week)

| **WBS** | **Workstream** | **Task** | **Role** | **Loc.** | **Hrs** |
| --- | --- | --- | --- | --- | --- |
| 2.L.01 | Governance | Sprint plan, release plan | Delivery Manager | On  | 32  |
| --- | --- | --- | --- | --- | --- |
| 2.L.02 | Governance | DM IST (50%) | Delivery Manager | Off | 12  |
| --- | --- | --- | --- | --- | --- |
| 2.L.03 | Architecture | LOE refinement | App Dev Architect | On  | 17  |
| --- | --- | --- | --- | --- | --- |
| 2.L.04 | Architecture | Refinement — offshore | App Dev Architect | Off | 35  |
| --- | --- | --- | --- | --- | --- |
| 2.L.05 | Engineering | Backlog grooming | Sr Back End Engineer | On  | 12  |
| --- | --- | --- | --- | --- | --- |
| 2.L.06 | Engineering | Backlog grooming — offshore | Sr Back End Engineer | Off | 38  |
| --- | --- | --- | --- | --- | --- |
| 2.L.07 | QA  | Test plan | QA Lead | Off | 32  |
| --- | --- | --- | --- | --- | --- |
| 2.L.08 | Cloud | Environment plan (namespace delta) | Cloud Engineer | Off | 5   |
| --- | --- | --- | --- | --- | --- |
| 2.L.09 | Governance | UAT scheduling with Frontdoor | Delivery Manager | On  | 17  |
| --- | --- | --- | --- | --- | --- |
|     | **Plan subtotal** |     |     |     | **205** |
| --- | --- | --- | --- | --- | --- |

### 6.4 Phase 2 — Value Checkpoint (<1 week)

| **WBS** | **Workstream** | **Task** | **Role** | **Loc.** | **Hrs** |
| --- | --- | --- | --- | --- | --- |
| 2.V.01 | Governance | VC deck + walkthrough | Delivery Manager | On  | 15  |
| --- | --- | --- | --- | --- | --- |
| 2.V.02 | Governance | Sponsor VC | Delivery Manager | On  | 3   |
| --- | --- | --- | --- | --- | --- |
| 2.V.03 | Architecture | Architecture VC | App Dev Architect | On  | 5   |
| --- | --- | --- | --- | --- | --- |
| 2.V.04 | Architecture | App Dev Architect walkthrough | App Dev Architect | On  | 10  |
| --- | --- | --- | --- | --- | --- |
|     | **Value Checkpoint subtotal** |     |     |     | **33** |
| --- | --- | --- | --- | --- | --- |

### 6.5 Phase 2 — Build (**8 weeks**, dual pods) · Translation, test, integration, deploy

Engineering build hours from **§3.3 (11,340 hrs for Phase 2)** are embedded in Sr/Assoc/QA sprint labor below; other rows are **overhead** per §2.5 (architect spikes, DM, **templated** mesh).

| **WBS** | **Workstream** | **Task** | **Role** | **Loc.** | **Hrs** |
| --- | --- | --- | --- | --- | --- |
| 2.B.01 | All svcs | Architecture leadership — onshore | App Dev Architect | On  | 135 |
| --- | --- | --- | --- | --- | --- |
| 2.B.02 | All svcs | Architecture leadership — offshore | App Dev Architect | Off | 330 |
| --- | --- | --- | --- | --- | --- |
| 2.B.03 | All svcs | Senior engineering — onshore | Sr Back End Engineer | On  | 380 |
| --- | --- | --- | --- | --- | --- |
| 2.B.05 | All svcs | Senior engineering — offshore | Sr Back End Engineer | Off | 2,537 |
| --- | --- | --- | --- | --- | --- |
| 2.B.06 | All svcs | Associate engineering — offshore (residual test-data prep / fixtures only) | Assoc Software Engineer | Off | 130 |
| --- | --- | --- | --- | --- | --- |
| 2.B.07 | All svcs | Cloud engineering — K8s manifests, CI/CD — onshore | Cloud Engineer | On  | 72  |
| --- | --- | --- | --- | --- | --- |
| 2.B.08 | All svcs | Cloud engineering — offshore (copy-modify against proven P1 templates) | Cloud Engineer | Off | 176 |
| --- | --- | --- | --- | --- | --- |
| 2.B.09 | Sprint coverage | QA Lead | QA Lead | Off | 290 |
| --- | --- | --- | --- | --- | --- |
| 2.B.10 | Sprint coverage | QA Engineer (×3) | QA Engineer | Off | 700 |
| --- | --- | --- | --- | --- | --- |
| 2.B.13 | Governance | DM — onshore | Delivery Manager | On  | 165 |
| --- | --- | --- | --- | --- | --- |
| 2.B.14 | Governance | DM — IST (25%) | Delivery Manager | Off | 40  |
| --- | --- | --- | --- | --- | --- |
| 2.B.15 | Governance | Sponsor governance | Delivery Manager | On  | 12  |
| --- | --- | --- | --- | --- | --- |
| 2.B.16 | Istio Routing | Strangler fig VirtualService/DestinationRule — **template reuse / batch gen** per P2 service; Cloud Eng HITL | Cloud Engineer | Off | 25  |
| --- | --- | --- | --- | --- | --- |
|     | **Build subtotal** |     |     |     | **~4,992** |
| --- | --- | --- | --- | --- | --- |

### 6.6 Phase 2 — Verify (2 weeks)

| **WBS** | **Workstream** | **Task** | **Role** | **Loc.** | **Hrs** |
| --- | --- | --- | --- | --- | --- |
| 2.Y.01 | QA  | Test execution | QA Lead | Off | 58  |
| --- | --- | --- | --- | --- | --- |
| 2.Y.02 | QA  | QA engineers (×3) | QA Engineer | Off | 265 |
| --- | --- | --- | --- | --- | --- |
| 2.Y.04 | Engineering | Defect remediation — onshore | Sr Back End Engineer | On  | 48  |
| --- | --- | --- | --- | --- | --- |
| 2.Y.05 | Engineering | Remediation — offshore | Sr Back End Engineer | Off | 180 |
| --- | --- | --- | --- | --- | --- |
| 2.Y.06 | Engineering | Associate remediation | Assoc Software Engineer | Off | 11  |
| --- | --- | --- | --- | --- | --- |
| 2.Y.07 | Architecture | App Dev Architect triage | App Dev Architect | On  | 18  |
| --- | --- | --- | --- | --- | --- |
| 2.Y.08 | Architecture | Architect triage — offshore | App Dev Architect | Off | 20  |
| --- | --- | --- | --- | --- | --- |
| 2.Y.09 | UAT | UAT coordination with Frontdoor | Delivery Manager | On  | 38  |
| --- | --- | --- | --- | --- | --- |
| 2.Y.10 | Cloud | Perf testing (API response time) | Cloud Engineer | Off | 38  |
| --- | --- | --- | --- | --- | --- |
| 2.Y.11 | Governance | DM  | Delivery Manager | On  | 32  |
| --- | --- | --- | --- | --- | --- |
| 2.Y.12 | Governance | DM IST (25%) | Delivery Manager | Off | 18  |
| --- | --- | --- | --- | --- | --- |
|     | **Verify subtotal** |     |     |     | **~726** |
| --- | --- | --- | --- | --- | --- |

### 6.7 Phase 2 — Operate (1 week)

| **WBS** | **Workstream** | **Task** | **Role** | **Loc.** | **Hrs** |
| --- | --- | --- | --- | --- | --- |
| 2.O.01 | Cloud | SRE handover (K8s manifests) | Cloud Engineer | Off | 30  |
| --- | --- | --- | --- | --- | --- |
| 2.O.02 | KT  | KT to Frontdoor | Sr Back End Engineer | On  | 35  |
| --- | --- | --- | --- | --- | --- |
| 2.O.03 | KT  | KT — offshore | Sr Back End Engineer | Off | 55  |
| --- | --- | --- | --- | --- | --- |
| 2.O.04 | KT  | Documentation finalization — review & publish AI-generated artifacts | Sr Back End Engineer | Off | 25  |
| --- | --- | --- | --- | --- | --- |
| 2.O.05 | Architecture | Architecture handover packet | App Dev Architect | On  | 22  |
| --- | --- | --- | --- | --- | --- |
| 2.O.06 | Governance | Phase close-out | Delivery Manager | On  | 25  |
| --- | --- | --- | --- | --- | --- |
| 2.O.07 | Governance | DM IST (25%) | Delivery Manager | Off | 6   |
| --- | --- | --- | --- | --- | --- |
| 2.O.08 | Governance | Sponsor close-out | Delivery Manager | On  | 12  |
| --- | --- | --- | --- | --- | --- |
|     | **Operate subtotal** |     |     |     | **200** |
| --- | --- | --- | --- | --- | --- |

### 6.8 Phase 2 — Subtotal

| **Stage** | **Hours** |
| --- | --- |
| Prepare | —   |
| --- | --- |
| Define | 161 |
| --- | --- |
| Design | 235 |
| --- | --- |
| Plan | 205 |
| --- | --- |
| Value Checkpoint | 33  |
| --- | --- |
| Build | 4,992 |
| --- | --- |
| Verify | 726 |
| --- | --- |
| Operate | 200 |
| --- | --- |
| **Phase 2 total** | **~6,552** |
| --- | --- |

## 7\. Phase 3 — Translate & decommission final 39 PL/SQL objects (**~14.5 weeks** active)

**Services touched in P3:** Customer · Contract · Renewal (extend) · Cancellation (new) · Dispatch (extend) · Service Order (extend, P3 dominant) · Work Order/Fulfillment (extend) · Invoicing (extend) · Payment (extend) · Vendor (extend) · Vendor Invoice (extend) · Sales Transaction (extend) · Finance Integration (extend) · Card Security (extend) · Notification (extend) · Reporting (extend) · Web Portal & User (extend). **17 services, mostly extensions.** AHS_QUEUE_FUNCTIONS delivered as a Phase 0 shared library and consumed in-process by Cancellation, Sales Transaction, Work Order & Fulfillment, and Customer services.

Phase 3 has **no dedicated Prepare stage** — the team and operating model carry over from Phase 2.

### 7.1 Phase 3 — Define (1 week)

| **WBS** | **Workstream** | **Task** | **Role** | **Loc.** | **Hrs** |
| --- | --- | --- | --- | --- | --- |
| 3.D.01 | Architecture | Contract scoping (extensions = delta only) | App Dev Architect | On  | 14  |
| --- | --- | --- | --- | --- | --- |
| 3.D.02 | Architecture | Contract scoping — offshore | App Dev Architect | Off | 21  |
| --- | --- | --- | --- | --- | --- |
| 3.D.03 | Engineering | PL/SQL deep-dive (39 packages; AI-driven, HITL validation) — onshore | Sr Back End Engineer | On  | 13  |
| --- | --- | --- | --- | --- | --- |
| 3.D.04 | Engineering | Deep-dive — offshore | Sr Back End Engineer | Off | 54  |
| --- | --- | --- | --- | --- | --- |
| 3.D.06 | Governance | Frontdoor-BA liaison, requirements validation gates | Delivery Manager | On  | 11  |
| --- | --- | --- | --- | --- | --- |
| 3.D.07 | Governance | DM IST (50%) | Delivery Manager | Off | 7   |
| --- | --- | --- | --- | --- | --- |
|     | **Define subtotal** |     |     |     | **120** |
| --- | --- | --- | --- | --- | --- |

### 7.2 Phase 3 — Design (2 weeks)

| **WBS** | **Workstream** | **Task** | **Role** | **Loc.** | **Hrs** |
| --- | --- | --- | --- | --- | --- |
| 3.S.01 | Architecture | API design (12 effective svcs; AI gen + HITL) — onshore | App Dev Architect | On  | 22  |
| --- | --- | --- | --- | --- | --- |
| 3.S.02 | Architecture | Design — offshore | App Dev Architect | Off | 52  |
| --- | --- | --- | --- | --- | --- |
| 3.S.03 | Engineering | Detailed design (DTO mapping, Oracle query patterns; delta for extensions) — onshore | Sr Back End Engineer | On  | 18  |
| --- | --- | --- | --- | --- | --- |
| 3.S.04 | Engineering | Detailed design — offshore | Sr Back End Engineer | Off | 62  |
| --- | --- | --- | --- | --- | --- |
| 3.S.05 | Cloud | K8s manifest reuse (templates from P1/P2) | Cloud Engineer | Off | 16  |
| --- | --- | --- | --- | --- | --- |
| 3.S.06 | Cloud | Cloud Architect review | Cloud Architect | On  | 3   |
| --- | --- | --- | --- | --- | --- |
| 3.S.07 | Security | Card Security extension review (PCI re-validation) | Security Architect | On  | 16  |
| --- | --- | --- | --- | --- | --- |
| 3.S.08 | Security | Security Engineer | Security Engineer | Off | 28  |
| --- | --- | --- | --- | --- | --- |
| 3.S.09 | QA  | Test strategy | QA Lead | Off | 20  |
| --- | --- | --- | --- | --- | --- |
| 3.S.10 | Governance | DM  | Delivery Manager | On  | 12  |
| --- | --- | --- | --- | --- | --- |
| 3.S.11 | Governance | DM IST (50%) | Delivery Manager | Off | 6   |
| --- | --- | --- | --- | --- | --- |
|     | **Design subtotal** |     |     |     | **255** |
| --- | --- | --- | --- | --- | --- |

### 7.3 Phase 3 — Plan (1 week)

| **WBS** | **Workstream** | **Task** | **Role** | **Loc.** | **Hrs** |
| --- | --- | --- | --- | --- | --- |
| 3.L.01 | Governance | Sprint/release plan | Delivery Manager | On  | 30  |
| --- | --- | --- | --- | --- | --- |
| 3.L.02 | Governance | DM IST (50%) | Delivery Manager | Off | 12  |
| --- | --- | --- | --- | --- | --- |
| 3.L.03 | Architecture | LOE refine | App Dev Architect | On  | 20  |
| --- | --- | --- | --- | --- | --- |
| 3.L.04 | Architecture | LOE — offshore | App Dev Architect | Off | 47  |
| --- | --- | --- | --- | --- | --- |
| 3.L.05 | Engineering | Backlog grooming | Sr Back End Engineer | On  | 13  |
| --- | --- | --- | --- | --- | --- |
| 3.L.06 | Engineering | Backlog grooming offshore | Sr Back End Engineer | Off | 40  |
| --- | --- | --- | --- | --- | --- |
| 3.L.07 | QA  | Test plan | QA Lead | Off | 38  |
| --- | --- | --- | --- | --- | --- |
| 3.L.08 | Cloud | Env plan (namespace delta from P2) | Cloud Engineer | Off | 9   |
| --- | --- | --- | --- | --- | --- |
| 3.L.09 | Governance | UAT scheduling with Frontdoor | Delivery Manager | On  | 21  |
| --- | --- | --- | --- | --- | --- |
|     | **Plan subtotal** |     |     |     | **230** |
| --- | --- | --- | --- | --- | --- |

### 7.4 Phase 3 — Value Checkpoint (<1 week)

| **WBS** | **Workstream** | **Task** | **Role** | **Loc.** | **Hrs** |
| --- | --- | --- | --- | --- | --- |
| 3.V.01 | Governance | VC deck | Delivery Manager | On  | 15  |
| --- | --- | --- | --- | --- | --- |
| 3.V.02 | Governance | Sponsor VC | Delivery Manager | On  | 3   |
| --- | --- | --- | --- | --- | --- |
| 3.V.03 | Architecture | Architecture VC | App Dev Architect | On  | 5   |
| --- | --- | --- | --- | --- | --- |
| 3.V.04 | Architecture | App Dev Architect walkthrough | App Dev Architect | On  | 10  |
| --- | --- | --- | --- | --- | --- |
|     | **Value Checkpoint subtotal** |     |     |     | **33** |
| --- | --- | --- | --- | --- | --- |

### 7.5 Phase 3 — Build (**7 weeks**, dual pods) · Translation, test, integration, deploy

Engineering build hours from **§3.3 (6,475 hrs for Phase 3)** are embedded in Sr/Assoc/QA sprint labor below; other rows are **overhead** per §2.5.

| **WBS** | **Workstream** | **Task** | **Role** | **Loc.** | **Hrs** |
| --- | --- | --- | --- | --- | --- |
| 3.B.01 | All svcs | Architecture — onshore | App Dev Architect | On  | 72  |
| --- | --- | --- | --- | --- | --- |
| 3.B.02 | All svcs | Architecture — offshore | App Dev Architect | Off | 185 |
| --- | --- | --- | --- | --- | --- |
| 3.B.03 | All svcs | Sr eng — onshore | Sr Back End Engineer | On  | 240 |
| --- | --- | --- | --- | --- | --- |
| 3.B.05 | All svcs | Sr eng — offshore | Sr Back End Engineer | Off | 1,267 |
| --- | --- | --- | --- | --- | --- |
| 3.B.06 | All svcs | Assoc eng — offshore (residual test-data prep / fixtures only) | Assoc Software Engineer | Off | 75  |
| --- | --- | --- | --- | --- | --- |
| 3.B.07 | All svcs | Cloud eng — K8s manifests, CI/CD — onshore | Cloud Engineer | On  | 48  |
| --- | --- | --- | --- | --- | --- |
| 3.B.08 | All svcs | Cloud eng — offshore (copy-modify against proven templates) | Cloud Engineer | Off | 116 |
| --- | --- | --- | --- | --- | --- |
| 3.B.09 | Card Security extension | Security engineer | Security Engineer | Off | 56  |
| --- | --- | --- | --- | --- | --- |
| 3.B.10 | Card Security extension | Security architect oversight | Security Architect | On  | 13  |
| --- | --- | --- | --- | --- | --- |
| 3.B.11 | Sprint coverage | QA Lead | QA Lead | Off | 225 |
| --- | --- | --- | --- | --- | --- |
| 3.B.12 | Sprint coverage | QA Engineer (×3) | QA Engineer | Off | 492 |
| --- | --- | --- | --- | --- | --- |
| 3.B.15 | Governance | DM — onshore | Delivery Manager | On  | 139 |
| --- | --- | --- | --- | --- | --- |
| 3.B.16 | Governance | DM — IST (25%) | Delivery Manager | Off | 35  |
| --- | --- | --- | --- | --- | --- |
| 3.B.17 | Governance | Sponsor governance | Delivery Manager | On  | 8   |
| --- | --- | --- | --- | --- | --- |
| 3.B.18 | Istio Routing | Strangler fig VirtualService/DestinationRule — **template reuse / batch gen** per P3 service | Cloud Engineer | Off | 28  |
| --- | --- | --- | --- | --- | --- |
|     | **Build subtotal** |     |     |     | **~2,999** |
| --- | --- | --- | --- | --- | --- |

### 7.6 Phase 3 — Verify (2 weeks)

| **WBS** | **Workstream** | **Task** | **Role** | **Loc.** | **Hrs** |
| --- | --- | --- | --- | --- | --- |
| 3.Y.01 | QA  | QA Lead execution | QA Lead | Off | 44  |
| --- | --- | --- | --- | --- | --- |
| 3.Y.02 | QA  | QA Engineer (×3) | QA Engineer | Off | 160 |
| --- | --- | --- | --- | --- | --- |
| 3.Y.04 | Engineering | Defect remediation — onshore | Sr Back End Engineer | On  | 38  |
| --- | --- | --- | --- | --- | --- |
| 3.Y.05 | Engineering | Remediation — offshore | Sr Back End Engineer | Off | 150 |
| --- | --- | --- | --- | --- | --- |
| 3.Y.06 | Engineering | Associate remediation | Assoc Software Engineer | Off | 10  |
| --- | --- | --- | --- | --- | --- |
| 3.Y.07 | Architecture | App Dev Architect triage | App Dev Architect | On  | 13  |
| --- | --- | --- | --- | --- | --- |
| 3.Y.08 | Architecture | Architect triage — offshore | App Dev Architect | Off | 17  |
| --- | --- | --- | --- | --- | --- |
| 3.Y.09 | UAT | UAT coordination with Frontdoor | Delivery Manager | On  | 32  |
| --- | --- | --- | --- | --- | --- |
| 3.Y.10 | Cloud | Perf testing (API response time) | Cloud Engineer | Off | 30  |
| --- | --- | --- | --- | --- | --- |
| 3.Y.11 | Governance | DM  | Delivery Manager | On  | 26  |
| --- | --- | --- | --- | --- | --- |
| 3.Y.12 | Governance | DM IST (25%) | Delivery Manager | Off | 13  |
| --- | --- | --- | --- | --- | --- |
|     | **Verify subtotal** |     |     |     | **493** |
| --- | --- | --- | --- | --- | --- |

### 7.7 Phase 3 — Operate (1 week)

| **WBS** | **Workstream** | **Task** | **Role** | **Loc.** | **Hrs** |
| --- | --- | --- | --- | --- | --- |
| 3.O.01 | Cloud | SRE handover (K8s manifests) | Cloud Engineer | Off | 22  |
| --- | --- | --- | --- | --- | --- |
| 3.O.02 | KT  | KT — final program | Sr Back End Engineer | On  | 35  |
| --- | --- | --- | --- | --- | --- |
| 3.O.03 | KT  | KT — offshore | Sr Back End Engineer | Off | 45  |
| --- | --- | --- | --- | --- | --- |
| 3.O.04 | KT  | Documentation finalization — review & publish AI-generated artifacts | Sr Back End Engineer | Off | 25  |
| --- | --- | --- | --- | --- | --- |
| 3.O.05 | Architecture | Final architecture handover packet | App Dev Architect | On  | 25  |
| --- | --- | --- | --- | --- | --- |
| 3.O.06 | Governance | Program close-out, final retrospective | Delivery Manager | On  | 32  |
| --- | --- | --- | --- | --- | --- |
| 3.O.07 | Governance | DM IST (25%) | Delivery Manager | Off | 6   |
| --- | --- | --- | --- | --- | --- |
| 3.O.08 | Governance | Sponsor final close-out | Delivery Manager | On  | 12  |
| --- | --- | --- | --- | --- | --- |
|     | **Operate subtotal** |     |     |     | **200** |
| --- | --- | --- | --- | --- | --- |

### 7.8 Phase 3 — Subtotal

| **Stage** | **Hours** |
| --- | --- |
| Prepare | —   |
| --- | --- |
| Define | 120 |
| --- | --- |
| Design | 255 |
| --- | --- |
| Plan | 230 |
| --- | --- |
| Value Checkpoint | 33  |
| --- | --- |
| Build | 2,999 |
| --- | --- |
| Verify | 493 |
| --- | --- |
| Operate | 200 |
| --- | --- |
| **Phase 3 total** | **~4,330** |
| --- | --- |

## 8\. Cross-phase governance & enabling tasks

These run continuously regardless of which program phase is active. Hours below span the **~6.0-month (~26-week)** program wall-clock (§2.5).

| **WBS** | **Task** | **Role** | **Loc.** | **Hrs** |
| --- | --- | --- | --- | --- |
| X.01 | Sponsor relationship, escalation, contract management (**monthly cadence** after mobilization) | Delivery Manager | On  | 25  |
| --- | --- | --- | --- | --- |
| X.02 | Cross-domain alignment, ADR review (quarterly reviews) | App Dev Architect | On  | 25  |
| --- | --- | --- | --- | --- |
| X.03 | AI tooling continuous improvement (tapers after P1; self-calibrates) | App Dev Architect | On  | 38  |
| --- | --- | --- | --- | --- |
| X.04 | AI tooling continuous improvement | App Dev Architect | Off | 58  |
| --- | --- | --- | --- | --- |
| X.05 | Platform team — shared lib maintenance, NuGet versioning, security patches (**release-window** cadence) | Sr Back End Engineer | Off | 125 |
| --- | --- | --- | --- | --- |
| X.06 | Platform team — associate (residual support only) | Assoc Software Engineer | Off | 7   |
| --- | --- | --- | --- | --- |
| X.07 | Cloud platform team — K8s manifest maintenance | Cloud Engineer | Off | 42  |
| --- | --- | --- | --- | --- |
| X.08 | Security ops — AWS KMS rotation procedures, internal security hygiene | Security Engineer | Off | 48  |
| --- | --- | --- | --- | --- |
| X.10 | Istio routing ops — per-service VirtualService/DestinationRule maintenance, strangler fig traffic-split tuning, cutover tracking (no control-plane upgrades) | Cloud Engineer | Off | 18  |
| --- | --- | --- | --- | --- |
| X.11 | Project mesh security ops — project-scoped AuthorizationPolicy updates for new services; coordination on Frontdoor mesh audits | Security Engineer | Off | 10  |
| --- | --- | --- | --- | --- |
|     | **Cross-phase subtotal** |     |     | **~368** |
| --- | --- | --- | --- | --- |

## 9\. Effort roll-up by phase

| **Phase** | **Calendar duration** | **Effort (hours)** |
| --- | --- | --- |
| Phase 0 — Foundation | **2.5 wks** | 983 |
| --- | --- | --- |
| Phase 1 — Translate & decommission 45 P1 objects | **~21 wks** | 8,517 |
| --- | --- | --- |
| Phase 2 — Translate & decommission 45 P2 objects | **~14.5 wks** | 6,552 |
| --- | --- | --- |
| Phase 3 — Translate & decommission 39 P3 objects | **~14.5 wks** | 4,330 |
| --- | --- | --- |
| Cross-phase governance | (parallel) | 368 |
| --- | --- | --- |
| **Subtotal** |     | **20,750** |
| --- | --- | --- |
| Less overlap savings (**5,730 h** — see §2.5 and assumption 32) |     | (5,730) |
| --- | --- | --- |
| **After overlap savings (net labor)** |     | **~15,020** |
| --- | --- | --- |
| **Acceleration premium** (dual-pod integration, parallel UAT coordination, incremental QA + architect HITL — §2.5) | (parallel) | **2,100** |
| --- | --- | --- |
| **Total program effort** | **~26 wks wall-clock** | **~17,120** |
| --- | --- | --- |

**Calendar:** **~6.0 months (~26 weeks)** end-to-end from Phase 0 start through Phase 3 Operate, using **phase overlap** and **dual Build pods** (§2.5). **Verify** remains **3 / 2 / 2** calendar weeks per phase. **Overlap-adjusted labor** is **~15,020 h** (~**$1.65M** at ~~$110/hr illustrative blend); the~~ **~~~2,100 h~~** ~~acceleration premium (~~**$0.23M**) funds parallel integration and acceptance capacity so scope and parity obligations are not traded for speed. If Frontdoor cannot staff **parallel UAT cohorts**, end-to-end calendar extends regardless of engineering parallelism (§2.5).

## 10\. Effort & cost roll-up by role

Effort aggregated across Phase 0–3 + cross-phase, with rates from §2.3. BA, SME, hypercare, P2/P3 Prepare, Terraform/IaC, Istio control-plane work, and contingency all excluded. Dead code factor 50% applied. K8s manifests + project Istio routing configuration — no Terraform infrastructure provisioning. 100% AI-driven code generation with HITL validation.

| **Role** | **On Hrs** | **Off Hrs** | **Total Hrs** | **Blended $/hr** | **Cost (USD)** |
| --- | --- | --- | --- | --- | --- |
| App Dev Architect | 944 | 1,298 | **2,242** | 164 | 367,700 |
| --- | --- | --- | --- | --- | --- |
| Cloud Architect | 144 | —   | 144 | 291 | 41,904 |
| --- | --- | --- | --- | --- | --- |
| Security Architect | 178 | —   | 178 | 314 | 55,930 |
| --- | --- | --- | --- | --- | --- |
| Sr Back End Engineer | 1,617 | 5,038 | **6,655** | 111 | 738,700 |
| --- | --- | --- | --- | --- | --- |
| Cloud Engineer | 185 | 1,283 | **1,468** | 99  | 145,300 |
| --- | --- | --- | --- | --- | --- |
| Security Engineer | —   | 556 | **556** | 92  | 51,152 |
| --- | --- | --- | --- | --- | --- |
| Associate Software Engineer | —   | 1,057 | **1,057** | 53  | 56,000 |
| --- | --- | --- | --- | --- | --- |
| QA Lead | —   | 820 | **820** | 68  | 55,560 |
| --- | --- | --- | --- | --- | --- |
| QA Engineer | —   | 2,690 | **2,690** | 63  | 169,070 |
| --- | --- | --- | --- | --- | --- |
| Delivery Manager | 1,058 | 468 | **1,526** | 179 | 273,200 |
| --- | --- | --- | --- | --- | --- |
| **Program total** | **4,964** | **12,156** | **~17,120** | **~114** | **~$1.95M** |
| --- | --- | --- | --- | --- | --- |

Mix check — Onshore : Offshore ≈ **29% : 71%** by hours (includes incremental onshore DM/Architect for parallel delivery). The cost-optimization pass removed mostly lower-rate offshore associate hours, so the effective blend rises slightly to **~$114/hr**.

**Reconciliation:** §4–§8 gross **20,750 h** before the **5,730 h** overlap credit → **~15,020 h**; **+2,100 h** acceleration premium (§2.5) → **~17,120 h**. Role costs are **illustrative** at ~~$114/hr effective blend (cost ÷ hours).~~ **~~Overlap-adjusted effort~~** ~~is~~ **~~~15,020 h~~** ~~(~~**$1.65M** at ~$110/hr) before the acceleration premium. IST DM tapered from 50% to 25% allocation in P2/P3 Build. No contingency is applied.

### 10.1 Cost by program phase

| **Phase** | **Effort (hrs)** | **Cost (USD)** |
| --- | --- | --- |
| Phase 0 — Foundation | 983 | $0.11M |
| --- | --- | --- |
| Phase 1 | 8,517 | $0.76M |
| --- | --- | --- |
| Phase 2 | 6,552 | $0.57M |
| --- | --- | --- |
| Phase 3 | 4,330 | $0.38M |
| --- | --- | --- |
| Cross-phase governance | 368 | $0.02M |
| --- | --- | --- |
| **Gross subtotal** | **~20,750** | **~$1.84M** |
| --- | --- | --- |
| Less overlap savings (5,730 hrs) | (5,730) | (~$0.13M) |
| --- | --- | --- |
| **After overlap (net labor)** | **~15,020** | **~$1.72M** |
| --- | --- | --- |
| **Acceleration premium** (§2.5) | **2,100** | **~$0.23M** |
| --- | --- | --- |
| **Total program** | **~17,120** | **~$1.95M** |
| --- | --- | --- |

**Planning anchor:** **~$1.95M** and **~17,120 h** for the **~6.0-month wall-calendar** program. Treat **~$1.85M–$2.05M** as the band. **~15,020 h** (~**$1.65M**) is overlap-adjusted labor before the acceleration premium. No contingency is applied.

## 11\. Per-service effort × phase matrix (for Microsoft Project / Smartsheet import)

**Upper-bound sizing reference (see §3 note).** This per-service matrix is a top-down tier inventory for planning/import. It is **not** the costed model and is **not** reconciled line-for-line into §9–§10; the per-stage tables in §4–§8 govern effort and cost. Treat these per-service totals as an upper bound for sequencing and capacity planning.

Engineering build hours per service per phase (at 50% effective LOC, excluding shared cross-svc PM/architect/QA overheads).

| **Service** | **P1 Build** | **P2 Build** | **P3 Build** | **Total Build** |
| --- | --- | --- | --- | --- |
| Customer Service | 540 | 900 | 360 | 1,800 |
| --- | --- | --- | --- | --- |
| Contract Service | 1,700 | 0   | 100 | 1,800 |
| --- | --- | --- | --- | --- |
| Pricing & Revenue Service | 625 | 275 | 0   | 900 |
| --- | --- | --- | --- | --- |
| Renewal Service | 0   | 1,285 | 515 | 1,800 |
| --- | --- | --- | --- | --- |
| Cancellation Service | 0   | 0   | 900 | 900 |
| --- | --- | --- | --- | --- |
| Dispatch Service | 1,400 | 1,050 | 350 | 2,800 |
| --- | --- | --- | --- | --- |
| Service Order Service | 85  | 400 | 1,415 | 1,900 |
| --- | --- | --- | --- | --- |
| Work Order & Fulfillment Service | 280 | 2,100 | 420 | 2,800 |
| --- | --- | --- | --- | --- |
| Invoicing Service | 900 | 770 | 130 | 1,800 |
| --- | --- | --- | --- | --- |
| Payment Service | 1,080 | 165 | 105 | 1,350 |
| --- | --- | --- | --- | --- |
| Vendor Service | 450 | 800 | 550 | 1,800 |
| --- | --- | --- | --- | --- |
| Vendor Invoice & Writeoff Service | 1,255 | 385 | 160 | 1,800 |
| --- | --- | --- | --- | --- |
| Sales Transaction Service | 0   | 400 | 500 | 900 |
| --- | --- | --- | --- | --- |
| Commission & Territory Service | 750 | 750 | 0   | 1,500 |
| --- | --- | --- | --- | --- |
| Campaign & Lead Service | 0   | 900 | 0   | 900 |
| --- | --- | --- | --- | --- |
| Finance Integration Service | 540 | 60  | 300 | 900 |
| --- | --- | --- | --- | --- |
| Card Security Service (PCI) | 150 | 0   | 150 | 300 |
| --- | --- | --- | --- | --- |
| Notification Service | 0   | 265 | 135 | 400 |
| --- | --- | --- | --- | --- |
| Reporting Service | 360 | 270 | 270 | 900 |
| --- | --- | --- | --- | --- |
| Web Portal & User Service | 0   | 600 | 300 | 900 |
| --- | --- | --- | --- | --- |
| **Microservice subtotal** | **9,935** | **11,340** | **6,475** | **27,750** |
| --- | --- | --- | --- | --- |
| Shared libs (Phase 0) | 1,700 | —   | —   | 1,700 |
| --- | --- | --- | --- | --- |
| **Engineering build totals** | **11,635** | **11,340** | **6,475** | **29,450** |
| --- | --- | --- | --- | --- |

## 12\. Risks, dependencies, and assumptions

### 12.1 Top risks

| **#** | **Risk** | **Likelihood** | **Impact** | **Mitigation** |
| --- | --- | --- | --- | --- |
| R1  | **STAR_CLOUD** and **COMMON_FUNC_EMAIL_ROB3** have no implementation in package or additional_code exports | Medium | Medium | STAR_CLOUD: adapter from call sites (Phase 0). COMMON_FUNC_EMAIL_ROB3: confirm retired vs never exported with Frontdoor before Phase 3 Define |
| --- | --- | --- | --- | --- |
| R2  | **prod.star_cloud** still has no local source; **clob_to_file** / **cost_center2department** now exported in additional_code | Medium | Medium | File & S3 Storage / Finance Integration adapters use supplemental SQL where available; star_cloud remains call-site reverse engineering |
| --- | --- | --- | --- | --- |
| R3  | AI translation introduces semantic drift in financial logic (commission, write-off, credit-memo) | Medium | High | Mandatory pair-review by sr engineer + dedicated parity test suite (test PL/SQL output vs .NET output for ≥90% scenarios) |
| --- | --- | --- | --- | --- |
| R4  | Caller systems span undocumented integration patterns (DBMS_AQ, JDE files, lockbox, etc.) | Medium | High | Phase 0 prod-dependency decomposition (WBS 0.33); affected packages either re-implemented as REST endpoints inside their owning service or owned by Frontdoor's caller systems |
| --- | --- | --- | --- | --- |
| R5  | PCI scope expansion from Card Security service forces enclaved infra | Medium | High | Phase 0 PCI scope analysis (WBS 0.12); AWS KMS-backed Crypto lib isolates key material |
| --- | --- | --- | --- | --- |
| R6  | UAT delays due to Frontdoor BA / SME / business-user availability | High | Medium | DM books **parallel UAT lanes** + standing SME windows (§2.5); DM books UAT slots at Plan and schedules Frontdoor BA / SME participation in Define and Verify; UAT failures attributed to Frontdoor-side capacity are change-control items |
| --- | --- | --- | --- | --- |
| R7  | Concurrent decommissioning by Frontdoor causes service contract churn | Low | High | Out of scope per SOW; communicated explicitly; change-control gate for in-flight contract changes |
| --- | --- | --- | --- | --- |
| R8  | AI productivity assumptions under-realized due to PL/SQL idioms | Medium | Medium | Dead code factor (50%) provides implicit buffer; Phase 0 golden-example exercise calibrates actual AI rate and dead-code ratio before Phase 1 commit; re-baseline at Value Checkpoint if variance >15% |
| --- | --- | --- | --- | --- |
| R9  | No 66degrees hypercare means production defects post go-live land on Frontdoor | Medium | Medium | 1-week stabilization within Operate (deploy → monitor → close-out); defect-severity gates inside Verify; Frontdoor SRE knowledge transfer is an explicit Operate deliverable |
| --- | --- | --- | --- | --- |
| R10 | Requirements drift without an embedded BA | Medium | Medium | App Dev Architect + Sr Engineer jointly own the AI-extracted PL/SQL specifications; weekly Frontdoor-BA validation gate inside Define |
| --- | --- | --- | --- | --- |
| R11 | No Phase 2 / Phase 3 Prepare stage compresses ramp-up windows for net-new P2/P3 services | Medium | Medium | Phase 1 Operate close-out includes lessons-learned & next-phase planning; Phase 2 / 3 Define expands scope to include light onboarding for any net-new service team members |
| --- | --- | --- | --- | --- |
| R12 | Single-week Plan stages reduce buffer for sprint-plan & test-strategy refinement | Medium | Low | Test plan re-uses Phase 1 templates from Phase 2 onward; sprint plan templates and capacity model are codified in cross-phase governance (X.09) |
| --- | --- | --- | --- | --- |
| R13 | Istio routing adds operational complexity (sidecar proxy behavior, existing mesh conventions, debugging through proxies during strangler cutover) | Medium | Medium | Frontdoor's existing Istio is mature and in production; Phase 0 POC validates integration, routing templates, and debugging workflows before Phase 1 commit. Rollback to 100% legacy via VirtualService weight is per service. Control-plane upgrades remain Frontdoor-owned. |
| --- | --- | --- | --- | --- |
| R14 | Removed Async Messaging wrapper leaves true DBMS_AQ-style workloads unowned (counter increments, AQ enqueue/dequeue) | Medium | Medium | Frontdoor's caller systems retain those queue patterns (Amazon SQS, internal app queues, or direct REST polling); not 66degrees scope. AHS_QUEUE_FUNCTIONS is excluded from this risk — it is preserved as a shared library so all dependent in-scope services keep working. |
| --- | --- | --- | --- | --- |
| R15 | Reduced architect staffing (1 offshore) creates single-point-of-failure for offshore architecture decisions | Medium | Medium | Onshore App Dev Architect lead provides continuity; Cloud Architect co-owns platform decisions; AI-assisted design artifacts reduce dependency on individual architects; architecture decisions documented as ADRs. Architecture frozen after Phase 0. |
| --- | --- | --- | --- | --- |
| R16 | Elimination of Associate QA reduces regression testing throughput during Verify phases | Low | Medium | **+2,100 h** program acceleration premium adds **QA Engineer + DM** capacity for parallel regression/parity execution (§2.5); QA Engineers absorb associate hours; AI-generated test suites provide broader coverage baseline; Verify phases sized for QA Engineer capacity. If defect density exceeds expectations during P1 Verify, QA headcount is re-evaluated at P2 Plan. |
| --- | --- | --- | --- | --- |
| R17 | **STAR_CLOUD** / **COMMON_FUNC_EMAIL_ROB3** may require scope change if source is recovered later | Low | Medium | Only 2 objects lack SQL after additional_code pass. Re-baseline at Value Checkpoint if Frontdoor supplies missing exports. |
| --- | --- | --- | --- | --- |
| R18 | Strangler fig pattern requires legacy PL/SQL to be exposed as REST services (ORDS or equivalent) — if Frontdoor cannot expose legacy endpoints, traffic splitting is blocked | Medium | High | Confirm Oracle REST Data Services (ORDS) capability or equivalent REST exposure mechanism during Phase 0 Design. If legacy REST exposure is delayed, services deploy behind mesh but use big-bang cutover (the mesh still provides mTLS and observability value). Rollback procedure validated in Phase 0 POC (WBS 0.38). |
| --- | --- | --- | --- | --- |
| R19 | Service mesh strangler fig traffic splitting introduces latency and complexity in the critical path during dual-running periods | Low | Medium | Phase 0 POC (WBS 0.38) benchmarks latency overhead of mesh routing. Weighted traffic splitting is per-service — only actively migrating services run dual-routed; fully cut-over services route 100% to .NET with no splitting overhead. Mesh sidecar proxy adds <2ms p99 per hop in typical Istio deployments. |
| --- | --- | --- | --- | --- |

### 12.2 Key assumptions

1.  The 132-object source scope is reduced to **122 confirmed objects** in this WBS: 3 truly inter-service queue objects removed (ENQUEUE_MESSAGE, DEQUEUE_MESSAGE, PROC_INCREMENT_USAGE_COUNT — SQL exists in src/additional_code/ but remains out of build scope), and 4 double-counted objects formally descoped as package-only aliases (PROC_CREDIT_MEMO_APPLY, PPUB_REVERSE_PAYMENT, PPUB_SERVICEFEE_WRITEOFF, VALIDATECREDITCARD — each also has a standalone export in additional_code). **CLOB_TO_FILE** and **COST_CENTER2DEPARTMENT** are implemented in additional_code and drive Phase 0 adapters. **STAR_CLOUD** remains external (no source). **COMMON_FUNC_EMAIL_ROB3** remains absent. Phase 1 JDE/staging procedures (CREATE_\*\_JDE_FILE, PPUB_INVOICE_DIST, PROC_\*\_STAGING) are in additional_code and in scope for Finance Integration / Vendor Invoice translation. AHS_QUEUE_FUNCTIONS is **retained** as the AHS Queue Persistence shared library.
2.  The two remaining missing-source objects (STAR_CLOUD, COMMON_FUNC_EMAIL_ROB3) are confirmed with Frontdoor before Phase 1 Build commit.
3.  Frontdoor provides production-equivalent test data, dual-run capability, and a domain SME per service at agreed allocation. For the target wall calendar, Frontdoor additionally staffs **≥2 parallel UAT cohorts** (or equivalent SME hours) during Verify so acceptance does not queue behind a single serial UAT team.
4.  AWS is the target cloud. Services deploy to an **existing Kubernetes cluster** (no new infrastructure provisioning). Only K8s manifests (Deployment, Service, Ingress, ConfigMap, Secrets, HPA) are generated per service — no Terraform, no AWS Fargate, no IaC. AWS KMS/AWS CloudHSM for crypto. **Data remains in Oracle — no new database provisioning, no data migration.** Oracle HA/backup/tuning remain Frontdoor's operational responsibility.
5.  .NET 8/9 LTS, C# 12, ASP.NET Core minimal APIs + MediatR pattern (in-process), Oracle EF Core provider for relational access (or Dapper with Oracle.ManagedDataAccess where micro-perf matters). **Frontdoor's existing Istio on EKS provides mTLS; this program adds project VirtualServices/DestinationRules for strangler fig traffic routing between legacy Oracle REST and new .NET microservices.** No SNS/SQS for inter-service async.
6.  **100% AI-driven code generation with HITL validation.** AI tooling (Cursor + Claude Sonnet/Opus class LLM agents) is licensed and authorized for use with PL/SQL source code. The AI generates all first-draft code, tests, OpenAPI specs, sequence diagrams, DTO mappings, and K8s manifests end-to-end. Humans review, validate, and iteratively refine AI output — they do not author code. This shifts the staffing model from author-centric (seniors write, associates support) to review-centric (seniors validate, associates perform test-data prep and manual testing).
7.  Onshore = NorthAM, Offshore = India per rate card; DM provides IST follow-the-sun shadow. No nearshore tier — all offshore work is performed from India.
8.  Decommissioning of source PL/SQL packages is performed by Frontdoor (out of scope).
9.  Security/compliance testing (PCI ROC, SOC2, pen-test) is performed by Frontdoor's existing programs (out of scope, except for in-build security engineering of Card Security service).
10. Business process re-engineering / optimization is out of scope; services replicate current PL/SQL behavior 1:1 except where idiomatic .NET patterns are demonstrably equivalent.
11. **Business analysis is provided by Frontdoor (out of scope for 66degrees).** 66degrees engages Frontdoor's BA pool through Delivery Manager + App Dev Architect for requirements validation, UAT planning, and acceptance sign-off.
12. **Subject Matter Experts are provided by Frontdoor (out of scope for 66degrees).** Frontdoor commits SME availability for Define-stage validation gates and Verify-stage UAT.
13. **Hypercare is provided by Frontdoor (out of scope for 66degrees).** 66degrees Operate stage delivers SRE handover, runbooks, dashboards, documentation, and knowledge transfer; Frontdoor's run-team owns post-go-live production support and any patching/incident response thereafter.
14. **Phase 2 and Phase 3 share team and operating model with Phase 1.** No discrete Prepare stage is run in P2 or P3 — kickoff, RAID refresh, and onboarding for net-new team members are absorbed into the Phase 1 Operate close-out and the next-phase Define stage.
15. **Single-week Plan stages** assume sprint-plan, release-plan, and test-strategy templates from Phase 1 are reusable in Phase 2 and Phase 3 with only delta updates.
16. **Target architecture is existing Istio with project strangler-routing.** Each service is a .NET REST API deployed on EKS integrated with Frontdoor's **existing** Istio mesh. The mesh provides mTLS, service discovery, and observability; **this program** adds: (a) **strangler fig traffic splitting** — per-service VirtualService/DestinationRule weighted routing between legacy PL/SQL exposed as REST from Oracle and new .NET microservices, enabling progressive, reversible migration, and (b) **project-scoped AuthorizationPolicy** for new microservice identities. **There is a high degree of inter-service communication** — 11 critical, 17 strong, and 28 moderate couplings across the 18 services. Calling applications route through the mesh. Cross-service data needs are addressed by all services sharing the existing Oracle database, composite read endpoints owned by the data-owning service, or caller-side orchestration. Async Messaging Service wrapper remains out of scope — only the 3 inter-service queue packages are excluded.
17. **Async Messaging Service wrapper is removed from the build scope.** Inter-process queue functionality backed by Oracle DBMS_AQ (ENQUEUE_MESSAGE, DEQUEUE_MESSAGE, PROC_INCREMENT_USAGE_COUNT) is retained by Frontdoor's caller systems; alternatives such as Amazon SQS, internal app queues, or direct REST polling are Frontdoor's responsibility. **AHS_QUEUE_FUNCTIONS is preserved** as the AHS Queue Persistence shared library (NuGet, in-process) because it is a database-table CRUD layer rather than inter-process messaging plumbing — Cancellation, Sales Transaction, Work Order & Fulfillment, and Customer services all consume it.
18. **Data remains in the existing Oracle database.** No new database instances (Amazon RDS or otherwise) are provisioned. No data migration or schema redesign is performed. Services connect to Oracle via ODP.NET / Oracle EF Core provider using connection strings managed in K8s Secrets per environment. Detailed design work is limited to DTO/entity mapping against existing Oracle tables rather than new schema design. **Cloud Engineering scope includes K8s manifest generation** (Deployment, Service, Ingress, ConfigMap, Secrets, HPA), **project Istio routing configuration** (VirtualService, DestinationRule, project AuthorizationPolicy), and CI/CD pipeline configuration — no Terraform, no infrastructure provisioning, no AWS Fargate, **no Istio control-plane installation**. The Kubernetes cluster and Istio mesh are pre-existing and managed by Frontdoor.
19. **Dead code factor: 50%.** Only 50% of source LOC is assumed to require conversion ("effective LOC"). The remaining 50% is dead, unreachable, or duplicated code that does not map to any active business function. This assumption is applied uniformly across all services and shared libraries, driving tier assignment and build hours via the effective LOC. Dead code identification occurs during Define (AI-assisted call-graph analysis, coverage profiling of production traces if available). If actual live-code ratio differs from 50% by more than ±15%, build hours should be re-baselined at the Phase 1 Value Checkpoint.
20. **No contingency is applied.** The dead code factor (50%) and AI productivity uplift together provide sufficient estimation buffer. If both assumptions hold, the program has inherent margin; if one under-performs, the other compensates. A formal contingency reserve is therefore excluded from the cost and effort totals.
21. **Architect staffing at ~1:6 ratio.** App Dev Architect allocation uses one offshore architect per active phase (not two) during Build, supported by one onshore lead. AI-assisted OpenAPI synthesis (~88% uplift) and sequence diagram drafting (~83% uplift) underwrite the reduced staffing.
22. **IST Shadow DM at 50% allocation in P1, tapering to 25% in P2/P3 Build.** The IST DM is on-call for blockers and escalations, not running continuous follow-the-sun parallel delivery management. After Phase 1 establishes operating rhythm, IST coverage tapers.
23. **No Associate QA Engineer role.** AI-generated tests and 1:1 behavioral translation parity reduce the need for paired QA testing. QA Engineers handle all test execution, coverage validation, and triage.
24. **3 trivial services merged into domain parents.** General Ledger and MIM Integration are absorbed into Finance Integration Service (renamed from JDE Sync). Service Request is absorbed into Service Order Service. Service count: 18 microservices + 6 shared libraries.
25. **Two objects remain without source** after the src/additional_code/ export (STAR_CLOUD, COMMON_FUNC_EMAIL_ROB3). **PROC_INCREMENT_USAGE_COUNT** is a discontinued stub. If additional source is recovered during Define, hours are re-baselined at the Value Checkpoint.
26. **Existing Istio + project routing in scope.** Frontdoor operates Istio on the existing EKS cluster. Phase 0 validates integration and authors **project-specific** VirtualService / DestinationRule templates and AuthorizationPolicies for strangler fig traffic routing (weighted splitting between legacy Oracle REST and new .NET services). **No Istio control-plane installation, upgrade, or cluster-wide mesh reconfiguration.** Per-service routing configuration is included in each Build phase. Cross-phase work covers **project** VirtualService maintenance and cutover tracking only — not control-plane upgrades or sidecar lifecycle (Frontdoor-operated).
27. **Strangler fig pattern assumes legacy PL/SQL is exposed as REST.** The strangler fig traffic splitting model requires legacy PL/SQL logic to be accessible as REST endpoints (via Oracle REST Data Services / ORDS or equivalent mechanism). Frontdoor is responsible for exposing legacy logic as REST. If legacy REST exposure is unavailable, the existing mesh still provides mTLS and observability value but cutover reverts to big-bang per service rather than progressive traffic splitting.
28. **Istio routing integration does not extend calendar duration.** Routing architecture, existing-mesh validation, and POC work in Phase 0 runs in parallel with shared-library and cloud-platform tracks. Per-phase VirtualService/DestinationRule configuration runs in parallel with Build sprints.
29. **Architect staffing tapers after Phase 1.** Phase 2 Build architect allocation is 20% below Phase 1 per-week intensity; Phase 3 is 25% below. Established patterns, proven templates, and AI-assisted design artifacts reduce the need for continuous architectural presence. Architects focus on exceptions and cross-cutting concerns; pattern-following work is engineer-led.
30. **Phase 2/Phase 3 Verify** each run **2 calendar weeks**; teams assume lower defect density after P1 (reusable templates, established regression suites). **Incremental QA/DM hours** inside those windows support **parallel lanes** — not fewer calendar days. If defect density exceeds expectations during P1 Verify, QA headcount is re-evaluated at P2 Plan.
31. **Associate SWE at residual level only.** Taking the 100% AI-generation assumption as firm, AI fully absorbs the traditional associate authoring role (scaffolding, CRUD, test stubs), and the residual associate tasks (test-data preparation, fixtures, manual smoke checks) overlap the QA Engineer pool and AI-generated documentation. The Associate SWE allocation is therefore cut to a residual ~1,057 h program-wide (from ~2,230 h) — Phase 0 shared-library associate support is eliminated entirely, and Build/Verify associate hours are reduced ~70%. If the 100% AI generation assumption proves overstated for complex PL/SQL idioms, associate capacity is re-evaluated at the Phase 1 Value Checkpoint.
32. **Overlap savings credit (5,730 h, ~28% of gross 20,750 h)** is applied in §9 between gross phase roll-up and overlap-adjusted net labor. Primary drivers: team continuity, shared libraries built once in Phase 0, calibrated AI and Istio routing templates, repeatable pipeline across 18 services, and behavioral 1:1 translation.
33. **Payment Service tiered at M+ (1,350 hours).** At 5,091 effective LOC with limited integration complexity and well-established .NET payment processing patterns.
34. **Phase 2/Phase 3 Define leverage team learning for extensions.** Extension services (majority of P2/P3 work) require only delta analysis, not full-package reverse engineering. AI spec extraction is calibrated from Phase 1.
35. **Cloud engineering tapers after Phase 1.** K8s manifest templates, Kustomize overlays, and service mesh routing patterns are proven after Phase 1's 13-service implementation. Subsequent services are copy-modify operations against established templates.
36. **DM absorbs sponsor relationship and commercial governance.** Fixed-scope behavioral translation with no scope negotiations does not require a dedicated Engagement Manager. The Delivery Manager handles sprint governance, UAT coordination, Frontdoor-BA liaison, and sponsor relationship management with a modest uplift.
37. **No Solution Architect role.** Architecture is frozen after Phase 0 ADRs; inter-service communication is mesh-mediated. The App Dev Architect provides day-to-day architectural leadership; Cloud Architect co-owns platform decisions. Cross-cutting concerns are resolved by App Dev Architect + Cloud Architect jointly. Quarterly architecture reviews are conducted by the App Dev Architect.
38. **Design stages compressed to reflect AI-generated artifacts.** OpenAPI specs (~88% AI uplift) and sequence diagrams (~83% AI uplift) are AI-generated with human review and refinement, not manually authored. P2/P3 extensions require delta design only.
39. **Build-stage architect hours reduced for 100% AI output.** AI-generated code follows Phase 0 templates consistently. Architects focus review on the ~15% of code involving complex business logic, not templated CRUD.
40. **Verify stages explicitly preserved as quality gates.** Verify-stage hours — test execution, UAT coordination, defect remediation, behavioral parity testing — are not compressed by the AI code generation model. These are quality gates on a $2B revenue system.
41. **Target wall calendar (~6 months)** uses dual Build pods, phase overlap (Phase 2 starts Phase 1 week 6; Phase 3 starts Phase 2 week 4), and **~2,100 h** acceleration premium while **Verify** remains **3 / 2 / 2** weeks and SOW parity obligations stay intact. End-to-end duration still depends on Frontdoor parallel UAT cohort capacity (§2.5).
42. **Phase §4 line items** sum to the Phase 0 figure in §9. **Gross program labor 20,750 h** (§4–§8) less the **5,730 h** overlap credit yields **~15,020 h** before the **~2,100 h** acceleration premium.
43. **Core Utilities (~70% near-automatic AI translation).** At 13,780 effective LOC, Core Utilities is predominantly string operations, date formatting, validation functions, and assertion frameworks — among the most well-documented patterns for AI translation. Allocated 700 hours (0.05 hrs/LOC).
44. **PCI attestation preparation is Frontdoor's responsibility.** Per the explicit scope exclusion of security/compliance testing. 66degrees provides in-build security engineering for Card Security and Crypto Primitives. AWS KMS key rotation is configured once in Phase 0 with automated rotation policies; ongoing security ops are limited to rotation procedure maintenance and internal security hygiene.

## 13\. Glossary of WBS columns

- **WBS ID:** &lt;phase&gt;.&lt;stage&gt;.&lt;seq&gt; — Phase 0 prefix 0; Phase 1 = 1; Phase 2 = 2; Phase 3 = 3. Stage codes: P Prepare · D Define · S Design · L Plan · V Value Checkpoint · B Build · Y Verify · O Operate.
- **Workstream:** logical bundle (governance / architecture / per-service / cross-cutting).
- **Loc.:** On = NorthAM Onshore · Off = India Offshore.
- **Hrs:** total effort in the row at the indicated location.