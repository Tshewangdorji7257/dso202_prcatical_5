# Practical: Environment-Specific Configuration with Kustomize on Kind

Complete step-by-step guide: setup, manifest files, every task in order, model answers for the report, and cleanup. Run each block from top to bottom.

---

## Task 0: Pre-flight

You need Docker, `kind`, and `kubectl` (v1.14+ for `-k`). Check them:

```bash
docker --version
kind --version
kubectl version --client -o yaml
```

Create the cluster (1 control-plane and 2 workers, matching the lab topology):

```bash
mkdir -p ~/kustomize-lab && cd ~/kustomize-lab

cat > kind-config.yaml <<'EOF'
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
  - role: worker
EOF

kind create cluster --name kustomize-lab --config kind-config.yaml
```

Run the required checks:

```bash
kubectl cluster-info --context kind-kustomize-lab
kubectl get nodes -o wide
kubectl version --client -o yaml
kubectl kustomize --help | head -5
```

Confirm all three nodes are `Ready` and the context is `kind-kustomize-lab`. Take screenshots for the report.

---

## Task 1: Create the repository (one script creates every file)

Run this from `~/kustomize-lab`. It creates the base and the dev, staging, and prod overlays.

```bash
mkdir -p examples/webapp/base examples/webapp/overlays
cd examples/webapp

# ---------- BASE ----------
cat > base/deployment.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  labels:
    app.kubernetes.io/name: webapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: webapp
  template:
    metadata:
      labels:
        app.kubernetes.io/name: webapp
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 50m
              memory: 32Mi
            limits:
              cpu: 100m
              memory: 64Mi
          volumeMounts:
            - name: web-content
              mountPath: /usr/share/nginx/html
      volumes:
        - name: web-content
          configMap:
            name: web-content
EOF

cat > base/service.yaml <<'EOF'
apiVersion: v1
kind: Service
metadata:
  name: webapp
  labels:
    app.kubernetes.io/name: webapp
spec:
  selector:
    app.kubernetes.io/name: webapp
  ports:
    - port: 80
      targetPort: 80
EOF

cat > base/index.html <<'EOF'
<h1>BASE page - should never be seen in a real environment</h1>
EOF

cat > base/kustomization.yaml <<'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
configMapGenerator:
  - name: web-content
    files:
      - index.html
EOF

# ---------- OVERLAY GENERATOR FUNCTION (dev/staging/prod) ----------
mk_overlay() {
  env=$1; replicas=$2; tag=$3
  d=overlays/$env
  mkdir -p $d

  cat > $d/namespace.yaml <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: webapp-$env
EOF

  cat > $d/index.html <<EOF
<h1>Hello from the ${env^^} environment</h1>
<p>Served by NGINX via Kustomize overlay: $env</p>
EOF

  cat > $d/kustomization.yaml <<EOF
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: webapp-$env
resources:
  - ../../base
  - namespace.yaml
labels:
  - pairs:
      environment: $env
    includeSelectors: false
    includeTemplates: true
replicas:
  - name: webapp
    count: $replicas
images:
  - name: nginx
    newTag: "$tag"
configMapGenerator:
  - name: web-content
    behavior: replace
    files:
      - index.html
EOF
}

mk_overlay dev     1 1.25-alpine
mk_overlay staging 2 1.25-alpine
mk_overlay prod    3 1.26-alpine

# ---------- PROD-ONLY PATCH (strategic merge) ----------
cat > overlays/prod/patch-resources.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  annotations:
    training.example.com/tier: production
spec:
  template:
    spec:
      containers:
        - name: nginx
          resources:
            requests:
              cpu: 200m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 256Mi
EOF

cat >> overlays/prod/kustomization.yaml <<'EOF'
patches:
  - path: patch-resources.yaml
EOF

cd ../..   # back to ~/kustomize-lab
```

