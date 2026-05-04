# Auto-TLS-Using-Existing-Certificates-with-Custom-Root-CA
This article describe how-to config auto-tls using Existing Certificates with Custom Root CA

**TECHNICAL GUIDE**

**Auto-TLS Configuration**

CDP Private Cloud Base 7.3.2

_Use Case 3: Existing Certificates with Custom Root CA_

| Platform       | CDP Private Cloud Base 7.3.2              |
| -------------- | ----------------------------------------- |
| CM Version     | Cloudera Manager 7.13.2                   |
| ECS Version    | CDP ECS 1.5.5-h2000-b238                  |
| CM Server      | drcedge.ds-inovasi.com (192.168.90.232)   |
| Domain         | ds-inovasi.com                            |
| Kerberos Realm | DS-INOVASI.COM                            |
| Root CA        | RootCA-DSI (self-signed, 5-year validity) |
| Prepared by    | DS-Inovasi Infrastructure Team            |
| Date           | May 4, 2026                               |
| Classification | INTERNAL - CONFIDENTIAL                   |

_DS-Inovasi | ds-inovasi.com_

# **1\. Introduction**

This document provides a comprehensive technical guide for configuring Auto-TLS (Transport Layer Security) on Cloudera Manager using Use Case 3 - deploying data-in-transit encryption across all CDP Private Cloud Base components with a custom Root CA generated and managed entirely by the DS-Inovasi infrastructure team.

Prior to Auto-TLS, all inter-component communication within the Cloudera cluster operated over unencrypted plaintext channels, representing a significant security exposure. Once Auto-TLS is enabled, Cloudera Manager automatically distributes certificates to every node and reconfigures each service to enforce TLS on all connections.

## **1.1 Document Objectives**

- Provide a step-by-step, reproducible procedure for deploying Auto-TLS across DR and RND clusters
- Document the TLS architecture, certificate hierarchy, and distribution workflow
- Serve as a troubleshooting reference for certificate-related incidents
- Record all critical file paths, passwords, and configuration parameters

## **1.2 Scope**

| **Component**                 | **Scope**                                 |
| ----------------------------- | ----------------------------------------- |
| Cloudera Manager Server       | In scope - CM Server on drcedge           |
| CM Agent (all nodes)          | In scope - 11 nodes across DR and RND     |
| CDP Services (HDFS, YARN…)    | In scope - auto-configured by CM          |
| Standalone NiFi (rndmaster01) | Partial - requires separate configuration |
| Nexus Docker Registry         | Out of scope - separate guide             |
| ECS / Kubernetes              | Out of scope - separate guide             |

# **2\. TLS Architecture**

## **2.1 Cluster Topology**

The DS-Inovasi environment consists of two clusters managed by a single Cloudera Manager Server instance running on drcedge.ds-inovasi.com:

**CLOUDERA MANAGER SERVER**

drcedge.ds-inovasi.com | 192.168.90.232 | Port 7183 (HTTPS)

| **DR CLUSTER** |     | **RND CLUSTER (ECS)** |
| -------------- | --- | --------------------- |

| **drcedge.ds-inovasi.com**<br><br>drcmtr01.ds-inovasi.com<br><br>drcmtr02.ds-inovasi.com<br><br>drcnifi01.ds-inovasi.com (192.168.90.231)<br><br>drcwrk02.ds-inovasi.com<br><br>drcwrk03.ds-inovasi.com |     | **rndmaster02.ds-inovasi.com (ECS Master)**<br><br>rndwrk01.ds-inovasi.com<br><br>rndwrk02.ds-inovasi.com<br><br>rndwrk03.ds-inovasi.com<br><br>rndmaster01.ds-inovasi.com (Standalone NiFi - partial scope) |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |

_Figure 2.1 - DS-Inovasi cluster topology: a single CM Server manages both DR and RND clusters_

## **2.2 TLS Architecture - Use Case 3**

In Use Case 3, host certificates are generated and signed manually against a custom Root CA before being submitted to Cloudera Manager via the generateCmca API. CM then distributes the certificates to all registered nodes and reconfigures every managed service to enforce TLS.

**Root CA (Self-Signed)**

RootCA-DSI | /root/cert/ca/RootCA-DSI.crt

**| Signs |**

| **drcedge**<br><br>CM Server | **drcmtr01**<br><br>DR Master 01 | **drcmtr02**<br><br>DR Master 02 | **drcnifi01**<br><br>NiFi / Nexus | **drcwrk02/03**<br><br>DR Workers |
| ---------------------------- | -------------------------------- | -------------------------------- | --------------------------------- | --------------------------------- |

