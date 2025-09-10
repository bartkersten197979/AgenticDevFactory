1) Harde limieten (relevant voor je instructies)

Modelcontext: houd rekening met het totale contextbudget (instructies + tools + history + inputs).

Per bericht: praktisch ruim; truncatie ontstaat meestal door je eigen prompt‑budget, niet door een hard 64k‑limiet zoals bij oudere modellen.

Tools per agent: max. ~128. Splits grote sets in logische bundles.

Berichten per thread: ~100.000. Ruim genoeg; archiveer lange runs.

OpenAPI/tool‑definitie‑grootte: grote specs splitsen (praktijk: >~300k tokens kan falen). Registreer tools los; niet in de system message embedden.

Token vuistregel: tokens ≈ karakters/4 (EN). NL/DE zijn vaak iets ongunstiger (meer tokens per woord).

Praktisch advies: mik voor de system‑instructies op ~2–4k karakters (≈500–1.000 tokens) zodat er ruime marge overblijft voor tools & history.

2) Praktische richtlijnen (efficiëntie & kosten)

Compacte system message: 2–4k karakters; geen dubbele secties/versies.

Structuur: maximaal 7 kopjes: Role · Hard rules (MUST/NEVER) · Inputs · Outputs · Tool‑use & timeouts · Failure policy · Limits.

Verplaats bulk (lange voorbeelden, schema’s, CI‑stappen) naar referentiebestanden (repo/attachments/vector store) en verwijs er met 1 regel naar.

Één taal (NL of EN) om tokeninflatie te voorkomen.

Codefences: niet nesten; per voorbeeld één correct afgesloten blok. JSON altijd valide en apart.

Tools: niet inline definities in je system message. Registreer OpenAPI/MCP tools los en verwijs er alleen naar.

Timeout‑patroon: sync tool‑call kort; voor >~30s gebruik async job + polling.

3) Aanbevolen target‑format (ultrakorte mal)

Titel & versieCodingAgent — Instructions (vX.Y.Z, YYYY‑MM‑DD)

Summary (≤ 600–900 karakters)1–3 zinnen: rol, scope, harde grenzen (geen prod‑wijzigingen, least‑privilege), outputvorm (ZIP of PR) en metadata‑eisen.

Hard rules (5–9 bullets, 1 regel per bullet)

MUST: …

MUST: …

NEVER: …

Inputs (3–7 bullets)

Vereist: repo, target‑branch, author, generation_id, artifact/source tree.

Optioneel: tests, lint config, build script.

Outputs (3–7 bullets)

generation.json (conform schema) + release_note.md.

Artifact (ZIP) of PR/branch + korte metadata.

Tool‑use & timeouts (≤ 4 bullets)

Tools via geregistreerde Connections; geen API‑keys in prompt.

Sync ≤ ~30s; langer ⇒ async job + polling ID.

Failure policy (3 bullets)

Triviaal: auto‑patch + nieuwe generation_id (child).

Niet‑triviaal: open review‑issue + payload.

Altijd traceerbare metadata (parent/child links).

Limits

Verwijs naar de Quotas & Limits pagina van de dienst + interne max‑groottes voor tools/specs.

4) Referentie‑URL’s (naslag)

Azure AI Foundry — Agents · Quotas & Limitshttps://learn.microsoft.com/en-us/azure/ai-foundry/agents/quotas-limits

Tools & Connections (OpenAPI)https://learn.microsoft.com/en-us/azure/ai-foundry/agents/how-to/tools/openapi-spechttps://learn.microsoft.com/en-us/azure/ai-foundry/agents/how-to/tools/openapi-spec-sampleshttps://learn.microsoft.com/en-us/azure/ai-foundry/how-to/connections-addhttps://learn.microsoft.com/en-us/azure/ai-foundry/agents/how-to/tools/overview

Evaluations & Observabilityhttps://learn.microsoft.com/en-us/azure/ai-foundry/how-to/develop/cloud-evaluationhttps://learn.microsoft.com/en-us/azure/ai-foundry/how-to/develop/evaluate-sdkhttps://learn.microsoft.com/en-us/azure/ai-foundry/concepts/observability

Content filters / Guardrailshttps://learn.microsoft.com/en-us/azure/ai-foundry/openai/how-to/content-filtershttps://learn.microsoft.com/en-us/azure/ai-foundry/concepts/content-filteringhttps://learn.microsoft.com/en-us/azure/ai-foundry/foundry-models/how-to/configure-content-filters

DevOps toegang (service connection / managed identity)https://learn.microsoft.com/en-us/azure/devops/pipelines/library/connect-to-azure?view=azure-devopshttps://learn.microsoft.com/en-us/azure/devops/integrate/get-started/authentication/service-principal-managed-identity?view=azure-devops

SAS (user‑delegation & best practices)https://learn.microsoft.com/en-us/rest/api/storageservices/create-user-delegation-sashttps://learn.microsoft.com/en-us/azure/storage/common/storage-sas-overviewhttps://msrc.microsoft.com/blog/2023/09/microsoft-mitigated-exposure-of-internal-information-in-a-storage-account-due-to-overly-permissive-sas-token/https://learn.microsoft.com/en-us/security/benchmark/azure/baselines/storage-security-baseline

EU Data Boundary & Azure OpenAI privacyhttps://learn.microsoft.com/en-us/privacy/eudb/eu-data-boundary-learnhttps://learn.microsoft.com/en-us/privacy/eudb/change-loghttps://learn.microsoft.com/en-us/azure/ai-foundry/responsible-ai/openai/data-privacy

Netwerkisolatie (Private Endpoints / Managed Network)https://learn.microsoft.com/en-us/azure/ai-foundry/openai/how-to/networkhttps://learn.microsoft.com/en-us/azure/ai-foundry/how-to/configure-managed-networkhttps://learn.microsoft.com/en-us/azure/ai-foundry/how-to/configure-private-linkhttps://learn.microsoft.com/en-us/azure/private-link/private-endpoint-dnshttps://learn.microsoft.com/en-us/azure/ai-foundry/agents/how-to/virtual-networks

Supply chain (SBOM / attestations)https://devblogs.microsoft.com/engineering-at-microsoft/microsoft-open-sources-software-bill-of-materials-sbom-generation-tool/https://marketplace.visualstudio.com/items?itemName=rhyskoedijk.sbom-tool

Timeouts (praktijk & tool‑specifiek)https://learn.microsoft.com/en-us/answers/questions/5523387/azure-ai-agent-service-connected-agent-timeouthttps://learn.microsoft.com/en-us/answers/questions/4379527/azure-ai-foundry-agents-playground-408-timeout-errhttps://learn.microsoft.com/en-us/azure/ai-foundry/agents/how-to/tools/code-interpreter

