# GitOps FluxCD — Copilot instructions

This repository is a FluxCD-driven GitOps infrastructure repo. The purpose of these instructions is to help AI coding agents (and contributors) be immediately productive — find where to make safe changes, understand common patterns, and run the commands required to validate modifications.

## Big picture (what this repo does)
- GitOps-style repository for managing Kubernetes infrastructure using FluxCD+Kustomize+Helm.
- Top-level folders:
  - `install/`: FluxCD bootstrap Kustomization and base manifests (includes FluxCD, Gateway API CRDs, Prometheus CRDs).
  - `components/`: Shared kustomize components (e.g., `common-repo.yaml`) used by Kustomizations.
  - `common/`: The primary collection of apps and core infra organized by category:
    - `cert-manager/`: TLS certificate management
    - `dbms/`: Database operators (cloudnative-pg, dragonfly-operator)
    - `external-secrets/`: External secrets management
    - `istio-system/`: Istio service mesh
    - `keda/`: Event-driven autoscaling
    - `kube-system/`: Core system apps (metrics-server, reloader)
    - `kube-tools/`: Kubernetes tooling (kyverno, node-feature-discovery)
    - `observability/`: Full observability stack (grafana, loki, tempo, alloy, prometheus, etc.)
  - `clusters/` (not always present): cluster-specific overlays (k8s provider or environment specific) — not included in this repo subset.

## How FluxCD is wired in this repo
- Flux is bootstrapped from `install/` (see `install/kustomization.yaml`); it deploys flux controllers into `flux-system`.
- The repository exposes a GitRepository resource configured in `components/common-repo.yaml`:
  - `name: common` (the repo name used by `install.yaml` Kustomizations)
  - The `ignore` filter is configured to include only paths in `/common/*` (so `components/common-repo.yaml` is important — do not widen it unintentionally).
- Applications are installed via Flux Kustomizations located at `common/<category>/<app>/install.yaml`. Those Kustomizations live in namespace `flux-system` and point to `path: ./common/<...>/app` in the repo.

## Conventions to follow (file & resource patterns)
- App layout (canonical):
  common/my-app/
  ├─ app/ (resource definitions)
  │  ├─ kustomization.yaml
  │  ├─ helmrepository.yaml | ocirepository.yaml | gitrepository.yaml
  │  ├─ helmrelease.yaml
  │  └─ additional resource manifests
  ├─ install.yaml (Kustomization for Flux)
  └─ namespace.yaml (optional)

- `install.yaml` is the Flux Kustomization for that app. It should:
  - Live in the app directory, set `metadata.namespace: flux-system`.
  - Set `spec.targetNamespace` to the app's target namespace (e.g., `observability`, `kube-system`, `dbms`).
  - Use `sourceRef.name: common` (GitRepository) and `sourceRef.namespace: flux-system`.
  - Use `prune: true` and configure `wait`, `interval`, and `timeout`.
  - Optionally include `commonMetadata.labels` for consistent labeling.

- Use `dependsOn` in Kustomizations to ensure ordering when resources must be created sequentially (e.g., cert-manager and its issuers).

- Namespaces: Each app should declare a namespace or use `targetNamespace` in `install.yaml` for namespace-targeting.

- Labels: There is a `common/labels.yaml` label transformer applied in `common/kustomization.yaml` — follow labeling conventions and do not remove base `labels.yaml` usage.

## Safe places to change
- `common/<category>/<app>/app/*`: Add or update resources there for a specific application.
- `common/<category>/<app>/install.yaml`: Only modify to correct source refs, target namespace, or to add `dependsOn`/timeouts.
- `common/kustomization.yaml`: Add your new app `resources: - <app>/install.yaml` so that parent kustomizations include it.

## Do not change without caution
- `install/kustomization.yaml` (Flux bootstrap): changes here affect cluster-wide Flux installation.
- `components/common-repo.yaml` `ignore` pattern: it scopes which repo paths Flux watches. Widening this will change Flux’s source set across clusters.
- Anything under `clusters/` (if present): cluster-specific overlays may be consumed by cluster operators. Consult owners before editing.