View the tree (use `find` if `tree` isn't installed):

```bash
tree examples/webapp || find examples/webapp | sort
```

Expected structure:

```text
examples/webapp/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── index.html
│   └── kustomization.yaml
└── overlays/
    ├── dev/      (index.html, kustomization.yaml, namespace.yaml)
    ├── staging/  (index.html, kustomization.yaml, namespace.yaml)
    └── prod/     (index.html, kustomization.yaml, namespace.yaml, patch-resources.yaml)
```

### Answers to the Task 1 questions (for the report)

1. **Files that exist only once:** `base/deployment.yaml`, `base/service.yaml`, `base/kustomization.yaml`, and the base `index.html`.
2. **Values that differ:** namespace, `environment` label, replica count, image tag, resource requests/limits (prod), web page content, annotations (prod).
3. **Where the differences live:**
   - Namespace, labels, replicas, images, and the ConfigMap are set in each overlay's `kustomization.yaml`.
   - Page content is in each overlay's `index.html`.
   - Resources and the annotation are in `overlays/prod/patch-resources.yaml`.

---

## Task 2: Render the base (do not apply)

```bash
kubectl kustomize examples/webapp/base
kubectl kustomize examples/webapp/base | grep '^kind:'
kubectl kustomize examples/webapp/base | grep 'name: web-content'
```

Find these in the output:

- The Deployment and the Service.
- The ConfigMap named `web-content-<hash>`.
- The Deployment's `volumes.configMap.name`, rewritten to `web-content-<hash>`.

**Checkpoint answer:** The `configMapGenerator` appends a suffix computed from a hash of the ConfigMap's contents, so the name is `web-content-xxxxxxxxxx`. Kustomize then rewrites every reference to `web-content` (here the Deployment's volume) to the hashed name. The hash makes the name content-addressed, so any content change produces a new name and forces Pods to roll.

---

## Task 3: Compare dev and prod without touching the cluster

```bash
kubectl kustomize examples/webapp/overlays/dev  > /tmp/webapp-dev.yaml
kubectl kustomize examples/webapp/overlays/prod > /tmp/webapp-prod.yaml
diff -u /tmp/webapp-dev.yaml /tmp/webapp-prod.yaml || true
```

**Differences you will see (pick at least five):**

| # | Category | dev | prod |
|---|---|---|---|
| 1 | Namespace | `webapp-dev` | `webapp-prod` |
| 2 | Environment label | `environment: dev` | `environment: prod` |
| 3 | Replicas | 1 | 3 |
| 4 | Image tag | `nginx:1.25-alpine` | `nginx:1.26-alpine` |
| 5 | Resources | requests 50m/32Mi, limits 100m/64Mi | requests 200m/128Mi, limits 500m/256Mi |
| 6 | Web content and ConfigMap hash | DEV page | PROD page |
| 7 | Annotation | none | `training.example.com/tier: production` |

**Principle:** An overlay is valuable when you can explain the environment difference by reading a small number of lines instead of reviewing a complete duplicate manifest.

---

## Task 4: Deploy dev safely (render, diff, apply, verify)

```bash
kubectl kustomize examples/webapp/overlays/dev
kubectl diff -k examples/webapp/overlays/dev || true
kubectl apply -k examples/webapp/overlays/dev

kubectl get all -n webapp-dev
kubectl get configmap -n webapp-dev
kubectl rollout status deployment/webapp -n webapp-dev
```

`kubectl diff` shows everything as new (`+`) on the first run, and it may complain that the namespace doesn't exist yet. That is expected, and `|| true` keeps the shell going. Screenshot the `kubectl get all -n webapp-dev` output for the report.

The Deployment keeps the base name `webapp`; the namespace separates the environment.

---

## Task 5: Reach the application

Terminal 1:

```bash
kubectl port-forward -n webapp-dev service/webapp 8080:80
```

Terminal 2:

```bash
curl http://127.0.0.1:8080
```

Expected output: `Hello from the DEV environment`. Press `Ctrl+C` in terminal 1 to stop the port-forward.

---

## Task 6: Prove the ConfigMap hash to rollout chain

**Before the change** (save this output for the report):

```bash
kubectl get configmap -n webapp-dev
kubectl get pods -n webapp-dev -o wide
```

