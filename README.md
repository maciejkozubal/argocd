# Argo CD Lab

Step-by-step GitOps lab for experimenting with Argo CD

## Goal

| Stage | Topic                    | Concept                   |
| ----- | ------------------------ | ------------------------- |
| 0     | Base install             | GitOps loop               |
| 1     | Multi-env                | App-of-apps               |
| 2     | Sync automation & Projects | Self-healing + boundaries |
| 3     | Helm vs Kustomize        | Templates vs overlays     |
| 4     | PR workflow & Rollbacks  | Promotion via Git         |
| 5     | Access                   | RBAC & SSO                |
| 6     | Multi-cluster            | Cluster management        |
| 7     | Observability            | Drift & metrics           |
| 8     | Image automation         | Auto-tag updates in Git   |
| 9     | Progressive delivery     | Argo Rollouts             |

---

## Stage 0 – Base (GitOps Loop)
### What
* install Argo CD
* expose UI
* deploy single echo app

### Why
* observe Git → cluster sync
* understand drift correction

### How
1. **Install Argo CD**
   ```bash
   kubectl create ns argocd
   kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.12.3/manifests/install.yaml
   kubectl -n argocd rollout status deploy/argocd-server
   ```
2. **UI access**
   ```bash
   kubectl -n argocd port-forward svc/argocd-server 8080:443
   # https://localhost:8080
   # user: admin
   kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d; echo
   ```
3. **Create app namespace**
   ```bash
   kubectl create ns echo-dev
   ```
4. **Deploy `echo` app**
   ```bash
   kubectl apply -f stage-0-base/echo-app.yaml
   ```
5. **Verify deployment**
   ```bash
   kubectl -n echo-dev get deploy,svc,pods
   kubectl -n echo-dev port-forward svc/echo 8081:80
   curl http://localhost:8081/
   # → "hello from dev"
   ```
6. **GitOps / UI tests**
   1. *Edit manifest in Git* → change message → commit → watch **Argo CD UI**
      * status: Out of Sync → Synced
      * pod restarts → new message visible
   2. *Manual drift*
      ```bash
      kubectl -n echo-dev scale deploy/echo --replicas=3
      ```
      * UI: Out of Sync → Auto-Sync → back to 1 replica
   3. *Delete resource*
      ```bash
      kubectl -n echo-dev delete pod -l app=echo
      ```
      * UI: red ❌ → recreated → green ✅
   4. *Pause auto-sync* – toggle **Auto-Sync Off** in UI → make a change in Git → click **Sync** manually

- **Cleanup**
    ```bash
    kubectl delete app echo-dev -n argocd --cascade=foreground
    kubectl delete ns echo-dev
    ```

---

## Stage 1 – Multi-Env (App-of-Apps)
### What
* dev/staging/prod apps
* Helm chart + values per env
* root app manages everything

### Why
* environment separation
* selective sync per environment

### How
1. **Create root app**
   ```bash
   kubectl apply -f 1-multi-env/apps/root-app.yaml
   ```
2. **Verify deployment**
   ```bash
   kubectl -n echo-dev get deploy,svc,pods
   kubectl -n echo-dev port-forward svc/echo 8081:80
   curl http://localhost:8081/
   # → "hello from dev"
   ```   
3. **Verify in UI**
   * Root app appears under **Applications**
   * Sub-apps (`echo-dev`, `echo-staging`, `echo-prod`) created automatically
   * Each has its own namespace and message version
4. **Test**
   * Change only `values-staging.yaml` → commit → Argo syncs *staging* only
   * Delete one sub-app manually → root recreates it (self-healing)
- **Cleanup**
    ```bash
      kubectl -n argocd delete app root echo-dev echo-staging echo-prod --cascade=foreground
      kubectl delete ns echo-dev echo-staging echo-prod
    ```

---

## Stage 2 – Sync Automation & Projects
### What
* Applications belong to apps project
* project restricts repos + namespaces (echo-*)
* consistent automation (prune + self-heal)

### Why
* predictable sync behaviour
* prevents accidental deployments
* foundation for RBAC later

### How
1. **Create**
   ```bash
   kubectl apply -f 2-sync-automation/projects/apps-project.yaml
   kubectl apply -f 2-sync-automation/root-app.yaml
   ```
   - Argo CD will discover and sync all child apps automatically

2. **Verify**
   * In Argo CD UI → **Settings → Projects** → project `apps` is visible
   * Root app appears and shows three children (dev/staging/prod)
      → All apps show **Synced** and **Healthy**

