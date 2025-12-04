---
layout: default
title: identifiabl
---

# identifiabl

**user-scoped identity and authentication for llm and agentic apps (apps sdk + mcp)**

```bash
npm i identifiabl
```

> 1-line jwt/jwks verification for mcp and apps sdk backends — without touching app logic
> user → apps sdk/mcp → identifiabl → your backend
>
> ✓ production-tested in [inner](https://innerdreamapp.com)  
> ✓ first [gatewaystack](https://gatewaystack.com) release  

until now, most llm systems have relied on shared api keys — not real user identity.

the shift toward agentic systems, personal data, and enterprise governance requires
every model call to be tied to a verified user, tenant, and scope.

identifiabl defines this layer.

```ts
import { createIdentifiablVerifier } from "identifiabl";

const identifiablVerifier = createIdentifiablVerifier({
  issuer: OAUTH_ISSUER,
  audience: OAUTH_AUDIENCE || "",
  jwksUri: JWKS_URI_FALLBACK,
  source: "auth0",
  scopeClaim: "scope",
  roleClaim: "permissions",
});

const result = await identifiablVerifier(accessToken)
```
→ req.identity is now available everywhere  
→ all requests are user-bound  
→ app logic stays untouched

**production example:**
identifiabl now powers all authentication in the [inner mcp server](https://innerdreamapp.com).
it replaced ~100 lines of custom jose logic with a single verifier call.
read the full before vs. after breakdown [here](https://reducibl.com/2025/12/03/one-line-jwt-jwks-verification-for-mcp-backends.html)

## at a glance

identifiabl is a **user-scoped identity and authentication gateway** for llm apps.  

**identifiabl sits between your app/agent and your model provider, forming the identity layer of gatewaystack.** 

it lets you:

- verify user identity via oidc / apps sdk tokens  
- attach user, org, tenant, and scope metadata  
- enforce that every model call is user-bound  
- emit identity-level audit events  
- normalizes identity into `{ user_id, org_id, tenant, roles, scopes }` with consistent claim mapping across providers  

> 📦 **implementation**: [`identifiabl`](https://github.com/davidcrowe/GatewayStack/tree/main/packages/identifiabl-core)  
> 📦 **npm package**: [`npm i identifiabl`](https://www.npmjs.com/package/identifiabl) (published)

## why now?

as organizations adopt agentic systems and user-specific ai workflows, identity, policy, and governance become mandatory. shared api keys cannot support enterprise-grade access control or compliance.

a new layer is required — **the user-scoped trust and governance gateway**.

it sits between applications (or agents) and model providers, ensuring that **every request is authenticated, authorized, observable, and governed.**

**gatewaystack** defines this layer. **identifiabl** is the linchpin... an authentication module that attaches user-scoped identity to a shared request context for ai model calls.

> ChatGPT Apps SDK and MCP provide verified user identity through signed tokens.
> identifiabl validates these tokens and exposes canonical user/org/tenant metadata to your gateway or agent runtime.

## example use cases

**a healthcare saas** with 10,000+ doctors needs to ensure every ai-assisted diagnosis is tied to the licensed physician who requested it, with full audit trails for hipaa and internal review.

today, most of their ai calls run through a shared openai key:

- the model doesn't know *which* doctor made the request  
- security can't answer "who saw which patient data?"  
- compliance teams can't reconstruct a reliable audit trail  

with identifiabl in front of the model provider:

- each request must carry a verified oidc / sso token  
- `user_id` maps to the specific physician; `org_id` to the clinic or hospital  
- roles/scopes encode permissions (for example, `role:physician`, `scope:diagnosis:write`)  
- every call is logged with identity metadata for audit and incident response  

the result: **every ai diagnosis is user-bound, tenant-aware, and fully auditable**, without the app having to manually thread identity through every model call.

**a global enterprise** rolls out an internal copilot that can search confluence, jira, google drive, and internal apis. employees authenticate with sso (okta / entra / auth0), but today the copilot usually calls the llm and tools with a shared api key.

with identifiabl, every request is bound to a specific employee:

- their user id (`sub`)  
- their org / business unit  
- their roles and permissions  
- their scopes for specific tools and data sources  

this lets security teams enforce "only finance analysts can run this tool", "legal can see these repositories but not those", and keep a full identity-level audit trail for every copilot interaction.

**a multi-tenant saas platform** offers ai features across free, pro, and enterprise tiers. today, most of the ai usage runs through a single openai key per environment — making it impossible to answer basic questions like:

- which customer used which model?  
- which tenant exceeded their budget?  
- which user triggered that problematic generation?  

with identifiabl, each model call is tied to a user and tenant:

- `user_id` and `org_id` are attached to every request  
- scopes reflect plan/tier (for example, `plan:free`, `plan:pro`, `feature:advanced-rag`)  
- downstream modules like `limitabl` and `explicabl` can enforce per-tenant limits and produce human-readable audit trails  

this turns "one big shared key" into **per-tenant, per-user accountability** without changing your app's business logic.

## competitive landscape

today, teams solve this problem in a few different ways:

- **identity providers (auth0, okta, cognito, entra id)**  
  handle login and token minting, but they stop at the edge of your app. they don't understand model calls, tools, or which provider a request is going to — and they don't enforce user identity *inside* the ai gateway.

- **api gateways and service meshes (kong, apigee, api gateway, istio, envoy)**  
  great at path/method-level auth and rate limiting, but they treat llms like any other http backend. they don't normalize user/org/tenant metadata, don't speak apps sdk / mcp, and don't provide a model-centric identity abstraction.

- **cloud ai gateways and guardrails (cloudflare ai gateway, azure openai + api management, vertex, bedrock guardrails)**  
  they focus on provider routing, quota, and safety filters at the **tenant** or **api key** level. user identity is usually out-of-band or left to the application.

- **hand-rolled middleware**  
  many teams glue together jwt validation, headers, and logging inside their app or a thin node/go proxy. it works… until you need to support multiple agents, providers, tenants, and audit/regulatory requirements.

**identifiabl** is different:

- it treats **user identity as the first-class input** to the gateway  
- it is **model- and provider-aware**, designed for apps sdk + mcp  
- it produces **normalized identity context** that all other gatewaystack modules consume (`transformabl`, `validatabl`, `limitabl`, `proxyabl`, `explicabl`)  

you can still run it *alongside* traditional api gateways — identifiabl is the user-scoped identity slice of the stack.

## designing user identity and authentication for ai

identifiabl is responsible for establishing the trust chain at the very start of every model call.

it validates identity tokens, extracts normalized user metadata, and binds that identity to all downstream governance modules.

### within the shared requestcontext

all gatewaystack modules operate on a shared `RequestContext` object.

**identifiabl is responsible for**:

- **reading**: http `Authorization` header, jwks endpoint
- **writing**: `identity` (canonical user/org/tenant/roles/scopes), `trace_id`, identity headers (`x-user-id`, `x-org-id`, `x-tenant`, `x-user-scopes`)

the `identity` object becomes the foundation for all downstream governance decisions in `transformabl`, `validatabl`, `limitabl`, `proxyabl`, and `explicabl`.

## what identifiabl does

- validates oidc / apps sdk identity tokens  
- extracts user, org, tenant, and scopes  
- attaches identity metadata to each request  
- creates a verifiable audit chain  

identity becomes a piece of runtime metadata that other gatewaystack modules rely on — policy checks (`validatabl`), routing (`proxyabl`), cost/budget enforcement (`limitabl`), and full audit trails (`explicabl`) all start from this canonical identity object.

## identifiabl works with

- `transformabl` for preprocess content (see `transformabl`)  
- `validatabl` to evaluate authorization policies (see `validatabl`)  
- `limitabl` to apply rate limits or quotas (see `limitabl`)  
- `proxyabl` to perform provider routing (see `proxyabl`)  
- `explicabl` to store or ship audit logs by itself (see `explicabl`)  

## when not to use identifiabl?

- you don’t need user-level audit trails.
- your app is single-tenant and already uses per-user API keys with minimal governance needs.
- you are not using agents, tools, or MCP.

## under the hood

identifiabl wraps a small, framework-agnostic core verifier:

- validates RS256 JWTs via JWKS
- enforces issuer and audience
- normalize identity into `{ user_id, org_id, tenant, roles, scopes }`

Together, these become a canonical `identity` object shared across
all GatewayStack modules via the `RequestContext`.

if you want to integrate at a lower level, use the core helper directly:

→ [identifiabl on npm](https://www.npmjs.com/package/identifiabl)  
→ [github readme with examples](https://github.com/davidcrowe/gatewaystack/tree/main/packages/identifiabl-core)

## end to end flow
```text
user
   → identifiabl       (who is calling?)
   → transformabl      (prepare, clean, classify, anonymize)
   → validatabl        (is this allowed?)
   → limitabl          (how much can they use? pre-flight constraints)
   → proxyabl          (where does it go? execute)
   → llm provider      (model call)
   → [limitabl]        (deduct actual usage, update quotas/budgets)
   → explicabl         (what happened?)
   → response
```

each module intercepts the request, adds or checks metadata, and guarantees that the call is:

**identified, transformed, validated, constrained, routed, and audited.**

this is the foundation of user-scoped ai.

## integrates with your existing stack

identifiabl plugs into gatewaystack and your existing llm stack without requiring application-level changes. it exposes http middleware and sdk hooks for:

- chatgpt apps sdk  
- model context protocol (mcp)  
- oauth2 / oidc identity providers  
- any llm provider (openai, anthropic, google, internal models)  

## links

want to explore the full gatewaystack architecture?  
→ [view the gatewaystack github repo](https://github.com/davidcrowe/gatewaystack)

want to contact us for enterprise deployments?  
→ [reducibl applied ai studio](https://reducibl.com)

<div class="arch-diagram">
  <div class="arch-row">
    <div class="arch-node">
      <div class="arch-node-title">app / agent</div>
      <div class="arch-node-sub">chat ui · internal tool · agent runtime</div>
    </div>

    <div class="arch-arrow">→</div>

    <div class="arch-node arch-node-gateway">
      <div class="arch-node-title">gatewaystack</div>
      <div class="arch-node-sub">user-scoped trust &amp; governance gateway</div>

      <div class="arch-pill-row">
        <span class="arch-pill">identifiabl</span>
        <span class="arch-pill">transformabl</span>
        <span class="arch-pill">validatabl</span>
        <span class="arch-pill">limitabl</span>
        <span class="arch-pill">proxyabl</span>
        <span class="arch-pill">explicabl</span>
      </div>
    </div>

    <div class="arch-arrow">→</div>

    <div class="arch-node">
      <div class="arch-node-title">llm providers</div>
      <div class="arch-node-sub">openai · anthropic · internal models</div>
    </div>
  </div>

  <p class="arch-caption">
    every request flows from your app through gatewaystack's modules before it reaches an llm provider —
    <strong>identified, transformed, validated, constrained, routed, and audited.</strong>
  </p>
</div>