## Developer workflows (commands & testing)
- Bootstrap Flux (if evaluating locally):
  - `kubectl apply --kustomize install` (install Flux controller stack)
  - Create Flux Git secret: `kubectl create secret generic flux-system --namespace=flux-system --from-file=identity=~/.ssh/id_rsa --from-file=identity.pub=~/.ssh/id_rsa.pub --from-literal=known_hosts="$(ssh-keyscan github.com 2>/dev/null)"`
  - Set up SOPS age secret (optional): `cat $HOME/.config/sops/age/keys.txt | kubectl -n flux-system create secret generic sops-age --from-file=age.agekey=/dev/stdin`

- Apply a local change and test:
  - Update or add resources under `common/<...>/app` and `install.yaml`.
  - Commit & push the branch.
  - Reconcile: `flux reconcile kustomization <kustomization-name> --namespace flux-system` or `flux reconcile source git common --namespace flux-system`.
  - Check status: `flux get kustomizations` and `flux get helmreleases`.

- Quick Kustomize apply locally (without Flux):
  - `kubectl apply -k common/<...>/app` to apply rendered resources directly for local debug (use with caution; Flux will reconcile and may override resources).

- Useful Flux commands for debugging:
  - `flux logs --follow`
  - `flux get kustomizations` and `flux reconcile kustomization <name> --namespace flux-system`
  - `flux get helmrelease <name>`

## Secrets and SOPS
- Secrets are encrypted using Mozilla SOPS with age keys. Prefer SOPS for repo-based secrets.
- Editing or creating encrypted secrets:
  - `sops --encrypt --age <AGE_PUBLIC_KEY> secret.yaml > secret.sops.yaml`
  - `sops secret.sops.yaml` to edit
  - `sops --decrypt secret.sops.yaml`
- The Flux SOPS secret is expected at `flux-system/sops-age` (created via command in README). Do not commit private keys.

## Integrations / external systems
- External Secrets Operator and HashiCorp Vault are configured under `common/external-secrets` and `common/vault` directories.
- Helm charts are pulled by defining `helmrepository.yaml` in each app's `app/` folder and referenced by `helmrelease.yaml`.
- `install/` includes additional third-party CRD and system installs such as Gateway API and Prometheus CRDs.

## Code patterns & conventions for AI agents
- When changing an app, prefer code-local edits under `common/<category>/app` and add/modify `install.yaml` to configure Flux Kustomization.
- Apply `labels.yaml` transformer and namespace isolation consistently for any new resource.
- When adding `HelmRepository` or `OCIRopository`, include them in the `app/kustomization.yaml` so the app Kustomization references them.
- Keep `install.yaml` in `flux-system` namespace and `targetNamespace` consistent with the target app’s `namespace.yaml`.
- Use `dependsOn` for Kustomizations when resource ordering matters (e.g., system setup before issuers or CRDs).

## Examples (referenced files in repo)
- `components/common-repo.yaml` — GitRepository for `common` (used by Kustomization `sourceRef`).
- `common/kustomization.yaml` — parent kustomization that includes `components` and groups of apps.
- `common/observability/grafana/install.yaml` — typical `install.yaml` Kustomization for an app (targetNamespace + sourceRef).
- `common/cert-manager/install.yaml` — example of `dependsOn` usage (cert-manager + cert-manager-issuers).

## Adding new apps (step-by-step)
1. Create `common/<category>/<name>/app` with `kustomization.yaml`, `helmrepository.yaml` (or other sources), and `helmrelease.yaml`.
2. Add `install.yaml` in `common/<category>/<name>/` with `metadata.namespace: flux-system` and `spec.targetNamespace`.
3. Update the category's `kustomization.yaml` to reference your new `install.yaml`.
4. Add the app to any cluster-specific overlays if needed.
5. Commit changes and create a PR for review.

## Safety and responsibility
- Avoid editing `install/kustomization.yaml` or `components/common-repo.yaml` unless you're prepared to handle cluster-wide changes.
- Do not commit private keys, age private keys, or other credentials. Use SOPS or external secret managers.

---

If anything here is unclear or you'd like more specific examples (e.g., a new Helm app or a cluster overlay patch), tell me which part to expand and I’ll iterate. ✨
