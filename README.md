# issuers

## Purpose

This chart replaces `cluster-config/apps/issuers` and its kustomize-envsubst
render. It installs two Let's Encrypt ClusterIssuers, an internal CA
ClusterIssuer, and a self-signed ClusterIssuer. Infisical supplies the Cloudflare
API token. The chart does not render a plain Secret.

There is no upstream chart or dependency archive. `dependencies: []`
allows the normal dependency-build check to run; its empty Chart.lock is kept.
`appVersion: 1.0.0` describes
these local templates, not the separately installed cert-manager controller.
The chart does not ship CRDs. It does not need `helm.skipCrds`.

## Before first install

1. Install cert-manager and its `cert-manager.io/v1` ClusterIssuer CRD.
2. Install the Infisical operator with the
   `secrets.infisical.com/v1beta1` InfisicalStaticSecret CRD.
3. Check that `default/infisical-prod-universal-auth` is ready. Its connection
   must permit reads from project `k3s-ntpo`, environment `prod`,
   path `/argocd/apps/env`, including child paths.
4. Confirm that `CLOUDFLARE_API_TOKEN` exists at that source. Do not copy its
   value into this chart or cluster-config.
5. Keep the existing CA key pair Secret
   `cert-manager/raulstechtips-internal-ca-key-pair`. This chart references it;
   it does not create or rotate it.
6. Publish the chart to `ghcr.io/raulstechtips/charts` through CI. Replace
   `PENDING_CI_VERSION` in `cluster-config/argo/dev/issuers.yaml` with the first
   CI version. Follow the adoption steps below before any sync.

## Values reference

Use `cluster-config/apps/dev/issuers.yaml` for environment values. It contains
the email and domain read from the live ClusterIssuers and the Infisical
connection. Fixed cluster-wide CA names are chart defaults. It contains no
secret values.

| Value | Default | Function |
| --- | --- | --- |
| `acme.enabled` | `false` | Render both ACME issuers. |
| `acme.email` | empty | Required account and Cloudflare email when ACME is on. |
| `acme.dnsZones` | `[]` | Required DNS zone selector when ACME is on. |
| `acme.issuers` | production and staging | Names and ACME directory URLs. Each account key Secret uses its issuer name. |
| `ca.enabled` | `false` | Render an issuer for an existing CA key pair. |
| `ca.name` | `raulstechtips-internal-ca-issuer` | Cluster-wide ClusterIssuer name. |
| `ca.secretName` | `raulstechtips-internal-ca-key-pair` | Existing cluster-wide CA key pair Secret; must not be empty when CA is on. |
| `selfSigned.enabled` | `true` | Render `selfsigned-issuer`. |
| `cloudflareToken.enabled` | `false` | Sync the Cloudflare token from Infisical. Can stay off if another system supplies the same Secret. |
| `infisical.authRef.name` | empty | Required InfisicalAuth name when token sync is on. |
| `infisical.authRef.namespace` | empty | Required InfisicalAuth namespace when token sync is on. |
| `infisical.projectSlug` | empty | Required source project when token sync is on. |
| `infisical.environmentSlug` | empty | Required source environment when token sync is on. |
| `infisical.secretPath` | `/argocd/apps/env` | Existing token source path. |
| `infisical.recursive` | `true` | Preserve the current source recursion setting. |

The target name and key stay `cloudflare-token-secret/cloudflare-token`.
The template maps only `CLOUDFLARE_API_TOKEN` to that key. Refresh is `60s`,
type is `Opaque`, and creation policy is `Orphan`.

## Upgrades

Keep the release name `issuers` and destination namespace `cert-manager`.
Run local checks with the required Helm version:

```sh
export PATH="/opt/homebrew/bin:$PATH"
export TMPDIR=/tmp/ulw-wrappers/issuers/
bash ../.omo/wrapper-charts/tools/checks.sh "$PWD" "$TMPDIR"
/opt/homebrew/bin/helm template issuers charts -n cert-manager \
  -f ../cluster-config/apps/dev/issuers.yaml
```

