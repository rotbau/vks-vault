# HashiCorp Vault on vSphere Kubernetes Service

This repository contains a proof-of-concept installation of HashiCorp Vault on VMware vSphere Kubernetes Service (VKS). This repository leverages Helm to install Vault.

This installation uses self-signed certificates for internal Raft communication and Vault connections, and it is not intended as an enterprise ready installation. While the `helm-values.yaml` file does adhere to security best practices, it is not intended for a production installation, and you should follow the official HashiCorp Vault documentation.

This documentation and all accompanying code, scripts, and manifests are provided **"AS IS" WITHOUT WARRANTY OF ANY KIND**, either expressed or implied, including but not limited to the implied warranties of merchantability, fitness for a particular purpose, or non-infringement. The entire risk as to the quality, execution, and performance of these steps is borne entirely by you. In no event shall the authors be liable for any damages, system failures, data loss, or production outages resulting from the use of this guide. Use these materials at your own discretion.

## Test Environment
- vSphere 9.1.0 (Should work on 8.0u3i or later)
- VKS 3.7 (Should work on most VKS versions)
- Kubernetes 1.36.1 (refer to Vault documentation for supported versions)
- Cert-Manager 1.21.1
- Contour 1.33.4 (VKS addon)
- Antrea CNI
- Self-Signed cluster-issuer and certificate for internal Vault communication
- Lets Encrypt certificate for Vault Web UI

## Repository Manifests

| File | Description | Required? |
| --- | --- | --- |
| `01-contour-addon.yaml` | Deploys Contour Gateway API for UI ingress | Optional |
| `02-vault-namespace.yaml` | Creates the `vault` namespace | Yes |
| `03-gateway.yaml` | GatewayClass and Gateway routing rules | Optional |
| `04-vault-self-signed-issuer.yaml` | cert-manager ClusterIssuer for internal Raft TLS | Yes |
| `05-vault-external-tls-secret.yaml` | Let's Encrypt TLS secret for external UI access | Optional |
| `06-vault-httproute.yaml` | Httproute object for Vault Web UI | Optional |
| `07-vault-csi-provider-class.yaml` | SecretProviderClass for CSI Driver testing | Optional |
| `helm-values.yaml` | Custom Helm values for Vault HA cluster | Yes |

## Prerequisites

1. **VKS Cluster:** Create a VKS Cluster if needed. The test environment uses a single control plane (2 vCPU x 8 GB) and 3 worker nodes (4 vCPU x 16 GB). Sizing should depend on requirements and Vault documentation.
2. **Ingress / Gateway API:** We are exposing the Vault Web UI using Gateway API via the VKS Contour addon. If you prefer to use another method, you can skip deploying `01-contour-addon.yaml` and `03-gateway.yaml` to your VKS cluster.

## Installation

### Step 1: Deploy Contour VKS Addon

**Note:** This is done from the Supervisor cluster context and created in the same vsphere namespace as the VKS Cluster.  This can be applied to an existing cluster.  See the VKS Addon documentation for more details.
```bash
kubectl apply -f 01-contour-addon.yaml

# You should see the tanzu-system-ingress namesapce created on your VKS Cluster with the contour and envoy pods
```
### Step 2: Create Vault Namespace and Deploy GatewayClass and Gateway Object
Create the Vault namespace and setup the Gateway API routing (skip Contour/Gateway steps if using an alternative ingress controller).

```bash
kubectl apply -f 02-vault-namespace.yaml
kubectl apply -f 03-gateway.yaml
```

### Step 3: Configure TLS Issuers

Apply the self-signed cluster issuer/secret for internal HA communication (requires SANs) and the Let's Encrypt TLS secret for the Web UI.

```bash
kubectl apply -f 04-vault-self-signed-issuer.yaml
kubectl apply -f 05-vault-external-tls-secret.yaml
```

### Step 4: Helm Install

An example HA-based `helm-values.yaml` file with appropriate hardening and configuration is provided. Consult the Vault documentation for specific configuration options.

