# PreProd Exploration Guide - CBP-26380

Step-by-step guide for connecting to CloudBees PreProd environment and exploring S3 log structure.

---

## PHASE 1: AWS Authentication

### Command 1: Login to AWS SSO for PreProd
```bash
aws sso login --profile saas-pp-dev
```
**What it does**: Opens browser to authenticate you with AWS SSO. You'll enter the device code shown.

**Expected output**: "Successfully logged into Start URL..."

---

## PHASE 2: Connect to Kubernetes Cluster

### Command 2: Configure kubectl for PreProd US-West-2 cluster
```bash
aws eks update-kubeconfig --name=eks-preprod-us-west-2 --region=us-west-2 --kubeconfig ~/.kube/pp-platform-west --profile saas-pp-dev
```
**What it does**: Downloads cluster credentials and saves them to `~/.kube/pp-platform-west` file

**Expected output**: "Updated context arn:aws:eks:us-west-2:..."

### Command 3: Verify cluster connection
```bash
kubectl --kubeconfig ~/.kube/pp-platform-west get nodes
```
**What it does**: Lists all Kubernetes nodes in the cluster

**What to look for**:
- Node names (e.g., `ip-10-228-1-99.us-west-2.compute.internal`)
- STATUS should be "Ready"
- AGE shows how long nodes have been running

---

## PHASE 3: Explore Kubernetes Resources

### Command 4: List all namespaces
```bash
kubectl --kubeconfig ~/.kube/pp-platform-west get namespaces
```
**What it does**: Shows all Kubernetes namespaces (logical groupings of resources)

**What to look for**:
- `platform` - where CloudBees services run
- `kube-system` - Kubernetes system components
- `cb-*` - CloudBees infrastructure (cassandra, nats, vault, etc.)

### Command 5: Find log-service pods
```bash
kubectl --kubeconfig ~/.kube/pp-platform-west get pods -n platform | grep log-service
```
**What it does**: Lists log-service pods in the platform namespace

**What to look for**:
- 3 replicas running (for high availability)
- STATUS "Running"
- Ready: 1/1

### Command 6: Look for fluent-bit in platform namespace
```bash
kubectl --kubeconfig ~/.kube/pp-platform-west get daemonsets -n platform
```
**What it does**: Lists DaemonSets (pods that run on EVERY node)

**Expected**: No fluent-bit here (returns empty)

### Command 7: Check kube-system for fluent-bit
```bash
kubectl --kubeconfig ~/.kube/pp-platform-west get daemonsets -n kube-system
```
**What it does**: Check system namespace for DaemonSets

**What to look for**:
- `aws-node`, `kube-proxy`, etc. (standard K8s components)
- No fluent-bit here either!

### Command 8: Search ALL namespaces for fluent-bit
```bash
kubectl --kubeconfig ~/.kube/pp-platform-west get pods -A | grep -i fluent
```
**What it does**: Searches every namespace for fluent-related pods

**Expected**: Might be empty or minimal - fluent-bit might be managed differently in preprod

### Command 9: Find workflow pods (to understand where logs come from)
```bash
kubectl --kubeconfig ~/.kube/pp-platform-west get pods -A | grep -E "workflow|tekton" | head -10
```
**What it does**: Finds workflow execution pods

**What to look for**:
- `ng-tekton-dispatch-service` - dispatches workflows
- `external-workflow-service` - manages external workflows

---

## PHASE 4: Explore S3 Log Storage

### Command 10: List all S3 buckets
```bash
aws s3 ls --profile saas-pp-dev | grep log
```
**What it does**: Shows all S3 buckets with "log" in the name

**What to look for**:
- `cloudbees-saas-preprod-us-west-2-logs` - main logs bucket
- `cloudbees-saas-preprod-us-east-1-logs` - east region bucket

### Command 11: See top-level structure of logs bucket
```bash
aws s3 ls s3://cloudbees-saas-preprod-us-west-2-logs/ --profile saas-pp-dev
```
**What it does**: Lists root folders in the bucket

**Expected output**:
```
PRE edge-logs/
PRE workflow-logs/
```

**Key observation**: TWO types of logs with different structures!

### Command 12: Count how many components exist
```bash
aws s3 ls s3://cloudbees-saas-preprod-us-west-2-logs/workflow-logs/ --profile saas-pp-dev | wc -l
```
**What it does**: Counts the number of component folders

**Expected**: ~4,600+ components (each is a componentID UUID)

**Why this matters**: ALL these components share `/workflow-logs/` prefix → rate limiting!