| **rndmaster01**<br><br>Standalone NiFi | **rndmaster02**<br><br>ECS Master | **rndwrk01**<br><br>ECS Worker 01 | **rndwrk02**<br><br>ECS Worker 02 | **rndwrk03**<br><br>ECS Worker 03 |
| -------------------------------------- | --------------------------------- | --------------------------------- | --------------------------------- | --------------------------------- |

_Figure 2.2 - CA hierarchy: RootCA-DSI signs individual host certificates for all 11 cluster nodes_

## **2.3 Certificate Distribution Workflow**

| **Step** | **Process**                                                                      | **Actor**            |
| -------- | -------------------------------------------------------------------------------- | -------------------- |
| 1        | Generate Root CA (private key + self-signed certificate)                         | Admin / Server       |
| 2        | Generate per-host private key and Certificate Signing Request (CSR)              | Admin / Server       |
| 3        | Sign all CSRs against the Root CA private key                                    | Admin / Server       |
| 4        | Stage signed certificates and keys in the required directory structure           | Admin / Server       |
| 5        | Invoke CM API: POST /api/v41/cm/commands/generateCmca                            | Admin / CM API       |
| 6        | CM validates and stores certificates under /opt/cloudera/AutoTLS/                | Cloudera Manager     |
| 7        | CM distributes certificates to all registered nodes via SSH                      | Cloudera Manager     |
| 8        | CM Agents are restarted and load the new certificates                            | CM Agent (all nodes) |
| 9        | CM reconfigures all managed services to enforce TLS                              | Cloudera Manager     |
| 10       | Cluster restart completes - all inter-service communication is now TLS-encrypted | CM + All Services    |

## **2.4 Auto-TLS Use Case Comparison**

| **Aspect**             | **Use Case 1**           | **Use Case 2**               | **Use Case 3 (Deployed)**     |
| ---------------------- | ------------------------ | ---------------------------- | ----------------------------- |
| CA Source              | CM-generated internal CA | External CA                  | Custom / internal CA          |
| CSR Generation         | Automated by CM          | Automated by CM              | Manual (admin)                |
| CSR Signing            | Automated by CM          | Manual (once) by ext. CA     | Manual by own Root CA         |
| Certificate Deployment | Automated                | Automated                    | Automated (post-API)          |
| CA Ownership           | None (CM-managed)        | Depends on external PKI      | Full - CA owned internally    |
| Best Suited For        | Lab / Dev                | Enterprise with existing PKI | Internal / air-gapped network |

# **3\. Prerequisites**

## **3.1 Software Requirements**

| **Component**    | **Version / Detail** | **Notes**                                     |
| ---------------- | -------------------- | --------------------------------------------- |
| Cloudera Manager | 7.13.2               | Installed on drcedge                          |
| CDH Parcel       | 7.3.2                | Distributed to all nodes                      |
| OpenSSL          | 1.1.1+ or 3.x        | Available at /usr/bin/openssl on drcedge      |
| Java Keytool     | JDK 8 or 11          | Required for JKS keystore creation (optional) |
| Bash             | 4.x+                 | For executing provisioning scripts            |
| curl             | 7.x+                 | For invoking the CM REST API                  |

## **3.2 Access and Permissions**

- Root access to all cluster nodes (passwordless SSH from drcedge to all nodes)
- Cloudera Manager admin credentials (default: admin / admin)
- Minimum 500 MB free disk space under /root/cert/ on drcedge
- Minimum 200 MB free disk space under /opt/cloudera/AutoTLS/ on drcedge
- All nodes registered in Cloudera Manager with COMMISSIONED status

## **3.3 Pre-Deployment Checklist**

| **#** | **Checklist Item**                                                     | **Status** |
| ----- | ---------------------------------------------------------------------- | ---------- |
| 1     | All nodes registered in CM and reachable via SSH                       | \[ \] OK   |
| 2     | Passwordless SSH from drcedge to all nodes is functional               | \[ \] OK   |
| 3     | OpenSSL is available: openssl version                                  | \[ \] OK   |
| 4     | curl is available: curl --version                                      | \[ \] OK   |
| 5     | No stale or decommissioned hosts remain registered in CM               | \[ \] OK   |
| 6     | CM API is accessible: curl -s <http://drcedge:7180/api/v41/cm/version> | \[ \] OK   |
| 7     | Sufficient disk space confirmed on /root and /opt/cloudera             | \[ \] OK   |
| 8     | CM configuration backup has been completed                             | \[ \] OK   |

