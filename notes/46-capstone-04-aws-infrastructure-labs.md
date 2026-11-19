# 46 · Capstone 04 · AWS Infrastructure Labs: EKS, Networking, RDS, SonarQube

> **Part H: Capstone** · Previous: [45 · Build and Validate the Backend and Frontend (Stage 6)](45-capstone-03-build-and-test.md) · Next: [47 · Code Analysis and Quality Gates with SonarQube (Stage 7)](47-capstone-05-sonarqube-analysis.md) · [Back to INDEX](INDEX.md)
>
> **Code for the whole capstone:** [`code/capstone/`](code/capstone/).
>
> ⏱ **Revising, not building?** Read **What it is → How it works inside → When it breaks → Quick recap** (about 5 minutes). The step-by-step build below is for when you actually build this stage.

---

## What it is

Five hands-on **AWS labs** that create the infrastructure the pipeline needs: **Lab 1** EKS cluster (eksctl), **Lab 2** networking (subnets and route tables in the same VPC), **Lab 3** security groups (SonarQube, RDS), **Lab 4** RDS PostgreSQL, **Lab 5** SonarQube on EC2 (Java 21, PostgreSQL client, systemd service) plus SonarQube projects, Quality Gates and a token.

## Why it exists

GitHub Actions orchestrates **other systems**. Here those systems are created: a cluster to deploy to, a database, and a code-quality server. These labs use **AWS console/CLI and Linux admin**, not GitHub Actions. The **"Configure GitHub Actions Authentication"** step at the end is where they meet: the SonarQube URL and token become repository variables and secrets.

## Analogy

Before the delivery service (CI/CD) can run, you **build the depots** (EKS), **lay the roads** (VPC, subnets, routes), **put up fences with gates** (security groups), **build the warehouse** (RDS), and **open the inspection office** (SonarQube).

## How it works inside

Recaps from your Linux/Kubernetes notes used here:

