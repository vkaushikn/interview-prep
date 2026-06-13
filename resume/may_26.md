# Kaushik Subramanian

narasimhan.kaushik@gmail.com | 608-698-8136 | LinkedIn: vkaushikn | Snoqualmie, WA

## Infrastructure Capacity Planning & Optimization Engineer

12+ years at Meta, Google, Microsoft, and Amazon building infrastructure capacity planning and resource allocation systems using large-scale constrained optimization, mixed-integer programming, and discrete event simulation. Delivered systems for cycle time reduction, optimal server placement, logistics network redesign, inventory control, and real-time operations optimization.

## EXPERIENCE

### Meta · Bellevue, WA — Jan 2024 – Present
**Capacity & Performance Engineer**

- **Strategic Planning — Engineering Lead:** Led engineering design of the 2–5-year Long Range Plan (LRP) infrastructure planning pipeline, working with ~8 engineers and XFN data science partners. Scenario analysis from this pipeline feeds directly into VP-level capital allocation decisions. Reduced planning cycle time from weeks to days and scaled scenario throughput by an order of magnitude through model automation, a waterfall impact analysis tool, and a plan-validation framework.
- **Unit Cost Model:** Designed and built the model that derives MW-per-unit and server-per-MW cost vectors for ~12 aggregated demand buckets, collapsed from 300+ raw demand lines — used for both power and server capex planning downstream. Prototype→build→operate ownership; serve as business lead for production runs.
- **Datacenter Fit Check:** Designed and directed implementation of the model that validates whether the LRP can physically fit within Meta's datacenter capacity — from model design through production runs. Own the executive-facing plan reviews where results are presented to VP-level planning leadership.
- **Scenario Modeling Platform & Agentic Interface — Engineering Lead:** Technical lead for the Scenario Modeling workstream within Meta's org-wide planning platform initiative. Designed and prototyped a domain-specific language (DSL) that creates a structured audit trail for planning intents previously locked in emails and chats, enabling reusability and agent-driven scenario generation. Demonstrated to VP-level stakeholders; production delivery in progress. Also leading the engineering transition of the core planning model from a Python/spreadsheet hybrid to fully version-controlled Python.
- **Tactical Planning (MRP) — Engineering Lead & Model Owner:** Designed and built the mid-range server planning model that blends the same 300+ demand lines with the LRP across tens of business rules to produce near-term server forecasts, working with a team of 4–6 engineers. Built to absorb upstream model changes without cascading failures. Prototype→build→operate ownership.
- **LP Solver & AI Inference Integration:** Tech-led integration of inference workload regionalization into the unified LP capacity solver, eliminating ~3 engineers of manual per-cycle coordination. Achieved 3× solver speedup (30→10 min) during the project; numerical stability improvements whose methodology informed the team's subsequent solver rewrite.
- **Q4 Fulfillment Sev Response:** Organically led the response to a Q4 Fulfillment Sev — identified the solver modifications needed to redirect capacity, orchestrated implementation across ~10 SWEs, and successfully redirected capacity within the deadline. Recognized in writing by principal-level engineering leadership.

### Google · Seattle, WA — Mar 2018 – Dec 2023
**Data Science Tech Lead**

- **Global Logistics Optimization:** Built MIP warehouse location model to determine the best locations for Google's server supply chain warehouses under transit time, space, and cost constraints. Model provided the analytical backing for a Director-level decision to redesign Google's server logistics network and consolidate warehouse locations. Guided a program manager through this project as their transition to a data science role.
- **TCO & Fleet Costing — Tech Lead, team of 4:** Scaled fleet cost model from ad-hoc SQL/spreadsheets to a production Python API with hardware refresh scheduling (optimization under power, labor, and cost constraints) and granular TCO breakdowns from commodity to server to rack level. Drove cost definition standardization across 4 internal teams; ~70% of ad-hoc requests handled by PM self-service.
- **Network Supply Chain:** Prototyped and productionalized network gear buffer model determining optimal inventory levels across 70+ metros. Built headcount planning model for network operations across the same 70+ metro footprint.
- **Industrial Robotics — Financial & Investment Modeling:** Built financial and discrete event simulation models to evaluate Google's industrial robotics portfolio investments. Analysis funded the investment decision for a hard disk repair robot. Managed and promoted one engineer from early to mid-level.

### Microsoft · Redmond, WA — Aug 2016 – Mar 2018
**Data Scientist & Program Manager**

PM for Microsoft's multi-echelon inventory optimization program while building stochastic models to improve it.

- **Multi-Echelon Inventory Optimization Program:** Ran the monthly S&OP planning cycle setting optimal inventory placement targets across Microsoft's supply chain. Reduced cycle time from 1.5 weeks to 2 days through process improvements, engineering of the planning tool, and scenario management and waterfall analysis tooling. Managed the planning cadence across teams — maintaining the internal clock and handoffs for each planning stage.
- **Stochastic MILP for Server Allocation:** Built a stochastic MILP prototype (Gurobi/Python) to improve the inventory program — modeling optimal server docking decisions under demand and supply uncertainty to meet service level requirements.

### Amazon · Seattle, WA — Jan 2013 – Jul 2016
**Operations Research Scientist**

Sole OR/DS engineer on both prototypes, working closely with SWEs toward productionalization.

- **Fulfillment Center Scheduling & Control:** Built combined MILP scheduling and dynamic control system (Xpress/Java) for Amazon fulfillment centers. Rolling-horizon shift scheduler assigned labor across Pick, Sort, and Pack to maximize throughput while meeting ship deadlines; dynamic control module continuously adjusted operations to track the plan. Launched in 3 FCs; subsequently adopted as the standard system for all new FC launches.
- **Transportation Resource Pricing:** Built dynamic pricing prototype minimizing transportation cost via LP duality applied to historical hindsight-optimal solutions. Discrete event simulation replaying network traffic demonstrated savings in the billions of dollars.

## EDUCATION

**Ph.D., Chemical Engineering** · University of Wisconsin – Madison — 2007 – 2012
Thesis: Integration of control theory and scheduling methods for supply chain management · Minor: Computer Sciences
Best Paper Award, Computers & Chemical Engineering (2012) — presented as plenary talk, FOCAPO/CPC VIII Joint Session, Savannah, GA

**M.Tech., Chemical Engineering** · Indian Institute of Technology, Bombay — 2005 – 2007

**B.E., Chemical Engineering** · R.V. College of Engineering (VTU), Bangalore — 2001 – 2005
