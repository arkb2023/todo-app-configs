## [Exercise 4.8. The project, step 24](https://courses.mooc.fi/org/uh-cs/courses/devops-with-kubernetes/chapter-5/gitops)

**Instructions:**  

Move the project to use GitOps so that when you commit to the repository, the application is automatically updated. In this exercise, it is enough that the main branch is deployed to cluster.

---

**Key Changes from Base**
- [`.github/workflows/gitops-project.yaml`](../.github/workflows/gitops-project.yaml): Defines GitHub Actions workflow that builds and pushes new Docker images for the project applications and updates Kustomize image references automatically on relevant path changes.
- [`environments/project-local/kustomization.yaml`](../environments/project-local/kustomization.yaml):  Updated the Kustomize configuration for project so images are driven by placeholders that the workflow replaces with immutable SHA tags, instead of manually managed version tags.
- Base application: [Todo App and Todo Backend v4.6](https://github.com/arkb2023/devops-kubernetes/tree/4.6/the_project)

---

**Directory and File Structure**
<pre>
.github/                                        # github workflows root folder
└── workflows
    └── gitops-project.yaml                     # GitOps workflow for project (todo (frontend) + todo backend)

environments/                                   # Multi-env overlays (k3dlocal/GKE)
├── project-gke                                 # GKE environment specific overlays
│   ├── gateway.yaml                            # Gateway API (GKE)
│   ├── kustomization.yaml                      # Top level kustomization entry point 
│   ├── namespace.yaml                          # Namespace
│   ├── persistentvolumeclaim.yaml              # Persistent Volume Claim
│   ├── podmonitoring-todo-backend.yaml         # Pod monitoring
│   ├── postgresql-backup-cronjob.yaml          # PostgreSQL Backup Cronbjob
│   ├── todo-app-route.yaml                     # Todo-App HTTPRoute (GKE)
│   └── todo-backend-route.yaml                 # Todo-backend App HTTPRoute (GKE)
└── project-local                               # k3d Local environment specific overlays
    ├── kustomization.yaml                      # Top level kustomization entry point 
    ├── namespace.yaml                          # Namespace
    ├── persistentvolume.yaml                   # Persistent Volume
    ├── persistentvolumeclaim.yaml              # Persistent Volume Claim
    ├── todo-ingress.yaml                       # Traefik ingress
    └── nats-dep.yaml                           # NATS deployment

apps/                                           # Shared base resources
└── the-project                                 # Consolidated app manifests + kustomization
    ├── cron_wiki_todo.yaml                     # Wiki Generator CronJob
    ├── kustomization.yaml                      # Base manifests for Todo App, Todo backend, Postgress and Wiki Generator
    ├── postgresql-configmap.yaml               # PostgreSQL ConfigMap
    ├── postgresql-dbsecret.yaml                # PostgreSQL Secret
    ├── postgresql-service.yaml                 # PostgreSQL Service
    ├── postgresql-statefulset.yaml             # PostgreSQL StatefulSet
    ├── todo-app-configmap.yaml                 # Todo Application ConfigMap
    ├── todo-app-deployment.yaml                # Todo Application Deployment
    ├── todo-app-service.yaml                   # Todo Application Service
    ├── todo-backend-configmap.yaml             # Todo Backend Application ConfigMap
    ├── todo-backend-deployment.yaml            # Todo Backend Application Deployment
    └── todo-backend-service.yaml               # Todo Backend Application Service

the_project/                                    # Project root
├── broadcaster                                 # Broadcaster application
│   ├── Dockerfile                              # Dockerfile for image
│   └── broadcaster.py                          # Subscribe to NATS and post to Slack
├── todo_app                                    # Frontend application
│   ├── Dockerfile                              # Dockerfile for image
│   ├── app                                     # Application code
│   │   ├── __init__.py
│   │   ├── cache.py
│   │   ├── main.py
│   │   ├── routes
│   │   │   ├── __init__.py
│   │   │   └── frontend.py
│   │   ├── static
│   │   │   └── scripts.js
│   │   └── templates
│   │       └── index.html
│   └── requirements.txt
└── todo_backend                                # Backend application
    ├── Dockerfile                              # Dockerfile for image
    ├── app                                     # Application code
    │   ├── __init__.py
    │   ├── main.py
    │   ├── models.py
    │   ├── routes
    │   │   ├── __init__.py
    │   │   └── todos.py
    │   └── storage.py
    └── requirements.txt
</pre>
  
***

**Base Setup**  
- Docker  
- k3d (K3s in Docker)  
- kubectl (Kubernetes CLI)
- Create Cluster 
  ```bash
  k3d cluster create dwk-local --agents 2 --port 8081:80@loadbalancer
  ```
- Install ArgoCD in the cluster
  ```bash
  kubectl create namespace argocd
  kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
  ```
- Port-forward ArgoCD server to localhost
  ```bash
  kubectl port-forward svc/argocd-server -n argocd 8080:80
  ```
- Get admin password:  
  ```bash
  kubectl get secret argocd-initial-admin-secret -n argocd \
    -o jsonpath="{.data.password}" | base64 -d
  ```
- Access ArgoCD web UI at: `http://localhost:8080/applications`, use `admin` as username and the password retrieved above.
---

devops-kubernetes [main]$ docker exec k3d-dwk-local-agent-0 mkdir -p /tmp/kube-staging
devops-kubernetes [main]$ docker exec k3d-dwk-local-agent-0 chmod 777 /tmp/kube-staging
devops-kubernetes [main]$ docker exec k3d-dwk-local-agent-0 mkdir -p /tmp/kube-production
devops-kubernetes [main]$ docker exec k3d-dwk-local-agent-0 chmod 777 /tmp/kube-production
devops-kubernetes [main]$

export NATS_URL="nats://127.0.0.1:4222"
# Create namespace + secrets
kubectl create ns staging
kubectl -n staging create secret generic slack-webhook-secret \
  --from-literal=webhook_url=$SLACK_WEBHOOK_URL
  
kubectl -n production create secret generic slack-webhook-secret \
  --from-literal=webhook_url=$SLACK_WEBHOOK_URL

# Deploy staging manually first
kustomize build overlays/staging | kubectl -n staging apply -f - 
# Verify
kubectl -n staging get all
kubectl -n staging logs -l app=broadcaster | grep "Staging: Logging only"















## **Perfect Plan! Baby Steps = Success**

**Your approach is textbook GitOps.** Sequential validation prevents surprises.

## **Execution Plan**

### **1. STAGING (main branch)**

#### **a. Manual Deploy + Test**
```bash
# Create namespace + secrets
kubectl create ns staging
kubectl create secret generic slack-webhook-secret \
  --from-literal=webhook_url=https://hooks.slack.com/DUMMY \
  -n staging

# Deploy staging manually first
kustomize build overlays/staging | kubectl apply -f - -n staging

# Verify
kubectl -n staging get all
kubectl -n staging logs -l app=broadcaster | grep "Staging: Logging only"
```

#### **b. Git Push → ArgoCD Auto-Deploy**
# 2. Apply ArgoCD app (if not already)
kubectl apply -n argocd -f apps/the-project/argocd-apps/staging-app.yaml

```bash
git add . && git commit -m "4.9: Staging environment" && git push origin main
# ArgoCD → staging-broadcaster-dep (logs only)
```

# 3. Verify ArgoCD sync
kubectl -n argocd get applications | grep staging
argocd app get the-project-staging  # Shows Healthy/Synced

#### **c/d. Validate**
```
✅ Logs NATS messages (no Slack)
✅ No DB backup (default PostgreSQL)
```

### **2. PRODUCTION (tagged releases)**

#### **a. Manual Deploy + Test** 
```bash
kubectl create ns production
# Same secrets (shared)
kubectl create secret generic slack-webhook-secret \
  --from-literal=webhook_url=https://hooks.slack.com/REAL \
  -n production

kustomize build overlays/prod | kubectl apply -f - -n production
kubectl -n production logs -l app=broadcaster | grep "Production: Forwarding"
```

#### **b. Tag + Push → ArgoCD Auto-Deploy**
```bash
git tag v1.0.0
git push origin v1.0.0
# ArgoCD → prod-broadcaster-dep (Slack + backups)
```

#### **c/d. Validate**
```
✅ Logs + Slack forwarding
✅ DB backup enabled (add prod patch)
```

## **Shared Secrets (Outside ArgoCD)**
```bash
# Common pattern - external secret management
kubectl create secret generic common-secrets \
  --from-literal=database-password=secret123 \
  -n staging --dry-run=client -o yaml > secrets.yaml
kubectl apply -f secrets.yaml -n production  # Same secret both envs
```

## **Quick Prod Backup Patch**
```bash
cat > overlays/prod/database-backup-patch.yaml << 'EOF'
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgresql-db
spec:
  template:
    spec:
      containers:
      - name: postgresql
        volumeMounts:
        - name: backup-storage
          mountPath: /backups
  volumeClaimTemplates:
  - metadata:
      name: backup-storage
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi
EOF

# Add to overlays/prod/kustomization.yaml
# patches: [database-backup-patch.yaml]
```

## **Status Checkpoints**
```
After staging manual: ✅ Logs only, no Slack
After main push:     ✅ ArgoCD staging healthy  
After prod manual:   ✅ Slack forwarding + backups
After tag push:      ✅ ArgoCD prod healthy
```

**Start with `kubectl create ns staging` + manual deploy. Share logs!**[1]

[1](https://courses.mooc.fi/org/uh-cs/courses/devops-with-kubernetes/chapter-5/gitops)






















### 1. Setup a new application for project in ArgoCD

  - Create a new ArgoCD Application pointing to:  
    | Field            | Value                                              |
    |------------------|----------------------------------------------------|
    | Application Name | project-local                                      |
    | Project          | default                                            |
    | SYNC POLICY      | Manual (change to Automatic after first sync)      |
    | Repository URL   | https://github.com/arkb2023/devops-kubernetes      |
    | Revision         | main                                               |
    | Path             | environments/project-local                         |
    | Cluster          | https://kubernetes.default.svc                     |
    | Namespace        | project                                            |

  - Application created in ArgoCD:

    ![caption](./images/01-application-pre-sync-summary.png)

  - Application details before first sync:  

    ![caption](./images/02-application-pre-sync-details.png)

  - Sync the application manually for the first time via ArgoCD Web UI by clicking **SYNC** button and confirming the sync.  

    ![caption](./images/04-application-post-sync-summary.png)  

  - Application details after first sync:

    ![caption](./images/03-application-post-sync-details.png)

  - Enable automatic sync for the application:  

    ![caption](./images/05-application-enable-auto-sync.png)

---

### 2: GitHub Actions workflow for GitOps
- Add the GitOps workflow: [`.github/workflows/gitops-project.yaml`](../.github/workflows/gitops-project.yaml).
- The workflow is triggered on pushes to `main` affecting the `the_project/`, `apps/the-project/`, or `environments/project-local/` paths (as defined in the `paths:` filter)  
- The workflow builds and pushes new `todo-app` and `todo-backend` images, runs `kustomize edit set image` to update `environments/project-local/kustomization.yaml`, and commits the change back to the `main` branch.

---
### 3: Test End-to-end GitOps: push → Actions → Argo auto-sync  
- Make a code change in `the_project/` folder, commit and push to `main` branch:  

  *Example:*  
  Update following files to scale up deployment pods from 1 to 2 replicas each for:  

  - Postgresql: [`postgresql-statefulset.yaml`](../apps/the-project/postgresql-statefulset.yaml)  
  - Todo backend:     [`todo-backend-deployment.yaml`](../apps/the-project/todo-backend-deployment.yaml)  
  - Todo app (frontend): [`todo-app-deployment.yaml`](../apps/the-project/todo-app-deployment.yaml)  

  ```bash
  git add .  
  git commit -m "Test01: deployment scaled up - 2 pods each"
  git push origin main
  ```
  Output:  
  ```
  Enumerating objects: 18, done.
  Counting objects: 100% (18/18), done.
  Delta compression using up to 22 threads
  Compressing objects: 100% (11/11), done.
  Writing objects: 100% (11/11), 125.06 KiB | 590.00 KiB/s, done.
  Total 11 (delta 7), reused 0 (delta 0), pack-reused 0
  remote: Resolving deltas: 100% (7/7), completed with 6 local objects.
  To github.com:arkb2023/devops-kubernetes.git
    e8c013d..cd7733e  main -> main
  ```
  > CommitID: `cd7733e` for reference in the next steps.

- GitHub Actions workflow is triggered: [Run #58384901408](https://github.com/arkb2023/devops-kubernetes/actions/runs/20323843484/job/58384901408)

- ArgoCD detects the Git change, automatically syncs the `project-local` application to  commitID `cd7733e` and updates the Deployment specs to use 2 replicas for PostgreSQL, todo-backend, and todo-app:  

  ![caption](./images/20-argocd-developer-commitid-synced.png)

- GitHub Actions workflow details:

  - High-level view (commitID `cd7733e`):
    ![caption](./images/11-github-workflow-highlevel-view-run-success.png)  
  
  - Step-level view  
    ![caption](./images/12-github-workflow-steplevel-view-success.png)  

  - Kustomize updated with SHA image references:  
    ![caption](./images/13-github-workflow-kustomize-set-image-with-sha.png)  

  - Git commit, pushed to main branch from gitops workflow:
    ![caption](./images/15-github-workflow-EndBug-add-and-commit-v9-step.png)
    > Note the commit message and commit sha:  
    >   Using "Project GitOps cd7733e49c7ff25739f1bdd11eaeea5471739fe7" as commit message.
    >   commit_sha: `e006308`

  - Corresponding to the gitops commit above, [`environments/project-local/kustomization.yaml`](https://github.com/arkb2023/devops-kubernetes/commit/e006308cf2c9add9c0aff62abae93599c7f2a204#diff-74ab6b5b77555130d39c5cc0d6c0228be210ab5e3e49a7413450ea7848b53633) updated with new SHA image references:
    ![caption](./images/14-kustomize-updated-with-sha.png)

- ArgoCD detects this Git change from Gitops workflow, automatically syncs the `project-local` application to commitID `e006308` and updates the application to use the new images:
   
  ![caption](./images/22-argocd-history-rollback-view.png)
    
- Application fully synced and healthy state with two replicas per deployment and new images:

  ![caption](./images/21-argocd--synced-view.png)

---

### 4. Validation
- Verify in the cluster that the Deployments and StatefulSet have 2 replicas each:  
  ```bash
  kubectl  -n project get pods
  ```
  Output:  
  ```text
  NAME                                 READY   STATUS      RESTARTS      AGE
  broadcaster-dep-d754bcc8-8l6v6       1/1     Running     1 (28h ago)   35h
  broadcaster-dep-d754bcc8-phk69       1/1     Running     1 (28h ago)   35h
  broadcaster-dep-d754bcc8-vnfk5       1/1     Running     1 (28h ago)   35h
  broadcaster-dep-d754bcc8-wcvfk       1/1     Running     1 (28h ago)   35h
  broadcaster-dep-d754bcc8-xjmkp       1/1     Running     1 (28h ago)   35h
  broadcaster-dep-d754bcc8-znwml       1/1     Running     1 (28h ago)   35h
  postgresql-db-0                      1/1     Running     4 (28h ago)   6d15h
  postgresql-db-1                      1/1     Running     0             3h46m
  todo-app-dep-774d9556c-2ksjb         2/2     Running     0             3h39m
  todo-app-dep-774d9556c-g86j4         2/2     Running     0             3h39m
  todo-backend-dep-6577bf8f76-fp2sv    1/1     Running     0             3h39m
  todo-backend-dep-6577bf8f76-szh4x    1/1     Running     0             3h39m
  wiki-todo-generator-29433840-l9wnq   0/1     Completed   0             113m
  wiki-todo-generator-29433900-khb6b   0/1     Completed   0             67m
  wiki-todo-generator-29433960-lh8vg   0/1     Completed   0             6m38s
  ```

- Verify deployments status:
  ```bash
  kubectl get deploy -n project
  ```
  Output:  
  ```text
  NAME               READY   UP-TO-DATE   AVAILABLE   AGE
  broadcaster-dep    6/6     6            6           35h
  todo-app-dep       2/2     2            2           6d15h
  todo-backend-dep   2/2     2            2           6d15h
  ```

- Verify statefulset status:
  ```bash
  kubectl get statefulset -n project
  ```
  Output:  
  ```text
  NAME            READY   AGE
  postgresql-db   2/2     6d15h
  ```
- Access the Todo App at: `http://localhost:8081`, verify functionality works with the scaled deployments.

  ![caption](./images/31-application-works.png)

---

**Cleanup**
- Delete ArgoCD application `project-local` via ArgoCD Web UI.
- Delete k3d cluster:  
  ```bash
  k3d cluster delete dwk-local
  ```