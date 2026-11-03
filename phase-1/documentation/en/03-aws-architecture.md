# ToggleMaster AWS Architecture (Phase 1)

This document describes the proposed architecture for deploying the ToggleMaster monolithic application on AWS and what was actually provisioned. In Phase 1, the application is deployed **as is**: an EC2 instance in a public subnet runs the API, a managed PostgreSQL database (RDS) sits in a private subnet, and the database credentials are stored in AWS Secrets Manager.

Sections 1 to 8 describe the architecture in its final state, as created in the AWS console. The adjustments made during deployment relative to the initial plan, with their reasons, are in section 9.

---

## 1. Overview

![ToggleMaster AWS architecture diagram](../../images/togglemaster_aws_architecture.png)

---

## 2. Design decisions

1. **Two private subnets.** RDS requires a *DB subnet group* with subnets in at least two Availability Zones, even for a Single-AZ instance. The instance is created in **one** of the subnets in the group. Since the *Availability Zone* field was left as *No preference*, AWS chose `us-east-1b`, so the database sits in private subnet B. Private subnet A remains as a spare in the group.
2. **PostgreSQL 16 on RDS.** Standard support for RDS PostgreSQL 13 ended on 02/28/2026. Creating the database with 13.23 was rejected by the console, which required *Extended Support* (additional charge). The chosen version was **16.15**, and the local `docker-compose.yaml` uses `postgres:16` to keep parity between development and production. The application only uses basic SQL (`SERIAL`, `VARCHAR`, `BOOLEAN`, `TIMESTAMPTZ`), so it is compatible with both versions.
3. **SSL on the database.** Starting with PostgreSQL 15, RDS requires SSL connections by default (`rds.force_ssl=1`). The application worked without changes, because the default `sslmode` of `psycopg2` (`prefer`) negotiates SSL. Certificate validation is not performed in this phase.
4. **No NAT Gateway.** RDS does not need internet access, and the EC2 instance already has outbound access through the route to the Internet Gateway. A NAT Gateway would add a fixed cost with no benefit.
5. **Credentials in AWS Secrets Manager.** RDS creates and maintains the secret holding the database username and password. The EC2 instance reads that secret through an IAM Role (`LabInstanceProfile`) and exports the environment variables before starting Gunicorn. The password is neither typed on the instance nor written in the code, and `app.py` does not need to change, since it keeps reading `DB_USER` and `DB_PASSWORD` from the environment.
6. **Amazon Linux 2023 on EC2.** The AMI's Python 3.9 is the same as the one in the Dockerfile's `python:3.9-slim` image, which brings the local environment closer to the EC2 one, and the AWS CLI comes preinstalled (needed to read the secret). The instance price is the same as plain Ubuntu, so the choice was technical, not cost-driven.

---

## 3. Network (VPC)

**Region:** `us-east-1` (N. Virginia). AWS Academy limits access to `us-east-1` and `us-west-2`, and the `vockey` key only exists in `us-east-1`.

| Resource | Configuration | Name |
|---|---|---|
| VPC | `10.0.0.0/16`, DNS hostnames enabled | `togglemaster-vpc` |
| Public subnet | `10.0.1.0/24`, `us-east-1a` (EC2), auto-assigned public IP | `togglemaster-public-1a` |
| Private subnet A | `10.0.2.0/24`, `us-east-1a` (spare in the subnet group) | `togglemaster-private-1a` |
| Private subnet B | `10.0.3.0/24`, `us-east-1b` (RDS) | `togglemaster-private-1b` |
| Internet Gateway | Attached to the VPC | `togglemaster-igw` |
| Public route table | `local` and `0.0.0.0/0 → IGW`, associated with the public subnet | `togglemaster-public-rt` |
| Private route table | `local` only, associated with both private subnets | `togglemaster-private-rt` |

---

## 4. Security Groups

### 4.1 SG-EC2 (`togglemaster-sg-ec2`)

**Inbound:**

| Type | Port | Source | Reason |
|---|---|---|---|
| SSH | 22 | `<operator IP>/32` | Administration, from the operator's IP only |
| Custom TCP | 5000 | `0.0.0.0/0` | The API (Gunicorn) listens on port 5000 |

**Outbound:** default (all traffic allowed), required for `dnf`, `pip` and for reading the secret from Secrets Manager (HTTPS).

