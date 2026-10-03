# Argo CD demo

A minimal GitOps repository that deploys the whoami HTTP application into an
`argocd-demo` namespace. Argo CD must already be installed in the `argocd`
namespace of the destination cluster.

## Repository layout

```text
apps/whoami/
  deployment.yaml      # One whoami replica
  service.yaml         # ClusterIP service on port 80
  kustomization.yaml   # Lists the workload files and sets their namespace
bootstrap/
  application.yaml     # Tells Argo CD which repository, branch, and path to follow
```

The Application points at `apps/whoami` on the `main` branch. Argo CD detects
`kustomization.yaml` and renders the Deployment and Service with Kustomize.
The `bootstrap` directory is outside the watched workload path; its Application
is applied once to connect Argo CD to this repository.

The repository is public, so Argo CD can fetch it without GitHub credentials.

## Preview the workload locally

Run these commands from the repository root:

```bash
kubectl kustomize apps/whoami
```

## Connect Argo CD

Check that your current Kubernetes context is the intended cluster, then apply
only the Application. Argo CD fetches and deploys the workload from GitHub:

```bash
kubectl config current-context
kubectl apply -f bootstrap/application.yaml
kubectl -n argocd get application argocd-demo -w
```

Wait for `Synced` and `Healthy`. `CreateNamespace=true` creates `argocd-demo`
during the initial sync. Automatic sync deploys Git changes, and `selfHeal`
restores tracked resources if they are changed directly in Kubernetes.
`prune: false` means removing a file from Git does not automatically delete the
corresponding live resource.

## Test the HTTP application

```bash
kubectl -n argocd-demo rollout status deployment/whoami
kubectl -n argocd-demo port-forward service/whoami 18081:80
```

In another terminal:

```bash
curl http://127.0.0.1:18081
```

The response includes the pod hostname and IP addresses. This test uses the
whoami Service directly; Gateway and Traefik configuration can be added later.

## Test a Git-driven change

Change `replicas: 1` to `replicas: 2` in `apps/whoami/deployment.yaml`, then:

```bash
git diff
git add apps/whoami/deployment.yaml
git commit -m "Scale whoami to two replicas"
git push
kubectl -n argocd-demo get deployment,pods -w
```

Argo CD detects the commit during its next repository refresh. To request an
immediate refresh:

```bash
kubectl -n argocd annotate application argocd-demo \
  argocd.argoproj.io/refresh=hard --overwrite
```

Revert the replica change in Git and push again to return to one replica.

## What makes a repository compatible with Argo CD?

Argo CD needs a reachable Git repository, a revision (for example `main`), and a
path containing valid Kubernetes configuration. That configuration can be plain
YAML, a Kustomize directory, or a Helm chart. Folder names are your choice; they
must match `spec.source.path` in the Application.

The Application also names a destination cluster, destination namespace, and
Argo CD project. This example uses the local cluster and the existing `default`
project. Kubernetes credentials and repository credentials belong outside this
public demo repository.

References:

- [Argo CD Application specification](https://argo-cd.readthedocs.io/en/stable/user-guide/application-specification/)
- [Argo CD Kustomize support](https://argo-cd.readthedocs.io/en/stable/user-guide/kustomize/)
