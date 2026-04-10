# Claude Code Instructions — olympus-gitops

> Full context in `AGENTS.md`. This file covers Claude Code-specific operations.

## Key Flux Commands

```bash
# Force-sync a specific kustomization (after pushing a change)
flux --context management-hub reconcile source git flux-system
flux --context management-hub reconcile kustomization <name>

# Force-sync a HelmRelease
flux --context management-hub reconcile helmrelease <name> -n <namespace>

# Check all kustomization status
kubectl --context management-hub get kustomization -n flux-system
kubectl --context management-hub get helmrelease -A

# compute-hub — use extracted kubeconfig (no direct context in local kubeconfig)
kubectl --context management-hub get secret -n headlamp headlamp-combined-kubeconfig \
  -o jsonpath='{.data.kubeconfig}' | base64 -d > /tmp/compute-hub.yaml
flux --kubeconfig /tmp/compute-hub.yaml --context compute-hub reconcile source git flux-system
```

## Dependency Chain Reference

```
external-secrets
  → cert-manager
    → cert-manager-config
  → external-dns
  → traefik-config
  → cloudflared
  → <apps> (dependsOn varies per app)
```

## Common Patterns

- **New app checklist**: namespace → helmrepository → helmrelease → (externalsecret if secrets needed) → (certificate + ingressroute if public)
- **Secret keys**: ExternalSecret `target.template` renames 1Password field names to what the chart expects
- **valuesFrom**: Use `HelmRelease.spec.valuesFrom` to inject secret values as Helm values
- **HTTP redirect**: Handled globally by Traefik HelmChartConfig — no per-route redirect needed

## Rules

- Do NOT hard-code IPs in manifests — use DNS names or ExternalName services
- Do NOT use `latest` image tags in HelmReleases — pin chart versions
- Always set `dependsOn` in Flux Kustomizations — ordering failures cause cascading reconciliation errors
- Commit plans to `../olympus-sdk/docs/plans/` before starting multi-step changes
