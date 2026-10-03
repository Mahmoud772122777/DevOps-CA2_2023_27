# Kubernetes Rolling Update and Rollback

These commands operate on the CA2 Deployment in the current `kubectl` context.
They are documented for manual use; no image update or rollback has been run as
part of preparing this guide.

## Check the current deployment

```powershell
kubectl get deployment devops-ca2 -o wide
kubectl get pods -l app=devops-ca2 -o wide
kubectl rollout status deployment/devops-ca2 --timeout=120s
kubectl rollout history deployment/devops-ca2
```

The Deployment manifest leaves `spec.strategy` unspecified. Kubernetes applies
its default `RollingUpdate` strategy. The current live Deployment reports
`maxSurge: 25%` and `maxUnavailable: 25%`; with two replicas this permits a
surge pod while preserving availability as readiness checks pass.

## Build and apply a new image version

Use a new, unique tag for each version. Build from the CA2 project root, then
load that exact tag into Minikube because the Deployment uses
`imagePullPolicy: Never`:

```powershell
docker build -f docker/Dockerfile -t devops-ca2:v2 .
minikube image load devops-ca2:v2
```

Replace `v2` with a tag that identifies your new image. Then update the
Deployment's container image to that tag:

```powershell
kubectl set image deployment/devops-ca2 devops-ca2=docker.io/library/devops-ca2:v2
kubectl rollout status deployment/devops-ca2 --timeout=120s
kubectl get deployment devops-ca2 -o wide
kubectl get pods -l app=devops-ca2 -o wide
kubectl rollout history deployment/devops-ca2
```

`kubectl set image` updates the live Deployment and creates a rollout revision;
it does not update `kubernetes/deployment.yaml`. To keep the version in source
control, also change the image in that manifest and apply it deliberately.

## Roll back

Review the revisions and, if needed, inspect a specific revision before
choosing a rollback target:

```powershell
kubectl rollout history deployment/devops-ca2
kubectl rollout history deployment/devops-ca2 --revision=4
```

Roll back to the immediately previous revision:

```powershell
kubectl rollout undo deployment/devops-ca2
kubectl rollout status deployment/devops-ca2 --timeout=120s
```

Or select a specific known-good revision:

```powershell
kubectl rollout undo deployment/devops-ca2 --to-revision=3
kubectl rollout status deployment/devops-ca2 --timeout=120s
```

Revision numbers change after each rollout. Choose the target based on the
current output of `kubectl rollout history`, not just the example numbers above.

## Verify after rollback

Confirm the Deployment and pods are available, and check which image is
currently assigned:

```powershell
kubectl get deployment devops-ca2 -o wide
kubectl get pods -l app=devops-ca2 -o wide
kubectl rollout history deployment/devops-ca2
```

If the CA2 port-forward is not already running, start it in a separate
PowerShell terminal and leave it running:

```powershell
kubectl port-forward service/devops-ca2 4000:4000
```

In another terminal, check the health endpoint and metrics:

```powershell
Invoke-RestMethod http://localhost:4000/health
curl.exe --fail http://localhost:4000/metrics
```

For submission evidence, capture the actual output of the status/history
commands before and after an approved rollout and rollback, plus the health
response after rollback. No rollout screenshots or results are included here;
those must be captured when you perform the commands.