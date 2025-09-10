Agentic Dev Factory — Interaction Overview (v0.1)
Goal: From user prompt → working Azure resources + app via Azure DevOps, reliably and idempotently — application first & seamless in Azure. Coordinator always elicits an application name and high level purpose; each app gets its own Resource Group named exactly as the app.
1) Scope & Non Goals
•	Scope: Describe how agents collaborate; define inputs/outputs per agent; outline the happy path.
•	POC focus: Webapps only (each user request results in a simple functional webapp). Backend technicalities (region, networking, tags, security, etc.) are auto handled by agents using defaults.
•	Non Goals: Deep implementation details, full schemas, or security runbooks (to be added later).
1b) POC Environment Defaults
•	Subscription: Azure subscription 1 (861c0915-7f88-43e1-8424-971a8bae2c42).
•	Main RG (Agentic Dev Factory shared): rg-BartKersten-0416.
•	Key Vault (Main RG): DefaultVaultAgentSetup — URL: https://defaultvaultagentsetup.vault.azure.net/.
o	Policy: All factory-level secrets/IDs/config are stored and read here.
•	API Center (Main RG): Lives alongside the Key Vault in the Main RG.
2) Primary Actors
•	Coordinator (orchestrator)
o	Purpose: Owns the end to end flow. Translates user intent to a deployable plan. Calls other agents/tools.
o	Always asks the user:
	Application name (used 1:1 as the Resource Group name for that app)
	Global purpose of the app (what it should roughly do)
	Process description: source(s) → action(s) → output(s) (e.g., “upload invoices → extract fields → propose booking”).
	Expected monthly usage (numeric + unit, e.g., 2000 invoices/month, 500 uploads/month).