**Edit the page:**

```bash
cat > examples/webapp/overlays/dev/index.html <<'EOF'
<h1>DEV v2 — configuration changed</h1>
<p>Served by NGINX via Kustomize overlay: dev</p>
EOF
```

**Render, apply, and wait:**

```bash
kubectl kustomize examples/webapp/overlays/dev | grep 'name: web-content'
kubectl apply -k examples/webapp/overlays/dev
kubectl rollout status deployment/webapp -n webapp-dev
```

**After the change** (compare with the earlier output):

```bash
kubectl get configmap -n webapp-dev
kubectl get pods -n webapp-dev -o wide
```

The ConfigMap name suffix and the Pod name will both differ. The old ConfigMap may still be listed briefly, since it is left behind unless you prune. Optionally re-run the port-forward and `curl` to see "DEV v2".

**Mandatory explanation for the report:**

```text
file content changed
→ generated ConfigMap content changed
→ generated ConfigMap name hash changed
→ Deployment reference changed
→ Deployment pod template changed
→ rollout occurred
```

> I changed `overlays/dev/index.html`, so the content of the generated ConfigMap changed. Kustomize computes the ConfigMap name hash from the content, so the name changed from `web-content-<old>` to `web-content-<new>`. Kustomize rewrote the Deployment's volume reference to the new name, which changed the Deployment's pod template. Kubernetes treats a pod template change as a new revision, so it created a new ReplicaSet and rolled the Pods, and the new Pods serve the new content.

---

## Task 7: Deploy staging and prod

```bash
kubectl diff -k examples/webapp/overlays/staging || true
kubectl apply -k examples/webapp/overlays/staging

kubectl diff -k examples/webapp/overlays/prod || true
kubectl apply -k examples/webapp/overlays/prod
```

Wait for the rollouts, then verify all namespaces:

```bash
kubectl rollout status deployment/webapp -n webapp-staging
kubectl rollout status deployment/webapp -n webapp-prod

kubectl get deploy -A -l app.kubernetes.io/name=webapp
kubectl get pods -A -l app.kubernetes.io/name=webapp -o wide
```

**Replica counts to record:**

| Environment | Replicas |
|---|---|
| dev | 1 |
| staging | 2 |
| prod | 3 |

---

## Task 8: Inspect the prod patch

```bash
cat examples/webapp/overlays/prod/patch-resources.yaml
kubectl kustomize examples/webapp/overlays/prod | grep -B2 -A8 'resources:'
```

**Answers:**

1. **Deleted or merged?** Merged. This is a strategic merge patch. Containers are matched by `name: nginx`, and only the fields in the patch (requests/limits values) were overridden. Everything else on the container (image, ports, volumeMounts) was kept.
2. **Who owns the production resource policy?** The prod overlay, in `overlays/prod/patch-resources.yaml`. The base only holds a small default.
3. **Why a patch is better than copying `deployment.yaml` into `prod/`:**
   - The patch is about 10 lines, so a reviewer sees exactly what differs.
   - A copy would fork the Deployment, so a base fix (a new probe, say) would have to be repeated in every copy or prod would silently drift.
   - The patch keeps a single source of truth in the base.

---

## Task 9: Create your own QA overlay

```bash
mkdir -p examples/webapp/overlays/qa
cd examples/webapp/overlays/qa

cat > namespace.yaml <<'EOF'
apiVersion: v1
kind: Namespace
metadata:
  name: webapp-qa
EOF

cat > index.html <<'EOF'
<h1>Hello from the QA environment</h1>
<p>Unique QA page - owned by qa-team</p>
EOF

cat > patch-annotation.yaml <<'EOF'
- op: add
  path: /metadata/annotations
  value:
    training.example.com/owner: qa-team
EOF

cat > kustomization.yaml <<'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: webapp-qa
resources:
  - ../../base
  - namespace.yaml
labels:
  - pairs:
      environment: qa
    includeSelectors: false
    includeTemplates: true
replicas:
  - name: webapp
    count: 2
configMapGenerator:
  - name: web-content
    behavior: replace
    files:
      - index.html
patches:
  - path: patch-annotation.yaml
    target:
      group: apps
      version: v1
      kind: Deployment
      name: webapp
EOF

cd ../../../..   # back to ~/kustomize-lab
```

