# HashiCorp Vault on vSphere Kubernetes Service

This repository contains a proof-of-concept installation of HashiCorp Vault on VMware vSphere Kubernetes Service (VKS). This repository leverages Helm to install Vault.

This installation uses self-signed certificates for internal Raft communication and Vault connections, and it is not intended as an enterprise ready installation. While the `helm-values.yaml` file does adhere to security best practices, it is not intended for a production installation, and you should follow the official HashiCorp Vault documentation.

This documentation and all accompanying code, scripts, and manifests are provided **"AS IS" WITHOUT WARRANTY OF ANY KIND**, either expressed or implied, including but not limited to the implied warranties of merchantability, fitness for a particular purpose, or non-infringement. The entire risk as to the quality, execution, and performance of these steps is borne entirely by you. In no event shall the authors be liable for any damages, system failures, data loss, or production outages resulting from the use of this guide. Use these materials at your own discretion.

## Test Environment
- vSphere 9.1.0 (Should work on 8.0u3i or later)
- VKS 3.7 (Should work on most VKS versions)
- Kubernetes 1.36.1 (refer to Vault documentation for supported versions)
- Cert-Manager 1.21.1
- Antrea CNI
- Self-Signed cluster-issuer and certificate for internal Vault communication and Web UI

## Repository Manifests

| File | Description | Required? |
| --- | --- | --- |
| `01-example-vks-cluster.yaml` | Example VKS cluster manifest with Cert-Manager addon | Optional |
| `02-vault-namespace.yaml` | Creates the `vault` namespace | Yes |
| `03-vault-self-signed-issuer.yaml` | cert-manager ClusterIssuer for internal Raft TLS | Yes |
| `04-vault-agent-test.yaml` | Test application for validate secret injection | Optional |
| `example-values.yaml` | Custom Helm values for Vault HA cluster | Yes |

## Prerequisites

- vSphere / VCF environment with Supervisor enabled
- Helm

## Installation

### Step 1: Create VKS Cluster 

We've provided an example VKS cluster manifest with the cert-manager addon.  You can choose to use this for testing or create your own and add cert-manager.
```bash
kubectl apply -f 01-example-vks-cluster.yaml
```

### Step 2: Create Vault Namespace

Create the namespace for the vault install.  Label for the correct pod security admission (PSA) if required.

```bash
kubectl apply -f 02-vault-namespace.yaml
```

### Step 3: Configure TLS Issuers

Apply the self-signed cluster issuer/secret for internal HA communication (requires SANs) and external Load Balancer

```bash
kubectl apply -f 04-vault-self-signed-issuer.yaml
```

### Step 4: Add Helm Repository

Add the Hashicorp helm repository and verify latest version. 

```bash
helm repo add hashicorp https://helm.releases.hashicorp.com
helm search repo hashicorp/vault

NAME                                    CHART VERSION   APP VERSION     DESCRIPTION
hashicorp/vault                         0.34.1          2.0.4           Official HashiCorp Vault Chart
```
### Step 5: Adjust Helm Values and Install Vault

An example HA-based `example-values.yaml` file with appropriate hardening and configuration is provided. Consult the Vault documentation for specific configuration options.  Based on the output of the `helm search` command, update the `example-values.yaml` with the correct app version to match the chart version you are installiing.

```
helm install vault hashicorp/vault --version 0.34.1 --values example-values.yaml --namespace vault
```

**Note:** Immediately after installing, if you check `kubectl get po -n vault`, the Vault pods will show as `0/1 READY`. This is expected behavior; they will fail their readiness probes until they are initialized and unsealed in the next section.

### Step 6: Create DNS record to match vault-ui service External-IP

```
k get svc -n vault
NAME                       TYPE           CLUSTER-IP       EXTERNAL-IP      PORT(S)             AGE
vault                      ClusterIP      10.98.11.67      <none>           8200/TCP,8201/TCP   10m
vault-active               ClusterIP      10.98.189.6      <none>           8200/TCP,8201/TCP   10m
vault-agent-injector-svc   ClusterIP      10.100.227.152   <none>           443/TCP             10m
vault-internal             ClusterIP      None             <none>           8200/TCP,8201/TCP   10m
vault-standby              ClusterIP      10.105.216.97    <none>           8200/TCP,8201/TCP   10m
vault-ui                   LoadBalancer   10.98.221.73     192.168.150.19   8200:31087/TCP      10m
```

## Initialization & Unsealing

### Step 1: Initialize Vault-0

```bash
# We need to supply the certificate since we used a self-signed cert
kubectl exec -i vault-0 -n vault -- vault operator init \
  -format=json > vault-keys.json

# Secure vault-keys.json  
chmod 600 vault-keys.json
```

**IMPORTANT:** Securely store vault-keys.json file which contains the unseal keys and root_token. If you lose these, you will lose access to Vault.

### Step 2: Unseal Raft Nodes

You must unseal each node in the HA cluster using 3 of the 5 generated keys.

#### Vault-0 (Active Leader)

```bash
kubectl exec -i vault-0 -n vault -- vault operator unseal $(jq -r '.unseal_keys_b64[0]' vault-keys.json)
kubectl exec -i vault-0 -n vault -- vault operator unseal $(jq -r '.unseal_keys_b64[1]' vault-keys.json)
kubectl exec -i vault-0 -n vault -- vault operator unseal $(jq -r '.unseal_keys_b64[2]' vault-keys.json)
```
Expected Output after Final Unseal

