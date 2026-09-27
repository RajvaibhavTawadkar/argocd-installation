# Argo CD Browser Access Issue — RCA


kubectl create namespace argocd

kubectl apply -n argocd --server-side --force-conflicts -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

to get password : $ kubectl get secret argocd-initial-admin-secret -n argocd -o yaml

sonaliraj1317@cloudshell:~ (project-5b757f62-dc19-4cae-98e)$ kubectl get secret argocd-initial-admin-secret -n argocd -o yaml 
apiVersion: v1
data:
  password: VURHbFAxUjFCOWFGLVpsSA==
kind: Secret
metadata:
  creationTimestamp: "2026-09-27T09:07:02Z"
  name: argocd-initial-admin-secret
  namespace: argocd
  resourceVersion: "1790500022272255002"
  uid: e7c1f80f-a5f6-434a-830b-219a75d013a7
type: Opaque

run this : sonaliraj1317@cloudshell:~ (project-5b757f62-dc19-4cae-98e)$ echo VURHbFAxUjFCOWFGLVpsSA== | base64 --decode 

## Overview

Argo CD was successfully deployed on a GKE Autopilot cluster, but the Argo CD UI was not opening correctly through Google Cloud Shell Web Preview.

Browser error:

```text
ERR_TOO_MANY_REDIRECTS
```

The issue was caused by an HTTP-to-HTTPS redirect interaction between Argo CD and the Cloud Shell Web Preview proxy.

---

## 1. Verify Argo CD Pods

```bash
kubectl get pods -n argocd
```

All Argo CD pods should be `Running`.

---

## 2. Verify Argo CD Service

```bash
kubectl get svc -n argocd
```

Important service:

```text
argocd-server   ClusterIP   ...   80/TCP,443/TCP
```

---

## 3. Initial Port-Forward Syntax Issue

Incorrect:

```bash
kubectl port-forward -n argocd svc argocd-server 8080:443
```

Error:

```text
Error from server (NotFound): pods "svc" not found
```

Correct:

```bash
kubectl port-forward -n argocd svc/argocd-server 8080:443
```

---

## 4. Verify Argo CD Locally

```bash
curl -k -I https://localhost:8080
```

Result:

```text
HTTP/1.1 200 OK
```

This confirmed:

* Argo CD server was healthy.
* Kubernetes service was working.
* Port-forward was working.
* The issue was not with the Argo CD pods.

---

## 5. Investigate Browser Redirect

Cloud Shell Web Preview showed:

```text
ERR_TOO_MANY_REDIRECTS
```

Test the HTTP service:

```bash
kubectl port-forward -n argocd svc/argocd-server 8081:80
```

Then:

```bash
curl -I http://localhost:8081
```

Result:

```text
HTTP/1.1 307 Temporary Redirect
Location: https://localhost:8081/
```

This confirmed that Argo CD was forcing HTTP traffic to HTTPS.

---

## 6. Root Cause

The issue was the interaction between:

```text
Browser
   ↓
Cloud Shell Web Preview
   ↓
Argo CD
```

Argo CD was redirecting HTTP requests to HTTPS while Cloud Shell Web Preview was already providing an HTTPS browser endpoint.

This resulted in a redirect loop:

```text
HTTP
 ↓
HTTPS
 ↓
Cloud Shell Proxy
 ↓
HTTP/HTTPS handling
 ↓
HTTPS redirect
 ↓
...
```

Browser result:

```text
ERR_TOO_MANY_REDIRECTS
```

### Important

A GCP firewall rule was **not required**.

The successful local `curl` test proved that Kubernetes networking and the port-forward were working.

---

## 7. Resolution

Configure Argo CD to run in insecure HTTP mode behind the Cloud Shell HTTPS proxy.

```bash
kubectl -n argocd patch configmap argocd-cmd-params-cm \
  --type merge \
  -p '{"data":{"server.insecure":"true"}}'
```

Verify:

```bash
kubectl -n argocd get configmap argocd-cmd-params-cm -o yaml
```

Expected:

```yaml
data:
  server.insecure: "true"
```

---

## 8. Restart Argo CD Server