o	Does NOT ask: technical backend choices (region, networking, tags, security, runtime, etc.) — these are decided by agents.
o	Pre flight completeness gate: Coordinator must collect/resolve all required functional info before hand off to CodingAgent. If anything is missing, Coordinator derives/decides it (consulting NamingAgent, ResourceAgent, or others) and documents the assumption.
o	Decision envelope: Defines small, non critical decisions that CodingAgent may take autonomously (see §5c) and sets constraints (no security/compliance impact; cost impact ≤ small; no new resource types).
o	No guess rule (CodingAgent): CodingAgent must not invent missing inputs. It may ask clarifying questions back to the Coordinator and wait until the Coordinator resolves them.
o	Resource & tier selection (POC): Coordinator selects the minimal resource set and a pricing tier; default Free/Flex Consumption unless required capabilities are missing, in which case Coordinator chooses the lowest viable tier and notes the rationale.
o	Key inputs: User prompt + constraints (region policy, tags, environments).
o	Key outputs: Concept summary of the app + monthly cost estimate (idle/low/normal/high) for user approval; execution plan; assignments to agents; consolidated evidence; ensures RG <appName> exists/verified before deployment.
•	CodingAgent (IaC + app scaffolding)
o	Purpose: Generates/updates Infrastructure as Code (Bicep/Terraform) and minimal app scaffolds/tests.
o	Within decision envelope (§5c): May choose minor implementation details (e.g., folder layout nuances, test harness structure, pagination size, retry delays within a safe range, function route names) so long as cost/security are unaffected and no new resources/tiers are introduced.
o	Must escalate: Any choice that affects cost (beyond “small”), security/compliance, resource types, pricing tier, or external system contracts.
o	Key inputs: Desired architecture (from Coordinator), naming & tags (via NamingAgent), resource reality (via ResourceAgent), decision envelope (from Coordinator).
o	Key outputs: PR to Repo with infra/main.bicep (or Terraform), parameters, working app code (Functions/Logic App workflows/triggers), smoke tests, and a decision_log.md documenting micro decisions taken under the envelope.
•	DeploymentAgent (DevOps runner)
o	Purpose: Executes a single standard Azure DevOps pipeline to validate → deploy → smoke. Idempotent create if not exists; retries/backoff.
o	Key inputs: Repo ref/commit, pipeline id, environment, parameters, region policy.
o	Key outputs: Run id/URL, logs, deployment evidence, health check results.
3) Supporting Actors (on demand)
•	NamingAgent: Supplies compliant names + standard tags for all resources (global uniqueness, Azure constraints).
•	ResourceAgent: Reports current state (subscriptions, RGs, resources) for diff/plan.
•	RepoAgent: Standardizes repo layout, branch/PR policies; can create repos if missing.
•	DevOpsAgent: Creates/connects pipelines, service connections, and variables/secrets (no secrets echoed).
•	ExactAgent: Domain knowledge + API guidance (Exact Online) used when the solution needs it.
•	AzureAIFoundryExpert: Product/feature guidance for Azure AI Foundry.
•	MetaAgent: Inactive by default (currently does nothing). Optional, read only advisor when enabled; can critique plans and suggest improvements, but never applies changes automatically.
4) Guardrails & Policies
•	Application first & RG strategy: For every requested application, create/use a dedicated Resource Group named exactly as the application (e.g., InvoiceReconcile). All app resources must be deployed into this RG.
•	Coordinator responsibility: Must ask for appName and purpose, validate/ensure the RG exists before planning/deploying.
•	Shared (hybrid) resources: Only Key Vault DefaultVaultAgentSetup and API Center live in Main RG rg-BartKersten-0416. All other resources for user apps live in their app RG = <appName>. Factory-level secrets/IDs/config are stored in https://defaultvaultagentsetup.vault.azure.net/.
•	Key Vault policy: Never create per app vaults. Always use DefaultVaultAgentSetup; secrets are stored as Key Vault references and must be tagged app=<appName> (plus standard tags) for traceability.
•	Repository per app: Every application gets its own Azure DevOps repository named exactly <appName>. RepoAgent must create if not exists, set default branch main, and seed the golden layout (/infra, /app, /tests, /.azure-pipelines/deploy.yml).
•	Single resource requests: If the user asks for a standalone resource, Coordinator clarifies whether it belongs to an app. If yes → place in that app’s RG. If truly standalone → create a small dedicated RG named for the task/resource.
•	Working App rule: Infrastructure must ship with runnable functionality. For compute/workflow resources (Function App, Logic Apps, Web App, etc.) the code/workflows/triggers are included and configured (bindings, schedules, webhook/event subscriptions) in the same PR and deployment. No empty shells.
•	Region policy (EU): Primary Germany West Central, fallback France Central, else EU mainland supporting the workload.
•	Idempotency: Always create if not exists, else patch/validate. No duplicate resources.
•	Standard tags: Provided by NamingAgent and applied to every resource (owner, env, cost center, ai_*…). NamingAgent must respect the RG name lock = appName.
•	Completeness gate: Coordinator resolves all open questions before handing to CodingAgent; CodingAgent must never guess and should ask back through Coordinator.
•	Tier selection policy (POC): Default Free/Flex Consumption; escalate to higher tiers only if required features are unmet; record rationale and cost impact.
•	Security defaults (POC): System-assigned Managed Identity = ON for compute; secrets via Key Vault references only (never inline); RBAC scoped minimally to app RG.
•	Observability defaults (POC): /healthz endpoint required; Application Insights enabled (retention 7 days); telemetry key via KV reference or MI.
•	Pipelines: One reusable pipeline runs validate → deploy → smoke, then publishes evidence back to Coordinator.
5) Happy Path (Prompt → Production)
1.	User → Coordinator: Provide initial idea. Coordinator asks app name, purpose, process description (source → action → output), and expected monthly usage (numeric + unit).
2.	Coordinator → User (gate): Drafts Concept: <appName> (functional summary) + Monthly Cost Estimate (idle / low / normal / high) and asks for approval.
3.	If approved: Coordinator ensures/creates RG = <appName>, then requests NamingAgent (names/tags; RG name lock) and ResourceAgent (current state for diff). Coordinator also ensures/creates Azure DevOps Repo = <appName> via RepoAgent (create if not exists, branch main, seed layout).
4.	Coordinator runs the completeness gate (all inputs resolved) and selects minimal resources + consumption tier (default Free/Flex unless features require higher). Then assigns CodingAgent to produce/update IaC + working app code (Functions/Logic App workflows) + configured triggers/bindings + smoke tests in Repo (PR → merge).
5.	Coordinator triggers DeploymentAgent with: repo commit, pipeline id, parameters, region, environment.
6.	DeploymentAgent runs DevOps pipeline: validate templates → deploy idempotently → run smoke tests.
7.	DeploymentAgent returns run id, logs, and evidence (URIs, outputs, health endpoints) to Coordinator.
8.	Coordinator summarizes results to the user; if gaps/failures, enters the feedback loop with CodingAgent and re-runs CI/CD via DeploymentAgent.
5a) POC Mode — Coordinator Interview (functional only)
Ask only functional items (no backend/infra details):
•	App name (becomes the Resource Group name)
•	Purpose (what problem does it solve?)
•	Process description (sources → actions → outputs)
•	Target user(s) (who will use it?)
•	Primary tasks (top 1–3 actions the user will perform)
•	Inputs (what data comes in? upload/file/API name)
•	Outputs (what must the app show/produce? e.g., a list, a CSV export, a booking suggestion)
•	Expected monthly usage → number + unit (e.g., 2,000 invoices/month or 500 uploads/month)
•	Sample data available? (yes/no; where)
•	External system names (functional: e.g., “Exact Online” if relevant)
•	Definition of Done (1–2 sentences for demo acceptance)
5b) Cost Estimation (POC)
Coordinator must produce a monthly cost indication before build/deploy:
•	Four figures:
o	Idle (0 usage): fixed/base hosting & required platform services for a minimal webapp.
o	Low / Normal / High usage: scale from the user’s expected monthly usage using default multipliers: low = 0.25×, normal = 1.0×, high = 3.0× (adjustable if the user provides explicit tiers).
•	Method: Use EU region defaults and a ratecard/pricing source when available; otherwise fall back to a transparent heuristic (show units, assumed rates, and which components are fixed vs usage based).
•	Scope (POC): web hosting, serverless executions/storage, basic data storage, and any per use AI/API calls if the process description implies them.
•	Output: A concise “Concept + Costs (€/mo)” card with Idle/Low/Normal/High totals and an assumptions list.
•	Gate: No deployment until the user approves the concept + cost indication.
5c) Decision Delegation (POC)
•	Goal: Keep Coordinator focused on the big picture while allowing CodingAgent to make small, non critical decisions safely.
•	Coordinator defines:
o	decisionEnvelope.allowed: examples → tests layout; pagination size (default 50); retry/backoff within [100ms..5s]; function route names; minor parameter defaults; CORS = none; upload limit = 10 MB; storage redundancy = LRS; diagnostics retention = 7 days
o	decisionEnvelope.constraints: noNewResources=true, tierChange=false, costImpact<=small, securityImpact=none.
o	decisionBudget: optional count of micro decisions before a checkpoint (e.g., 5).
•	CodingAgent must:
o	Act within the envelope; record each micro decision in decision_log.md with rationale.
o	Escalate to Coordinator when a choice breaches constraints or impacts cost/security.
•	Coordinator reviews: decision log during PR; approves or requests changes.
5e) Coordinator Prompt Template (copy paste)
App name:
Purpose (1–2 lines):
Process (sources → actions → outputs):
Expected monthly usage (number + unit):
Target users & primary tasks (1–3):
Definition of Done (2 lines):
5f) Health Endpoints — Minimal Templates (POC)
Goal: simple HTTP 200 at /healthz with a tiny JSON body. No auth. Fast. No external calls.
A) Web App — Node (Express)
import express from 'express';
const app = express();
app.get('/healthz', (req, res) => {
  res.json({ status: 'ok', ts: new Date().toISOString(), version: process.env.APP_VERSION || 'dev' });
});
app.listen(process.env.PORT || 8080);
B) Web App — .NET 8 Minimal API (Program.cs)
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();
app.MapGet("/healthz", () => Results.Json(new {
    status = "ok",
    ts = DateTime.UtcNow,
    version = Environment.GetEnvironmentVariable("APP_VERSION") ?? "dev"
}));
app.Run();
C) Azure Function — .NET 8 Isolated
Functions/Health.cs
using System.Net;
using Microsoft.Azure.Functions.Worker;
using Microsoft.Azure.Functions.Worker.Http;

