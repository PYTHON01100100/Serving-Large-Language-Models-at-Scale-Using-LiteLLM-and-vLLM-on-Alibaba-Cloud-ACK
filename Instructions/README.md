# Kubernetes Access Setup (WSL + Alibaba Cloud ACK)

This repository provides a complete guide for configuring Kubernetes access from **Windows using WSL 2**, preparing Persistent Volumes (PV/PVC), uploading models to OSS, and deploying Large Language Models (LLMs) such as **Llama 3**, **Qwen 2.5**, and **LiteLLM Proxy** on **Alibaba Cloud ACK**.

---

# 📌 Requirements

### ✔ Windows Environment
- Windows 10/11
- WSL 2 enabled
- Ubuntu installed from Microsoft Store

### ✔ Tools Inside WSL
- `kubectl`  
  Install from official guide:  
  https://kubernetes.io/docs/tasks/tools/
- `git`
- `git-lfs`
- `ossutil` (OSS CLI)
- Access to your ACK cluster (private or public endpoint)

### ✔ Alibaba Cloud Permissions
- RAM user with required permissions
- Access to OSS
- Ability to create or use existing PV/PVC
- ACK cluster admin or developer role

---

# ⚠️ Important Notes Before Deploying Models

### ✅ 1. Upload All Model Folders to OSS  
All models **must** exist in OSS before deploying them.

Example structure:

```
oss://modelshugging/
 ├── Qwen2.5-7B-Instruct/
 ├── Qwen2.5-14B-Instruct/
 ├── Qwen2.5-VL-32B-Instruct/
 ├── Llama-3.1-8B-Instruct/
 └── Llama-3.2-3B-Instruct/
```

---

### ✅ 2. Persistent Volume (PV) + PVC Required

Models must be mounted at:

```
/mnt/OSSbuket
```

Use OSS, NAS, or CPFS to create PV/PVC.

📌 **Preferred Method:**  
Create PV & PVC from the **Alibaba Cloud Console → Storage**.  
(It is easier, validated, and prevents YAML errors.)

Volume mount example:

```yaml
volumeMounts:
  - mountPath: /mnt/OSSbuket
    name: model-storage

volumes:
  - name: model-storage
    persistentVolumeClaim:
      claimName: <your-pvc-name>
```

---

# 🧱 4. Model Preparation & Upload to OSS

## **4.1 Install OSSUTIL**

```bash
curl -o ossutil-2.2.0-linux-amd64.zip https://gosspublic.alicdn.com/ossutil/v2/2.2.0/ossutil-2.2.0-linux-amd64.zip
unzip ossutil-2.2.0-linux-amd64.zip
chmod 755 ossutil
sudo mv ossutil /usr/local/bin/
```

Configure OSSUTIL:

```bash
ossutil config
```

Provide:
- **Endpoint**: `oss-me-central-1.aliyuncs.com`
- **AccessKeyId**
- **AccessKeySecret**

---

## ⚠️ Credential Security Note

Always use **RAM User AccessKeys**, not Root Account keys.

Recommended:
- Create RAM user  
- Grant: `AliyunOSSFullAccess` (or more restricted custom policy)  
- Use RAM AccessKeyId & AccessKeySecret when configuring OSSUTIL

Root keys should NEVER be used for CLI or automation.

---

## **4.2 Download and Prepare Model (Example: Qwen 7B Chat Int8)**

Install Git LFS:

```bash
sudo apt install git-lfs -y
git lfs install
```

Clone model:

```bash
git clone https://huggingface.co/Qwen/Qwen-7B-Chat-Int8
```

Check model size:

```bash
du -sh Qwen-7B-Chat-Int8
```

---

## **4.3 Upload Model to OSS**

Create bucket directory:

```bash
ossutil mkdir oss://modelshugging/Qwen-7B-Chat-Int8 --region me-central-1
```

Upload:

```bash
ossutil cp -r ./Qwen-7B-Chat-Int8 oss://modelshugging/Qwen-7B-Chat-Int8
```

Verify:

```bash
ossutil ls oss://modelshugging/Qwen-7B-Chat-Int8
```

---

# 🚀 Connect to Alibaba Cloud ACK Using kubectl

## 1. Install kubectl
https://kubernetes.io/docs/tasks/tools/

## 2. Obtain kubeconfig from ACK console  
Use “Temporary” or “Long-term” kubeconfig.