### 4.2 SG-RDS (`togglemaster-sg-rds`)

**Inbound:**

| Type | Port | Source |
|---|---|---|
| PostgreSQL | 5432 | `togglemaster-sg-ec2` (a reference to the Security Group, not a CIDR) |

### 4.3 Notes

- **HTTP/HTTPS:** the rules for ports 80 and 443 were initially created, as stated in the assignment, but were removed because nothing listens on them: the application serves HTTP directly on port 5000. Opening 80/443 would only make sense with a reverse proxy or Load Balancer, which are out of scope for Phase 1.
- **Operator IP:** with a dynamic IP, the SSH rule must be updated (*My IP* option) whenever the IP changes.

---

## 5. EC2

| Item | Choice | Reason |
|---|---|---|
| AMI | Amazon Linux 2023 (kernel 6.1), x86_64 | Python 3.9 matching the Dockerfile and preinstalled AWS CLI. Login user: `ec2-user` |
| Type | `t3.micro` (2 vCPU, 1 GiB) | Enough for a low-load Gunicorn; allowed in AWS Academy |
| CPU credits | Standard | *Unlimited* mode can incur extra charges |
| Disk | 8 GiB gp3 (default) | The application is small |
| Network | Public subnet, auto-assigned public IP, SG-EC2 | External access |
| SSH key | `vockey` (Academy), file `labsuser.pem` | SSH access |
| IAM Role | `LabInstanceProfile` (AWS Academy) | Lets the EC2 instance read the database secret without fixed credentials. In a personal account, a role with `secretsmanager:GetSecretValue` restricted to the RDS secret |
| Metadata | IMDSv2 required | More secure |
| Tools | AWS CLI 2.38 (preinstalled), Python 3.9.25 | Reading the secret and running the application |
| Elastic IP | Not used | The dynamic public IP changes when the instance is stopped and started; AWS charges for public IPv4 addresses |

**Why not Ubuntu.** Ubuntu 24.04 ships with Python 3.12, and `psycopg2-binary==2.9.5` in `requirements.txt` dates from 2022, which poses a compatibility risk. Version 22.04 only appeared in the console's quick selector in the SQL Server variant (paid license). Amazon Linux 2023 avoided both problems.

---

## 6. RDS

| Item | Choice |
|---|---|
| Engine | PostgreSQL 16.15 (*Extended Support* disabled) |
| Template | Free tier |
| Deployment | Single instance, no Multi-AZ |
| Class | `db.t3.micro` |
| Storage | gp3, 20 GiB, no autoscaling, encrypted (`aws/rds`) |
| Identifier | `togglemaster-db` |
| Master user | `postgres` |
| Password | Managed by RDS in AWS Secrets Manager (*Managed in AWS Secrets Manager*) |
| Initial database name | `togglemaster` |
| Network | `togglemaster-vpc`, subnet group `togglemaster-db-subnet-group` (`private-1a` and `private-1b`), AZ *No preference* (AWS chose `us-east-1b`) |
| Public access | No |
| Security Group | `togglemaster-sg-rds` |
| Backup | 1-day retention |
| Extras | Enhanced Monitoring and *Log exports* off; Database Insights in Standard mode (7 days, no charge); Deletion protection off to make cleanup easier |

The **Initial database name** field is essential: if left blank, RDS does not create the `togglemaster` database, and the application fails with `database "togglemaster" does not exist`.

---

## 7. Deploying on EC2

The code was sent from the local machine to the EC2 instance with `scp`, using the same key as SSH (without placing GitHub credentials on the instance).

```powershell
# On the local machine (PowerShell)
scp -i C:\Users\<user>\.ssh\labsuser.pem -r "<path>\toggle-master-monolith" ec2-user@<public-ip>:~/
ssh -i C:\Users\<user>\.ssh\labsuser.pem ec2-user@<public-ip>
```