**⚠ Warning:**

Ensure no hosts are registered in CM other than the 11 nodes that will receive certificates. Any host without a corresponding certificate entry in hostCerts will cause the generateCmca API call to fail with: 'No directory found for host'.

# **4\. Configuration Procedure**

## **4.1 Generate the Root CA**

The Root CA (Certificate Authority) is the trust anchor that signs all host certificates in the cluster. The Root CA generated here is self-signed and managed entirely by the DS-Inovasi infrastructure team.

### **4.1.1 Create the CA directory and generate Root CA**

mkdir -p /root/cert/ca

cd /root/cert/ca

\# Generate the Root CA private key (4096-bit RSA)

openssl genrsa -out RootCA-DSI.key 4096

\# Generate the self-signed Root CA certificate (5 years = 1825 days)

openssl req -new -x509 \\

\-key RootCA-DSI.key \\

\-out RootCA-DSI.crt \\

\-days 1825 \\

\-sha256 \\

\-subj "/C=ID/ST=DSI/L=DSI/O=DSI/OU=DSI/CN=RootCA DSI"

\# Verify the output

openssl x509 -in RootCA-DSI.crt -noout -text | \\

grep -E "Issuer|Subject|Not Before|Not After"

### **4.1.2 Expected output**

Issuer: C=ID, ST=DSI, L=DSI, O=DSI, OU=DSI, CN=RootCA DSI

Subject: C=ID, ST=DSI, L=DSI, O=DSI, OU=DSI, CN=RootCA DSI

Not Before: Apr 30 03:59:33 2026 GMT

Not After : Apr 29 03:59:33 2031 GMT

**✖ Important:**

RootCA-DSI.key is the most sensitive file in this TLS infrastructure. Anyone in possession of this file can issue certificates trusted by the entire cluster. Store an offline backup in a secure vault immediately after generation.

## **4.2 Generate CSRs for All Hosts**

Each cluster node requires a dedicated private key and Certificate Signing Request (CSR). The CSR encodes the node's identity (CN and SAN) that will appear in the issued certificate.

### **4.2.1 CSR generation script - /root/cert/gencert.sh**

# !/usr/bin/env bash

set -e

OUTDIR="/root/cert/csr_out"

mkdir -p \$OUTDIR

C="ID"; ST="DSI"; L="DSI"; O="DSI"; OU="DSI"

nodes=(

drcedge.ds-inovasi.com drcmtr01.ds-inovasi.com

drcmtr02.ds-inovasi.com drcnifi01.ds-inovasi.com

drcwrk02.ds-inovasi.com drcwrk03.ds-inovasi.com

rndmaster01.ds-inovasi.com rndmaster02.ds-inovasi.com

rndwrk01.ds-inovasi.com rndwrk02.ds-inovasi.com

rndwrk03.ds-inovasi.com

)

for node in "\${nodes\[@\]}"; do

echo "Processing: \$node"

openssl genrsa -out \$OUTDIR/\$node.key 4096

openssl req -new \\

\-key \$OUTDIR/\$node.key \\

\-out \$OUTDIR/\$node.csr \\

\-subj "/C=\$C/ST=\$ST/L=\$L/O=\$O/OU=\$OU/CN=\$node"

done

echo "All CSRs generated in \$OUTDIR"

chmod +x /root/cert/gencert.sh && /root/cert/gencert.sh

## **4.3 Sign All CSRs Against the Root CA**

Sign each CSR using the Root CA generated in Step 4.1. Each host certificate MUST include a Subject Alternative Name (SAN) and Extended Key Usage (EKU) extension - both are mandatory requirements enforced by Cloudera Manager.

### **4.3.1 Certificate signing script - /root/cert/signcert.sh**

# !/bin/bash

set -e

CA_CRT="/root/cert/ca/RootCA-DSI.crt"

CA_KEY="/root/cert/ca/RootCA-DSI.key"

CSR_DIR="/root/cert/csr_out"

OUT="/root/cert/signed"

DAYS=1825

mkdir -p \$OUT

for CSR in \$CSR_DIR/\*.csr; do

NODE=\$(basename \$CSR .csr)

echo "Signing: \$NODE"

\# Create OpenSSL extension file (SAN + EKU - mandatory for Cloudera)