3. **Tests**
   * **Environment-specific update**
     * modify only `values-staging.yaml`
     * commit + push
       → only *staging* should re-sync
   * **Self-healing (drift correction)**
     ```bash
     kubectl -n echo-dev delete svc echo
     ```
     → Argo restores it automatically

   * **Pruning**
     * remove a resource from the Helm chart
     * commit + push
       → Argo deletes it from the cluster

   * **Scaling drift**
     ```bash
     kubectl -n echo-prod scale deploy echo --replicas=10
     ```
     → Argo returns it to the declared replica count

   * **Project boundary check**
     * change a child app’s namespace to something not matching `echo-*`
       → Argo rejects it with a project error


   * **Promotion Test (dev → staging → prod)**
     * Goal – practice moving a change safely across environments using Git (the GitOps promotion workflow)
     1. **Make a change in dev**
         * edit a field only in: `values-dev.yaml`, e.g. change the message or replica count
         * commit + push
         * → Argo CD updates **echo-dev** automatically

     2. **Promote to staging**
        * copy the same change into: `values-staging.yaml` *(yes — just a simple copy/paste of the relevant lines)*
        * commit + push
        * → only **echo-staging** resyncs

     3. **Promote to prod**
        * copy the same change into: `values-prod.yaml`
        * commit + push
        * → **echo-prod** updates after staging is validated

     4. **Verify in Argo CD**
        * each environment updates only when its values file changes
        * root app remains stable
        * sync history shows three separate syncs (dev → staging → prod)

     5. **Rollback test**
        * revert the last commit
        * push
        * → Argo CD rolls back all environments to previous values

- **Cleanup**
    ```bash
      kubectl delete appproj apps -n argocd
      kubectl -n argocd delete app root echo-dev echo-staging echo-prod --cascade=foreground
      kubectl delete ns echo-dev echo-staging echo-prod
    ```


---

## Stage 3 – Kustomize vs Helm (Overlays vs Values)
### What
* replicate Stage 2 environment using Kustomize overlays
* compare Kustomize overlays vs Helm values
* apply same echo app with a Kustomize setup

### Why
* understand when to use Helm (templates) vs Kustomize (patching)
* see strengths/weaknesses side-by-side
* learn both dominant GitOps packaging tools

Kustomize and Helm solve a similar problem (reusable k8s manifests) with different models:

