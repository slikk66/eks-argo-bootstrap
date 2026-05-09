# AFK worker rules — eks-argo-bootstrap (local additions)

## Public-repo sanitization (HARD invariant)

Before any commit, run:

```bash
grep -RIn 'arn:aws:iam::[0-9]\{12\}\|[0-9]\{12\}\.dkr\.ecr' . --exclude-dir=.git --exclude-dir=.dangeresque
```

Zero matches required. Also re-check for: bare 12-digit numbers, OIDC URLs, VPC/subnet/SG IDs, domain names. Use `{{metadata.annotations.<key>}}` templating from the cluster Secret instead.

If you find a real value that slipped in, **stop, do NOT amend, do NOT push** — file a finding in your run result and let the human rotate the credential before any history is rewritten.

## ApplicationSet contract — one canonical shape

Reference: `bootstrap/applicationsets/cert-manager.yaml`. Every new chart copies this shape. Do NOT introduce alternative ApplicationSet shapes (no Git generator, no list generator, no SCM generator) without an explicit issue + sign-off.

Required fields per ApplicationSet:
- `kind: ApplicationSet`, `namespace: argocd`
- `spec.syncPolicy.preserveResourcesOnDeletion: true`
- Cluster generator with selector `argocd.argoproj.io/secret-type: cluster`
- `template.metadata.annotations.argocd.argoproj.io/sync-wave: "<wave>"`
- Two-source pattern: values-ref (this repo) + Helm chart
- `helm.releaseName: <chart>` explicit (default-derived would break service discovery)
- `helm.valueFiles: [$values/charts/<chart>/values.yaml]`
- `helm.valuesObject` for any `{{metadata.annotations.X}}` templating
- `destination.name: '{{name}}'`, `destination.namespace: <chart-namespace>`
- `syncPolicy.automated: { prune: true, selfHeal: true }`
- `syncOptions: [CreateNamespace=true, ServerSideApply=true]`

## Annotation templating idiom

Use `{{metadata.annotations.<key>}}` to read the 13 cluster Secret annotations. NOT `{{values.<key>}}` (that's the cluster-generator `values:` block, a separate intermediary that is NOT used in this repo unless explicitly chosen). The issue's `{{values.X}}` wording was loose shorthand — the canonical pattern matches AWS Blueprints reference.

## Toleration discipline

Every Deployment / Job in any chart's values that targets `system-ng` MUST tolerate `CriticalAddonsOnly:NoSchedule`. Read the chart's upstream `values.yaml` for the FULL list of toleration keys (controllers, webhooks, jobs, cainjectors, …) before claiming completeness. Missing one toleration silently breaks scheduling — and for Helm post-install Jobs, hangs the install indefinitely.

cert-manager example: FOUR keys needed (`tolerations`, `webhook.tolerations`, `cainjector.tolerations`, `startupapicheck.tolerations`).

## Sync wave convention

See `docs/sync-waves.md`. The registry is the source of truth.
- New chart? Pick a wave from the table; if none fits, propose an addition in your run result, don't unilaterally invent.
- Sync wave on the **Application object** (via `template.metadata.annotations` in the ApplicationSet) is documentation-grade unless an app-of-apps wrapper exists. ArgoCD's retry semantics handle ordering races. Do not preemptively add a wrapper.
- For inside-Application ordering (CRDs before CRs, etc.), put the annotation on the **resource** in the chart values or via Helm post-renderer.

## Pinned versions

`targetRevision` for every Helm chart source MUST be a pinned `v*.*.*` string. NO `*`, `HEAD`, blank, or `latest`. Verify in your run result.

The values-ref source's `targetRevision` may be `HEAD` for development; pin to a tag/SHA before any production cut.

## Repo URL hardcode

`bootstrap/applicationsets/*.yaml` hardcodes `repoURL: https://github.com/slikk66/eks-argo-bootstrap` for the values-ref source. This is intentional — forks search-replace. The eks-pulumi annotation contract is currently 14 keys (was 13; extended to add `vpc_id`). Extending again should be a deliberate cross-repo PR, not a slice-internal decision.

## Deny patterns

Do NOT introduce:
- Hardcoded ARNs (any AWS service)
- Hardcoded account IDs (12-digit sequences)
- Hardcoded OIDC URLs
- Hardcoded VPC / subnet / SG / IGW / NAT / ENI IDs
- `kubectl apply` instructions (this repo is GitOps-only)
- ApplicationSets without `argocd.argoproj.io/sync-wave`
- Helm `targetRevision: '*'`, `HEAD`, or unset
- Alternative ApplicationSet generator types (Git/list/SCM)
- App-of-apps wrappers (preemptive complexity — only add if retry noise bites)

## CI gates (local mirror)

Run before every commit:

```bash
kustomize build bootstrap/ > /tmp/rendered.yaml || exit 1
kubeconform -strict -summary -ignore-missing-schemas \
  -schema-location default \
  -schema-location 'https://raw.githubusercontent.com/datreeio/CRDs-catalog/main/{{.Group}}/{{.ResourceKind}}_{{.ResourceAPIVersion}}.json' \
  -kubernetes-version 1.35.0 /tmp/rendered.yaml || exit 1
grep -RIn 'arn:aws:iam::[0-9]\{12\}\|[0-9]\{12\}\.dkr\.ecr' . --exclude-dir=.git --exclude-dir=.dangeresque && exit 1
```