cat > /tmp/\${NODE}.ext <<EOF

\[req_ext\]

subjectAltName = DNS:\${NODE}

extendedKeyUsage = serverAuth, clientAuth

EOF

openssl x509 -req \\

\-in \$CSR \\

\-CA \$CA_CRT -CAkey \$CA_KEY -CAcreateserial \\

\-out \$OUT/\${NODE}.pem \\

\-days \$DAYS -sha256 \\

\-extfile /tmp/\${NODE}.ext -extensions req_ext

cp \$CSR_DIR/\${NODE}.key \$OUT/\${NODE}.key

cat \$OUT/\${NODE}.pem \$CA_CRT > \$OUT/\${NODE}\_chain.pem

cat \$OUT/\${NODE}.key \$OUT/\${NODE}.pem \$CA_CRT \\

\> \$OUT/\${NODE}\_key_chain.pem

echo " Done: \$NODE"

done

echo "Completed. Output directory: \$OUT"

chmod +x /root/cert/signcert.sh && /root/cert/signcert.sh

### **4.3.2 Verify signed certificates**

openssl x509 -in /root/cert/signed/drcedge.ds-inovasi.com.pem \\

\-noout -text | grep -A3 -E "Subject Alternative|Extended Key"

Expected output:

X509v3 Subject Alternative Name:

DNS:drcedge.ds-inovasi.com

X509v3 Extended Key Usage:

TLS Web Server Authentication, TLS Web Client Authentication

**ℹ Note:**

If either SAN or Extended Key Usage is absent from the certificate output, Cloudera Manager will reject the certificate. Regenerate the affected certificates with a correctly formatted extension file.

## **4.4 Prepare the Staging Directory**

Organize all certificate files into the directory structure expected by the generateCmca API.

\# Create staging directory structure

mkdir -p /tmp/auto-tls/certs /tmp/auto-tls/keys /tmp/auto-tls/ca-certs

\# Copy signed certificates and private keys

for NODE in \$(ls /root/cert/signed/\*.pem | grep -v '\_chain'); do

BASE=\$(basename \$NODE .pem)

cp /root/cert/signed/\${BASE}.pem /tmp/auto-tls/certs/

cp /root/cert/signed/\${BASE}.key /tmp/auto-tls/keys/\${BASE}-key.pem

done

\# Copy Root CA certificate

cp /root/cert/ca/RootCA-DSI.crt /tmp/auto-tls/ca-certs/RootCA-DSI.pem

\# Create keystore and truststore password files

\# IMPORTANT: No special characters allowed (!, @, #, \$ etc.)

echo "cloudera1234" > /tmp/auto-tls/keys/key.pwd

echo "cloudera1234" > /tmp/auto-tls/ca-certs/truststore.pwd

\# Set ownership to cloudera-scm service account

chown -R cloudera-scm:cloudera-scm /tmp/auto-tls/

\# Provision AutoTLS base directory

mkdir -p /opt/cloudera/AutoTLS

chown cloudera-scm:cloudera-scm /opt/cloudera/AutoTLS

**⚠ Warning:**

