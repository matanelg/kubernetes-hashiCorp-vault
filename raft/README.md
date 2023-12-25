# Vault installation to minikube via Helm with Integrated Storage

## Source

## Notes

- Source
  - [Vault installation to minikube via Helm with TLS enabled](https://developer.hashicorp.com/vault/tutorials/kubernetes/kubernetes-minikube-tls)
- Main Changes
  - Using RAFT storage instead consul.
    - Deploy 3 vault servers in ha mode.
    - Apply latest chart & app versions.
  - TLS
    - Change host domains, needed to include raft cluster & services dns. (\*.vault-internal)
    - Generate certificate via openssl.
    - Use kubernetes CertificateSigningRequest.
  - Application
    - Fix injected dynamic variable (MYSQL_PASSWORD > MYSQL_PORT)

## Create minikube cluster

```bash
minikube start -p vault-raft --cpus 4 --memory 4096
```

## TLS

Create tls secrets & apply to vault namespace

```bash
mkdir -p ./tmp/vault

#
export VAULT_K8S_NAMESPACE="vault" \
export VAULT_HELM_RELEASE_NAME="vault" \
export VAULT_SERVICE_NAME="vault-internal" \
export K8S_CLUSTER_NAME="cluster.local" \
export WORKDIR=./tmp/vault

#
openssl genrsa -out ${WORKDIR}/vault.key 2048

#
cat > ${WORKDIR}/vault-csr.conf <<EOF
[req]
default_bits = 2048
prompt = no
encrypt_key = yes
default_md = sha256
distinguished_name = kubelet_serving
req_extensions = v3_req
[ kubelet_serving ]
O = system:nodes
CN = system:node:*.${VAULT_K8S_NAMESPACE}.svc.${K8S_CLUSTER_NAME}
[ v3_req ]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth, clientAuth
subjectAltName = @alt_names
[alt_names]
DNS.1 = *.${VAULT_SERVICE_NAME}
DNS.2 = *.${VAULT_SERVICE_NAME}.${VAULT_K8S_NAMESPACE}.svc.${K8S_CLUSTER_NAME}
DNS.3 = *.${VAULT_K8S_NAMESPACE}
IP.1 = 127.0.0.1
EOF

#
openssl req -new -key ${WORKDIR}/vault.key -out ${WORKDIR}/vault.csr -config ${WORKDIR}/vault-csr.conf

#
cat > ${WORKDIR}/csr.yaml <<EOF
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
   name: vault.svc
spec:
   signerName: kubernetes.io/kubelet-serving
   expirationSeconds: 8640000
   request: $(cat ${WORKDIR}/vault.csr|base64|tr -d '\n')
   usages:
   - digital signature
   - key encipherment
   - server auth
EOF

#
kubectl create -f ${WORKDIR}/csr.yaml

#
kubectl certificate approve vault.svc

#
kubectl get csr vault.svc

#
kubectl get csr vault.svc -o jsonpath='{.status.certificate}' | openssl base64 -d -A -out ${WORKDIR}/vault.crt

#
kubectl config view \
--raw \
--minify \
--flatten \
-o jsonpath='{.clusters[].cluster.certificate-authority-data}' \
| base64 -d > ${WORKDIR}/vault.ca

#
kubectl create namespace $VAULT_K8S_NAMESPACE
kubectl config set-context --current --namespace=$VAULT_K8S_NAMESPACE

#
kubectl create secret generic vault-ha-tls \
   -n $VAULT_K8S_NAMESPACE \
   --from-file=vault.key=${WORKDIR}/vault.key \
   --from-file=vault.crt=${WORKDIR}/vault.crt \
   --from-file=vault.ca=${WORKDIR}/vault.ca
```

```bash
# certificatesigningrequest.certificates.k8s.io/vault.svc created
# certificatesigningrequest.certificates.k8s.io/vault.svc approved
# NAME        AGE   SIGNERNAME                      REQUESTOR       REQUESTEDDURATION   CONDITION
# vault.svc   15s   kubernetes.io/kubelet-serving   minikube-user   100d                Approved,Issued
# namespace/vault created
# Context "vault-raft" modified.
# secret/vault-ha-tls created
```

## Configure Vault With RAFT Storage

Get chart values & Edit configuration (vault-raft-values.yml)

```bash
mkdir -p ./helm/manifests
helm show values hashicorp/vault > ./helm/helm-vault-raft-values.yml
# cat ./helm/vault-raft-values.yml
```

Generate & Deploy manifests with configured chart values.

```bash
helm template vault hashicorp/vault \
  --namespace vault \
  --version 0.27.0 \
  -f ./helm/vault-raft-values.yml \
  > ./helm/manifests/vault-raft-storage.yaml
kubectl apply -f ./helm/manifests/vault-raft-storage.yaml
watch kubectl get pods
```

```bash
# NAME                                    READY   STATUS    RESTARTS   AGE
# vault-0                                 1/1     Running   0          33s
# vault-1                                 1/1     Running   0          33s
# vault-2                                 1/1     Running   0          33s
# vault-agent-injector-54cf4f74c4-v2mqx   1/1     Running   0          33s
# vault-server-test                       1/1     Running   0          33s
```

Initialize & Unseal vault servers

- Initialize vault-0 with one key share and one key threshold.

```bash
kubectl exec vault-0 -- vault operator init \
    -key-shares=1 \
    -key-threshold=1 \
    -format=json > ${WORKDIR}/cluster-keys.json

# jq -r ".unseal_keys_b64[]" ${WORKDIR}/cluster-keys.json
VAULT_UNSEAL_KEY=$(jq -r ".unseal_keys_b64[]" ${WORKDIR}/cluster-keys.json)

kubectl exec vault-0 -- vault operator unseal $VAULT_UNSEAL_KEY
```

```bash
# Key                     Value
# ---                     -----
# Seal Type               shamir
# Initialized             true
# Sealed                  false
# Total Shares            1
# Threshold               1
# Version                 1.15.2
# Build Date              2023-11-06T11:33:28Z
# Storage Type            raft
# Cluster Name            vault-cluster-63ff686d
# Cluster ID              40dd2760-0bc6-e18d-571d-b0d1a3a4fd8c
# HA Enabled              true
# HA Cluster              https://vault-0.vault-internal:8201
# HA Mode                 active
# Active Since            2023-12-25T06:20:02.77143743Z
# Raft Committed Index    52
# Raft Applied Index      52
```

- Join vault-1 pod to the Raft cluster.

```bash

kubectl exec -n $VAULT_K8S_NAMESPACE -it vault-1 -- sh

#
vault operator raft join -address=https://vault-1.vault-internal:8200 -leader-ca-cert="$(cat /vault/userconfig/vault-ha-tls/vault.ca)" -leader-client-cert="$(cat /vault/userconfig/vault-ha-tls/vault.crt)" -leader-client-key="$(cat /vault/userconfig/vault-ha-tls/vault.key)" https://vault-0.vault-internal:8200

#
exit
```

```bash
# Key       Value
# ---       -----
# Joined    true
```

- Unseal vault-1 server.

```bash
kubectl exec -n $VAULT_K8S_NAMESPACE -ti vault-1 -- vault operator unseal $VAULT_UNSEAL_KEY
```

```bash
# Key                     Value
# ---                     -----
# Seal Type               shamir
# Initialized             true
# Sealed                  false
# Total Shares            1
# Threshold               1
# Version                 1.15.2
# Build Date              2023-11-06T11:33:28Z
# Storage Type            raft
# Cluster Name            vault-cluster-63ff686d
# Cluster ID              40dd2760-0bc6-e18d-571d-b0d1a3a4fd8c
# HA Enabled              true
# HA Cluster              https://vault-0.vault-internal:8201
# HA Mode                 standby
# Active Node Address     https://10.244.0.7:8200
# Raft Committed Index    58
# Raft Applied Index      58
```

- Join vault-2 pod to the Raft cluster.

```bash
kubectl exec -n $VAULT_K8S_NAMESPACE -it vault-2 -- sh
#
vault operator raft join -address=https://vault-2.vault-internal:8200 -leader-ca-cert="$(cat /vault/userconfig/vault-ha-tls/vault.ca)" -leader-client-cert="$(cat /vault/userconfig/vault-ha-tls/vault.crt)" -leader-client-key="$(cat /vault/userconfig/vault-ha-tls/vault.key)" https://vault-0.vault-internal:8200
```

```bash
# Key       Value
# ---       -----
# Joined    true
```

- Unseal vault-2 server.

```bash
kubectl exec -n $VAULT_K8S_NAMESPACE -ti vault-2 -- vault operator unseal $VAULT_UNSEAL_KEY
```

```bash
# Key                     Value
# ---                     -----
# Seal Type               shamir
# Initialized             true
# Sealed                  false
# Total Shares            1
# Threshold               1
# Version                 1.15.2
# Build Date              2023-11-06T11:33:28Z
# Storage Type            raft
# Cluster Name            vault-cluster-63ff686d
# Cluster ID              40dd2760-0bc6-e18d-571d-b0d1a3a4fd8c
# HA Enabled              true
# HA Cluster              https://vault-0.vault-internal:8201
# HA Mode                 standby
# Active Node Address     https://10.244.0.7:8200
# Raft Committed Index    61
# Raft Applied Index      61
```

- Login to one of the vault server for start using.

```bash
export CLUSTER_ROOT_TOKEN=$(cat ${WORKDIR}/cluster-keys.json | jq -r ".root_token")
kubectl exec -n $VAULT_K8S_NAMESPACE vault-0 -- vault login $CLUSTER_ROOT_TOKEN
```

```bash
# Key                  Value
# ---                  -----
# token                hvs.6uxSSoEjm531B59abs2pibby
# token_accessor       K5rx5BOYsgbQKCCVAFK6y3N1
# token_duration       ∞
# token_renewable      false
# token_policies       ["root"]
# identity_policies    []
# policies             ["root"]
```

- Check that all servers are peers to the raft cluster.

```bash
kubectl exec -n $VAULT_K8S_NAMESPACE vault-0 -- vault operator raft list-peers
```

```bash
# Node       Address                        State       Voter
# ----       -------                        -----       -----
# vault-0    vault-0.vault-internal:8201    leader      true
# vault-1    vault-1.vault-internal:8201    follower    true
# vault-2    vault-2.vault-internal:8201    follower    true
```

- Check vault-0 status

```bash
kubectl exec -n $VAULT_K8S_NAMESPACE vault-0 -- vault status
```

```bash
# Key                     Value
# ---                     -----
# Seal Type               shamir
# Initialized             true
# Sealed                  false
# Total Shares            1
# Threshold               1
# Version                 1.15.2
# Build Date              2023-11-06T11:33:28Z
# Storage Type            raft
# Cluster Name            vault-cluster-63ff686d
# Cluster ID              40dd2760-0bc6-e18d-571d-b0d1a3a4fd8c
# HA Enabled              true
# HA Cluster              https://vault-0.vault-internal:8201
# HA Mode                 active
# Active Since            2023-12-25T06:20:02.77143743Z
# Raft Committed Index    65
# Raft Applied Index      65
```

Summary

- as we can see vault-0 is in active mode & the leader on the raft cluster, were both the other servers vault-1 & vault-2 are the follower and in standby mode.

## Setup kubernetes engine & authentication

```bash
kubectl exec -it vault-0 -- sh
vault auth enable kubernetes
vault write auth/kubernetes/config \
  token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
  kubernetes_host=https://${KUBERNETES_PORT_443_TCP_ADDR}:443 \
  kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
```

```bash
# Success! Enabled kubernetes auth method at: kubernetes/
# Success! Data written to: auth/kubernetes/config
```

## Mysql

Create mysql server

```bash
kubectl create ns mysql
kubectl config set-context --current --namespace mysql
kubectl apply -f ./mysql/
watch kubectl get pods
```

```bash
# NAME                     READY   STATUS    RESTARTS   AGE
# mysql-6c78bd9c8f-gpnbq   1/1     Running   0          21s
```

Create a superuser

```bash
kubectl exec -it mysql-6c78bd9c8f-gpnbq bash
mysql -u root --password=password
```

```bash
# Check users
mysql> select user from mysql.user;
# +------------------+
# | user             |
# +------------------+
# | root             |
# | mysql.infoschema |
# | mysql.session    |
# | mysql.sys        |
# | root             |
# +------------------+
# 5 rows in set (0.00 sec)

# Create superuser (username=superuser password=superuser)
mysql> CREATE USER 'superuser'@'%' IDENTIFIED BY 'superuser';
# Query OK, 0 rows affected (0.02 sec)

mysql> GRANT ALL PRIVILEGES ON *.* TO 'superuser' WITH GRANT OPTION;
# Query OK, 0 rows affected (0.02 sec)

# Check users
mysql> select user from mysql.user;
# +------------------+
# | user             |
# +------------------+
# | root             |
# | superuser        |
# | mysql.infoschema |
# | mysql.session    |
# | mysql.sys        |
# | root             |
# +------------------+
# 6 rows in set (0.00 sec)
```

## Dynamic Secrets

Summary - Enable database engine & create config, role, policy and checking user creation.

```bash
kubectl config set-context --current --namespace vault
kubectl exec -it vault-0 vault -- sh

# Enable the database engine
vault secrets enable database

# Configure Database for MYSQL configuration
vault write database/config/mysql-db \
  plugin_name=mysql-database-plugin \
  allowed_roles="mysql-role" \
  connection_url="{{username}}:{{password}}@tcp(mysql.mysql.svc.cluster.local:3306)/" \
  max_open_connections="20" \
  max_connection_lifetime="7200s" \
  username="superuser" \
  password="superuser"

# Create role
vault write database/roles/mysql-role \
  db_name=mysql-db \
  creation_statements="CREATE USER '{{name}}'@'%' IDENTIFIED BY '{{password}}';GRANT SELECT ON *.* TO '{{name}}'@'%';" \
  default_ttl="72h" \
  max_ttl="72h"

# Create new user - check
vault read database/creds/mysql-role

# Create policy with read access for this role
cat <<EOF > /home/vault/mysql-app-policy.hcl
path "database/creds/mysql-role" {
  capabilities = ["read"]
}
EOF
vault policy write mysql-app-policy /home/vault/mysql-app-policy.hcl

# Bind our role to this policy, application service account.
# service-account of the app is app & will deployed on default ns
vault write auth/kubernetes/role/mysql-role \
   bound_service_account_names=app \
   bound_service_account_namespaces=default \
   policies=mysql-app-policy \
   ttl=72h
```

```bash
# Success! Enabled the database secrets engine at: database/
# Success! Data written to: database/config/mysql-db
# Success! Data written to: database/roles/mysql-role
# Key                Value
# ---                -----
# lease_id           database/creds/mysql-role/PhVaNGxrflQlOsjaoBh0X3IE
# lease_duration     72h
# lease_renewable    true
# password           qS2qte5KmVC6KhN4-ZJb
# username           v-root-mysql-role-e9pPiUIZKCkdc6
# Success! Uploaded policy: mysql-app-policy
# Success! Data written to: auth/kubernetes/role/mysql-role
```

## Application

Apply application in the default namespace

```bash
kubectl config set-context --current --namespace default
kubectl apply -f ./app/
kubectl get pods
```

```bash
# NAME                            READY   STATUS    RESTARTS   AGE
# mysql-client-6cbcd77f96-wrtmc   2/2     Running   0          4s
```

Check injected values

```bash
kubectl exec -it mysql-client-6cbcd77f96-wrtmc bash
cat /vault/secrets/mysql-role
```

```bash
# {"MYSQL_USER": v-kubernetes-mysql-role-yjOXaDuB
#   "MYSQL_PASSWORD": xmrosX5-s1ZU9e9653wD
#   "MYSQL_HOST": mysql.mysql.svc.cluster.local
#   "MYSQL_PORT": 3306
# }
```

Connect to db with the injected generated user & list all users.

```bash
mysql -u v-kubernetes-mysql-role-yjOXaDuB --password=xmrosX5-s1ZU9e9653wD -h mysql.mysql.svc.cluster.local --port 3306
select user from mysql.user;
```

```bash
# +----------------------------------+
# | user                             |
# +----------------------------------+
# | root                             |
# | superuser                        |
# | v-kubernetes-mysql-role-yjOXaDuB |
# | v-kubernetes-mysql-role-zwm4GBcs |
# | v-root-mysql-role-e9pPiUIZKCkdc6 |
# | mysql.infoschema                 |
# | mysql.session                    |
# | mysql.sys                        |
# | root                             |
# +----------------------------------+
# 9 rows in set (0.00 sec)
```