```text
Seal Type               shamir
Initialized             true                  <---- Initialized True
Sealed                  false                 <---- Sealed False
Total Shares            5
Threshold               3
Version                 2.0.3
Build Date              2026-06-17T12:39:45Z
Storage Type            raft
Cluster Name            vault-raft01
Cluster ID              f11790eb-9494-b956-7acf-d73279fde946
Removed From Cluster    false
HA Enabled              true                                    <---- HA Enabled
HA Cluster              https://vault-0.vault-internal:8201     <---- HA Leader is Vault-0
HA Mode                 active
Active Since            2026-09-30T22:33:59.36880955Z
Raft Committed Index    38
Raft Applied Index      38
```
**Note:** Occasionaly I'd notice the Sealed field didn't show false after the final unseal command; so I reran the same command again which then showed unsealed.

#### Vault-1 (Standby)

```bash
kubectl exec -i vault-1 -n vault -- vault operator unseal $(jq -r '.unseal_keys_b64[0]' vault-keys.json)
kubectl exec -i vault-1 -n vault -- vault operator unseal $(jq -r '.unseal_keys_b64[1]' vault-keys.json)
kubectl exec -i vault-1 -n vault -- vault operator unseal $(jq -r '.unseal_keys_b64[2]' vault-keys.json)
```
Expected Output after Final Unseal (truncated)
```text
Seal Type               shamir
Initialized             true                  <---- Initialized True
Sealed                  false                 <---- Sealed False
...

**Note:** Occasionaly I'd notice the Sealed field didn't show false so I reran the command.
```

#### Vault-2 (Standby)

```bash
kubectl exec -i vault-2 -n vault -- vault operator unseal $(jq -r '.unseal_keys_b64[0]' vault-keys.json)
kubectl exec -i vault-2 -n vault -- vault operator unseal $(jq -r '.unseal_keys_b64[1]' vault-keys.json)
kubectl exec -i vault-2 -n vault -- vault operator unseal $(jq -r '.unseal_keys_b64[2]' vault-keys.json)
```

Expected Output after Final Unseal (truncated)
```text
Seal Type               shamir
Initialized             true                  <---- Initialized True
Sealed                  false                 <---- Sealed False
...
```

### Step 3: Verify Pod and UI Readiness

```bash
kubectl get po -n vault

NAME                                   READY   STATUS    RESTARTS   AGE
vault-0                                1/1     Running   0          27m
vault-1                                1/1     Running   0          27m
vault-2                                1/1     Running   0          27m
vault-agent-injector-c94798c8d-22sj8   1/1     Running   0          27m

```

Validate the Web UI is reachable via the ingress route configured:

```bash
curl -vv https://vault.example.com:8200

```

You can now log into the Web UI using the `root_token` found in the `vault-keys.json` file.
https://vault.example.com:8200
## Workload Testing

### Vault Base Configuration (KV Engine, Policy, Auth)

This section covers enabling the KV V2 engine, creating a sample secret, enabling Kubernetes Auth, and creating an application policy and role. This sets the foundation for testing and is **NOT INTENDED** as a production example.

1. Login to vault-0 with root_token from `vault-keys.json`:
```bash
kubectl exec -it vault-0 -n vault -- vault login <ROOT_TOKEN>
```

2. Enable KV V2 Engine:
```bash
kubectl exec -it vault-0 -n vault -- vault secrets enable -path=secret kv-v2
```

3. Create a sample secret:
```bash
kubectl exec -it vault-0 -n vault -- vault kv put secret/config api_key="secret-key-9988" env="production"
```

4. Enable Kubernetes Auth:
```bash
kubectl exec -it vault-0 -n vault -- vault auth enable kubernetes
```

5. Configure Kubernetes Auth:
```bash
kubectl exec -it vault-0 -n vault -- vault write auth/kubernetes/config \
    kubernetes_host="https://kubernetes.default.svc:443"
```

6. Create an App Policy:
```bash
kubectl exec -it vault-0 -n vault -- vault policy write app-policy - <<EOF
path "secret/data/config" {
  capabilities = ["read"]
}
EOF
```

7. Write the Role:
```bash
kubectl exec -it vault-0 -n vault -- vault write auth/kubernetes/role/app-role \
    bound_service_account_names=vault-test-sa \
    bound_service_account_namespaces=default \
    policies=app-policy \
    ttl=1h

# You can safely ignore the audience warning for testing. For production, consult official Vault documentation.
```

8. Copy the TLS secret to the `default` namespace. This is required to allow the Vault Agent sidecar to validate the self-signed certificate:
```bash
kubectl get secret vault-tls-secret -n vault -o json | \
  jq 'del(.metadata.namespace, .metadata.uid, .metadata.resourceVersion, .metadata.creationTimestamp)' | \
  kubectl apply -n default -f -
```

### Test Vault Agent Injector (Sidecar)

1. Deploy the test app in the default namespace:
```bash
kubectl apply -f vault-agent-test.yaml
```

2. Verify the Pod is running (2/2 containers ready) and the secret was populated:
```bash
kubectl get po

NAME               READY   STATUS    RESTARTS   AGE
vault-agent-test   2/2     Running   0          30s

# Exec into Pod and validate secret was written
kubectl exec -it vault-agent-test -n default -c app -- cat /vault/secrets/config.txt

data: map[api_key:secret-key-9988 env:production]
metadata: map[created_time:2026-10-01T00:57:22.265091976Z custom_metadata:<nil> deletion_time: destroyed:false version:1]
```


## Troubleshooting

**Error: INSTALLATION FAILED: conflict occurred while applying object**

If you receive a webhook conflict error (e.g., `conflict with "vault-k8s" using admissionregistration.k8s.io/v1`), it is typically caused by a leftover webhook from a previous installation.

```bash
kubectl delete mutatingwebhookconfiguration vault-agent-injector-cfg
helm delete vault -n vault
helm install vault hashicorp/vault --values helm-values.yaml --namespace vault

```