Security Notes:
- Kubeconfig does NOT expire with RAM users  
- Revoke old kubeconfigs  
- Restrict allowed IPs

## 3. Create kubeconfig directory

```bash
mkdir -p ~/.kube
```

## 4. Add your kubeconfig

```bash
nano ~/.kube/config
```

Paste → Save.

## 5. Fix permissions

```bash
chmod 600 ~/.kube/config
```

## 6. Test

```bash
kubectl get nodes
kubectl get ns
```

---

# 🧠 Deploy & Redeploy All LLM Services (Namespace: your-namespace)

## 🔥 Delete All Deployments

```bash
kubectl delete -f litellm-config.yaml -n your-namespace
kubectl delete -f litellm-deployment.yaml -n your-namespace

kubectl delete -f Llama-3.1-8B-Instruct.yaml -n your-namespace
kubectl delete -f Llama-3.2-3B-Instruct.yaml -n your-namespace

kubectl delete -f Qwen2.5-7B-Instruct.yaml -n your-namespace
kubectl delete -f Qwen2.5-14B-Instruct.yaml -n your-namespace
kubectl delete -f Qwen2.5-VL-32B-Instruct.yaml -n your-namespace

# If you have Qwen3 models:
kubectl delete -f Qwen3-32B-AWQ.yaml -n your-namespace
kubectl delete -f Qwen3-30B-A3B.yaml -n your-namespace
```

---

## 🚀 Apply (Deploy) All Models

```bash
kubectl apply -f litellm-config.yaml -n your-namespace
kubectl apply -f litellm-deployment.yaml -n your-namespace

kubectl apply -f Llama-3.1-8B-Instruct.yaml -n your-namespace
kubectl apply -f Llama-3.2-3B-Instruct.yaml -n your-namespace

kubectl apply -f Qwen2.5-7B-Instruct.yaml -n your-namespace
kubectl apply -f Qwen2.5-14B-Instruct.yaml -n your-namespace
kubectl apply -f Qwen2.5-VL-32B-Instruct.yaml -n your-namespace

# Qwen3 models only:
kubectl apply -f Qwen3-32B-AWQ.yaml -n your-namespace
kubectl apply -f Qwen3-30B-A3B.yaml -n your-namespace
```

---

## ♻️ Update/Redeploy All AI Models (kubectl)

Use these commands to update running deployments after changing YAMLs or container images.

### Option A: Re-apply all manifests
```bash
kubectl apply -f litellm-config.yaml -n your-namespace
kubectl apply -f litellm-deployment.yaml -n your-namespace

kubectl apply -f Llama-3.1-8B-Instruct.yaml -n your-namespace
kubectl apply -f Llama-3.2-3B-Instruct.yaml -n your-namespace

kubectl apply -f Qwen2.5-7B-Instruct.yaml -n your-namespace
kubectl apply -f Qwen2.5-14B-Instruct.yaml -n your-namespace
kubectl apply -f Qwen2.5-VL-32B-Instruct.yaml -n your-namespace

# If you have Qwen3 models:
kubectl apply -f Qwen3-32B-AWQ.yaml -n your-namespace
kubectl apply -f Qwen3-30B-A3B.yaml -n your-namespace
```



# 📂 Suggested Repository Structure

```
.
├── README.md
├── k8s/
│   ├── litellm-config.yaml
│   ├── litellm-deployment.yaml
│   ├── Llama-3.1-8B-Instruct.yaml
│   ├── Llama-3.2-3B-Instruct.yaml
│   ├── Qwen2.5-7B-Instruct.yaml
│   ├── Qwen2.5-14B-Instruct.yaml
│   ├── Qwen2.5-VL-32B-Instruct.yaml
├── scripts/
│   ├── deploy.sh
│   ├── cleanup.sh
└── assets/
    ├── screenshots/
    └── diagrams/
```

---

# 📘 Useful Commands

Check pods:

```bash
kubectl get pods -n your-namespace
```

Check services:

```bash
kubectl get svc -n your-namespace
```

Check storage:

```bash
kubectl get pv
kubectl get pvc -n your-namespace
```

Describe a pod:

```bash
kubectl describe pod <name> -n your-namespace
```

Restart a deployment (e.g. litellm-proxy):

```bash
kubectl rollout restart deployment litellm-proxy -n your-namespace
```

Scale a deployment (e.g. qwen3-32b-awq to 1 replica):

```bash
kubectl scale deployment qwen3-32b-awq -n your-namespace --replicas=1
```

