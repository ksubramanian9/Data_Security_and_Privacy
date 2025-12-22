# Lab Manual: Data Privacy and Security

## Lab 1: Implement Secure Data Communication with AES and RSA

**Overview:** Generate RSA keys, wrap an AES key with OAEP, encrypt data with AES-256-CBC, and verify integrity before cleanup.

### Environment Setup
- Linux/macOS shell (or WSL) with OpenSSL 1.1+ or 3.x (`openssl version`).
- Shell utilities: `chmod`, `xxd`, `diff`, `sha256sum`.
- Dedicated working directory with write access.
- Plan secure storage for temporary keys on shared machines.

### Steps
1. **Generate RSA Keys (2048-bit)**
   ```sh
   openssl genpkey -algorithm RSA -out private.pem -pkeyopt rsa_keygen_bits:2048
   openssl pkey -in private.pem -pubout -out public.pem
   chmod 600 private.pem
   ```
2. **Wrap a 256-bit AES Key with RSA-OAEP**
   ```sh
   openssl rand -out aes.key 32
   openssl pkeyutl -encrypt -inkey public.pem -pubin -in aes.key \
     -pkeyopt rsa_padding_mode:oaep -out aes.key.enc
   ```
3. **Encrypt a File with AES-256-CBC**
   ```sh
   echo "Hello, World!" > plaintext.txt
   openssl rand -out iv.bin 16
   openssl enc -aes-256-cbc -K $(xxd -p aes.key) -iv $(xxd -p iv.bin) \
     -in plaintext.txt -out ciphertext.enc
   ```
4. **Unwrap and Decrypt**
   ```sh
   openssl pkeyutl -decrypt -inkey private.pem -in aes.key.enc \
     -pkeyopt rsa_padding_mode:oaep -out aes.key.dec
   openssl enc -d -aes-256-cbc -K $(xxd -p aes.key.dec) -iv $(xxd -p iv.bin) \
     -in ciphertext.enc -out decrypted_ciphertext.txt
   diff plaintext.txt decrypted_ciphertext.txt
   sha256sum plaintext.txt decrypted_ciphertext.txt
   ```
5. **Cleanup & Safety**
   - Shred keys after use in shared labs: `shred -u private.pem aes.key*`.
   - Keep `iv.bin` and `aes.key.enc` with ciphertext; keep `private.pem` offline.
   - Document key ownership and rotation plans.

### Deliverables
- Archive with `public.pem`, `aes.key.enc`, `iv.bin`, and encrypted payload (no private keys).
- Terminal log showing `diff` and optional `sha256sum` verification.
- Note describing key handling, permissions, and rotation.

---

## Lab 2: Configure Access Control Policies in a Database

**Overview:** Create sample tables, roles, and grants in MySQL or PostgreSQL; verify and revoke privileges; inspect audit events.

### Environment Setup
- MySQL or PostgreSQL installed locally (or in Docker) with privilege to create databases/roles/users.
- CLI client works: `mysql -u root -p` or `psql -U postgres` with host/port info for remote instances.
- Audit/logging mechanism enabled (MySQL general log table or PostgreSQL `pgaudit`).
- Clean lab schema/database; admin credentials recorded securely.

### Steps
1. **Create a Database & Tables**
   ```sql
   -- MySQL example
   CREATE DATABASE big_data_project;
   USE big_data_project;

   CREATE TABLE patient_records (
     id INT PRIMARY KEY AUTO_INCREMENT,
     name VARCHAR(50),
     diagnosis VARCHAR(100)
   );

   CREATE TABLE transaction_logs (
     id INT PRIMARY KEY AUTO_INCREMENT,
     actor VARCHAR(50),
     action VARCHAR(50)
   );

   INSERT INTO patient_records (name, diagnosis) VALUES
   ('Ada', 'Hypertension'), ('Grace', 'Migraine');
   ```
2. **Define Roles and Users**
   ```sql
   -- Create roles
   CREATE ROLE analyst;
   CREATE ROLE data_engineer;
   CREATE ROLE admin;

   -- Create users
   CREATE USER 'analyst_user'@'%' IDENTIFIED BY 'P@ssw0rd!';
   CREATE USER 'data_engineer_user'@'%' IDENTIFIED BY 'P@ssw0rd!';

   -- Assign roles to users
   GRANT analyst TO 'analyst_user'@'%';
   GRANT data_engineer TO 'data_engineer_user'@'%';
   ```
3. **Grant & Verify Privileges**
   - Grant privileges to roles (not users) and test with `SET ROLE`.
4. **Review Audit Events**
   - Query audit/general logs to confirm activity is recorded.