public class Health
{
    [Function("Health")] 
    public HttpResponseData Run([HttpTrigger(AuthorizationLevel.Anonymous, "get", Route = "healthz")] HttpRequestData req)
    {
        var res = req.CreateResponse(HttpStatusCode.OK);
        res.Headers.Add("Content-Type", "application/json");
        var version = Environment.GetEnvironmentVariable("APP_VERSION") ?? "dev";
        res.WriteString($"{{\"status\":\"ok\",\"ts\":\"{DateTime.UtcNow:o}\",\"version\":\"{version}\"}}
");
        return res;
    }
}
host.json (optional: make Functions route /healthz instead of /api/healthz)
{
  "extensions": { "http": { "routePrefix": "" } }
}
D) Azure Function — Node v4
functions/healthz/function.json
{
  "bindings": [
    { "authLevel": "anonymous", "type": "httpTrigger", "direction": "in", "name": "req", "methods": ["get"], "route": "healthz" },
    { "type": "http", "direction": "out", "name": "res" }
  ]
}
functions/healthz/index.js
module.exports = async function (context, req) {
  context.res = {
    headers: { 'Content-Type': 'application/json' },
    body: { status: 'ok', ts: new Date().toISOString(), version: process.env.APP_VERSION || 'dev' }
  };
};
Smoke test expectation: CI/CD pings GET /healthz and expects HTTP 200 with { status: "ok", ... }. Keep it dependency free.
5d) Decision Table — Coordinator vs CodingAgent (POC)
•	Goal: Keep Coordinator focused on the big picture while allowing CodingAgent to make small, non critical decisions safely.
•	Coordinator defines:
o	decisionEnvelope.allowed: examples → tests layout; pagination size (default 50); retry/backoff within [100ms..5s]; function route names; minor parameter defaults; CORS = none; upload limit = 10 MB; storage redundancy = LRS; diagnostics retention = 7 days
o	decisionEnvelope.constraints: noNewResources=true, tierChange=false, costImpact<=small, securityImpact=none.
o	decisionBudget: optional count of micro decisions before a checkpoint (e.g., 5).
•	CodingAgent must:
o	Act within the envelope; record each micro decision in decision_log.md with rationale.
o	Escalate to Coordinator when a choice breaches constraints or impacts cost/security.
•	Coordinator reviews: decision log during PR; approves or requests changes.
5d) Decision Table — Coordinator vs CodingAgent (POC)
Decision item	Default for POC	Who decides	Escalation triggers	Notes
App / RG name	<appName> (RG = appName)	Coordinator	—	RG is created/verified before plan.
Repo name & creation	Azure DevOps repo named <appName>	RepoAgent (policy by Coordinator)	Name collision; repo exists with different purpose	Create-if-not-exists; default branch main; seed golden layout.
Resource set	Webapp + Storage (+ Function if needed)	Coordinator	New resource types required	Keep minimal for POC.
Pricing tier	Free / Flex Consumption	Coordinator	Feature not in tier → higher tier	Rationale recorded in plan.md.
Region choice	EU policy → Germany West Central → France Central	Agents (per policy)	Data residency override	Not asked to user; automatic.
Naming	From NamingAgent	NamingAgent	—	RG name lock = appName.
Tags	From NamingAgent	NamingAgent	—	Standard tags applied everywhere.
Runtime stack	Node LTS / .NET 8 (serverless-friendly)	CodingAgent	Non-default runtime/perf demands	Within decision envelope.
Routes & health	/api/*, /healthz	CodingAgent	Breaking API contract	Within envelope.
Pagination default	50 items	CodingAgent	Explicit UX requirement	Within envelope.
Retry/backoff	100ms–5s	CodingAgent	Integration SLA requires change	Within envelope.
Storage redundancy	LRS	CodingAgent	GRS/ZRS requested	Escalate (cost/DR impact).
Storage perf tier	Standard	CodingAgent	Premium needed	Escalate (tier change/cost).
CORS	None by default	CodingAgent	Cross-domain required	Escalate with allowed origins list.
Diagnostics retention	7 days	CodingAgent	Legal/compliance retention	Escalate (cost/log volume).
Key Vault usage	Use Main KV DefaultVaultAgentSetup only; no new vaults	Policy (Coordinator)	HSM isolation/regulatory need → explicit approval	Tag secrets app=<appName> (+ standard tags).
App settings & secrets	Key Vault references only	CodingAgent	Plaintext secrets	Never allowed; use KV; reference & tag properly.
Upload limit	10 MB	CodingAgent	Larger files needed	May affect tier/storage; escalate.
Cost estimate	Idle/Low/Normal/High €/mo	Coordinator	—	Approval gate before build.
External systems	Names only (e.g., “Exact Online”)	Coordinator	Credentials/contracted APIs	Coordinator must approve scope.
Rule of thumb: CodingAgent may decide small, non critical details within the envelope (no new resources/tiers, cost impact ≤ small, no security/compliance impact) and must record them in decision_log.md. This table is not exhaustive; other minor decisions are allowed if they fit the envelope and are logged.
6) Minimal Interaction Contract (illustrative)
message.flow:
  - from: User
    to: Coordinator
    intent: "High-level goal + constraints"
  - from: Coordinator
    to: User
    ask:
      appName: "Required"
      purpose: "Required"
      process: "Required (sources → actions → outputs)"
      expectedUsagePerMonth:
        value: "number"
        unit: "e.g., invoices, uploads"
  - from: Coordinator
    to: User
    content: "Concept summary + Monthly cost estimate (idle/low/normal/high)"
    ask: { approve: "yes/no" }
  - from: User
    to: Coordinator
    approve: true
  - from: Coordinator
    to: ResourceAgent
    ask: "current state for diff"
  - from: Coordinator
    to: NamingAgent
    ask: "names+tags for resources (RG name is locked to appName)"
  - from: Coordinator
    to: RepoAgent
    action: "ensure repo exists"
    payload: { name: "<appName>", branch: "main", seedGoldenLayout: true }
  - from: Coordinator
    to: CodingAgent
    payload:
      resolvedInputs: { appName, purpose, process, expectedUsagePerMonth }
      plan:
        resources: [ "webapp", "storage", "function (if needed)", "ai/service (if implied)" ]
        tier: "free|flex|other"
      decisionEnvelope:
        allowed: [ "tests layout", "pagination size (50)", "retry delays 100ms..5s", "function routes", "minor parameter defaults", "CORS none", "upload limit 10MB", "storage LRS", "diagnostics retention 7d" ]
        constraints: { noNewResources: true, tierChange: false, costImpact: "<=small", securityImpact: "none" }
        decisionBudget: 5
      deliverables: "PR with infra/main.bicep, params, app scaffold, smoke tests, decision_log.md"
  - from: CodingAgent
    to: Coordinator
    ask: "clarifications (if any inputs are ambiguous or missing)"
  - from: Coordinator
    to: DeploymentAgent
    payload:
      appName: <string>
      resourceGroup: <appName>
      pipelineId: <id>
      repo: <org/project>/<appName>@<commit>
      environment: <dev|test|prod>
      regionPolicy: [germanywestcentral, francecentral, eu-mainland]
  - from: DeploymentAgent
    to: Coordinator
    result: { runId: <url>, status: "Succeeded", evidence: {...} }
  # Failure feedback loop (CI/CD)
  - from: DeploymentAgent
    to: Coordinator
    result: { runId: <url>, status: "Failed", logsUri: <uri>, evidence: {...}, failedStep: <validate|deploy|smoke> }
  - from: Coordinator
    to: CodingAgent
    action: "Analyze logs and patch IaC/app accordingly"
  - from: Coordinator
    to: DeploymentAgent
    action: "Re-run pipeline with new commit (retry)"
6b) Failure Feedback Loop (CI/CD)
•	On failure in validate/deploy/smoke:
1.	DeploymentAgent posts logs + errors + failed step to Coordinator.
2.	Coordinator summarizes and assigns CodingAgent to patch IaC/app.
3.	After PR merge, Coordinator re-triggers DeploymentAgent (CI/CD retry).
4.	Repeat until Succeeded or abort per policy (max attempts/backoff).
•	Signals & artifacts: status, runId, logsUri, evidence are always returned.
6c) Smoke Tests — Minimum (POC)
•	Webapp root (/) returns HTTP 200 and /healthz returns HTTP 200.
•	At least one Function trigger runs successfully (HTTP or Timer) and is visible in logs.
•	Logic App (if present) completes one run; link included in evidence.
6d) Rollback & Cleanup (POC)
•	After 2 failed retries, Coordinator commands cleanup of partial app resources in RG <appName> (Main RG unaffected). A summary of removed resources is appended to evidence.
7) Repo Layout (golden path)
/infra
  └─ main.bicep (or main.tf)
/app
  ├─ src/...                 # minimal working code
  ├─ functions/*             # Azure Functions, bindings, host.json
  └─ workflows/*             # Logic App workflow JSON, connection refs
/tests
  └─ smoke/*                 # post-deploy checks (HTTP /healthz, function invoke, logicapp run)
/.azure-pipelines
  └─ deploy.yml              # build → validate → deploy → smoke
8) Evidence & Outputs
•	IDs/URIs: resourceIds, endpoints, pipeline run URL.
•	Artifacts: build/deploy logs, smoke results, decision_log.md.
•	Functional checks: list of deployed functions/endpoints, trigger binding status, Logic App run link or history.
•	Costing: cost_estimate.json (idle/low/normal/high), assumptions.md (rates, SKUs used, usage basis), and the Concept summary presented to the user.
•	Plan & Tier: Selected resource set and consumption tier with rationale (plan.md).
•	State file: final parameters + applied tags (for provenance).
8a) cost_estimate.json — schema (POC)
{
  "appName": "<string>",
  "basisUsage": { "value": <number>, "unit": "invoices|uploads|..." },
  "tiers": {
    "idle": { "totalEur": <number>, "components": [ { "name": "webapp", "eur": <number> } ] },
    "low":  { "totalEur": <number>, "components": [ { "name": "functions", "eur": <number> } ] },
    "normal":{ "totalEur": <number>, "components": [ { "name": "storage", "eur": <number> } ] },
    "high": { "totalEur": <number>, "components": [ { "name": "ai_calls", "eur": <number> } ] }
  },
  "assumptions": [ "EU pricing defaults", "low=0.25x, normal=1x, high=3x of basisUsage" ]
}
9) Open Items / To Do
•	Fill in pipelineId, org/project/repo, and standard tag set.
•	Add error handling patterns (retry/backoff matrix).
•	Expand ExactAgent integration points (if this solution needs ERP features).
________________________________________
Appendix A — Simple ASCII Flow
User → Coordinator → (NamingAgent, ResourceAgent) → CodingAgent → Repo → DeploymentAgent → DevOps → Azure
(Coordinator ensures RG=<appName>)

[Clarifications]
CodingAgent ⇄ Coordinator  (rare: only if functional info is missing or ambiguous)
Coordinator ⇄ (NamingAgent / ResourceAgent / other domain agents)  (re-ask if prior answer is unsatisfactory)

[Failure path]
DeploymentAgent/DevOps ── logs+errors ──> Coordinator ── fix request ──> CodingAgent ── PR ──> DeploymentAgent → DevOps (retry)

