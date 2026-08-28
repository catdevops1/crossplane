# Cloudflare as Code with Crossplane

Cloudflare DNS managed as Kubernetes resources on a bare-metal cluster.
Argo CD syncs this repo, Crossplane reconciles it against the Cloudflare API.

Every record lives in Git. Change one in the dashboard and Crossplane puts it back.

## What's here

```
providers/
  provider-cloudflare.yaml   # the Crossplane Provider package
  provider-config.yaml       # wires the Vault-sourced token to the provider

cloudflare/
  dns-records.yaml           # 13 records: tunnel CNAMEs, MX, DKIM, SPF, DMARC
  access-argocd.yaml         # planned - Zero Trust Access as code
  tunnel.yaml                # planned - tunnel entity
  tunnel-config.yaml         # planned - tunnel ingress rules
```

The Argo CD applications that deploy this repo live in
[k8s-config-pub](https://github.com/catdevops1/k8s-config-pub) under
`argocd/applications/`.

## Stack

| Component | Version |
|---|---|
| Crossplane | v2.3.2 |
| provider-cloudflare-dns | v0.1.3 (wildbitca) |
| Argo CD | v3.1.4 |
| Kubernetes | v1.35.0 (kubeadm, bare metal) |

## Credentials

The API token never touches Git. It lives in Vault and reaches the provider
through External Secrets Operator:

```
Vault -> ExternalSecret -> Kubernetes Secret -> ProviderConfig
```

The provider expects JSON, not a bare string:

```bash
vault kv put secret/cloudflare/credentials \
  credentials='{"api_token":"YOUR_TOKEN"}'
```

Token scopes:

- Zone > DNS > Edit
- Zone > Zone > Read
- Account > Access: Apps and Policies > Edit

## Deployment order

Three separate Argo CD applications, each waiting on the previous:

```
crossplane              Helm chart, Crossplane core
crossplane-cloudflare   providers/  - Provider + ProviderConfig
crossplane-dns          cloudflare/ - DNS records
```

One app pointing at the repo root does not work. Argo CD tries to apply the
ProviderConfig before its CRD exists, and changing an existing app's path
makes it lose track of resources it already manages and prune them.

Required sync options on the provider app:

```yaml
syncOptions:
  - ServerSideApply=true
  - SkipDryRunOnMissingResource=true
```

`ServerSideApply` is needed because provider CRDs exceed the annotation size
limit. `SkipDryRunOnMissingResource` is needed because the ProviderConfig CRD
does not exist until the provider finishes installing.

Crossplane also needs annotation-based resource tracking in `argocd-cm`:

```yaml
data:
  application.resourceTrackingMethod: annotation
```

Argo CD reads that at startup, so restart `argocd-repo-server`,
`argocd-server` and `argocd-application-controller` after changing it.

## Adopting existing records

Records already in Cloudflare are adopted in place, never recreated, so there
is no DNS gap. Fetch the real IDs first:

```bash
curl -s "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/dns_records" \
  -H "Authorization: Bearer $TOKEN" \
  | jq '.result[] | {id, name, type, content, proxied}'
```

Reference the ID in the manifest:

```yaml
metadata:
  annotations:
    crossplane.io/external-name: "<existing record id>"
```

Crossplane finds that record and takes ownership. Confirm with `SYNCED: True`
and `READY: True`, then check the Cloudflare record count is unchanged. If it
went up, something created a duplicate instead of adopting.

## Email records are read-only

MX, DKIM, SPF and DMARC use `Observe` so Crossplane watches them but never
writes or deletes:

```yaml
spec:
  managementPolicies:
    - Observe
```

Tunnel CNAMEs keep the default (`["*"]`) so drift correction stays active
where records actually change.

Note that `deletionPolicy: Orphan` is the Crossplane v1 way of doing this and
is not supported by this provider. `managementPolicies` replaces it and covers
more: it blocks accidental updates as well as deletion.

### Observe records need a compound external-name

This one is easy to miss. A record under `Observe` cannot be created, so the
only way Crossplane can learn about it is to import it. The Terraform provider
underneath requires a compound ID for imports:

```yaml
# fully managed - bare record id is fine
crossplane.io/external-name: "726dffa4a8d7f9bcaa026b36d4f64511"

# Observe only - needs zone_id/record_id
crossplane.io/external-name: "6ee56644798bada42b5503d52ff27754/726dffa4a8d7f9bcaa026b36d4f64511"
```

With the bare ID the record reports `SYNCED: False` and the provider logs:

```
Error: invalid ID
expected urlencoded segments "<zone_id>/<dns_record_id>"
```

`READY` stays `True` throughout and the real Cloudflare record is never
touched, so this looks alarming but is harmless. Fully managed records do not
hit this because they never take the import path.

## Recovering stuck terminating records

If Argo CD prunes the Record objects they enter a terminating state held by
the Crossplane finalizer. A `deletionTimestamp` cannot be removed, so the
objects have to finish dying. The goal is making sure the real Cloudflare
records survive that.

Back up first:

```bash
curl -s "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/dns_records" \
  -H "Authorization: Bearer $TOKEN" | jq '.result' > cf-dns-backup.json
```

Set every record to `Observe`, then verify it actually applied:

```bash
for r in $(kubectl get record.dns.upjet-cloudflare.m.upbound.io -n crossplane-system -o name); do
  kubectl patch $r -n crossplane-system --type merge \
    -p '{"spec":{"managementPolicies":["Observe"]}}'
done

kubectl get record.dns.upjet-cloudflare.m.upbound.io -n crossplane-system \
  -o jsonpath='{range .items[*]}{.metadata.name}{": "}{.spec.managementPolicies}{"\n"}{end}'
```

Only once every record reads `["Observe"]`, clear the finalizers:

```bash
for r in $(kubectl get record.dns.upjet-cloudflare.m.upbound.io -n crossplane-system -o name); do
  kubectl patch $r -n crossplane-system --type merge \
    -p '{"metadata":{"finalizers":null}}'
done
```

Argo CD recreates the objects from Git within seconds and the `external-name`
annotations make Crossplane re-adopt the same records.

Clearing finalizers before setting `Observe` deletes the real DNS records.

## Notes

- `managementPolicies` is beta in this provider. `EnableBetaManagementPolicies`
  appears in the provider logs at startup. Patching it onto a live object
  works, but verify rather than assuming the manifest value took effect.
- Anything patched by hand is drift. Argo CD `selfHeal` will revert it to
  whatever Git says, so declared state belongs in the manifest.
- Zone IDs, record IDs and the tunnel ID appear in this repo. They are
  identifiers, not credentials, and the tunnel ID is already public in DNS.
  The API token is the only secret and it stays in Vault.
- Footprint is small: roughly 22m CPU and 208Mi memory across Crossplane core,
  the RBAC manager and both providers.

## Related

- [k8s-config-pub](https://github.com/catdevops1/k8s-config-pub) - cluster infrastructure and Argo CD applications
- [vault-config-pub](https://github.com/catdevops1/vault-config-pub) - Vault and External Secrets Operator