Delete pods by label (e.g. qwen3-32b-awq):

```bash
kubectl delete pods -n your-namespace -l app=qwen3-32b-awq
```

Watch pods in real time:

```bash
watch kubectl get po -n your-namespace
```

---

# 🔧 Troubleshooting

## Replica Not Ready / Pod Stuck in Pending

If pods stay in `Pending` or replicas show `0/1 Ready`:

```bash
kubectl describe pod <pod-name> -n your-namespace
kubectl get events -n your-namespace --sort-by='.lastTimestamp'
```

Common causes:
- **Insufficient GPU/CPU/Memory** — check node resources with `kubectl describe node <node-name>`
- **PVC not bound** — verify PV/PVC status with `kubectl get pvc -n your-namespace`
- **Image pull error** — check image name and registry access

---

## Apply Fails (YAML Errors)

If `kubectl apply` returns an error:

```bash
# Validate YAML before applying
kubectl apply -f <file>.yaml -n your-namespace --dry-run=client
```

Common causes:
- **Indentation or syntax error** in YAML
- **Duplicate resource name** — delete the old resource first then re-apply
- **Namespace does not exist** — create it with `kubectl create ns your-namespace`
- **Invalid field** — check API version and resource spec

---

## Restart Fails / Rollout Stuck

If `kubectl rollout restart` hangs or the new pods never become ready:

```bash
# Check rollout status
kubectl rollout status deployment <deployment-name> -n your-namespace

# Check why new pods are failing
kubectl get pods -n your-namespace
kubectl logs <new-pod-name> -n your-namespace
```

To undo a bad rollout:

```bash
kubectl rollout undo deployment <deployment-name> -n your-namespace
```

---

## Pod in CrashLoopBackOff (Looping)

If a pod keeps restarting and shows `CrashLoopBackOff`:

```bash
# Check logs from the current crash
kubectl logs <pod-name> -n your-namespace

# Check logs from the previous crash
kubectl logs <pod-name> -n your-namespace --previous

# Get detailed pod info
kubectl describe pod <pod-name> -n your-namespace
```

Common causes:
- **Model path not found** — verify the model exists in OSS and PVC is mounted correctly
- **Out of Memory (OOM)** — increase memory limits in the YAML or use a smaller model
- **Wrong container arguments** — check the `args` or `command` in the deployment YAML
- **GPU not available** — confirm GPU nodes are scheduled and `nvidia.com/gpu` resource is set

Quick fix — delete the crashing pods and let the deployment recreate them:

```bash
kubectl delete pods -n your-namespace -l app=<app-label>
```

If the issue persists, scale down, fix the YAML, then scale back up:

```bash
kubectl scale deployment <deployment-name> -n your-namespace --replicas=0
# Fix the YAML file
kubectl apply -f <file>.yaml -n your-namespace
kubectl scale deployment <deployment-name> -n your-namespace --replicas=1
```

---

## General Debugging Tips

```bash
# View all pod statuses at a glance
kubectl get po -n your-namespace -o wide

# Stream live logs from a running pod
kubectl logs -f <pod-name> -n your-namespace

# Open a shell inside a running pod
kubectl exec -it <pod-name> -n your-namespace -- /bin/bash

# Check resource usage on nodes
kubectl top nodes
kubectl top pods -n your-namespace
```

---

# 🛠 Next Enhancements (Optional)

- Auto-deploy script  
- Full GPU workload documentation  
- Architecture diagram (OSS → PV/PVC → vLLM → LiteLLM → OpenWebUI)  
- Health checks & probes  
- Internal/External LoadBalancer configuration  

# References

- [Kubernetes Tools Installation](https://kubernetes.io/docs/tasks/tools/)
- [LiteLLM Documentation](https://docs.litellm.ai/docs/)
- [vLLM Documentation](https://docs.vllm.ai/en/latest/)
- [Redis Documentation](https://redis.io/docs/latest/)
- [ACK – Install KServe Components](https://www.alibabacloud.com/help/en/ack/cloud-native-ai-suite/user-guide/installing-ack-kserve-components?spm=a2c63.p38356.help-menu-85222.d_2_4_5_0.30f34092KTTqGa)
- [ACK – Deploy vLLM Inference](https://www.alibabacloud.com/help/en/ack/cloud-native-ai-suite/user-guide/deploy-a-vllm-inference-application?spm=a2c63.p38356.help-menu-85222.d_2_4_5_1.1f5a4092AO8xmj)