Render it and **do not proceed until the annotation appears**:

```bash
kubectl kustomize examples/webapp/overlays/qa | grep -B3 -A3 'training.example.com/owner'
```

If you see `training.example.com/owner: qa-team` under the Deployment's `metadata.annotations`, continue:

```bash
kubectl diff -k examples/webapp/overlays/qa || true
kubectl apply -k examples/webapp/overlays/qa
kubectl rollout status deployment/webapp -n webapp-qa
kubectl get deployment webapp -n webapp-qa -o yaml
```

The `add` on `/metadata/annotations` is safe here because the base Deployment has no annotations. If a base Deployment already had annotations, use the JSON Pointer path `/metadata/annotations/training.example.com~1owner` instead (`/` is escaped as `~1`).

---

## Task 10: Cleanup

```bash
kubectl delete -k examples/webapp/overlays/dev
kubectl delete -k examples/webapp/overlays/staging
kubectl delete -k examples/webapp/overlays/prod
kubectl delete -k examples/webapp/overlays/qa

kubectl get ns | grep 'webapp-' || true
```

Empty output from the last command means everything is gone, because each overlay includes its namespace and deleting it removes everything inside. To delete the cluster at the very end (optional, only after collecting all evidence):

```bash
kind delete cluster --name kustomize-lab
```

---

## Challenge extension: `namePrefix` sandbox (render only)

```bash
mkdir -p examples/webapp/overlays/sandbox

cat > examples/webapp/overlays/sandbox/index.html <<'EOF'
<h1>SANDBOX</h1>
EOF

cat > examples/webapp/overlays/sandbox/kustomization.yaml <<'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: webapp-sandbox
namePrefix: sandbox-
resources:
  - ../../base
configMapGenerator:
  - name: web-content
    behavior: replace
    files:
      - index.html
EOF
```

**Write your prediction first** (before rendering):

| Resource | Predicted new name |
|---|---|
| Deployment | `sandbox-webapp` |
| Service | `sandbox-webapp` |
| ConfigMap | `sandbox-web-content-<hash>` |
| Deployment's `volumes.configMap.name` | `sandbox-web-content-<hash>` (reference updated automatically) |
| Selectors and labels | unchanged, since `namePrefix` only touches names and references |

Then render and compare:

```bash
kubectl kustomize examples/webapp/overlays/sandbox | grep -E 'name:|kind:'
```

The takeaway is that Kustomize updates references (the ConfigMap volume reference) along with names, so nothing breaks.

---

## Practical report checklist

Use the same section structure as your previous practicals (aim, objectives, tasks, output, conclusion), and include:

1. Repository `tree` output (Task 1).
2. Rendered dev output excerpt (Task 4).
3. Dev vs prod `diff` excerpt (Task 3).
4. `kubectl get all -n webapp-dev` (Task 4).
5. ConfigMap names before and after the change (Task 6).
6. Pod names before and after the rollout (Task 6).
7. QA overlay files: all four (Task 9).
8. **Strategic merge vs JSON 6902:**
   - *Strategic merge* patches are written as a partial resource. Kubernetes-aware merge rules match list items by key (containers by `name`) and merge maps. It's easy to read, and it was used for `patch-resources.yaml`.
   - *JSON 6902* patches are an explicit list of operations (`add`, `replace`, `remove`) against exact paths. It's precise and works on any resource, but list items are addressed by index and the syntax is more fragile. It was used for the QA annotation.
9. **Reflection (write your own):** deliberately make one mistake, for example change `name: webapp` to `name: webap` in the QA patch target, or misspell `configMapGenerator`. Run `kubectl kustomize` and note the error or missing output. That gives you a real example of rendered output exposing the problem before it reached the cluster. Then fix it and record both the mistake and the fix.