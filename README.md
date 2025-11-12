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

7. **Cleanup**
    ```bash
    kubectl delete app echo-dev -n argocd
    kubectl delete ns echo-dev
    ```

---

## Stage 1 – Multi-Env (App-of-Apps)

Goal – extend the base app into **dev / staging / prod** environments using the **App-of-Apps** pattern.

1. **Create root app**
   ```bash
   kubectl apply -f 1-multi-env/apps/root-app.yaml
   ```
2. **Verify in UI**
   * Root app appears under **Applications**
   * Sub-apps (`echo-dev`, `echo-staging`, `echo-prod`) created automatically
   * Each has its own namespace and message version
3. **Test**
   * Change only `values-staging.yaml` → commit → Argo syncs *staging* only
   * Delete one sub-app manually → root recreates it (self-healing)

---

