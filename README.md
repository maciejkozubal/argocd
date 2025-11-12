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

## Stage 0 – Base 

1. Install Argo CD
    ```bash
    kubectl create ns argocd
    kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.12.3/manifests/install.yaml
    ```
2. UI
    ```bash
    kubectl port-forward -n argocd svc/argocd-server 8080:443
    # login at https://localhost:8080, user: admin, password:
    kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d; echo
    ```
3. Create app namespace
   ```bash
   kubectl create ns echo-dev
   ```
4. Deploy `echo` app using `Application` manifest
5. Edit manifest in Git → Argo syncs automatically
6. ?

    ```bash
    kubectl apply -f stage-0-base/echo-app.yaml
    ```

---