```bash
kubectl -n argocd rollout restart deployment argocd-server
```

Check rollout:

```bash
kubectl -n argocd rollout status deployment argocd-server
```

Expected:

```text
deployment "argocd-server" successfully rolled out
```

---

## 9. Final Port-Forward

```bash
kubectl port-forward -n argocd svc/argocd-server 8081:80
```

Test:

```bash
curl -I http://localhost:8081
```

The HTTP-to-HTTPS redirect should no longer occur.

---

## 10. Access From Browser

Use:

```text
Cloud Shell → Web Preview → Port 8081
```

The URL will look like:

```text
https://8081-<cloud-shell-id>.cloudshell.dev
```

---

## 11. Why `127.0.0.1:8081` Did Not Work in the Local Browser

`kubectl port-forward` was running inside Cloud Shell.

Therefore:

```text
127.0.0.1:8081
```

refers to the Cloud Shell environment, not the Windows machine.

Traffic flow:

```text
Windows Browser
       ↓
Cloud Shell Web Preview
       ↓
Cloud Shell
       ↓
kubectl port-forward
       ↓
Argo CD Service
       ↓
Argo CD Server
```

---

## 12. Useful Commands

### Check Pods

```bash
kubectl get pods -n argocd
```

### Check Services

```bash
kubectl get svc -n argocd
```

### Check Argo CD Logs

```bash
kubectl logs -n argocd deployment/argocd-server --tail=50
```

### Check Configuration

```bash
kubectl -n argocd get configmap argocd-cmd-params-cm -o yaml
```

### Restart Argo CD

```bash
kubectl -n argocd rollout restart deployment argocd-server
```

### Check Rollout

```bash
kubectl -n argocd rollout status deployment/argocd-server
```

### HTTPS Port-Forward

```bash
kubectl port-forward -n argocd svc/argocd-server 8080:443
```

### HTTP Port-Forward

```bash
kubectl port-forward -n argocd svc/argocd-server 8081:80
```

### Test HTTP

```bash
curl -I http://localhost:8081
```

### Test HTTPS

```bash
curl -k -I https://localhost:8080
```

---

## 13. Argo CD Admin Password

Username:

```text
admin
```

Get the initial password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

---

## 14. Final RCA Summary

| Area            | Result                                                         |
| --------------- | -------------------------------------------------------------- |
| GKE Autopilot   | Working                                                        |
| Argo CD Pods    | Running                                                        |
| Argo CD Service | Working                                                        |
| Port-forward    | Working                                                        |
| Local curl      | HTTP 200                                                       |
| GCP Firewall    | Not the issue                                                  |
| Browser         | Redirect loop                                                  |
| Root Cause      | HTTP → HTTPS redirect interaction with Cloud Shell Web Preview |
| Resolution      | `server.insecure=true`                                         |
| Final Access    | Cloud Shell Web Preview                                        |

---

## 15. Interview-Style RCA

> I deployed Argo CD on GKE Autopilot and verified that all pods and services were healthy. Initially, there was a port-forward syntax issue, which I corrected. I then tested Argo CD using curl from Cloud Shell and received HTTP 200, so I ruled out Kubernetes and firewall issues. When accessing Argo CD through Cloud Shell Web Preview, I received an `ERR_TOO_MANY_REDIRECTS` error. Further investigation showed that Argo CD was redirecting HTTP traffic to HTTPS while Cloud Shell Web Preview was already providing an HTTPS proxy endpoint. This created a redirect loop. I configured Argo CD with `server.insecure=true`, restarted the Argo CD server, and exposed it through the HTTP service port. After that, Argo CD was accessible through Cloud Shell Web Preview.

---

## 16. Troubleshooting Pattern

```text
Browser Error
     ↓
Check Pods
     ↓
Check Service
     ↓
Check Port-Forward
     ↓
Test with curl
     ↓
Isolate Kubernetes vs Browser/Proxy
     ↓
Identify HTTP → HTTPS Redirect
     ↓
Configure server.insecure=true
     ↓
Restart Argo CD
     ↓
Port-Forward HTTP
     ↓
Open Cloud Shell Web Preview
     ↓
Verify Argo CD UI
```