```bash
helm install vault hashicorp/vault --values helm-values.yaml --namespace vault
```

**Note:** Immediately after installing, if you check `kubectl get po -n vault`, the Vault pods will show as `0/1 READY`. This is expected behavior; they will fail their readiness probes until they are initialized and unsealed in the next section.

### Step 5: Deploy httproute (skip if not using contour)
```bash
kubectl apply -f 06-vault-httproute.yaml
```

## Initialization & Unsealing

### Step 1: Initialize Vault-0

```bash
# We need to supply the certificate since we used a self-signed cert
kubectl exec -i vault-0 -n vault -- vault operator init \
  -ca-cert=/vault/userconfig/vault-internal-tls/ca.crt \
  -format=json > vault-keys.json

# Secure vault-keys.json  
chmod 600 vault-keys.json
```

**IMPORTANT:** Securely store vault-keys.json file which contains the unseal keys and root_token. If you lose these, you will lose access to Vault.

### Step 2: Unseal Raft Nodes

You must unseal each node in the HA cluster using 3 of the 5 generated keys.

#### Vault-0 (Active Leader)

```bash
kubectl exec -i vault-0 -n vault -- vault operator unseal -ca-cert=/vault/userconfig/vault-internal-tls/ca.crt $(jq -r '.unseal_keys_b64[0]' vault-keys.json)
kubectl exec -i vault-0 -n vault -- vault operator unseal -ca-cert=/vault/userconfig/vault-internal-tls/ca.crt $(jq -r '.unseal_keys_b64[1]' vault-keys.json)
kubectl exec -i vault-0 -n vault -- vault operator unseal -ca-cert=/vault/userconfig/vault-internal-tls/ca.crt $(jq -r '.unseal_keys_b64[2]' vault-keys.json)
```
Expected Output

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

#### Vault-1 (Standby)

```bash
kubectl exec -i vault-1 -n vault -- vault operator unseal -ca-cert=/vault/userconfig/vault-internal-tls/ca.crt $(jq -r '.unseal_keys_b64[0]' vault-keys.json)
kubectl exec -i vault-1 -n vault -- vault operator unseal -ca-cert=/vault/userconfig/vault-internal-tls/ca.crt $(jq -r '.unseal_keys_b64[1]' vault-keys.json)
kubectl exec -i vault-1 -n vault -- vault operator unseal -ca-cert=/vault/userconfig/vault-internal-tls/ca.crt $(jq -r '.unseal_keys_b64[2]' vault-keys.json)
```
Expected Output (truncated)
```text
Seal Type               shamir
Initialized             true                  <---- Initialized True
Sealed                  false                 <---- Sealed False
...
```

#### Vault-2 (Standby)

```bash
kubectl exec -i vault-2 -n vault -- vault operator unseal -ca-cert=/vault/userconfig/vault-internal-tls/ca.crt $(jq -r '.unseal_keys_b64[0]' vault-keys.json)
kubectl exec -i vault-2 -n vault -- vault operator unseal -ca-cert=/vault/userconfig/vault-internal-tls/ca.crt $(jq -r '.unseal_keys_b64[1]' vault-keys.json)
kubectl exec -i vault-2 -n vault -- vault operator unseal -ca-cert=/vault/userconfig/vault-internal-tls/ca.crt $(jq -r '.unseal_keys_b64[2]' vault-keys.json)
```

Expected Output (truncated)
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
curl -vv https://vault01.vtechk8s.com

