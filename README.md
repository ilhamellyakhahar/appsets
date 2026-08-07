# Argo CD ApplicationSet Structure

This repository uses a project and service folder layout.

- ApplicationSet manifests are in `applicationsets/`
- Workload manifests are in `development/`
- Each service folder becomes one Argo CD Application

## Current structure

```text
.
├── applicationsets
│   ├── project-1.yml
│   └── project-2.yml
└── development
    ├── project-1
    │   ├── service-1
    │   │   ├── configmap.yml
    │   │   ├── deployment.yml
    │   │   └── service.yml
    │   └── service-2
    │       ├── configmap.yml
    │       ├── deployment.yml
    │       └── service.yml
    └── project-2
        ├── service-1
        │   ├── configmap.yml
        │   ├── deployment.yml
        │   └── service.yml
        └── service-2
            ├── configmap.yml
            ├── deployment.yml
            └── service.yml
```

## How manifests map

- `applicationsets/project-1.yml` scans `development/project-1/*`
- `applicationsets/project-2.yml` scans `development/project-2/*`

Generated Application names are unique:

- `project-1-service-1`, `project-1-service-2`
- `project-2-service-1`, `project-2-service-2`

Destination namespaces:

- Project 1 services deploy to namespace `project-1`
- Project 2 services deploy to namespace `project-2`

Both ApplicationSets use automated sync with prune and self-heal.

## Apply

```bash
kubectl apply -f applicationsets/project-1.yml
kubectl apply -f applicationsets/project-2.yml
```

## Verify

```bash
kubectl get applications -n default
kubectl get ns project-1 project-2
kubectl get all -n project-1
kubectl get all -n project-2
```