```bash
# On the EC2 instance
cd ~/toggle-master-monolith
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

### Environment variables

`DB_HOST`, `DB_PORT` and `DB_NAME` are not secrets and remain ordinary variables. `DB_USER` and `DB_PASSWORD` come from the secret created by RDS, which contains the `username` and `password` fields. The password is never typed, so it does not end up in the shell history or in a file:

```bash
export SECRET_ARN='<secret-arn>'
export DB_HOST='<rds-endpoint>'
export DB_PORT='5432'
export DB_NAME='togglemaster'
export SECRET_JSON=$(aws secretsmanager get-secret-value --secret-id "$SECRET_ARN" --region us-east-1 --query SecretString --output text)
export DB_USER=$(echo "$SECRET_JSON" | python3 -c "import sys,json; print(json.load(sys.stdin)['username'])")
export DB_PASSWORD=$(echo "$SECRET_JSON" | python3 -c "import sys,json; print(json.load(sys.stdin)['password'])")
unset SECRET_JSON
```

`app.py` does not read `DB_PORT` and uses the default port 5432 (a point already noted in the 12-Factor analysis).

### Creating the table and starting the application

With Docker, `entrypoint.sh` runs `flask init-db` automatically. On EC2, running only Gunicorn, this step must be done once:

```bash
flask init-db                              # creates the "flags" table in RDS
gunicorn --bind 0.0.0.0:5000 app:app       # runs in the foreground
```

### Validation

From outside the EC2 instance, using the public IP:

```powershell
curl.exe http://<public-ip>:5000/health
curl.exe -X POST -H "Content-Type: application/json" -d '{\"name\":\"new-feature\",\"is_enabled\":true}' http://<public-ip>:5000/flags
curl.exe http://<public-ip>:5000/flags
```

### Limitations

- The variables only last for the terminal session. After reconnecting over SSH, they must be exported again and Gunicorn restarted.
- The application reads the password only once, at startup. The secret managed by RDS is rotated automatically (every 7 days by default), and after a rotation the application must be restarted to pick up the new password.
- Gunicorn runs in the foreground: if the SSH session closes, the application goes down. In production, `systemd` would be used (factor IX of the 12-Factor analysis).

---

## 8. Environment considerations

- **AWS Academy (Learner Lab):** IAM is restricted (the `LabRole` is used) and the lab session lasts 4 hours. When it ends, EC2 and RDS are stopped and the **EC2 public IP changes**; the RDS endpoint does not. The `LabInstanceProfile` allowed reading the secret from Secrets Manager.
- **Personal account:** accounts created on or after 07/15/2025 receive credits (up to US$200) for 6 months on the free plan, instead of the 12 months of 750 hours for `t2.micro`/`t3.micro` on older accounts. Current rules should be checked in the Billing dashboard.
- **Public IPv4 has a cost**, even while in use (about US$0.005 per hour). This amount goes into the cost estimate.
- **Secrets Manager has a cost:** about US$0.40 per secret per month, plus API calls. This amount also goes into the cost estimate.
- **Cross-AZ traffic:** the EC2 instance (`us-east-1a`) accesses RDS (`us-east-1b`). It works normally through the VPC's `local` route, and the data transfer cost is negligible at this volume.

---

## 9. Decisions that changed during implementation

| Topic | Planned | Implemented | Reason |
|---|---|---|---|
| PostgreSQL | 13 (parity with compose) | **16.15**, compose updated to `postgres:16` | 13.23 left standard support and the console required Extended Support (extra charge) |
| Private subnets | 1 | **2** | The DB subnet group requires 2 AZs |
| RDS placement | Private A (AZ a) | **Private B (`us-east-1b`)** | AZ *No preference*: AWS chose 1b |
| EC2 operating system | Ubuntu 22.04 | **Amazon Linux 2023** | Python 3.9 matching the Dockerfile; preinstalled AWS CLI; 22.04 only appeared with paid SQL Server |
| SG-EC2 ports | 22, 80, 443 | **22 and 5000** | Nothing listens on 80/443; the API uses 5000 |
| Credentials | Typed environment variables | **Secrets Manager** read via IAM Role | Avoids the password in history, files or code |
| Table creation | Automatic (entrypoint) | **Manual `flask init-db`** | On EC2, Gunicorn is started without `entrypoint.sh` |

---

## 10. Out of scope for Phase 1

To keep the architecture simple, the following are not part of this phase: NAT Gateway, Load Balancer, HTTPS (ACM), Multi-AZ, Auto Scaling, Elastic IP, `systemd` and CloudWatch Agent. These items appear as future improvements in the 12-Factor analysis recommendations.

Since the API has no authentication and port 5000 is open to the public, it is recommended to keep the environment running only during development and evaluation, and to remove the resources afterwards.
