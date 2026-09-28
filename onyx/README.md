# Onyx (Community Edition)

Self-hosted knowledge agent over the SharePoint `Knowledge/` library. NAI serves
the chat, embedding, and reranker models; Onyx provides search, chat, agents
(Craft), and the doc-feed automation. Search index is OpenSearch (chart
0.8.38+), not Vespa.

Deployed via Flux like the rest of this repo — commit this directory plus
`../src/onyx.yaml` and `tools-flux.yaml`'s root Kustomization picks it up.
Complete the steps below **before** the first sync, or the release will
install with empty/default secrets.

## 1. Prerequisites (cluster-level, not part of this chart)

- Kubernetes 1.33+ and a default RWO StorageClass. No ingress controller or
  cert-manager needed — `ingress.enabled` stays `false` (chart default) and
  the chart's bundled `nginx` Service (ClusterIP/LoadBalancer, port 80) is
  reached via a Cloudflare Tunnel (`cloudflared`) instead of a public
  Ingress/TLS cert. Point the tunnel's ingress rule at that Service
  (`kubectl get svc -n onyx` to confirm its name) — tunnel config itself
  lives outside this directory.
- A small labelled node pool for Craft sandboxes:

  ```
  kubectl label nodes <node-name> onyx.app/workload=sandbox
  # optional, keeps other pods off the pool:
  kubectl taint nodes <node-name> workload=sandbox:NoSchedule
  ```

  These match the chart's `sandboxPod.nodeSelector` / `tolerations` defaults
  exactly — no values override needed.
- Entra ID app registrations: SharePoint read (client secret, connector),
  SharePoint write (`Sites.Selected`, scoped to one site, for Craft/Graph),
  and the Slack app from Onyx's manifest (pending workspace admin
  approval). SSO (Entra ID OIDC) is deferred for now — see "Open items";
  add an app registration for it when that's revisited.

## 2. Secrets

**Test env only:** credentials are inlined directly under `auth.*.values`
in `helmrelease.yaml` (the chart renders its own Secret from them) instead
of pointing at `existingSecret` — no `kubectl create secret` step needed,
but it means plaintext credentials live in git. Do not do this outside a
disposable environment.

To go back to out-of-band secrets (recommended for anything real): drop
each `auth.<x>.values` block, add `existingSecret: onyx-<x>` instead, and
create the secret yourself, e.g.:

```
kubectl create secret generic onyx-postgresql \
  --namespace onyx \
  --from-literal=username='postgres' \
  --from-literal=password="$(openssl rand -base64 24)"
```

— same idea for `onyx-opensearch` (`opensearch_admin_username`/
`opensearch_admin_password`), `onyx-objectstorage` (`rootUser`/
`rootPassword`/`s3_aws_access_key_id`/`s3_aws_secret_access_key` — MinIO
chart and backend S3 client share this one secret), `onyx-userauth`
(`user_auth_secret`), and `onyx-sandbox-push-secret` (`private_key`, an
Ed25519 PEM from `openssl genpkey -algorithm ed25519`).

Redis auth is left on the chart's generated secret (`auth.redis`) since
Redis is ClusterIP-only within the `onyx` namespace — tighten this too if
that stops being true.

## 3. NAI

Deploy chat, embedding, and reranker endpoints behind the Envoy gateway
(API keys + the guardrail ext_proc). Create **separate API keys** for Onyx
chat vs. Onyx indexing so usage splits cleanly in NAI's audit logs — those
keys get entered in the admin panel (step 4), not here.

If NAI's reranker API isn't Cohere/Jina-shaped (`{model, query, documents}`
→ `results[].index, relevance_score}`) — e.g. NIM's `/v1/ranking` — a small
translation service needs to sit between Onyx and NAI. Not included here;
add it once NAI's actual reranker schema is confirmed (see "Open items").

## 4. Admin panel (after the release is healthy), in order

Auth defaults to Onyx's own email/password login (`auth.userauth` above).
SSO is deferred — see "Open items" — add it later via Organization → SSO
Providers → OIDC provider at `https://login.microsoftonline.com/<tenant>/v2.0`.

1. **LLM** — OpenAI-compatible provider, NAI base URL + chat API key +
   model name, set as default.
2. **Embedding model** — LiteLLM provider type, full NAI
   `/v1/embeddings` URL + model name. Do this before the first index —
   changing it later triggers a full re-index.
3. **Reranking** — LiteLLM provider type, pointed at the NAI reranker (or
   the adapter from step 3 above).
4. **SharePoint connector** — client-secret auth against the site holding
   `Knowledge/`; access Public or restricted to a group (per-doc
   permission sync is a paid feature); keep the 30 min refresh, drop prune
   frequency to daily.
5. **Document set** "Knowledge" from that connector.
6. **Agent** "SharePoint Assistant" — knowledge = Knowledge document set,
   web search off, must cite sources, say "not found" rather than guess.
7. **Chat retention** — set to match company policy.
8. **Personal access tokens** — each MCP user creates their own.
9. **Slack bot** (once approved) — enter tokens, default config →
    SharePoint Assistant, then flip `slackbot.enabled: true` here and
    add per-channel configs.
10. **Craft** — workspace instructions; a custom Graph App
    (`graph.microsoft.com`) using the SharePoint-write app's credentials,
    unattended use allowed; upload the `notes-to-doc` skill (SKILL.md +
    reference.docx + pandoc render script — confirm the sandbox image has
    pandoc); test the prompt in a normal session first; then create the
    hourly Scheduled Task (process `Notes inbox/` → upload to `Drafts/` →
    move source to `processed/`), created from a dedicated service user
    since it runs as its creator.

## Community Edition limits

Per-document permission sync, query history UI, RBAC, and service-account
API keys are paid features. Phase 1 indexes only content every user may
see. For query history, read the chat tables directly in Postgres.

## Open items

- SSO (Entra ID OIDC) — deferred; using Onyx's built-in email/password
  auth for now.
- Slack admin approval.
- NAI's reranker API format (decides whether the adapter in step 3 is
  needed).
- Craft moves fast — the chart version above is pinned deliberately; test
  each upgrade before rolling it out.
