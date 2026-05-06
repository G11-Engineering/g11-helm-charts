# Local Argo CD Demo On kind

This repository includes a separate local demo path that does not affect the existing production or staging app-of-apps setup.

## Demo Files

- `app-of-apps-kind.yaml` - bootstrap application for the local demo
- `argo-apps-kind/g11-kind-postgres.yaml` - PostgreSQL application
- `argo-apps-kind/g11-kind.yaml` - main G11 application
- `postgres/values-kind.yaml` - kind-compatible Postgres values
- `g11/values-kind.yaml` - kind-compatible app values

## What This Demo Assumes

The local demo still requires the controllers that these charts depend on:

- Istio
- Argo Rollouts
- Crunchy Data PostgreSQL Operator
- Metrics Server

For local demo purposes, the included values files disable:

- Sealed Secrets
- cert-manager / ACME certificates
- AWS EFS storage class creation

## Before You Apply It

Argo CD syncs from a Git-accessible repository, not from your uncommitted working tree.

If you are demoing from a fork or a non-`main` branch, update the `repoURL` and `targetRevision` fields in:

- `app-of-apps-kind.yaml`
- `argo-apps-kind/g11-kind-postgres.yaml`
- `argo-apps-kind/g11-kind.yaml`

## Bootstrap The Demo

```bash
kubectl apply -f app-of-apps-kind.yaml
kubectl get applications -n argocd
```

Expected applications:

- `app-of-apps-kind`
- `g11-kind-postgres`
- `g11-kind`

The Postgres application uses the Helm release name `hippo-postgres` so that the generated database secret names match what the G11 chart expects.

## Accessing The Demo

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
kubectl port-forward svc/frontend -n g11-kind 3000:3000
```

Open the Argo CD UI at `https://localhost:8080`.

The frontend chart is configured with `gateway.hostname=g11.127.0.0.1.nip.io` for local demo use. Istio 1.28 rejects `localhost` as a Gateway/VirtualService host because it is not an FQDN, so use the loopback `nip.io` hostname instead. If you need a full end-to-end auth callback flow, replace the demo secrets and use a hostname or ingress setup that matches your OAuth callback configuration.