Keystore and truststore passwords MUST NOT contain special characters (!, @, #, \$, etc.). Cloudera will fail to open the keystore if the password includes such characters. Use alphanumeric characters only, with a minimum length of 12.

## **4.5 Validate Registered Hosts in CM**

Before invoking the API, confirm that the set of hosts registered in CM exactly matches the set of hosts with prepared certificates.

curl -s -u admin:admin \\

'<http://drcedge.ds-inovasi.com:7180/api/v41/hosts>' | \\

python3 -m json.tool | grep '"hostname"'

If any unexpected host appears (e.g., support.ds-inovasi.com), remove it from CM:

\# Retrieve the hostId for the unwanted host

curl -s -u admin:admin \\

'<http://drcedge.ds-inovasi.com:7180/api/v41/hosts>' | \\

python3 -m json.tool | grep -A5 'support'

\# Delete the host by its hostId

curl -s -u admin:admin -X DELETE \\

'<http://drcedge.ds-inovasi.com:7180/api/v41/hosts/<hostId>&gt;'

## **4.6 Invoke the generateCmca API**

Submit the certificate configuration to Cloudera Manager. CM will validate all certificates, generate JKS keystores and truststores, and distribute them to every registered node.

**⚠ Warning:**

After this API call succeeds, Cloudera Manager will restart automatically and switch exclusively to HTTPS on port 7183. HTTP access on port 7180 will no longer be available.

curl -i -v -u admin:admin \\

\-X POST \\

\--header 'Content-Type: application/json' \\

\--header 'Accept: application/json' \\

\-d '{

"location": "/opt/cloudera/AutoTLS",

"customCA": true,

"interpretAsFilenames": true,

"cmHostCert": "/tmp/auto-tls/certs/drcedge.ds-inovasi.com.pem",

"cmHostKey": "/tmp/auto-tls/keys/drcedge.ds-inovasi.com-key.pem",

"caCert": "/tmp/auto-tls/ca-certs/RootCA-DSI.pem",

"keystorePasswd": "/tmp/auto-tls/keys/key.pwd",

"truststorePasswd": "/tmp/auto-tls/ca-certs/truststore.pwd",

"hostCerts": \[

{"hostname":"drcedge.ds-inovasi.com",

"certificate":"/tmp/auto-tls/certs/drcedge.ds-inovasi.com.pem",

"key":"/tmp/auto-tls/keys/drcedge.ds-inovasi.com-key.pem"},

{"hostname":"drcmtr01.ds-inovasi.com",

"certificate":"/tmp/auto-tls/certs/drcmtr01.ds-inovasi.com.pem",

"key":"/tmp/auto-tls/keys/drcmtr01.ds-inovasi.com-key.pem"},

{"hostname":"drcmtr02.ds-inovasi.com",

"certificate":"/tmp/auto-tls/certs/drcmtr02.ds-inovasi.com.pem",

"key":"/tmp/auto-tls/keys/drcmtr02.ds-inovasi.com-key.pem"},

{"hostname":"drcnifi01.ds-inovasi.com",

"certificate":"/tmp/auto-tls/certs/drcnifi01.ds-inovasi.com.pem",

"key":"/tmp/auto-tls/keys/drcnifi01.ds-inovasi.com-key.pem"},

{"hostname":"drcwrk02.ds-inovasi.com",

"certificate":"/tmp/auto-tls/certs/drcwrk02.ds-inovasi.com.pem",

"key":"/tmp/auto-tls/keys/drcwrk02.ds-inovasi.com-key.pem"},

{"hostname":"drcwrk03.ds-inovasi.com",

"certificate":"/tmp/auto-tls/certs/drcwrk03.ds-inovasi.com.pem",

"key":"/tmp/auto-tls/keys/drcwrk03.ds-inovasi.com-key.pem"},

{"hostname":"rndmaster01.ds-inovasi.com",

"certificate":"/tmp/auto-tls/certs/rndmaster01.ds-inovasi.com.pem",

"key":"/tmp/auto-tls/keys/rndmaster01.ds-inovasi.com-key.pem"},

{"hostname":"rndmaster02.ds-inovasi.com",

"certificate":"/tmp/auto-tls/certs/rndmaster02.ds-inovasi.com.pem",

"key":"/tmp/auto-tls/keys/rndmaster02.ds-inovasi.com-key.pem"},

{"hostname":"rndwrk01.ds-inovasi.com",

"certificate":"/tmp/auto-tls/certs/rndwrk01.ds-inovasi.com.pem",

"key":"/tmp/auto-tls/keys/rndwrk01.ds-inovasi.com-key.pem"},

{"hostname":"rndwrk02.ds-inovasi.com",

"certificate":"/tmp/auto-tls/certs/rndwrk02.ds-inovasi.com.pem",

"key":"/tmp/auto-tls/keys/rndwrk02.ds-inovasi.com-key.pem"},

{"hostname":"rndwrk03.ds-inovasi.com",

"certificate":"/tmp/auto-tls/certs/rndwrk03.ds-inovasi.com.pem",

"key":"/tmp/auto-tls/keys/rndwrk03.ds-inovasi.com-key.pem"}

\],

"configureAllServices": "true",

"sshPort": 22,

"userName": "root",

"password": "&lt;root_password&gt;"

}' '<http://drcedge.ds-inovasi.com:7180/api/v41/cm/commands/generateCmca>'

### **4.6.1 Expected success response**

{

"id": 1546364875,

"name": "GenerateCMCACommand",

"active": false,

"success": true,

"resultMessage": "Successfully generated CMCA and enabled Auto-TLS"

}

### **4.6.2 Monitor CM Server logs during execution**

tail -f /var/log/cloudera-scm-server/cloudera-scm-server.log | \\

grep -iE "tls|cert|autotls|error|success"

## **4.7 Post-Activation Steps**