| Item | Meaning |
|---|---|
| **VPC / subnet / route table** | Your private network / an IP range in one Availability Zone / rules for where traffic goes (public subnets route to an Internet Gateway, private ones don't) |
| **Security group** | A stateful virtual firewall attached to resources. Rules can reference **other security groups** (better than IPs) |
| **eksctl + `eks-config.yaml`** | Creates the EKS control plane and node groups (and `withOIDC: true` = the **EKS** OIDC provider for IRSA. That's **not** the GitHub one, see topic 36) |
| **systemd unit** | `/etc/systemd/system/sonarqube.service`, managed with `systemctl` (start/enable/status) and `journalctl -u sonarqube` |
| **Quality Gate** | SonarQube pass/fail rules (coverage ≥ X%, no new bugs…) that CI enforces in Stage 7 |

Config files: [`code/capstone/infrastructure/eks-config.yaml`](code/capstone/infrastructure/eks-config.yaml), [`setup-mega-project-network.sh`](code/capstone/infrastructure/setup-mega-project-network.sh).

> **Lab-only simplification:** opening SonarQube port 9000 to everyone so GitHub-hosted runners can reach it. In production, use a **self-hosted runner in the VPC** (topic 38), larger runners with static IPs on an allowlist, or a private connection.

## Workflow YAML / commands: build it step by step

The order matters: **EKS** (it creates the VPC) → **subnets** → **security groups** → **RDS** (SonarQube needs a database) → **SonarQube**.

> 💸 Everything here costs money. Delete it when you finish (see topic 54).

### Lab 1: The EKS cluster

[`code/capstone/infrastructure/eks-config.yaml`](code/capstone/infrastructure/eks-config.yaml) ([schema](https://schema.eksctl.io/)):

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: cwvj-mega-demo-eks
  region: ap-south-1
  version: "1.36"
  tags:
    owner: cloudwithvarjosh
    project: cwvj-mega-demo
    purpose: github-actions-mega-project
    managed-by: eksctl

availabilityZones:
  - ap-south-1a
  - ap-south-1b

iam:
  withOIDC: true

managedNodeGroups:
  - name: cwvj-mega-demo-ng
    instanceType: c7i-flex.large
    minSize: 2
    maxSize: 2
    desiredCapacity: 2

    volumeSize: 20
    volumeType: gp3
    volumeEncrypted: true

    # Workers use public subnets created by eksctl.
    privateNetworking: false

    # SSH intentionally disabled.
    # Add only workload-specific IAM permissions when required.

    labels:
      project: cwvj-mega-demo
      workload: general

    tags:
      owner: cloudwithvarjosh
      project: cwvj-mega-demo
      purpose: github-actions-mega-project
      managed-by: eksctl
      node-type: public-worker
```

```bash
eksctl create cluster -f eks-config.yaml      # takes several minutes (CloudFormation)
```

- `iam.withOIDC: true` is the **cluster's** OIDC provider, for pods (IRSA). The demo doesn't need it; it's kept so you can contrast it with **GitHub's** OIDC provider for workflows (topic 36, Stage 9).
- `privateNetworking: false` puts the workers in public subnets to keep the lab simple. In production, workers go in private subnets.
- **AWS tags** (billing, ownership, governance) aren't **Kubernetes labels** (scheduling and selection). Check them separately.

Check: EKS → `cwvj-mega-demo-eks` is **Active**, node group `cwvj-mega-demo-ng` has 2 nodes, and the VPC (`192.168.0.0/16`), subnets, route tables and instances are visible in the VPC and EC2 consoles.

### Lab 2: Subnets for SonarQube and RDS (same VPC)

| AZ | Public IP | CIDR | Name | Route table |
|---|---|---|---|---|
| ap-south-1a | yes | `192.168.128.0/27` | `cwvj-mega-demo-devops-public-a` | existing `eksctl-…/PublicRouteTable` (has an IGW route) |
| ap-south-1b | yes | `192.168.128.32/27` | `cwvj-mega-demo-devops-public-b` | same |
| ap-south-1a | no | `192.168.128.64/27` | `cwvj-mega-demo-db-private-a` | new `cwvj-mega-demo-db-private-rt` (**local route only**) |
| ap-south-1b | no | `192.168.128.96/27` | `cwvj-mega-demo-db-private-b` | same |

- A subnet is **public because its route table points at an Internet Gateway**, not because instances get public IPs.
- DB subnets with only the local route are **isolated**. An RDS subnet group needs subnets in **at least two AZs**, even for a Single-AZ database.
- Using one VPC gives private connectivity without peering. Using one RDS instance with two databases (SonarQube and the app) keeps the lab cheap. Production often separates them per environment or application for isolation.

```bash
cd code/capstone/infrastructure
chmod +x setup-mega-project-network.sh
./setup-mega-project-network.sh
```

Check in the VPC console: 4 subnets with the right CIDRs, AZs and route-table associations, and **no IGW route** for the DB subnets.

### Lab 3: Security groups

**SonarQube SG** (in the EKS VPC):

| Direction | Port | Source / destination | Why |
|---|---|---|---|
| Inbound | TCP 9000 | `0.0.0.0/0`, `::/0` | The UI, and analysis uploads from GitHub-hosted runners. **Lab only.** |
| Outbound | all (default) | default | Package downloads, and connections to RDS |

**`cwvj-rds-sg`:**

| Direction | Port | Source | Why |
|---|---|---|---|
| Inbound | TCP 5432 | **the SonarQube SG** (an SG reference, not an IP or the whole VPC CIDR) | Only SonarQube may connect |
| Outbound | none needed | — | Security groups are **stateful**, so replies are allowed automatically |

**How CI reaches SonarQube** (network access and the token are separate controls):

| Runner | Network rule | Notes |
|---|---|---|
| Self-hosted in the VPC (**preferred**) | Allow the runner's SG on 9000 | Private, but you run the runner (topic 38) |
| Larger hosted runner with static IPs | Allowlist those IPs | Needs a paid plan |
| Standard hosted runner | Allowlist the `actions` ranges from the [Meta API](https://api.github.com/meta) | Many ranges that change weekly; not recommended |

### Lab 4: RDS for PostgreSQL

1. **RDS → Subnet groups → Create:** `cwvj-subnet-group`, the EKS VPC, AZs 1a and 1b, the two **private DB subnets**.
2. **RDS → Create database:** Full configuration, **PostgreSQL** (not Aurora), default version, Free tier template if offered, **Single-AZ** (production: Multi-AZ).

| Setting | Value |
|---|---|
| DB instance identifier | `cwvj-mega-postgres` |
| Master username / credentials | `postgres` / self-managed password (production: Secrets Manager or Vault, with rotation) |
| VPC / subnet group / SG | EKS VPC / `cwvj-subnet-group` / `cwvj-rds-sg` |
| AZ | 1a or 1b |
| Detailed monitoring | off (demo) |

Wait for **Available**, then copy the **endpoint** from Connectivity & security.

### Lab 5: SonarQube on EC2

**Launch** `cwvj-mega-sonarqube`: Ubuntu 26.04, `c7i-flex.large` (SonarQube needs memory for its web server, compute engine and Elasticsearch), a 20 GiB+ disk, the DevOps public subnet, the SonarQube SG and your key pair. Free-tier and credit eligibility depend on your account, so check it first.

```bash
chmod 600 ~/.ssh/<key>.pem
ssh -i ~/.ssh/<key>.pem ubuntu@<sonarqube-ec2-public-ip>

sudo hostnamectl set-hostname sonarqube && exec bash
sudo timedatectl set-timezone Asia/Kolkata

# Java 21 runtime and the PostgreSQL *client* (this box is just a convenient place to run psql)
sudo apt-get update
sudo apt-get install -y openjdk-21-jre postgresql-client unzip
java --version && psql --version
```

**Create SonarQube's database and user** (connect to the default `postgres` database first):

```bash
psql -h <YOUR_RDS_ENDPOINT> -p 5432 -U postgres -d postgres
```

```sql
CREATE USER sonar WITH PASSWORD 'replace-with-a-strong-password';  -- CREATE USER = role with LOGIN
GRANT sonar TO postgres;                 -- lets the master user create a DB owned by sonar
CREATE DATABASE sonarqube OWNER sonar;
\l
\du
\q
```

A dedicated `sonar` owner means SonarQube never uses the master account.

**Install it** under a version-independent path, owned by a non-root service user:

```bash
cd /tmp
curl -L -o sonarqube.zip \
  'https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-26.11.0.134175.zip'
sudo unzip sonarqube.zip -d /opt
sudo mv /opt/sonarqube-26.11.0.134175 /opt/sonarqube-current
rm -f /tmp/sonarqube.zip

sudo groupadd sonar
sudo useradd --system --gid sonar --shell /bin/false sonar     # no interactive login
sudo chown -R sonar:sonar /opt/sonarqube-current
sudo chmod +x /opt/sonarqube-current/bin/linux-x86-64/sonar.sh
id sonar && ls -ld /opt/sonarqube-current
```

**Kernel settings the embedded Elasticsearch needs** (without them SonarQube exits at startup, and `es.log` complains about `vm.max_map_count`):

```bash
echo -e "vm.max_map_count=524288\nfs.file-max=131072" | sudo tee /etc/sysctl.d/99-sonarqube.conf
sudo sysctl --system
```

**Point it at RDS:** in `/opt/sonarqube-current/conf/sonar.properties`, set:

```properties
sonar.jdbc.username=sonar
sonar.jdbc.password=replace-with-a-strong-password
sonar.jdbc.url=jdbc:postgresql://<YOUR_RDS_ENDPOINT>:5432/sonarqube
```

**Run it as a systemd service:** create `/etc/systemd/system/sonarqube.service`:

```ini
[Unit]
Description=SonarQube service
After=network.target

[Service]
Type=forking
User=sonar
Group=sonar
ExecStart=/opt/sonarqube-current/bin/linux-x86-64/sonar.sh start
ExecStop=/opt/sonarqube-current/bin/linux-x86-64/sonar.sh stop
Restart=on-failure
LimitNOFILE=131072
LimitNPROC=8192

[Install]
WantedBy=multi-user.target
```

`Type=forking` is used because `sonar.sh start` launches the server and returns. The `Limit*` values are for Elasticsearch. `WantedBy=multi-user.target` starts it at boot.

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now sonarqube
sudo systemctl status sonarqube

# If it isn't up after a few minutes:
sudo journalctl -u sonarqube -n 100 --no-pager
sudo tail -n 100 /opt/sonarqube-current/logs/{sonar,web,ce,es}.log
```

Open `http://<sonarqube-ec2-public-ip>:9000`, log in as `admin` / `admin`, and change the password. **Administration → System** confirms the database connection.

### Configure SonarQube for the pipeline

**Projects** (**Projects → Create Project**). The key must equal `sonar.projectKey` in the workflow, so keep it stable:

| Project key = display name | For |
|---|---|
| `cwvj-mega-backend` | Spring Boot (JaCoCo coverage) |
| `cwvj-mega-frontend` | React (LCOV coverage) |

**Quality Gates** (**Quality Gates → Create**), one per project so each has its own rules:

| Gate | Condition | Assign to |
|---|---|---|
| `cwvj-backend-qg` | **Coverage** on **Overall Code** · fails when **less than 80%** | `cwvj-mega-backend` |
| `cwvj-frontend-qg` | same | `cwvj-mega-frontend` |

> ⚠️ Corrected: was "Operator: is greater than, Value: 80%" (and "exactly 80% will not satisfy this condition"), now "fails when coverage is **less than** 80%", because a SonarQube gate condition describes when the gate **fails**. For coverage, the condition reads *"Coverage is less than 80.0%"*, so **exactly 80% passes**.

- **Overall Code vs New Code:** this is a fresh project, so gating the whole codebase shows the effect clearly. On a legacy codebase, gate **New Code** so teams improve step by step.
- 80% is a **demo target**, not a law. Coverage shows which code ran, not whether the tests are good. Gates can also include bugs, vulnerabilities and code smells.

**Token:** **Administration → Security → Users**. Create a service user `sonar-user` with analysis-only permissions, generate the token `github-actions-token`, and copy it now (it's shown once).

**GitHub** (**Settings → Secrets and variables → Actions**):

| Type | Name | Value |
|---|---|---|
| Secret | `SONAR_TOKEN` | the token above |
| Variable | `SONAR_HOST_URL` | `http://<sonarqube-ec2-public-ip>:9000` |

---

## Hands-on lab

Do Labs 1–5 in order. Checkpoints:

1. `kubectl get nodes` shows Ready nodes.
2. `psql -h <rds-endpoint> -U postgres -d postgres` connects from the SonarQube EC2.
3. `systemctl status sonarqube` is active, and `http://<ec2-ip>:9000` opens the UI.
4. Backend and frontend SonarQube projects plus Quality Gates exist, the token is generated, and `SONAR_HOST_URL` (variable) + `SONAR_TOKEN` (secret) are set in GitHub.
5. **Tear down** when you finish the capstone (cost!), using the steps at the end of topic 54.

## Where you'll use it

- Platform/DevOps work: the infrastructure your pipelines target.
- Interviews: "how does your CI reach SonarQube or a private DB?" (network paths, security groups, runner placement).

## When it breaks

| Symptom | Meaning | Fix |
|---|---|---|
| eksctl stuck or failed | Quotas, region or IAM permissions. | Check CloudFormation events. Retry with the correct region and quotas. |
| `psql: could not connect … timeout` | Security group or subnet routing. | Allow 5432 from the SonarQube SG to the RDS SG. Use the same VPC. |
| SonarQube won't start | `vm.max_map_count` too low, wrong JDBC URL, or file ownership. | `journalctl -u sonarqube`, check `logs/web.log` and `es.log`. Fix sysctl and config. |
| CI can't reach SonarQube | Port 9000 not reachable from GitHub-hosted runners. | The lab rule above, or a self-hosted runner. |
| Bills | Resources left running. | Delete the cluster, RDS and EC2 after use. |

## Quick recap

- Labs create EKS, networking, security groups, RDS and SonarQube. This is AWS and Linux work that the pipeline depends on.
- Two different OIDC providers: EKS (pods) vs GitHub (workflows).
- The output for GitHub Actions: `SONAR_HOST_URL` + `SONAR_TOKEN`, plus cluster and DB details used later.

**Interview questions**

1. **Q: Why reference security groups instead of IPs in rules?**
   A: Private IPs change. SG references keep working and make access intent clear.
2. **Q: How would you let CI reach a private SonarQube without opening it to the internet?**
   A: A self-hosted runner in the same VPC (or ARC on EKS), or hosted larger runners with static IPs on an allowlist, or a VPN/private link.
3. **Q: What's the EKS OIDC provider for?**
   A: IRSA: letting Kubernetes service accounts assume IAM roles. It's separate from GitHub's OIDC provider.