5. **Revoke & Cleanup**
   - Revoke unneeded privileges, drop test users/roles, and archive audit excerpts.

### Deliverables
- SQL scripts for schema creation, roles, grants, and revocations.
- Proof of privilege tests and audit log excerpts.
- Short report on least-privilege design and cleanup steps.

---

## Lab 3: Implement Multi-Factor Authentication (MFA)

**Overview:** Create a virtual environment, onboard a TOTP secret with a QR code, and verify codes with drift and rate-limit protections.

### Environment Setup
- Python 3.9+ with `pip` and `venv` (`python3 --version`).
- Ability to install `pyotp` and `qrcode[pil]` (online or wheel cache).
- Authenticator app ready to scan the QR code.
- Workspace with permissions to create the virtual environment and output images.

### Steps
1. **Isolate & Install Dependencies**
   ```sh
   python3 -m venv .venv
   source .venv/bin/activate
   pip install --upgrade pip
   pip install pyotp qrcode[pil]
   ```
2. **Provision a User Secret**
   ```python
   import pyotp, qrcode

   # Generate per-user secret
   secret = pyotp.random_base32()
   totp = pyotp.TOTP(secret)

   # Provisioning URI (scan with authenticator app)
   uri = totp.provisioning_uri(name="student@example.com", issuer_name="AD23631-MFA-Lab")
   qrcode.make(uri).save("totp-qr.png")
   print("Secret:", secret)
   print("URI saved to totp-qr.png")
   ```
3. **Verify with Clock Drift Allowance**
   ```python
   import time

   MAX_ATTEMPTS = 3
   attempts = 0
   while attempts < MAX_ATTEMPTS:
       code = input("Enter the 6-digit code: ")
       if totp.verify(code, valid_window=1):  # allows +/-30s drift
           print("Authentication Successful")
           break
       attempts += 1
       print("Invalid code. Attempts left:", MAX_ATTEMPTS - attempts)
   else:
       print("Too many failures — account locked, alert security team")
   ```
4. **Secure Secret Handling**
   - Store secrets in env vars or a secrets manager, not source control.
   - Rotate secrets on device loss; enforce per-session nonce/state to prevent replay.
   - Deactivate the virtual environment after testing (`deactivate`).

### Deliverables
- TOTP provisioning URI QR image (do not share raw secret).
- Verification log with at least one success and one failure plus retry limits.
- Write-up on secret protection and clock drift handling.

---

## Lab 4: Case Study – Capital One Data Breach

**Overview:** Build a timeline, map controls, and run a tabletop drill based on the 2019 Capital One breach.

### Environment Setup
- Note-taking tool and spreadsheet/table for timelines and roles.
- Internet access to reference incident reports and remediation write-ups.
- Collaboration space for tabletop drill (shared doc/whiteboard).
- Dedicated folder for evidence logs, role assignments, and postmortem outputs.

### Incident Overview
- **Target:** Capital One (2019)
- **Impact:** 100M+ customers exposed.
- **Root causes:** SSRF exploit plus misconfigured WAF & IAM roles.

### Steps
1. **Build a Timeline**
   - Collect events (exploit, exfiltration, detection, response, disclosure).
   - Record sources (CloudTrail, WAF logs) and note detection gaps.
2. **Map Controls to Frameworks**
   - Map missing controls to NIST CSF categories.
   - Propose fixes: strict IAM roles, IMDS protections, patched WAF rules, VPC egress limits.
3. **Tabletop Drill Script**
   - Roles: Incident Commander, Comms Lead, Cloud Engineer, Security Analyst.
   - Decision points: contain access keys, rotate credentials, isolate workloads, notify regulators.
   - Deliverables: evidence log, owner/action/deadline table, hardening checklist.
4. **Postmortem & Hardening**
   - Document blast radius, data classification, and customer impact.
   - Implement compensating controls: IMDSv2, key rotation, least-privilege WAF/IAM, anomaly alerts.
   - Schedule retests and track owners in a playbook template.

### Deliverables
- Incident timeline with timestamps, sources, and detection gaps.
- Control mapping table to NIST CSF with proposed remediations.
- Tabletop drill notes (roles, decisions, follow-up actions/owners).
- Postmortem summary covering blast radius, customer impact, and prioritized hardening.

---

## Lab 5: Cloud IAM (AWS)

**Overview:** Set up a sandbox account, create IAM users/groups/policies via CLI, validate permissions, and clean up safely.

### Environment Setup
- Sandbox/training AWS account with MFA enabled.
- AWS CLI v2 installed and configured (`aws --version`, `aws configure`); credentials stored securely.
- Optional local practice: Docker for LocalStack/MinIO/Keycloak instead of AWS.
- Network egress allowed for AWS API calls or CLI pointed at local simulator.

