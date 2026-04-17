# AWS Three-Tier Application Deployment on EKS

A complete three-tier application (Frontend, Backend, Database) deployed on Amazon EKS with automated CI/CD pipeline using GitHub Actions.

## 📋 Table of Contents

- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Issues Faced & Resolutions](#issues-faced--resolutions)
- [Quick Start](#quick-start)
- [Deployment Guide](#deployment-guide)
- [CI/CD Pipeline](#cicd-pipeline)
- [Application Access](#application-access)
- [Troubleshooting](#troubleshooting)

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────┐
│          Internet (Users/Browsers)              │
└────────────────┬────────────────────────────────┘
                 │
                 ▼
    ┌────────────────────────────┐
    │   AWS Application Load     │
    │   Balancer (ALB) + Ingress │
    └────────┬───────────────────┘
             │
    ┌────────┴──────────┐
    │                   │
    ▼                   ▼
┌─────────────┐   ┌──────────────┐
│  Frontend   │   │   Backend    │
│  (React)    │   │  (Express)   │
│  Port 3000  │   │  Port 3500   │
└─────────────┘   └──────┬───────┘
                         │
                         ▼
                 ┌─────────────────┐
                 │   MongoDB       │
                 │   Port 27017    │
                 │  (Persistent)   │
                 └─────────────────┘
```

**Stack:**
- **Frontend**: React with Material-UI (Todo List UI)
- **Backend**: Node.js Express API (CRUD operations)
- **Database**: MongoDB with persistent storage
- **Orchestration**: Amazon EKS (Kubernetes)
- **Ingress**: AWS ALB (Application Load Balancer)
- **CI/CD**: GitHub Actions
- **Registry**: AWS ECR (Elastic Container Registry)

---

## 📋 Prerequisites

- AWS Account with EKS cluster
- `kubectl` configured to access EKS cluster
- Docker installed locally
- AWS CLI configured with credentials
- GitHub repository with this code
- Node.js 14+ (for local development)

---

## 🔧 Issues Faced & Resolutions

### Issue 1: MongoDB Pod Stuck in Pending State

**Problem:**
```
NAME                      READY   STATUS    RESTARTS   AGE
mongodb-55d4c585b-g8b86   0/1     Pending   0          6m24s
```

**Root Causes:**
1. PersistentVolume (PV) manifest included `namespace: three-tier` (PVs are cluster-scoped)
2. PVC and PV had mismatched `storageClassName`
3. MongoDB container command used `numactl` which was unavailable in the image

**Resolution:**
- ✅ Removed `namespace` from PV manifest
- ✅ Added `storageClassName: manual` to both PV and PVC for deterministic binding
- ✅ Added `hostPath.type: DirectoryOrCreate` to auto-create missing host directories
- ✅ Replaced `command: numactl` with `args` for safer MongoDB startup

**File Changes:**
- [`Generic_Application_forDemo/Kubernetes-Manifests-file/Database/pv.yaml`](Generic_Application_forDemo/Kubernetes-Manifests-file/Database/pv.yaml)
- [`Generic_Application_forDemo/Kubernetes-Manifests-file/Database/pvc.yaml`](Generic_Application_forDemo/Kubernetes-Manifests-file/Database/pvc.yaml)
- [`Generic_Application_forDemo/Kubernetes-Manifests-file/Database/deployment.yaml`](Generic_Application_forDemo/Kubernetes-Manifests-file/Database/deployment.yaml)

---

### Issue 2: AWS Load Balancer Controller IAM Permission Denied

**Problem:**
```
AccessDenied: User: arn:aws:sts::...:assumed-role/AmazonEKSLoadBalancerControllerRoleEKS/... 
is not authorized to perform: elasticloadbalancing:DescribeListenerAttributes
```

**Root Cause:**
The IAM role attached to the Load Balancer Controller was missing required ELBv2 permissions.

**Resolution:**
- ✅ Added `elasticloadbalancing:DescribeListenerAttributes` and `ModifyListenerAttributes` to controller IAM role
- ✅ Restarted the controller deployment to pick up new permissions
- ✅ Controller successfully created ALB and provisioned Ingress

**Commands Used:**
```bash
aws iam put-role-policy \
  --role-name AmazonEKSLoadBalancerControllerRoleEKS \
  --policy-name ALBCExtraListenerAttributesPolicy \
  --policy-document file:///tmp/albc-extra-policy.json

kubectl rollout restart deployment aws-load-balancer-controller -n kube-system
```

---

### Issue 3: Ingress Host-Based Routing Required Custom DNS

**Problem:**
- Ingress was configured with host rule: `backend.arnabadhikarydevops.preparations`
- ALB DNS alone (`k8s-....elb.amazonaws.com`) returned `404`
- Custom DNS record was not resolving

**Root Cause:**
Host-based routing in Ingress requires the request `Host` header to match the configured hostname. Direct ALB DNS did not match.

**Resolution:**
- ✅ Removed host-based routing from Ingress manifest
- ✅ Kept only path-based routing (`/api` → backend, `/` → frontend)
- ✅ ALB DNS now serves both frontend and backend directly

**File Changes:**
- [`Generic_Application_forDemo/Kubernetes-Manifests-file/ingress.yaml`](Generic_Application_forDemo/Kubernetes-Manifests-file/ingress.yaml)

---

### Issue 4: Frontend Tasks.map is Not a Function

**Problem:**
```
TypeError: tasks.map is not a function
```

**Root Causes:**
1. `REACT_APP_BACKEND_URL` (React build-time env var) was not baked into the Docker image
2. Frontend called undefined URL, received HTML response instead of JSON array
3. No guard to check if API response is actually an array

**Resolution:**
- ✅ Updated frontend Dockerfile to accept `REACT_APP_BACKEND_URL` as build argument:
  ```dockerfile
  ARG REACT_APP_BACKEND_URL
  ENV REACT_APP_BACKEND_URL=$REACT_APP_BACKEND_URL
  ```
- ✅ Added fallback URL in `taskServices.js`: `process.env.REACT_APP_BACKEND_URL || "/api/tasks"`
- ✅ Added array guard in `Tasks.js`: `Array.isArray(data) ? data : []`
- ✅ Rebuilt frontend with ALB URL baked in during build

**File Changes:**
- [`Generic_Application_forDemo/frontend/Dockerfile`](Generic_Application_forDemo/frontend/Dockerfile)
- [`Generic_Application_forDemo/frontend/src/services/taskServices.js`](Generic_Application_forDemo/frontend/src/services/taskServices.js)
- [`Generic_Application_forDemo/frontend/src/Tasks.js`](Generic_Application_forDemo/frontend/src/Tasks.js)

---

### Issue 5: Tasks List Rendered but Text Blank

**Problem:**
- Task checkboxes and delete buttons appeared
- Task text field was empty

**Root Cause:**
Tasks created via old API calls had different field names or empty `task` field in MongoDB.

**Resolution:**
- ✅ Verified MongoDB schema uses `task` field (not `title`, `text`, etc.)
- ✅ Cleared old/malformed records from MongoDB:
  ```bash
  db.tasks.deleteMany({ task: { $in: [null, ""] } })
  ```
- ✅ Added new task via UI with correct field name
- ✅ Task text now displays correctly

---

## 🚀 Quick Start

### 1. Clone Repository
```bash
git clone https://github.com/your-org/AWS_Three_Tier_App_Deployment_EKS.git
cd AWS_Three_Tier_App_Deployment_EKS
```

### 2. Set Environment Variables
```bash
export AWS_ACCOUNT_ID=356302822804
export AWS_REGION=us-east-1
export EKS_CLUSTER_NAME=your-cluster-name
export ALB=http://k8s-threetie-mainlb-1dd958d0ec-2137855790.us-east-1.elb.amazonaws.com
```

### 3. Configure AWS Credentials
```bash
aws configure
# Enter your AWS Access Key ID and Secret Access Key
```

### 4. Update kubeconfig
```bash
aws eks update-kubeconfig --region $AWS_REGION --name $EKS_CLUSTER_NAME
```

### 5. Deploy Application
```bash
# Create namespace
kubectl create namespace three-tier

# Deploy all layers
kubectl apply -f Generic_Application_forDemo/Kubernetes-Manifests-file/Database/
kubectl apply -f Generic_Application_forDemo/Kubernetes-Manifests-file/Backend/
kubectl apply -f Generic_Application_forDemo/Kubernetes-Manifests-file/Frontend/
kubectl apply -f Generic_Application_forDemo/Kubernetes-Manifests-file/ingress.yaml

# Verify deployment
kubectl get pods -n three-tier
kubectl get ingress -n three-tier
```

---

## 📦 Deployment Guide

### Manual Deployment Steps

#### Step 1: Deploy Database Layer
```bash
kubectl apply -f Generic_Application_forDemo/Kubernetes-Manifests-file/Database/secrets.yaml
kubectl apply -f Generic_Application_forDemo/Kubernetes-Manifests-file/Database/pv.yaml
kubectl apply -f Generic_Application_forDemo/Kubernetes-Manifests-file/Database/pvc.yaml
kubectl apply -f Generic_Application_forDemo/Kubernetes-Manifests-file/Database/service.yaml
kubectl apply -f Generic_Application_forDemo/Kubernetes-Manifests-file/Database/deployment.yaml

# Wait for MongoDB
kubectl rollout status deployment/mongodb -n three-tier --timeout=120s
```

#### Step 2: Deploy Backend API
```bash
kubectl apply -f Generic_Application_forDemo/Kubernetes-Manifests-file/Backend/service.yaml
kubectl apply -f Generic_Application_forDemo/Kubernetes-Manifests-file/Backend/deployment.yaml

# Wait for Backend
kubectl rollout status deployment/api -n three-tier --timeout=120s
```

#### Step 3: Deploy Frontend
```bash
kubectl apply -f Generic_Application_forDemo/Kubernetes-Manifests-file/Frontend/deployment.yaml
kubectl apply -f Generic_Application_forDemo/Kubernetes-Manifests-file/Frontend/service.yaml

# Wait for Frontend
kubectl rollout status deployment/frontend -n three-tier --timeout=120s
```

#### Step 4: Apply Ingress & Load Balancer
```bash
kubectl apply -f Generic_Application_forDemo/Kubernetes-Manifests-file/ingress.yaml

# Get ALB DNS
kubectl get ingress mainlb -n three-tier -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```

#### Step 5: Verify Deployment
```bash
kubectl get all -n three-tier
kubectl get ingress -n three-tier -o wide
```

---

## 🔄 CI/CD Pipeline

### GitHub Actions Workflow

The pipeline is defined in [`.github/workflows/cicd.yaml`](.github/workflows/cicd.yaml).

**Workflow Steps:**

1. **Build Backend** (Parallel with Frontend)
   - Build Docker image from `Generic_Application_forDemo/backend/`
   - Push to ECR with tags: `<commit-sha>` and `latest`

2. **Build Frontend** (Parallel with Backend)
   - Build Docker image with `REACT_APP_BACKEND_URL` baked in
   - Push to ECR with tags: `<commit-sha>` and `latest`

3. **Deploy to EKS** (After both builds complete)
   - Update kubeconfig
   - Deploy all three layers
   - Update image tags to `<commit-sha>` for reproducibility
   - Verify rollout status

### Required GitHub Secrets

Go to GitHub → **Settings** → **Secrets and variables** → **Actions** → **New repository secret**:

| Secret Name | Value | Example |
|---|---|---|
| `AWS_ACCESS_KEY_ID` | IAM user access key | `AKIA...` |
| `AWS_SECRET_ACCESS_KEY` | IAM user secret key | `wJal...` |
| `AWS_ACCOUNT_ID` | AWS Account ID | `356302822804` |
| `EKS_CLUSTER_NAME` | EKS cluster name | `mycluster` |
| `REACT_APP_BACKEND_URL` | ALB endpoint | `http://k8s-threetie-mainlb.../api/tasks` |

### Trigger Pipeline

Commit and push to `main` branch:
```bash
git add .
git commit -m "Add new feature"
git push origin main
```

Pipeline automatically triggers on push to `main`. Pull requests also build images but skip deployment.

---

## 🌐 Application Access

### Access via ALB DNS (Recommended)

**Frontend:** 
```
http://k8s-threetie-mainlb-1dd958d0ec-2137855790.us-east-1.elb.amazonaws.com/
```

**API Endpoints:**
```
http://k8s-threetie-mainlb-1dd958d0ec-2137855790.us-east-1.elb.amazonaws.com/api/tasks
http://k8s-threetie-mainlb-1dd958d0ec-2137855790.us-east-1.elb.amazonaws.com/ok
```

### Test via curl

```bash
# Get all tasks
curl http://k8s-threetie-mainlb-1dd958d0ec-2137855790.us-east-1.elb.amazonaws.com/api/tasks

# Health check
curl http://k8s-threetie-mainlb-1dd958d0ec-2137855790.us-east-1.elb.amazonaws.com/ok

# Add task (from browser or curl)
curl -X POST http://k8s-threetie-mainlb-1dd958d0ec-2137855790.us-east-1.elb.amazonaws.com/api/tasks \
  -H "Content-Type: application/json" \
  -d '{"task":"My new task","completed":false}'
```

---

## 🐛 Troubleshooting

### Pod Stuck in Pending

```bash
# Check pod events
kubectl describe pod <pod-name> -n three-tier

# Check PVC binding
kubectl get pvc -n three-tier
kubectl describe pvc <pvc-name> -n three-tier

# Check PV availability
kubectl get pv
kubectl describe pv <pv-name>
```

### Frontend Not Loading Tasks

1. **Check API URL in browser console:**
   ```js
   console.log(process.env.REACT_APP_BACKEND_URL)
   ```

2. **Check frontend logs:**
   ```bash
   kubectl logs -f deployment/frontend -n three-tier
   ```

3. **Check backend logs:**
   ```bash
   kubectl logs -f deployment/api -n three-tier
   ```

4. **Check API connectivity:**
   ```bash
   kubectl exec -it deployment/frontend -n three-tier -- \
     curl http://api:3500/ok
   ```

### MongoDB Connection Issues

```bash
# Check MongoDB pod status
kubectl describe pod -l app=mongodb -n three-tier

# Check MongoDB logs
kubectl logs -f deployment/mongodb -n three-tier

# Test MongoDB from another pod
kubectl run -it --rm debug --image=mongo:4.4.6 --restart=Never -- \
  mongosh mongodb://mongodb-svc:27017/todo \
  -u admin -p password123 --authenticationDatabase admin
```

### Ingress Not Getting Address

```bash
# Check ingress events
kubectl describe ingress mainlb -n three-tier

# Check ALB controller logs
kubectl logs -f deployment/aws-load-balancer-controller -n kube-system

# Check security groups
aws ec2 describe-security-groups --filters "Name=tag:kubernetes.io/cluster/<cluster-name>,Values=owned"
```

### Clean Everything & Restart

```bash
# Delete all resources
kubectl delete namespace three-tier

# Recreate and deploy
kubectl create namespace three-tier
kubectl apply -f Generic_Application_forDemo/Kubernetes-Manifests-file/
```

---

## 📁 Project Structure

```
Generic_Application_forDemo/
├── backend/
│   ├── Dockerfile
│   ├── index.js
│   ├── package.json
│   ├── models/
│   │   └── task.js
│   └── routes/
│       └── tasks.js
├── frontend/
│   ├── Dockerfile
│   ├── package.json
│   ├── public/
│   └── src/
│       ├── App.js
│       ├── Tasks.js
│       ├── App.css
│       └── services/
│           └── taskServices.js
└── Kubernetes-Manifests-file/
    ├── ingress.yaml
    ├── Database/
    │   ├── deployment.yaml
    │   ├── pv.yaml
    │   ├── pvc.yaml
    │   ├── secrets.yaml
    │   └── service.yaml
    ├── Backend/
    │   ├── deployment.yaml
    │   └── service.yaml
    └── Frontend/
        ├── deployment.yaml
        └── service.yaml
```

---

## 🔐 Security Notes

- MongoDB credentials stored in Kubernetes Secret: `mongo-sec`
- ECR images are private; authentication via IAM role
- ALB security group allows inbound HTTP 80 from anywhere (open to internet)
- For production: Use HTTPS (TLS), restrict ALB ingress, add authentication layer

---

## 📝 Environment Variables

### Frontend (React)
- `REACT_APP_BACKEND_URL` — Backend API base URL (baked at build time)

### Backend (Node.js)
- `MONGO_CONN_STR` — MongoDB connection string
- `MONGO_USERNAME` — MongoDB user
- `MONGO_PASSWORD` — MongoDB password
- `PORT` — Express server port (default: 3500)

---

## 🛠️ Local Development

### Run Locally Without Docker

#### Backend:
```bash
cd Generic_Application_forDemo/backend
npm install
export MONGO_CONN_STR=mongodb://localhost:27017/todo
export MONGO_USERNAME=admin
export MONGO_PASSWORD=password123
node index.js
```

#### Frontend:
```bash
cd Generic_Application_forDemo/frontend
npm install
export REACT_APP_BACKEND_URL=http://localhost:3500/api/tasks
npm start
```

---

## 📚 References

- [Amazon EKS Documentation](https://docs.aws.amazon.com/eks/)
- [AWS Load Balancer Controller](https://kubernetes-sigs.github.io/aws-load-balancer-controller/)
- [Kubernetes Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [Docker Best Practices](https://docs.docker.com/develop/dev-best-practices/)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)

---

## 👨‍💻 Contributing

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Commit changes (`git commit -m 'Add amazing feature'`)
4. Push to branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

---

## 📄 License

This project is licensed under the MIT License - see the LICENSE file for details.

---

## 🤝 Support

For issues or questions:
1. Check the [Troubleshooting](#troubleshooting) section
2. Open a GitHub Issue with details and logs
3. Contact the maintainers

---

**Last Updated:** April 18, 2026

**Version:** 1.0.0