### **4.7.1 Restart Cloudera Manager Server**

systemctl restart cloudera-scm-server

\# Allow 30-60 seconds for CM to start on port 7183 (HTTPS)

sleep 60

curl -sk <https://drcedge.ds-inovasi.com:7183/api/v41/cm/version> | \\

python3 -m json.tool

### **4.7.2 Restart all CM Agents**

for host in drcedge drcmtr01 drcmtr02 drcnifi01 \\

drcwrk02 drcwrk03 rndmaster01 rndmaster02 \\

rndwrk01 rndwrk02 rndwrk03; do

echo "Restarting agent on: \$host"

ssh \${host}.ds-inovasi.com 'systemctl restart cloudera-scm-agent'

done

### **4.7.3 Restart CM Management Service**

\# Use HTTPS + port 7183 for all CM API calls after the restart

curl -sk -u admin:admin -X POST \\

'<https://drcedge.ds-inovasi.com:7183/api/v41/cm/service/commands/restart>'

### **4.7.4 Restart the cluster (via CM UI)**

- Log in to <https://drcedge.ds-inovasi.com:7183>
- Navigate to the DR Cluster page
- Click Actions > Restart Stale Services
- Confirm and wait for all services to reach a healthy state

**ℹ Note:**

After Auto-TLS is enabled, ALL access to Cloudera Manager must use HTTPS on port 7183. Update all bookmarks, scripts, and monitoring tools accordingly.

## **4.8 Import Root CA into Browser / Client OS**

To allow browsers and CLI tools to trust the CM server certificate without warnings, the RootCA-DSI certificate must be imported into the Windows Certificate Store on each client workstation.

### **4.8.1 Download Root CA from server**

\# From your workstation / laptop

scp root@192.168.90.232:/root/cert/ca/RootCA-DSI.crt .

### **4.8.2 Import via PowerShell (Run as Administrator)**

certutil -addstore "Root" "C:\\path\\to\\RootCA-DSI.crt"

### **4.8.3 Import via Windows GUI**

- Double-click RootCA-DSI.crt
- Click Install Certificate
- Select Local Machine, click Next
- Choose Place all certificates in the following store
- Click Browse, select Trusted Root Certification Authorities, click OK
- Click Next, then Finish, and confirm Yes
- Restart your browser and navigate to <https://drcedge.ds-inovasi.com:7183>

**⚠ Warning:**

Verify the SHA-256 fingerprint of RootCA-DSI.crt before importing it: openssl x509 -in RootCA-DSI.crt -noout -fingerprint -sha256 - confirm this matches the fingerprint on the server before trusting the file.

# **5\. File and Path Reference**

## **5.1 Administrator-Managed Files**

| **File Path**                                 | **Description**                                 | **Sensitive** |
| --------------------------------------------- | ----------------------------------------------- | ------------- |
| /root/cert/ca/RootCA-DSI.crt                  | Root CA certificate - distribute to all clients | No            |
| /root/cert/ca/RootCA-DSI.key                  | Root CA private key - KEEP OFFLINE BACKUP       | **YES**       |
| /root/cert/csr_out/&lt;node&gt;.key           | Per-host private key                            | YES           |
| /root/cert/csr_out/&lt;node&gt;.csr           | Per-host Certificate Signing Request            | No            |
| /root/cert/signed/&lt;node&gt;.pem            | Signed host certificate                         | No            |
| /root/cert/signed/&lt;node&gt;\_chain.pem     | Certificate + CA chain                          | No            |
| /root/cert/signed/&lt;node&gt;\_key_chain.pem | Key + certificate + CA chain                    | YES           |
| /root/cert/gencert.sh                         | CSR generation script                           | No            |
| /root/cert/signcert.sh                        | Certificate signing script                      | No            |

## **5.2 Cloudera Manager-Managed Files**

