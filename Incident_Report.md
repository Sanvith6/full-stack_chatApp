# 🛠️ POST-MORTEM: Backend Crash in Minikube

**Date:** 2025-12-28  
**Project:** Full-Stack Chat App (3-Tier Architecture)  
**Status:** ✅ Resolved  

---

## 1. The Problem
The backend pod `backend-deployment` was stuck in a `CrashLoopBackOff` state in Kubernetes. 
- **Symptoms:** The pod started, ran for a few seconds, and then crashed repeatedly.
- **Error Status:** `BackOff restarting failed container`.

## 2. Root Cause Analysis (RCA)
Upon investigation, we identified three distinct issues causing the failure:

### A. Code Failure (The Immediate Crash) 🚨
*   **Issue:** The file `src/index.js` was written using **CommonJS** syntax (`const express = require('express')`), but the `package.json` was configured with `"type": "module"`, which forces **ES Modules** (ESM).
*   **Result:** Node.js threw a `ReferenceError: require is not defined` immediately upon startup, causing the process to exit.
*   **Why strict Docker builds didn't catch it:** Docker builds usually just copy files and run `npm install`. They don't typically *run* the app during the build (unless you have a `RUN` step that tests it). The error only surfaced at *runtime* inside the cluster.

### B. Networking Failure (The Database Connection) 🔗
*   **Issue:** The backend was trying to connect to a MongoDB hostname, but there was no **Kubernetes Service** exposing the MongoDB pod.
*   **Result:** Even if the app started, it would have timed out trying to resolve or connect to the database host.

### C. Configuration Failure (Missing Environment Variables) ⚙️
*   **Issue:** Critical environment variables (`MONGODB_URI`, `JWT_SECRET`, `PORT`) were missing from `backend-deployment.yml`.
*   **Result:** The application would default to unsafe configurations or fail to connect to external services.

---

## 3. The Solution (How we fixed it)

### Step 1: Code Refactoring
We rewrote `src/index.js` to use modern ES Module syntax:
- Changed `require(...)` to `import ... from ...`.
- Properly imported `connectDB` and `socket.io` logic to ensure the server starts with all dependencies.

### Step 2: Infrastructure Fixes
- **Created `k8s/mongodb-service.yml`**: A Kubernetes Service to verify stable networking between the backend and database.
- **Updated `k8s/backend-deployment.yml`**: Injected the correct `ENV` variables so the backend knows *where* the database is and *how* to authenticate.

---

## 4. DevOps Engineer Perspective: How to Debug This Next Time 🕵️‍♂️

 If you face a `CrashLoopBackOff` in the future, follow this "DevOps First Responder" workflow:

### 1️⃣ Check the Logs (The "Black Box" Recorder)
The most important step. Don't guess; look at why the app died.
```bash
# Get the pod name
kubectl get pods -n chat-app

# Read the logs of the crashed pod
kubectl logs <pod-name> -n chat-app --previous
```
*In this case, the logs would have clearly printed: `ReferenceError: require is not defined`.*

### 2️⃣ Verify Environment Parity
"It works on my machine" is the enemy.
- **Check:** Does your local `npm run dev` use the exact same command/variables as the Dockerfile?
- **Action:** Try running the Docker container locally before pushing to K8s:
  `docker run -e NODE_ENV=production my-image:v7`
  *If it crashes locally in Docker, it will crash in Kubernetes.*

### 3️⃣ Validate Connectivity (DNS & Services)
If the app starts but hangs:
- Exec into the pod and try to ping/curl the database.
```bash
kubectl exec -it <backend-pod> -- sh
# Inside pod:
ping mongodb-service
```

### 4️⃣ Use Liveness & Readiness Probes
Always define these in your YAML. configuration.
- **Liveness:** "Is the app dead? Restart me."
- **Readiness:** "Am I ready to take traffic? If not, don't send requests."

---

## 5. Summary
We moved the application from a misconfigured local state to a production-ready Kubernetes deployment by aligning the code module system with the config and correctly defining the infrastructure wiring (Services/Env Vars).
