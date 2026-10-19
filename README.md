# podinfo-deploy

A GitOps demo: [podinfo](https://github.com/stefanprodan/podinfo) deployed with Argo CD
through an **umbrella Helm chart**, plus a **preview environment for every open pull
request**. Argo CD reports each preview back to the PR as a comment and as a GitHub
environment.

What you end up with:

| What | Where |
|---|---|
| The main deployment | `http://podinfo.k8s.orb.local` |
| One preview per open PR | `http://pr-<number>.k8s.orb.local` |
| Per PR, on GitHub | a comment with the preview link, and an entry under **Environments** |

It targets a local [OrbStack](https://orbstack.dev) Kubernetes cluster. Other clusters work
too, but then the `*.k8s.orb.local` hostnames and the ingress step need adapting.

## How the pieces fit together

```
 PR opened / new commit pushed on GitHub
          |
          | (1) polled every 60s, authenticated as the GitHub App
          v
 ApplicationSet controller --(2)--> one Application per PR: podinfo-pr-<n>
                                          |
                                          | (3) Argo CD renders charts/podinfo-umbrella
                                          |     at the PR's head commit
                                          v
                           namespace preview-pr-<n>: podinfo + Ingress pr-<n>.k8s.orb.local
                                          |
                                          | (4) sync succeeded and app is healthy
                                          v
 Notifications controller --(5)--> GitHub App --> PR comment + Deployment (environment)
```

- **ApplicationSet** (`argocd/applicationset.yaml`): its Pull Request generator lists the
  open PRs of this repo and renders one Argo CD Application per PR. When a PR closes, the
  Application disappears from the list and is deleted, which tears the preview down.
- **Umbrella chart** (`charts/podinfo-umbrella`): wraps the upstream podinfo chart as a
  dependency. Values for podinfo live under a `podinfo:` key, because Helm routes values to
  a subchart by its name. `values.yaml` is the base, `values-orbstack.yaml` the overlay that
  enables the ingress. Previews layer per-PR overrides (UI message, hostname) on top.
- **Notifications** (`argocd/argocd-values.yaml`): the Argo CD Notifications controller
  watches the Applications. When a preview becomes healthy it calls GitHub through the same
  GitHub App.

## Repository layout

```
.
├── argocd/
│   ├── argocd-values.yaml     # values for the argo-helm/argo-cd chart (notifications)
│   ├── application.yaml       # Application for the main deployment
│   └── applicationset.yaml    # ApplicationSet generating the PR previews
└── charts/podinfo-umbrella/
    ├── Chart.yaml             # declares podinfo as a dependency (pinned)
    ├── Chart.lock             # generated in step 1
    ├── values.yaml            # base values
    └── values-orbstack.yaml   # ingress on podinfo.k8s.orb.local
```

## Prerequisites

- OrbStack with Kubernetes enabled, and `kubectl config current-context` showing `orbstack`.
- `kubectl`, `helm` (v3) and `git` installed.
- Push access to `github.com/the-technat/podinfo-deploy`. A public repo is simplest. For a
  private repo, see the notes in steps 2 and 5.

## Setup

### 1. Push the repo

```bash
cd charts/podinfo-umbrella && helm dependency update && cd ../..   # writes Chart.lock
git init -b main
git add . && git commit -m "Initial commit"
git remote add origin git@github.com:the-technat/podinfo-deploy.git
git push -u origin main
```

`Chart.lock` records the exact resolved podinfo dependency, so every render is reproducible.
Check that the chart renders and the ingress is on:

```bash
helm template podinfo charts/podinfo-umbrella -f charts/podinfo-umbrella/values-orbstack.yaml \
  | grep -A12 'kind: Ingress'
```

### 2. Create the GitHub App

One GitHub App does three jobs: the ApplicationSet lists PRs with it, the notifications
controller comments with it, and (for a private repo) Argo CD clones with it.

1. GitHub profile picture, then **Settings**, **Developer settings**, **GitHub Apps**,
   **New GitHub App**.
2. Fill in the form:
   - **GitHub App name:** globally unique, max 34 characters, e.g. `technat-argocd-previews`.
   - **Homepage URL:** required but arbitrary, e.g. `https://github.com/the-technat/podinfo-deploy`.
   - **Webhook:** untick **Active**. Argo CD calls GitHub, never the other way round.
3. **Repository permissions**, everything else stays at No access:

   | Permission | Access | Used for |
   |---|---|---|
   | Pull requests | Read & write | list PRs (ApplicationSet), post the PR comment |
   | Deployments | Read & write | create the GitHub Deployment / environment |
   | Contents | Read-only | **private repos only**: let Argo CD clone the repo |
   | Metadata | Read-only | added automatically |

4. **Where can this GitHub App be installed?** Choose **Only on this account**, then
   **Create GitHub App**.
5. On the App's settings page, note the **App ID**, and under **Private keys** click
   **Generate a private key**. Save the downloaded file as `app.private-key.pem` in this
   directory (`.gitignore` already excludes `*.pem`). GitHub keeps only the public half, so
   this file is the only copy of the private key.
6. In the App's left sidebar click **Install App**, then **Install**. Choose **Only select
   repositories** and pick `podinfo-deploy`. The page you land on has a URL ending in
   `/settings/installations/<installation-id>`; note that number.

### 3. Install an ingress controller

OrbStack ships without one. This installs ingress-nginx and marks it as the **default**
IngressClass, which is what `values-orbstack.yaml` relies on (it sets no `ingressClassName`):

```bash
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  -n ingress-nginx --create-namespace \
  --set controller.ingressClassResource.default=true
kubectl get ingressclass          # nginx should be marked as default
```

OrbStack makes LoadBalancer services reachable under the wildcard domain `*.k8s.orb.local`,
so any ingress host ending in that works without DNS changes. Check:

```bash
curl -si http://anything.k8s.orb.local | head -1     # expect HTTP/1.1 404 from nginx
```

### 4. Install Argo CD

Open `argocd/argocd-values.yaml` and replace `appID` and `installationID` (under
`notifiers`) with the values from step 2. Then:

```bash
helm repo add argo https://argoproj.github.io/argo-helm && helm repo update
helm upgrade --install argocd argo/argo-cd -n argocd --create-namespace \
  -f argocd/argocd-values.yaml \
  --set-file notifications.secret.items.github-privateKey=./app.private-key.pem
kubectl -n argocd rollout status deploy/argocd-server
```

Why `--set-file`: the chart creates the Secret `argocd-notifications-secret`, and the
notifier config refers to its `github-privateKey` key. `--set-file` reads the `.pem`
verbatim (multi-line) so the key never lands in a values file. **Pass it on every
`helm upgrade`**, otherwise the key is dropped from the Secret again.

Open the UI:

```bash
kubectl -n argocd port-forward svc/argocd-server 8080:443 &
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d; echo
```

Browse to `https://localhost:8080` (accept the self-signed certificate) and log in as
`admin`. The port-forward must stay running for the links in PR comments to work, because
`argocdUrl` in the values points there.

### 5. Give Argo CD the GitHub App credentials

The ApplicationSet authenticates through a Secret in Argo CD's repo-creds format
(`appSecretName` in `argocd/applicationset.yaml`):

```bash
kubectl -n argocd create secret generic github-app-repo-creds \
  --from-literal=type=git \
  --from-literal=url=https://github.com/the-technat \
  --from-literal=githubAppID=<app-id> \
  --from-literal=githubAppInstallationID=<installation-id> \
  --from-file=githubAppPrivateKey=./app.private-key.pem
kubectl -n argocd label secret github-app-repo-creds argocd.argoproj.io/secret-type=repo-creds
```

The label also makes it a credential template for every repo under
`https://github.com/the-technat`. For a **private** repo this is what lets Argo CD clone it,
which is why the App needs the Contents permission in that case.

### 6. Deploy

```bash
kubectl apply -f argocd/application.yaml        # main deployment
kubectl apply -f argocd/applicationset.yaml     # PR previews
kubectl -n argocd get applications
curl http://podinfo.k8s.orb.local               # JSON from podinfo
```

Until the first PR exists, only the `podinfo` Application is listed.

## Try the demo

```bash
git switch -c demo
# change something visible, e.g. podinfo.ui.message in charts/podinfo-umbrella/values.yaml
git commit -am "Change greeting" && git push -u origin demo
gh pr create --fill          # or open the PR in the GitHub UI
```

Within about a minute (the generator polls every 60s):

1. `kubectl -n argocd get applications` lists `podinfo-pr-<number>`.
2. `curl http://pr-<number>.k8s.orb.local` answers with the preview, and its UI message reads
   `PR #<number>: <title>`.
3. The PR gets a **Preview deployed** comment, and the repo's **Environments** list
   (right sidebar of the repo's main page) shows `podinfo-pr-<number>`.
4. Push another commit: the preview updates to the new commit and the same comment is
   edited rather than duplicated.
5. Close or merge the PR: the Application and its resources are removed.

## Troubleshooting

| Symptom | Where to look |
|---|---|
| No Application appears for the PR | `kubectl -n argocd logs deploy/argocd-applicationset-controller`. Auth errors point at the `github-app-repo-creds` Secret or the App's Pull requests permission; also check `owner`/`repo` in `applicationset.yaml` and that the App is installed on this repo. |
| Application exists but "authentication required" / "repository not found" | Private repo: add Contents (read) to the App, accept the changed permissions on the installation, and check the `url` prefix in the repo-creds Secret. |
| Synced, but the hostname gives nginx 404 or no response | `kubectl get ingressclass` must show a default class; `kubectl -n preview-pr-<n> get ingress` shows the host. |
| No PR comment or environment | `kubectl -n argocd logs deploy/argocd-notifications-controller`. A 403/404 from GitHub means missing Pull requests or Deployments write access, or the App isn't installed on the repo. |
| Notifications log a key error | The Secret lacks `github-privateKey`: re-run the step 4 command including `--set-file`, then `kubectl -n argocd rollout restart deploy/argocd-notifications-controller`. |
| App has no notification at all | `kubectl -n argocd get app podinfo-pr-<n> -o jsonpath='{.metadata.annotations}'` must contain the `notifications.argoproj.io/subscribe...` and `preview-url` annotations. |
| Notified once, not again | Intended: the trigger fires once per revision (`oncePer`). Push a new commit to get a new notification. |

## Clean up

```bash
kubectl delete -f argocd/applicationset.yaml    # removes all previews
kubectl delete -f argocd/application.yaml
helm uninstall argocd -n argocd
helm uninstall ingress-nginx -n ingress-nginx
```

Then, on GitHub, delete the left-over environments (repo **Settings**, **Environments**) and,
if you like, the GitHub App (Settings, Developer settings, GitHub Apps).

## Known limitations

- **Environments are not removed when a PR closes.** Argo CD deletes the app, but nothing
  tells GitHub the environment is gone. A workflow on `pull_request: closed` could delete it.
- **Empty namespaces can remain.** Argo CD creates `preview-pr-<n>` through the
  `CreateNamespace` sync option but does not own it, so it typically stays after the app is
  deleted. `kubectl delete ns preview-pr-<n>` removes it.
- **Local-only URLs.** `*.k8s.orb.local` and `localhost:8080` resolve only on your machine,
  so teammates following the links in a PR comment won't reach them.
- **PR titles end up in Helm values.** A title containing quotes or other special characters
  could break that preview's values. Test with such a title before relying on it.
- **Polling.** New PRs appear after up to 60s. A GitHub webhook to the ApplicationSet
  controller makes it instant.

## Changing things later

| I want to... | Change... |
|---|---|
| Upgrade podinfo | `version` in `charts/podinfo-umbrella/Chart.yaml`, then `helm dependency update` and commit `Chart.lock` |
| Change replicas, UI message, resources | `podinfo:` block in `charts/podinfo-umbrella/values.yaml` |
| Change the main hostname / ingress | `charts/podinfo-umbrella/values-orbstack.yaml` |
| Change the PR comment text | `templates` in `argocd/argocd-values.yaml`, then `helm upgrade` (with `--set-file`) |
| Only preview labelled PRs | uncomment `labels` under the generator in `argocd/applicationset.yaml` |
