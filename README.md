# Argo CD ApplicationSet Git Generator Example

This repository now uses plain manifest folders only.

- Every folder under `apps/*` becomes one Argo CD `Application`.
- Each app folder contains only `deployment.yml`, `service.yml`, and `configmap.yml`.
- Pushing a new folder with that same structure creates a new `Application` automatically.
- The generated `Application` is created in Argo CD, but it does not auto-sync by default.

## Repository layout

```text
.
├── bootstrap/
│   └── folder-applicationset.yaml
└── apps/
    ├── guestbook/
    │   ├── configmap.yml
    │   ├── deployment.yml
    │   └── service.yml
    └── payments/
        ├── configmap.yml
        ├── deployment.yml
        └── service.yml
```

## Prerequisites

- Argo CD is already running in your local cluster.
- The ApplicationSet CRD/controller is available.
- Argo CD can access the Git repository that stores these files.

## Step 1: Put these files in a Git repository

Use this repository itself, or move the same structure into a Git repository that your Argo CD instance can read.

If you use GitHub, the repository can be public for a quick test. If it is private, add the repo to Argo CD first.

## Step 2: Update the Git URL in the ApplicationSet

Edit `bootstrap/folder-applicationset.yaml` and replace both placeholder values:

- `https://github.com/YOUR_ORG/YOUR_REPO.git`

If your default branch is not `main`, change `revision` and `targetRevision` too.

## Step 3: Why `directory.recurse` is in the template

This example does not use Helm or Kustomize.

Because each app folder contains only raw YAML manifests, the template sets:

```yaml
      source:
        repoURL: https://github.com/YOUR_ORG/YOUR_REPO.git
        targetRevision: main
        path: '{{path}}'
        directory:
          recurse: true
```

That tells Argo CD to treat each folder as a plain manifest directory.

## Step 4: Apply the ApplicationSet

Apply this manifest into the `argocd` namespace:

```bash
kubectl apply -n argocd -f bootstrap/folder-applicationset.yaml
```

What happens next:

- Argo CD scans `apps/*` in the configured Git repo.
- It finds `apps/guestbook` and `apps/payments`.
- It creates two Argo CD applications: `guestbook` and `payments`.
- Each application targets its own namespace.

## Step 5: Confirm the generated Applications exist

Check the generated Argo CD applications:

```bash
kubectl get applications -n argocd
```

You should see applications named `guestbook` and `payments`.

Because this example does not enable automated sync, the applications are created but not deployed until you sync them.

## Step 6: Sync the applications

From the Argo CD UI or CLI, sync `guestbook` and `payments`.

If you use the CLI:

```bash
argocd app sync guestbook
argocd app sync payments
```

## Step 7: Push a new folder with the same structure

Create a new app folder by copying one of the examples:

```bash
cp -R apps/guestbook apps/orders
```

Inside `apps/orders`, keep exactly this structure:

```text
apps/orders/
├── configmap.yml
├── deployment.yml
└── service.yml
```

Then replace the names inside those files:

- change `guestbook` to `orders`
- change `guestbook-config` to `orders-config`
- update any config values you want

Then commit and push:

```bash
git add apps/orders
git commit -m "Add orders app"
git push
```

After the push:

- ApplicationSet rescans the repo
- it finds `apps/orders`
- it creates a new Argo CD `Application` named `orders`

## Step 8: Example raw manifest structure

Each folder should contain exactly these three files.

Example `configmap.yml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: orders-config
data:
  app-name: orders
  app-message: welcome-to-orders
```

Example `deployment.yml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orders
spec:
  replicas: 1
  selector:
    matchLabels:
      app: orders
  template:
    metadata:
      labels:
        app: orders
    spec:
      containers:
        - name: orders
          image: nginx:1.27-alpine
          envFrom:
            - configMapRef:
                name: orders-config
          ports:
            - containerPort: 80
```

Example `service.yml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: orders
spec:
  selector:
    app: orders
  ports:
    - name: http
      port: 80
      targetPort: 80
```

## Step 9: Remove a folder to remove the application

If you delete a folder and push that change, ApplicationSet stops generating that application.

Example:

```bash
git rm -r apps/orders
git commit -m "Remove orders app"
git push
```

Whether live Kubernetes resources are also pruned depends on the generated application's sync and deletion behavior.

## Notes about sync behavior

This example intentionally creates the Argo CD `Application` only.

It does that by avoiding:

- `spec.template.spec.syncPolicy.automated.prune: true`
- `spec.template.spec.syncPolicy.automated.selfHeal: true`

If you want new folders to deploy automatically after the Application is created, change the template to:

```yaml
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
```

## Common mistakes

- The new folder does not match the `apps/<name>/` pattern.
- The new folder is missing one of the YAML files.
- Resource names inside the new folder still use the old app name.
- The `repoURL` in the template does not match the actual Git repo.
- Argo CD cannot access the Git repo.
- The branch in Argo CD is not the branch you pushed.