### Optional Local IAM Practice
- **LocalStack:** `docker run -d -p 4566:4566 localstack/localstack`; set endpoint to `http://localhost:4566`.
- **MinIO:** `docker run -p 9000:9000 -p 9001:9001 minio/minio server /data --console-address ":9001"`; log into `http://localhost:9001` (`minioadmin/minioadmin`) and create users/bucket policies.
- **Keycloak:** `docker run -p 8080:8080 -e KEYCLOAK_ADMIN=admin -e KEYCLOAK_ADMIN_PASSWORD=admin quay.io/keycloak/keycloak:latest start-dev`; explore MFA/user policies.

### Steps
1. **Sandbox & CLI Setup**
   - Use non-production account; verify CLI and configure with sandbox keys.
   - Ensure MFA on the root account.
2. **Define a Least-Privilege Policy**
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Sid": "AllowS3ReadAccess",
         "Effect": "Allow",
         "Action": ["s3:GetObject", "s3:ListBucket"],
         "Resource": [
           "arn:aws:s3:::my-secure-bucket",
           "arn:aws:s3:::my-secure-bucket/*"
         ]
       }
     ]
   }
   ```
3. **Create Groups, Users, and Attach Policy**
   ```sh
   aws iam create-group --group-name DataAnalysts
   aws iam create-policy --policy-name S3ReadOnly --policy-document file://s3-read.json
   aws iam attach-group-policy --group-name DataAnalysts \
     --policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/S3ReadOnly

   aws iam create-user --user-name analyst-user
   aws iam add-user-to-group --user-name analyst-user --group-name DataAnalysts
   aws iam create-access-key --user-name analyst-user
   ```
4. **Validate & Simulate**
   ```sh
   aws sts get-caller-identity
   aws iam get-user --user-name analyst-user
   AWS_PROFILE=analyst aws s3 ls s3://my-secure-bucket
   aws iam simulate-principal-policy --policy-source-arn \
     arn:aws:iam::<ACCOUNT_ID>:user/analyst-user --action-names s3:GetObject
   ```
5. **Cleanup & Rotation**
   - Rotate access keys every 90 days; prefer federation with MFA for long-term use.
   - Teardown after the lab: detach policies, remove keys, delete users/groups.
   - Confirm CloudTrail logging and review IAM actions for the lab period.

### Deliverables
- JSON policy files and CLI commands for groups/users/policy attachments.
- Evidence of validation: `aws sts get-caller-identity`, policy simulation output, and sample S3 access.
- CloudTrail excerpt or screenshot showing IAM actions during the lab.
- Cleanup log showing detached policies, deleted keys, and removed identities.

---

## Lab 6: Differential Privacy

**Overview:** Implement Laplace noise correctly, bound sensitivity, and observe privacy/utility trade-offs.

### Environment Setup
- Python 3.9+ with `numpy`; optional `matplotlib` for plotting.
- Jupyter Notebook or Python script runner with package install permissions.
- Optional fixed random seed for reproducibility.
- Sample dataset with values bounded to [0, 100].

### Steps
1. **Define the Query & Sensitivity**
   - Example: class average. Bound each student's score to [0, 100]; L1 sensitivity of sum is 100; average sensitivity = 100 / n.
2. **Implement Laplace Mechanism**
   - Scale = sensitivity / epsilon.
   ```python
   import numpy as np

   def laplace_mech(true_answer, sensitivity, epsilon):
       scale = sensitivity / epsilon
       noise = np.random.laplace(loc=0, scale=scale)
       return true_answer + noise

   # Example usage
   scores = [88, 92, 77, 95, 84]
   bounded = [min(100, max(0, s)) for s in scores]
   true_avg = sum(bounded) / len(bounded)
   sensitivity = 100 / len(bounded)
   epsilon = 1.0
   noisy_avg = laplace_mech(true_avg, sensitivity, epsilon)
   print(f"True: {true_avg:.2f}, Noisy: {noisy_avg:.2f}")
   ```
3. **Explore Composition & Accuracy**
   - Run the query repeatedly; track total ε budget spent.
   - Plot error distributions for ε = 0.1, 0.5, 1, 5 to visualize utility.
   - Discuss clipping/normalization choices and their impact on sensitivity.

### Deliverables
- Notebook or script implementing the Laplace mechanism with sensitivity notes.
- Plots or tables showing noisy results for multiple ε values with an accuracy discussion.
- Privacy budget tracker and reflections on utility trade-offs.
