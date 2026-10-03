# Ephemeral Self-Hosted GitHub Actions Runners on Kubernetes (ARC)

**Tool:** Actions Runner Controller (ARC) · **Supporting:** GitHub Actions, Minikube, Helm, kubectl

## 1. Tool selection
Hosted runners are shared, capped in CPU/RAM, and cost money at scale. Static self-hosted VMs sit idle
and keep state between jobs (security and "works on my runner" problems). ARC is a Kubernetes operator that
runs each job in a **fresh, single-use pod** and scales the pool with the job queue.

## 2. Architecture
```
GitHub (workflow queued) --long-poll--> Listener pod (arc-systems)
                                          | tells controller desired count
                                          v
 Controller (arc-systems) --creates--> EphemeralRunnerSet --> EphemeralRunner --> Runner POD (arc-runners)
                                                                                    runs 1 job, then is deleted
```
Namespaces: `arc-systems` = controller + listener; `arc-runners` = runner pods.
Runner label (`runs-on`) = Helm release name = `arc-runner-set`.
Note: this uses the current ARC (`gha-runner-scale-set`, GitHub-supported). The older community
`RunnerDeployment` + `HorizontalRunnerAutoscaler` is the legacy equivalent: `minRunners/maxRunners` here
= HRA `minReplicas/maxReplicas`; runners are always ephemeral.

## 3. Prerequisites
- Docker Desktop running; PowerShell
- A GitHub repo you own (push `.github/workflows/heavy-build.yml` to it)
- A classic Personal Access Token with `repo` scope (never commit it)

## 4. Steps
```powershell
.\00-install-tools.ps1                       # once, admin shell; reopen terminal
.\01-start-cluster.ps1                       # minikube, 4 CPU / 6 GB
.\02-install-arc.ps1 -Repo "https://github.com/<user>/<repo>" -Token "<PAT>"
kubectl get pods -n arc-systems              # controller + listener Running
kubectl get pods -n arc-runners              # NONE (minRunners: 0)
```
In GitHub: repo → Settings → Actions → Runners shows `arc-runner-set` (listener online).

### Demo run (two terminals)
- Terminal B: `.\03-watch.ps1`
- GitHub → Actions → "Compute-heavy pipeline on ARC" → **Run workflow**
- Observe: 3 pods appear (`arc-runner-set-xxxxx-runner-yyyyy`) Pending → Running → job logs show the pod hostname → pods go
  Terminating and vanish when done. `kubectl get pods -n arc-runners` is empty again.
- Extra proof: `kubectl get ephemeralrunners -n arc-runners` and
  `kubectl logs -n arc-systems -l actions.github.com/scale-set-name=arc-runner-set` (scaling decisions).

## 5. Comparison
| Aspect | GitHub-hosted | Static self-hosted VM | **ARC on Kubernetes** |
|---|---|---|---|
| Cost idle | none | pays 24/7 | ~zero (scale to 0) |
| Isolation | clean VM | shared, dirty state | clean pod per job |
| Scaling | managed, limited concurrency | manual | automatic min/max |
| Custom hardware/images | no | yes | yes (pod spec, node pools) |
| Ops effort | none | high | medium (cluster to maintain) |
| Private network access | no | yes | yes |

Alternatives: Jenkins agents on K8s (heavier, plugin-driven), Tekton (pipeline engine replacement, not a GitHub Actions runner),
GitLab Runner K8s executor (equivalent idea for GitLab). Limits of ARC: needs a cluster, PAT/App auth management,
Docker-in-Docker needs extra config (`containerMode: dind`/kubernetes).

## 6. Video script (~5-6 min)
1. (0:30) Problem + why ARC; show architecture diagram.
2. (1:00) `minikube start`, `kubectl get nodes`.
3. (1:30) Helm install controller + scale set; show `values.yaml` (min 0 / max 4), pods in both namespaces.
4. (0:30) GitHub Runners page shows the scale set; no runner pods running.
5. (2:00) Trigger workflow with `watch` on screen: pods spawn, job logs, pods terminate.
6. (0:30) Comparison table + conclusion.

## 7. Troubleshooting
- Pod `Pending`: not enough CPU → `minikube start --cpus=4` or lower `maxRunners`.
- Job stuck "Waiting for a runner": label mismatch; `runs-on` must equal the Helm release name.
- Listener `401/403`: PAT lacks `repo` scope or wrong repo URL.
- Image pull slow first time: pre-pull with `minikube ssh "docker pull ghcr.io/actions/actions-runner:latest"`.

## 8. Cleanup
`.\04-cleanup.ps1`