### Command 13: List first 20 component IDs
```bash
aws s3 ls s3://cloudbees-saas-preprod-us-west-2-logs/workflow-logs/ --profile saas-pp-dev | head -20
```
**What it does**: Shows first 20 component folders

**What to see**: Each folder is a UUID (componentID)

### Command 14: Pick one component and explore its structure
```bash
aws s3 ls s3://cloudbees-saas-preprod-us-west-2-logs/workflow-logs/00098741-ebe8-4bb7-a8fb-7deb774850ba/ --profile saas-pp-dev
```
**What it does**: Shows runs within one component

**Expected**: Folders with UUIDs (runIDs)

### Command 15: Go deeper - see run attempts
```bash
aws s3 ls s3://cloudbees-saas-preprod-us-west-2-logs/workflow-logs/00098741-ebe8-4bb7-a8fb-7deb774850ba/3d2d414a-4e1d-4901-a9c6-8b6c8cf5b61d/ --profile saas-pp-dev
```
**What it does**: Shows attempt numbers for a specific run

**Expected**: Folders like `1/`, `2/`, etc. (attempt numbers)

### Command 16: See job folders
```bash
aws s3 ls s3://cloudbees-saas-preprod-us-west-2-logs/workflow-logs/00098741-ebe8-4bb7-a8fb-7deb774850ba/3d2d414a-4e1d-4901-a9c6-8b6c8cf5b61d/1/ --profile saas-pp-dev
```
**What it does**: Shows job names in this run attempt

**Expected**: Folders like `build/`, `test/`, etc. (job names)

### Command 17: Finally - see the actual log files!
```bash
aws s3 ls s3://cloudbees-saas-preprod-us-west-2-logs/workflow-logs/00098741-ebe8-4bb7-a8fb-7deb774850ba/3d2d414a-4e1d-4901-a9c6-8b6c8cf5b61d/1/build/ --profile saas-pp-dev
```
**What it does**: Lists actual compressed log files

**Expected output**:
```
2025-07-08 08:38:07   104 2983-prepare-39db48d9....log.gz
2025-07-08 08:38:22  1306 2985-step-s002-0-0-checkout-7e5e.log.gz
```

**Full path structure confirmed**:
```
/workflow-logs/{componentID}/{runID}/{runAttempt}/{job}/{index}-{step}-{containerID}.log.gz
```

---

## PHASE 5: Compare with Edge Logs (Already Using OrgID!)

### Command 18: Explore edge-logs structure
```bash
aws s3 ls s3://cloudbees-saas-preprod-us-west-2-logs/edge-logs/ --profile saas-pp-dev | head -10
```
**What it does**: Shows edge-logs top-level folders

**What to see**: UUIDs (these are orgIDs!)

### Command 19: Pick one orgID and explore
```bash
aws s3 ls s3://cloudbees-saas-preprod-us-west-2-logs/edge-logs/5afc0079-9f2b-4c8d-80ad-bea07e4bfbc9/ --profile saas-pp-dev | head -5
```
**What it does**: Shows jobs within one org's edge logs

**Structure**:
```
/edge-logs/{orgID}/{jobID}/{stepIdx}/{file}.log.gz
```

**Key difference**: OrgID is FIRST in path → better S3 distribution!

---

## PHASE 6: Optional - Download and View a Log

### Command 20: Download one log file
```bash
aws s3 cp s3://cloudbees-saas-preprod-us-west-2-logs/workflow-logs/00098741-ebe8-4bb7-a8fb-7deb774850ba/3d2d414a-4e1d-4901-a9c6-8b6c8cf5b61d/1/build/2983-prepare-39db48d9.log.gz /tmp/sample.log.gz --profile saas-pp-dev
```
**What it does**: Downloads a sample log file

### Command 21: Decompress and view it
```bash
gunzip -c /tmp/sample.log.gz | head -20
```
**What it does**: Shows first 20 lines of the log

**What you'll see**: Actual container logs in JSON format with timestamps

---

## CURRENT vs PROPOSED STRUCTURE

### Current Structure (Problematic):
```
s3://cloudbees-saas-preprod-us-west-2-logs/
└── workflow-logs/                                    ⚠️ SHARED PREFIX
    ├── {componentID-1}/                              (4,646 components)
    │   └── {runID}/
    │       └── {runAttempt}/
    │           └── {job}/
    │               └── {index}-{step}-{containerID}.log.gz
    └── ...

Example:
/workflow-logs/00098741-ebe8-4bb7-a8fb-7deb774850ba/3d2d414a-4e1d-4901-a9c6-8b6c8cf5b61d/1/build/2983-prepare-39db48d9.log.gz
```

