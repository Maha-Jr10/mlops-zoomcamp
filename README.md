# 🚀 MLflow Remote Tracking Server on AWS

> **Status:** 🟢 Live & Operational  
> **Region:** `eu-north-1` (Stockholm)  
> **UI:** [http://ec2-16-16-201-85.eu-north-1.compute.amazonaws.com:5000](http://ec2-16-16-201-85.eu-north-1.compute.amazonaws.com:5000)

A fully configured, shared MLflow tracking server hosted on AWS. Data scientists can log experiments, compare runs, and store model artifacts from their local machines — no local server needed.

---

## 📓 Example Notebook

**[`mlflow_taxi.ipynb`](./mlflow_taxi.ipynb)** — NYC Taxi Duration Prediction

A complete end-to-end experiment tracking example using the NYC Green Taxi dataset. Covers:

- Linear Regression, Lasso, Ridge, and XGBoost model training
- Parameter and metric logging for every run
- Hyperparameter sweeps (grid search + Bayesian optimization via Hyperopt)
- Artifact logging (prediction plots, model binaries) → stored automatically in S3
- Programmatic run comparison and best-model loading

The notebook is pre-configured to log directly to this remote server — just open it and run. No local MLflow setup needed.

---

## 📐 Architecture

```
Your Machine (Python)
        │
        │  HTTP :5000
        ▼
┌─────────────────────────┐
│  EC2 Instance (t3.micro)│   ← MLflow Server Process
│  Amazon Linux 2023      │
│  eu-north-1             │
└────────┬────────────────┘
         │                         │
    :5432 PostgreSQL           S3 (boto3)
         │                         │
         ▼                         ▼
┌────────────────┐     ┌──────────────────────────┐
│   RDS Postgres │     │  S3 Bucket               │
│ mlflow-backend │     │  mlflow-artifacts-remote  │
│ -db            │     │  -maha                   │
└────────────────┘     └──────────────────────────┘
  Metadata store            Artifact store
  (params, metrics)         (models, plots, files)
```

---

## ⚡ Quick Start (For Data Scientists)

### 1. Install dependencies

```bash
pip install mlflow boto3
```

### 2. Add to your script or notebook

```python
import mlflow

# Point to the shared server
mlflow.set_tracking_uri("http://ec2-16-16-201-85.eu-north-1.compute.amazonaws.com:5000")

# Create or select an experiment
mlflow.set_experiment("your-experiment-name")
```

### 3. Log your training run

```python
with mlflow.start_run():
    mlflow.log_param("learning_rate", 0.01)
    mlflow.log_param("epochs", 50)

    # ... your training code ...

    mlflow.log_metric("accuracy", 0.94)
    mlflow.log_metric("loss", 0.12)

    # Optionally log the model (saved to S3 automatically)
    mlflow.sklearn.log_model(model, "model")
```

### 4. View results in the browser

Open: [http://ec2-16-16-201-85.eu-north-1.compute.amazonaws.com:5000](http://ec2-16-16-201-85.eu-north-1.compute.amazonaws.com:5000)

---

## 🔁 MLflow Autolog (Recommended)

MLflow can automatically capture parameters, metrics, and models for most popular frameworks:

```python
import mlflow

mlflow.set_tracking_uri("http://ec2-16-16-201-85.eu-north-1.compute.amazonaws.com:5000")
mlflow.set_experiment("autolog-demo")

mlflow.sklearn.autolog()        # scikit-learn
# mlflow.xgboost.autolog()     # XGBoost
# mlflow.pytorch.autolog()     # PyTorch Lightning
# mlflow.tensorflow.autolog()  # TensorFlow / Keras

with mlflow.start_run():
    model.fit(X_train, y_train)  # everything logged automatically
```

---

## 🏗️ Infrastructure Overview

| Component | AWS Service | Detail |
|-----------|-------------|--------|
| Compute | EC2 | `t3.micro`, Amazon Linux 2023, `eu-north-1` |
| Backend Store | RDS PostgreSQL | `mlflow-backend-db` |
| Artifact Store | S3 | `mlflow-artifacts-remote-maha` |
| Auth Method | IAM Role | `AmazonS3FullAccess` — no static keys |

---

## 🛠️ Full Setup Guide (For Admins)

This section documents every step taken to build this infrastructure from scratch, so it can be reproduced or recovered at any time.

### Phase 1 — EC2 Instance Setup

#### Instance configuration
- **OS:** Amazon Linux 2023 (not Amazon Linux 2 — uses `dnf` not `yum`)
- **Type:** t3.micro (free-tier eligible)
- **Region:** eu-north-1
- **Key pair:** Created at launch — store the `.pem` file in a secure password manager

#### Security Group rules for EC2

| Port | Protocol | Source | Purpose |
|------|----------|--------|---------|
| 22 | TCP | Your IP | SSH access |
| 5000 | TCP | 0.0.0.0/0 | MLflow UI & API |

#### Connect via SSH

```bash
chmod 400 your-key.pem
ssh -i your-key.pem ec2-user@ec2-16-16-201-85.eu-north-1.compute.amazonaws.com
```

#### Install dependencies on the instance

```bash
# 1. Update system (use dnf, NOT yum — Amazon Linux 2023)
sudo dnf update -y

# 2. Install Python and network tools
sudo dnf install python3 python3-pip nmap-ncat -y

# 3. Install MLflow and AWS libraries
python3 -m pip install mlflow boto3 psycopg2-binary

# 4. Fix python-dateutil conflict with system AWS CLI
python3 -m pip install "python-dateutil==2.9.0"
```

> **Why nmap-ncat?** It provides the `nc` command used to test database connectivity.

---

### Phase 2 — S3 Artifact Store

**Bucket name:** `mlflow-artifacts-remote-maha`  
**Region:** eu-north-1  
**Access:** Private (default settings)

> S3 bucket names must be globally unique across all AWS accounts. If the name is taken, add a suffix like `-2` or use your team name.

#### IAM Role (preferred over Access Keys)

Instead of running `aws configure` (which stores credentials as plaintext on disk), we attach an IAM Role directly to the EC2 instance.

| Method | Security |
|--------|----------|
| `aws configure` with Access Keys | ⚠️ Keys stored in `~/.aws/credentials` |
| IAM Role on EC2 ✅ | ✅ No keys on disk, auto-rotated by AWS |

**Steps to create the role:**
1. IAM Console → Roles → **Create Role**
2. Trusted entity: **AWS service → EC2**
3. Attach policy: **AmazonS3FullAccess**
4. Name it: `mlflow-ec2-role`
5. EC2 Console → select instance → **Actions → Security → Modify IAM Role** → attach `mlflow-ec2-role`

**Remove any old credentials file (critical):**

```bash
# If you previously ran 'aws configure', this file blocks the IAM Role
rm -rf ~/.aws/credentials
```

**Verify access:**

```bash
aws s3 ls
# Expected: lists your bucket — mlflow-artifacts-remote-maha
```

---

### Phase 3 — RDS PostgreSQL Backend Store

#### Database configuration

| Setting | Value |
|---------|-------|
| Engine | PostgreSQL |
| Template | Free Tier |
| Instance identifier | `mlflow-backend-db` |
| Master username | `mlflow` |
| Password | *(see team password manager)* |
| Initial database | `postgres` |
| Endpoint | `mlflow-backend-db.c7kawga28bo8.eu-north-1.rds.amazonaws.com` |
| Port | `5432` |

> ⚠️ The auto-generated password is shown **only once** in the RDS Console. It cannot be retrieved again — only reset.

#### The critical networking fix

The first connection attempt failed with **"Connection timed out"**. This is a Security Group problem. Fix:

1. RDS Console → select `mlflow-backend-db`
2. Under **Connectivity & security** → click the VPC Security Group
3. **Edit inbound rules → Add rule**
4. Type: **PostgreSQL** (auto-fills port 5432)
5. Source: **Custom** → paste the **Security Group ID** of your EC2 instance
6. Save

> **Why Security Group ID instead of an IP?** EC2 public IPs can change on restart. The Security Group ID is permanent and survives reboots.

#### Test the connection

```bash
nc -zv mlflow-backend-db.c7kawga28bo8.eu-north-1.rds.amazonaws.com 5432

# Success: Ncat: Connected to ...5432
# Failure: Ncat: Connection timed out  ← check the Security Group rule
```

---

### Phase 4 — Launch the MLflow Server

#### The launch command

SSH into the EC2 instance and run:

```bash
nohup mlflow server \
    --backend-store-uri postgresql://mlflow:<PASSWORD>@mlflow-backend-db.c7kawga28bo8.eu-north-1.rds.amazonaws.com:5432/postgres \
    --default-artifact-root s3://mlflow-artifacts-remote-maha/ \
    --host 0.0.0.0 \
    --port 5000 > mlflow.log 2>&1 &
```

> Replace `<PASSWORD>` with the actual RDS password from the team password manager.

| Flag | Purpose |
|------|---------|
| `nohup ... &` | Keeps running after SSH session closes |
| `--host 0.0.0.0` | Accepts connections from all IPs (required for external access) |
| `--port 5000` | The port everyone connects to |
| `> mlflow.log 2>&1` | Sends all logs to `mlflow.log` for debugging |

#### Verify it started

```bash
ps aux | grep mlflow
tail -f mlflow.log

# Expected log lines:
# [INFO] Starting gunicorn
# [INFO] Listening at: http://0.0.0.0:5000
```

---

## 🔄 Restart Script

Save this on the EC2 instance so you never have to remember the full command:

```bash
# Create the script (run this once)
cat > ~/start_mlflow.sh << 'EOF'
#!/bin/bash
pkill -f 'mlflow server' 2>/dev/null
sleep 2
nohup mlflow server \
    --backend-store-uri "postgresql://mlflow:<PASSWORD>@mlflow-backend-db.c7kawga28bo8.eu-north-1.rds.amazonaws.com:5432/postgres" \
    --default-artifact-root s3://mlflow-artifacts-remote-maha/ \
    --host 0.0.0.0 \
    --port 5000 > ~/mlflow.log 2>&1 &
echo "MLflow server started. PID: $!"
EOF

chmod +x ~/start_mlflow.sh

# Run it any time you need to start or restart
./start_mlflow.sh
```

---

## 🆘 Troubleshooting

### Emergency diagnostics (run on EC2)

```bash
# Is the process alive?
ps aux | grep mlflow

# What do the recent logs say?
tail -n 50 mlflow.log

# Can we still reach the database?
nc -zv mlflow-backend-db.c7kawga28bo8.eu-north-1.rds.amazonaws.com 5432
```

### Common problems

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| Browser shows "This site can't be reached" | Port 5000 not open, or process crashed | Add EC2 Security Group rule for port 5000; restart server |
| `Connection timed out` on port 5432 | RDS Security Group rule missing | Re-add inbound rule for port 5432 from EC2 Security Group |
| `403 Forbidden` on artifact upload | IAM Role not attached or credentials file overriding it | `rm -rf ~/.aws/credentials` |
| `python-dateutil` conflict error | AWS CLI version conflict | `pip install "python-dateutil==2.9.0"` |
| `mlflow: command not found` | PATH issue | Use `python3 -m mlflow server ...` |
| EC2 DNS changes after reboot | No Elastic IP | Allocate an Elastic IP in EC2 Console and associate it |

---

## 📋 Quick Reference

### Key endpoints

| Resource | Value |
|----------|-------|
| MLflow UI | `http://ec2-16-16-201-85.eu-north-1.compute.amazonaws.com:5000` |
| Tracking URI (Python) | `http://ec2-16-16-201-85.eu-north-1.compute.amazonaws.com:5000` |
| S3 artifact root | `s3://mlflow-artifacts-remote-maha/` |
| RDS endpoint | `mlflow-backend-db.c7kawga28bo8.eu-north-1.rds.amazonaws.com` |
| SSH | `ec2-user@ec2-16-16-201-85.eu-north-1.compute.amazonaws.com` |

### Data scientist checklist

- [ ] `pip install mlflow boto3`
- [ ] Set `mlflow.set_tracking_uri(...)` at the top of every script
- [ ] Set `mlflow.set_experiment("your-experiment-name")`
- [ ] Wrap training code in `with mlflow.start_run():`
- [ ] View results at the MLflow UI link above
- [ ] See [`mlflow_taxi.ipynb`](./mlflow_taxi.ipynb) for a complete worked example

### Admin checklist (after reboot or failure)

- [ ] SSH into EC2 instance
- [ ] Run `./start_mlflow.sh`
- [ ] Confirm with `tail -f mlflow.log`
- [ ] Test browser access to the UI
- [ ] Test DB with `nc -zv <rds-endpoint> 5432`

---

## 🔐 Security Notes

- **No static credentials** on the EC2 instance — authentication is handled via the IAM Role
- **RDS is not publicly accessible** — it only accepts connections from within the EC2 Security Group
- **Port 5000 is open to the internet** — the MLflow UI has no login by default; consider adding authentication if the server contains sensitive model data
- Rotate the RDS password periodically via the RDS Console → **Modify**

---

*Maintained by the MLOps Team. For issues, open a GitHub Issue or contact the team directly.*