The checks cover dependency build, strict lint with defaults and all features,
Helm unit tests, and kubeconform. InfisicalStaticSecret is skipped by the shared
kubeconform check; ClusterIssuer uses the cert-manager schema in the CRD catalog.
Review parity evidence in `../.omo/evidence/wrappers/issuers/` before a manual sync.

## Adoption notes

### Expected differences

- The four ClusterIssuer specifications and names must remain the same.
  Their existing metadata has no chart labels. The templates preserve this
  state to avoid changes during adoption. The labels helper is used only for
  the new InfisicalStaticSecret.
- `NEW secrets.infisical.com/InfisicalStaticSecret cert-manager/cloudflare-token-secret`:
  the operator now reads `CLOUDFLARE_API_TOKEN` directly from `/argocd/apps/env`.
  It writes the existing Secret with the existing key. It uses `Orphan` so
  removal of the sync resource does not remove the Secret.
- `PRUNE core/Secret cert-manager/cloudflare-token-secret`: the live Application
  still tracks the old plain Secret, but this chart does not render it.
  **Do not prune this object.** It must remain for cert-manager and the operator.
- The Application uses manual sync, project `platform`, GHCR, and two sources
  in place of the old automated kustomize-envsubst source. The old Secret
  `ignoreDifferences` and its only sync option, `RespectIgnoreDifferences=true`,
  are removed. Argo will no longer manage that Secret's data after it is
  orphaned. Keeping a data ignore rule would not protect it from prune.
- There is no old InfisicalSecret in this app to delete. Do not delete the
  shared `argocd/infisical-secret-argocd-app-env`; other apps can still use it.

### Operator runbook (not executed by local checks)

1. In Argo CD, open `issuers`, then **App Details > Summary > Sync Policy**.
   Disable automated sync on the old Application before the source handover.
   Confirm that automatic prune and self-heal are off. Do not sync the old
   source again after the Secret is orphaned.
2. Set the published CI version in the new Application file. Review the
   root-dev change. Adopt the same Application name, `issuers`, with its new
   sources and manual sync. Do not delete and recreate the Application.
3. Open the issuers sync dialog. Leave **Prune** unchecked. Sync the new
   resources. The old Secret can show as requiring prune; leave it alone.
4. Check the new sync resource without reading any Secret values:

   ```sh
   kubectl get infisicalstaticsecret cloudflare-token-secret -n cert-manager -o yaml
   ```

   Confirm successful reconciliation in its status and confirm the target is
   `cert-manager/cloudflare-token-secret`. Resolve source/authentication errors
   before continuing.
5. Remove only the old Argo tracking annotation from the target Secret:

   ```sh
   kubectl annotate secret cloudflare-token-secret -n cert-manager \
     argocd.argoproj.io/tracking-id-
   ```

   This is an operator action, not part of local verification. Do not delete
   the Secret. Live discovery found annotation-based tracking and no labels
   on this Secret. If tracking settings change before adoption, inspect
   metadata only and remove the corresponding Argo tracking label too; do not
   remove unrelated labels or the Infisical sync resource's tracking.
6. Refresh the Application. Confirm that the target Secret is no longer
   tracked by `issuers` and no longer appears in its prune list. Do not enable
   prune while it remains tracked. Check issuer status:

   ```sh
   kubectl get clusterissuer letsencrypt-production letsencrypt-staging \
     raulstechtips-internal-ca-issuer selfsigned-issuer
   ```

   All four must remain ready. Keep manual sync for subsequent upgrades.

If an older InfisicalSecret is found in a later environment, first confirm that
it manages this exact target. Before the first new sync, orphan-delete that
old resource with `kubectl delete infisicalsecret <name> -n <namespace>
--cascade=orphan` so its Secret stays. No such resource was found for this
Application during this local build.
