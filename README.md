# podinfo-umbrella

GitOps repo that deploys [podinfo](https://github.com/stefanprodan/podinfo) with Argo CD
through an **umbrella Helm chart**.

```
.
├── argocd/
│   └── application.yaml                  # Argo CD Application (apply once)
└── charts/
    └── podinfo-umbrella/
        ├── Chart.yaml                    # declares podinfo as a dependency
        ├── Chart.lock                    # generated, see "First-time setup"
        ├── values.yaml                   # base values (nested under `podinfo:`)
        └── values-orbstack.yaml          # env overlay: ingress on podinfo.k8s.orb.local
```

## How the umbrella pattern works

The chart in `charts/podinfo-umbrella` contains no Kubernetes templates of its own.
It only declares the upstream `podinfo` chart as a **dependency** in `Chart.yaml`.
When Helm renders the umbrella, it pulls the dependency and renders it as a subchart.

Values reach the subchart through a key named after the dependency, so all podinfo
settings live under `podinfo:` in the values files. Several value files are merged in
order, later ones win: `values.yaml` (base) and then `values-orbstack.yaml` (overlay).

Why bother with a wrapper instead of pointing Argo CD at the upstream chart directly?
Your configuration lives in Git, is versioned and reviewable, the upstream version is
pinned, and you can add your own templates (ConfigMaps, NetworkPolicies, ...) next to it
later.

## First-time setup

1. Generate the lock file and commit it (needs Helm installed locally):

   ```bash
   cd charts/podinfo-umbrella
   helm dependency update
   git add Chart.lock && git commit -m "Add Chart.lock"
   ```

   `Chart.lock` records the exact resolved dependency, so every render gets the same
   podinfo. Without it Argo CD still works, but it re-resolves the dependency each time.

2. Check the rendering locally:

   ```bash
   helm template podinfo . -f values-orbstack.yaml | grep -A12 'kind: Ingress'
   ```

3. Push the repo, set `repoURL` in `argocd/application.yaml`, and apply it:

   ```bash
   kubectl apply -f argocd/application.yaml
   ```

   For a private repo, first register it in Argo CD (Settings -> Repositories).

## Accessing it

The Ingress uses host `podinfo.k8s.orb.local` and no explicit `ingressClassName`, so the
cluster's default IngressClass serves it. This needs an ingress controller (for example
ingress-nginx) installed and marked as default. Check with
`kubectl get ingressclass`: the default one shows `(default)` in its annotations.

```bash
curl http://podinfo.k8s.orb.local
```

## Day-to-day changes

| I want to...                    | Change...                                              |
|---------------------------------|--------------------------------------------------------|
| Upgrade podinfo                 | `version` in `Chart.yaml`, then `helm dependency update` |
| Change replicas, UI message ... | `podinfo:` block in `values.yaml`                      |
| Change the hostname / ingress   | `values-orbstack.yaml`                                 |
| Add another environment         | new `values-<env>.yaml` + a second Application         |

Commit and push; Argo CD detects the change and syncs automatically.