| **Path**                                                                       | **Description**                            |
| ------------------------------------------------------------------------------ | ------------------------------------------ |
| /opt/cloudera/AutoTLS/                                                         | AutoTLS root directory (CM-managed)        |
| /opt/cloudera/AutoTLS/hosts-key-store/&lt;host&gt;/cm-auto-host_cert_chain.pem | Per-host certificate chain                 |
| /opt/cloudera/AutoTLS/hosts-key-store/&lt;host&gt;/cm-auto-host_key.pem        | Per-host private key                       |
| /opt/cloudera/AutoTLS/hosts-key-store/&lt;host&gt;/cm-auto-host_keystore.jks   | Per-host JKS keystore                      |
| /opt/cloudera/AutoTLS/hosts-key-store/&lt;host&gt;/cm-auto-host_key.pw         | Per-host keystore password                 |
| /opt/cloudera/AutoTLS/trust-store/cm-auto-global_cacerts.pem                   | Global CA certificate (PEM)                |
| /opt/cloudera/AutoTLS/trust-store/cm-auto-global_truststore.jks                | Global truststore (JKS)                    |
| /opt/cloudera/AutoTLS/trust-store/cm-auto-in_cluster_ca_cert.pem               | In-cluster CA certificate                  |
| /var/lib/cloudera-scm-agent/agent-cert/                                        | Certificates deployed to each node's agent |

## **5.3 Key Configuration Parameters**

| **Parameter**           | **Value**                                      |
| ----------------------- | ---------------------------------------------- |
| Keystore Password       | cloudera1234 (min 12 chars, alphanumeric only) |
| Truststore Password     | cloudera1234 (same as keystore)                |
| Certificate Validity    | 1825 days (5 years - expires April 2031)       |
| Key Size                | 4096-bit RSA                                   |
| Signature Algorithm     | SHA-256                                        |
| CM Port before Auto-TLS | 7180 (HTTP)                                    |
| CM Port after Auto-TLS  | 7183 (HTTPS)                                   |
| AutoTLS Location        | /opt/cloudera/AutoTLS                          |
| API Endpoint            | POST /api/v41/cm/commands/generateCmca         |

# **6\. Troubleshooting**

## **6.1 Common Errors and Resolutions**

| **Error Message**                                       | **Root Cause**                                                            | **Resolution**                                                                                      |
| ------------------------------------------------------- | ------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------- |
| No directory found for host 'X.ds-inovasi.com'          | Host is registered in CM but has no corresponding entry in hostCerts      | Delete the host from CM via DELETE /api/v41/hosts/&lt;hostId&gt;, or add a certificate entry for it |
| Keystore was tampered with, or password was incorrect   | Stale keystore from a previous CM state does not match the new password   | Restart CM Server: systemctl restart cloudera-scm-server                                            |
| Certificate does not have TLS Web Client Authentication | extendedKeyUsage extension is absent from the certificate                 | Regenerate the certificate with extendedKeyUsage = serverAuth, clientAuth in the .ext file          |
| certmanager not running as root                         | File ownership on staging directory prevents CM from reading certificates | Run: chown -R cloudera-scm:cloudera-scm /tmp/auto-tls/ and retry the API call                       |
| Failed to load keystore file with given password        | Special characters in keystore password                                   | Use an alphanumeric-only password. Remove /opt/cloudera/AutoTLS and re-run the API                  |
| CM Agent does not heartbeat after restart               | Agent fails to bind on IPv6-disabled host (OPSAPS-77429 / CM 7.13.2)      | Add listening_ip=&lt;node_IPv4&gt; to /etc/cloudera-scm-agent/config.ini and restart the agent      |

## **6.2 Diagnostic Commands**

### **Check CM Agent status across all nodes:**

for host in drcedge drcmtr01 drcmtr02 drcnifi01 drcwrk02 drcwrk03 \\

rndmaster01 rndmaster02 rndwrk01 rndwrk02 rndwrk03; do

echo -n "\$host: "

ssh \${host}.ds-inovasi.com 'systemctl is-active cloudera-scm-agent'

done

### **Search CM Server log for errors:**

grep -E "ERROR|Failed|AutoTLS|generateCmca" \\

/var/log/cloudera-scm-server/cloudera-scm-server.log | tail -30

### **Verify certificates deployed to a node's agent:**

ls -lh /var/lib/cloudera-scm-agent/agent-cert/

### **Validate certificate chain against Root CA:**

openssl verify -CAfile /root/cert/ca/RootCA-DSI.crt \\

/root/cert/signed/drcedge.ds-inovasi.com.pem

\# Expected output: drcedge.ds-inovasi.com.pem: OK

### **Reset and re-run generateCmca if required:**

\# Archive the failed AutoTLS directory

mv /opt/cloudera/AutoTLS /opt/cloudera/AutoTLS.bak.\$(date +%Y%m%d%H%M)

\# Recreate the directory with correct ownership

mkdir -p /opt/cloudera/AutoTLS

chown cloudera-scm:cloudera-scm /opt/cloudera/AutoTLS