```

You can now log into the Web UI using the `root_token` found in the `vault-keys.json` file.

## Workload Testing

### Vault Base Configuration (KV Engine, Policy, Auth)

This section covers enabling the KV V2 engine, creating a sample secret, enabling Kubernetes Auth, and creating an application policy and role. This sets the foundation for testing and is **NOT INTENDED** as a production example.

1. Login to vault-0 with root_token from `vault-keys.json`:
```bash
kubectl exec -it vault-0 -n vault -- vault login -ca-cert=/vault/userconfig/vault-internal-tls/ca.crt <ROOT_TOKEN>
```

2. Enable KV V2 Engine:
```bash
kubectl exec -it vault-0 -n vault -- vault secrets enable -ca-cert=/vault/userconfig/vault-internal-tls/ca.crt -path=secret kv-v2
```

3. Create a sample secret:
```bash
kubectl exec -it vault-0 -n vault -- vault kv put -ca-cert=/vault/userconfig/vault-internal-tls/ca.crt secret/config api_key="secret-key-9988" env="production"
```

4. Enable Kubernetes Auth:
```bash
kubectl exec -it vault-0 -n vault -- vault auth enable -ca-cert=/vault/userconfig/vault-internal-tls/ca.crt kubernetes
```

5. Configure Kubernetes Auth:
```bash
kubectl exec -it vault-0 -n vault -- vault write -ca-cert=/vault/userconfig/vault-internal-tls/ca.crt auth/kubernetes/config \
    kubernetes_host="https://kubernetes.default.svc:443"
```

6. Create an App Policy:
```bash
kubectl exec -it vault-0 -n vault -- vault policy write -ca-cert=/vault/userconfig/vault-internal-tls/ca.crt app-policy - <<EOF
path "secret/data/config" {
  capabilities = ["read"]
}
EOF
```

7. Write the Role:
```bash
kubectl exec -it vault-0 -n vault -- vault write -ca-cert=/vault/userconfig/vault-internal-tls/ca.crt auth/kubernetes/role/app-role \
    bound_service_account_names=vault-test-sa \
    bound_service_account_namespaces=default \
    policies=app-policy \
    ttl=1h

# You can safely ignore the audience warning for testing. For production, consult official Vault documentation.
```

8. Copy the TLS secret to the `default` namespace. This is required to allow the Vault Agent sidecar to validate the self-signed certificate:
```bash
kubectl get secret vault-internal-tls-secret -n vault -o json | \
  jq 'del(.metadata.namespace, .metadata.uid, .metadata.resourceVersion, .metadata.creationTimestamp)' | \
  kubectl apply -n default -f -
```

### Pattern A: Vault Agent Injector (Sidecar)

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

### Pattern B: Vault Secrets Store CSI Driver (Optional)

This test uses the Vault Secrets Store CSI driver instead of the sidecar Agent pattern. The `helm-values.yaml` has the CSI driver commented out. To use this, you must uncomment the CSI section in your values file and update the Vault Helm installation.

```bash
kubectl delete mutatingwebhookconfiguration vault-agent-injector-cfg --ignore-not-found
helm upgrade vault hashicorp/vault --values helm-values.yaml --namespace vault
```

1. Install the Core Secrets Store Driver:
```bash
helm repo add secrets-store-csi-driver https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts
helm repo update

helm install csi-secrets-store secrets-store-csi-driver/secrets-store-csi-driver --namespace kube-system

kubectl get csidriver
```

2. Create the `SecretProviderClass`:
```bash
kubectl apply -f 07-vault-csi-provider-class.yaml
```

3. Deploy the test app in the default namespace:
```bash
kubectl apply -f vault-csi-test.yaml
```

4. Verify the Pod is running and the secret was mounted:
```bash
kubectl get pod vault-csi-test -n default

NAME               READY   STATUS    RESTARTS   AGE
vault-csi-test     1/1     Running   0          3s

kubectl exec -it vault-csi-test -n default -- cat /vault/secrets/config.txt

secret-key-9988
```

## Troubleshooting

**Error: INSTALLATION FAILED: conflict occurred while applying object**

If you receive a webhook conflict error (e.g., `conflict with "vault-k8s" using admissionregistration.k8s.io/v1`), it is typically caused by a leftover webhook from a previous installation.

```bash
kubectl delete mutatingwebhookconfiguration vault-agent-injector-cfg
helm delete vault -n vault
helm install vault hashicorp/vault --values helm-values.yaml --namespace vault

```