| Aspect       | Helm                                                       | Kustomize                                                   |
| ------------ | ---------------------------------------------------------- | ----------------------------------------------------------- |
| Core idea    | Template engine + chart packaging                          | Patch/overlay engine, no templating                         |
| Main files   | Chart.yaml, values*.yaml, templates/*.yaml, _helpers.tpl, NOTES.txt | kustomization.yaml, base/, overlays/, patch files           |
| Structure    | One chart with templates rendered using `.Values`          | One base with overlays layering patches on top              |
| Composition  | Go templates (`{{ ... }}`) with helpers in `_helpers.tpl`   | YAML patches (strategic merge / json6902) applied on base   |
| Tooling      | `helm template/install/upgrade`, repos of packaged charts  | `kustomize build` or `kubectl kustomize` (built into kubectl) |
| Use cases    | Apps with lots of parameters, reusable “products” (nginx, grafana) | Cluster/platform configs, env-specific tweaks, low magic    |
| Popularity   | Very popular for app charts in ecosystem                   | Very common inside platforms, built into kubectl            |

Rule of thumb:

- **Helm** – “I want to generate manifests from a parameterised template.”
- **Kustomize** – “I want to modify/overlay existing manifests for different envs.”

### How
1. **Create**
   ```bash
   kubectl apply -f 3-kustomize/projects/apps-project.yaml -n argocd
   kubectl apply -f 3-kustomize/root-app.yaml -n argocd
   ```
   - Argo CD will detect and sync all Kustomize-based envs
   - each env renders using its own overlay instead of Helm values

2. **Verify**
   * Argo CD UI shows three new apps (dev/staging/prod) under the Kustomize root
   * manifests differ slightly from the Helm version (patches instead of templates)
   * each env should show **Synced** and **Healthy**

3. **Compare**
   * **Helm**
     - values files
     - templating (`{{ }}`)
     - single chart
   * **Kustomize**
     - base + overlays
     - patches (json6902 / strategic merge)
     - directory hierarchy, no templating

4. **Tests**
   * **Environment-specific patch**
     * modify only `kustomize/overlays/staging/patch.yaml`
     * commit + push
       → only *staging* updates
   * **Self-heal check**
     ```bash
     kubectl -n echo-dev delete deploy echo
     ```
     → Argo restores it using Kustomize-rendered manifests
   * **Overlay diff check**
     * in Argo UI → click **Diff**
       → shows rendered YAML differences between base and overlays
   * **Promotion (optional)**
     * copy a patch from dev → staging → prod
       → test manual promotion using overlays (mirrors Helm workflow)

- **Cleanup**
    ```bash
    kubectl -n argocd delete appproj apps
    kubectl -n argocd delete app echo-kustomize-root echo-dev-kustomize echo-staging-kustomize echo-prod-kustomize --cascade=foreground
    kubectl delete ns echo-dev echo-staging echo-prod
    ```

---

## Stage 4 – PR & Rollback Workflow
### What
* use GitHub Pull Requests for all env changes
* demonstrate promotion dev → staging → prod via PR merges
* rollback via git revert
* inspect diff + sync history in Argo CD

### Why
* real promotion workflow (GitFlow for manifests)
* traceability, approvals, auditability
* safe rollbacks with zero cluster-side commands

### How
1. **Apply project + root app**

   ```bash
   kubectl apply -f 2-sync-automation/projects/apps-project.yaml
   kubectl apply -f 4-access-and-rollbacks/root-app.yaml
   ```

2. **Verify apps in Argo CD**

   In Argo CD UI:

   * Settings → Projects → confirm `apps` project exists and lists your repo + `echo-*` namespaces
   * All three env apps should become **Synced** and **Healthy**

3. **Switch to PR-based changes (no direct pushes)**

   From now on, for *this stage*:

   * **Do not push straight to `main`**
   * Workflow per change:

     1. Create feature branch, e.g. `feat/change-echo-timeout`
     2. Edit only one env file (e.g. `4-access-and-rollbacks/workloads/.../values-dev.yaml` *or* Kustomize overlay)
     3. Commit to branch
     4. Open a GitHub PR → review → merge to `main`
     5. Argo CD sees commit on `main` and syncs automatically (or you click Sync)

   This matches common GitOps guidance: **Git is the source of truth, PR is the gate, Argo CD just syncs**.

4. **Promotion via PRs (dev → staging → prod)**

   For an actual “release”:

   1. **Dev**

      * On a feature branch, change only the dev env config
      * PR → review → merge
      * Argo updates **dev** app

   2. **Staging**

      * Separate PR that copies the same change into staging config
      * Merge → Argo updates **staging**

   3. **Prod**

      * Final PR updates prod config
      * Merge → Argo updates **prod**

   This gives you:

   * separate review points per environment
   * clear Git history of how a change moved dev → staging → prod

5. **Rollback using Git**

   When you intentionally break prod (do it 😈):

   * Find the last “good” commit in Git history for `main`
   * Run locally:

     ```bash
     git revert <bad-commit-sha>
     git push origin main
     ```
   * Argo CD:

     * detects new commit
     * syncs back to the reverted manifests
     * **prod rolls back** to previous config automatically

   This is the canonical GitOps rollback: you undo in Git, Argo forces the cluster back to that state.

6. **Rollback using Argo’s history (cluster-side)**

   Complementary to Git revert, you can test Argo’s built-in rollback:

   * In Argo CD UI → `echo-prod` → **History and Rollback**
   * Pick an earlier revision → click **Rollback**
   * Or CLI:

     ```bash
     argocd app history echo-prod
     argocd app rollback echo-prod <ID>
     ```
   * Argo reapplies the manifests from that history entry

   In a “pure” GitOps setup you’d still commit a matching revert in Git, but for the lab it’s enough to see that Argo can restore older revisions on its own.

7. **Small test plan for Stage 4**

   * change dev via PR → merge → only dev updates
   * promote same change via PR to staging → only staging updates
   * introduce bad prod change via PR → see it break →

     * fix with `git revert` **and**
     * separately try Argo UI “History and Rollback”
   * confirm Git history + Argo history tell the same story (who changed what, when)

---

## Stage 5 – Access (RBAC & SSO)
### What
* disable default admin
* configure OIDC (GitHub/Google/Azure)
* add RBAC roles (argocd-rbac-cm)
* project-level permissions

### Why
* secure the platform
* enforce least privilege
* allow multi-team setups

---

## Stage 6 – Multi-Cluster Management
### What
* register external clusters in Argo CD
* deploy echo app into multiple clusters
* root app manages all clusters

### Why
* real-world multi-cluster GitOps
* central management from one Argo CD
* cluster-scoped AppProjects

---

## Stage 7 – Observability, Drift & Notifications
### What
* enable Argo CD Notifications (Slack/webhooks)
* integrate Prometheus/Grafana dashboards
* track drift and sync events
* read controller + repo-server logs

### Why
* operational insight
* drift detection alerts
* visibility into GitOps pipelines

---

## Stage 8 – Image Automation
### What
* enable Argo CD Image Updater
* auto-bump container tags in Git
* PR mode or direct-write mode
* verify flow: registry → Git → Argo

### Why
* automate version bumps
* remove manual tag updates
* reliable hands-off workflow

---

## Stage 9 – Progressive Delivery
### What
* introduce Argo Rollouts
* replace Deployment with Rollout
* canary / blue-green strategies
* optional metric-based promotion (Prometheus)

### Why
* safer deploys
* automatic rollback on failures
* advanced rollout patterns
