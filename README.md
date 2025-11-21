# Argo CD Lab

Step-by-step GitOps lab for experimenting with Argo CD

## Goal

| Stage | Topic                | Concept            |
| ----- | -------------------- | ------------------ |
| 0     | Base install         | GitOps loop        |
| 1     | Multi-env            | App-of-apps        |
| 2     | Sync automation      | Self-healing       |
| 3     | Access               | RBAC & SSO         |
| 4     | Multi-cluster        | Cluster management |
| 5     | Observability        | Drift & metrics    |
| 6     | Progressive delivery | Safe rollouts      |

---

## Stage 0 – Base (GitOps Loop)
* install Argo CD
* expose via port-forward
* deploy a single app (echo)
* verify GitOps loop (sync, drift correction)


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
    kubectl delete app echo-dev -n argocd
    kubectl delete ns echo-dev
    ```

---

## Stage 1 – Multi-Env (App-of-Apps)
* add dev / staging / prod apps 
* use app-of-apps pattern - single root app manages all
* helm chart + values per env
* verify selective sync per env

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
      kubectl -n argocd delete app root echo-dev echo-staging echo-prod
      kubectl delete ns echo-dev echo-staging echo-prod
    ```

---

## Stage 2 – Sync Automation & Projects
- **What**
   * Applications now belong to the `apps` project
   * The project restricts which repos can be used and which namespaces apps may deploy into (`echo-*`)
   * All child apps inherit the same safety and automation rules
- **Why**
   * self-healing and pruning behave consistently across all environments
   * accidental deploys outside `echo-*` are blocked
   * cluster-wide resources are not allowed
   * future RBAC becomes possible (project-scoped roles)

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