**Problem**: All 4,646 components share `/workflow-logs/` prefix → single 3,500 PUT/sec S3 rate limit

### Proposed Structure (Solution):
```
s3://cloudbees-saas-preprod-us-west-2-logs/
└── {orgID}/                                          ✅ NEW: ORG PREFIX
    └── workflow-logs/
        └── {componentID}/
            └── {runID}/
                └── {runAttempt}/
                    └── {job}/
                        └── {index}-{step}-{containerID}.log.gz

Example:
/{orgID}/workflow-logs/00098741-ebe8-4bb7-a8fb-7deb774850ba/3d2d414a-4e1d-4901-a9c6-8b6c8cf5b61d/1/build/2983-prepare-39db48d9.log.gz
```

**Benefit**: Each org gets independent 3,500 PUT/sec capacity → scales linearly

### Edge Logs (Already Correct):
```
s3://cloudbees-saas-preprod-us-west-2-logs/
└── edge-logs/
    └── {orgID}/                                      ✅ Already using orgID!
        └── {jobID}/
            └── {stepIdx}/
                └── {file}.log.gz

Example:
/edge-logs/5afc0079-9f2b-4c8d-80ad-bea07e4bfbc9/21e18c37-1d21-454d-a119-2fb5bb93d106/0/1.log.gz
```

---

## OTHER USEFUL COMMANDS

### Connect to different clusters:

**PreProd US-East-1:**
```bash
aws eks update-kubeconfig --name=eks-preprod-us-east-1 --region=us-east-1 --kubeconfig ~/.kube/pp-platform-east --profile saas-pp-dev
```

**QA US-West-2:**
```bash
aws sso login --profile saas-qa-dev
aws eks update-kubeconfig --name=eks-qa-us-west-2 --region=us-west-2 --kubeconfig ~/.kube/qa-platform-west --profile saas-qa-dev
```

**QA US-East-1:**
```bash
aws eks update-kubeconfig --name=eks-qa-us-east-1 --region=us-east-1 --kubeconfig ~/.kube/qa-platform-east --profile saas-qa-dev
```

### Quick reference commands:

**Set active kubeconfig:**
```bash
export KUBECONFIG=~/.kube/pp-platform-west
kubectl get nodes
```

**View log-service logs:**
```bash
kubectl --kubeconfig ~/.kube/pp-platform-west logs -n platform -l app=log-service --tail=100
```

**Describe a pod:**
```bash
kubectl --kubeconfig ~/.kube/pp-platform-west describe pod -n platform <pod-name>
```

**Get pod labels (to see orgID, componentID, etc.):**
```bash
kubectl --kubeconfig ~/.kube/pp-platform-west get pods -n platform --show-labels
```

---

## SUMMARY OF FINDINGS

✅ **Authentication**: AWS SSO login → kubectl config → cluster access
✅ **Current Structure**: `/workflow-logs/{component}/{run}/{attempt}/{job}/...`
✅ **Problem**: 4,600+ components ALL share `/workflow-logs/` prefix
✅ **Rate Limit**: AWS S3 allows 3,500 PUT/sec per prefix
✅ **Evidence**: Multiple nodes across regions hitting SlowDown errors
✅ **Edge Logs**: Already use `/{orgID}/edge-logs/...` (correct pattern!)
✅ **Solution**: Move to `/{orgID}/workflow-logs/...` to match edge-logs

---

## ROOT CAUSE (CBP-26380)

AWS S3 applies rate limits (3,500 PUT/sec) per prefix. While the full path varies, AWS recommends "high-cardinality prefixes at the START of key names" for optimal partitioning.

All workflow logs share `/workflow-logs/` as the common leading prefix. During peak usage, aggregate PUT requests from all fluent-bit instances (across all nodes, regions, and tenants) exceed the 3,500 PUT/sec threshold for this shared prefix space, causing SlowDown errors.

**Evidence**: Multiple nodes across us-west-2 and us-east-1 experiencing SlowDown errors over 2-week period, indicating cluster-wide shared bottleneck.

**Solution**: Prefix with orgID (`/{orgID}/workflow-logs/...`) per AWS best practice. Distributes writes across multiple prefix spaces, providing 3,500 PUT/sec per org instead of shared across all orgs.

---

## NEXT STEPS

1. Update fluent-bit ConfigMap to extract `cloudbees.io/organization` label and prefix S3 path
2. Update log-service to read from new path with fallback to old path
3. Deploy changes to PreProd
4. Monitor S3 PUT rate distribution across org prefixes
5. Deploy to Prod after validation

---

**Document created**: 2026-03-26
**Ticket**: CBP-26380
**Author**: Engineering Team