\# Re-run the generateCmca API call (Section 4.6)

# **7\. Security Guidelines**

## **7.1 Root CA Key Management**

| **Practice**      | **Details**                                                           |
| ----------------- | --------------------------------------------------------------------- |
| Storage location  | Restricted to /root/cert/ca/ on drcedge - never copied to other nodes |
| File permissions  | chmod 600 /root/cert/ca/RootCA-DSI.key (root read-only)               |
| Offline backup    | Back up to an encrypted USB device immediately after generation       |
| Rotation planning | Plan CA rotation at least 6 months before expiry (April 2031)         |
| Audit logging     | Log all CA key usage events whenever signing new certificates         |

## **7.2 Certificate Best Practices**

- **Avoid wildcard certificates:** use individual per-host certificates to limit the blast radius of any compromise
- **Monitor expiry:** configure alerts 90 days before certificate expiration (April 2031)
- **Never share private keys:** each host must have its own dedicated key pair
- **Restrict SAN scope:** include only the hostnames actually in use for each node
- **Store passwords securely:** use a password manager; never hard-code credentials in scripts

# **8\. References**

## **8.1 Cloudera Documentation**

| **Title**                                      | **URL / Reference**                                                                                                             |
| ---------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| Encrypting Data in Transit - Auto-TLS Overview | <https://docs.cloudera.com/cdp-private-cloud-base/7.3.1/security-encrypting-data-in-transit/topics/cm-security-auto-tls.html>   |
| Use Case 3: Existing Certs with Custom Root CA | <https://docs.cloudera.com/cdp-private-cloud-base/7.3.1/security-encrypting-data-in-transit/topics/cm-security-use-case-3.html> |
| CM API Reference: generateCmca                 | <https://docs.cloudera.com/cdp-private-cloud-base/7.3.1/api/v41/apidocs.html>                                                   |
| CM TLS/SSL Configuration Guide                 | <https://docs.cloudera.com/cloudera-manager/7.13.2/security-encrypting-data-in-transit/>                                        |
| CM Agent IPv6 Bind Bug - OPSAPS-77429          | Cloudera Support KB: CM Agent fails to bind when IPv6 is disabled (CM 7.13.2)                                                   |

## **8.2 OpenSSL Command Reference**

| **Command**                                                         | **Purpose**                           |
| ------------------------------------------------------------------- | ------------------------------------- |
| openssl genrsa -out key.pem 4096                                    | Generate a 4096-bit RSA private key   |
| openssl req -new -x509 -key ca.key -out ca.crt -days N              | Generate a self-signed CA certificate |
| openssl req -new -key host.key -out host.csr                        | Generate a CSR from a private key     |
| openssl x509 -req -in host.csr -CA ca.crt -CAkey ca.key             | Sign a CSR with the CA                |
| openssl x509 -in cert.pem -noout -text                              | Display full certificate details      |
| openssl x509 -in cert.pem -noout -fingerprint -sha256               | Display SHA-256 fingerprint           |
| openssl verify -CAfile ca.crt cert.pem                              | Verify certificate against a CA       |
| openssl pkcs12 -export -in cert.pem -inkey key.pem -out out.p12     | Convert PEM to PKCS12                 |
| keytool -importkeystore -srckeystore p12 -srcstoretype PKCS12       | Convert PKCS12 to JKS                 |
| keytool -importcert -alias CA -file ca.crt -keystore truststore.jks | Create a JKS truststore               |

## **8.3 CM REST API Endpoints Used**

| **Method** | **Endpoint**                         | **Purpose**                               |
| ---------- | ------------------------------------ | ----------------------------------------- |
| POST       | /api/v41/cm/commands/generateCmca    | Enable Auto-TLS with custom certificates  |
| GET        | /api/v41/hosts                       | List all hosts registered in CM           |
| DELETE     | /api/v41/hosts/&lt;hostId&gt;        | Remove a host from CM                     |
| GET        | /api/v41/cm/version                  | Verify CM is running and retrieve version |
| POST       | /api/v41/cm/service/commands/restart | Restart the CM Management Service         |

# **9\. Change Log**

| **Version** | **Date**    | **Author**     | **Changes**                                               |
| ----------- | ----------- | -------------- | --------------------------------------------------------- |
| 1.0         | May 4, 2026 | DSI Infra Team | Initial release - Auto-TLS Use Case 3 with new RootCA-DSI |

_- End of Document -_

DS-Inovasi Infrastructure Team | Internal Document - Confidential
