# Setup Consul on Kubernetes Cluster


Setup Consul
```bash
helm repo add hashicorp https://helm.releases.hashicorp.com
kubectl create secret generic consul-gossip-encryption-key --from-literal=key=$(consul keygen)
helm install consul -f consul.yaml hashicorp/consul
```

Get the bootstrap token for authenticating to Consul

```bash
kubectl get secrets consul-bootstrap-acl-token -o json | jq -cr .data.token | base64 -d
```


Enable port forwarding so that you can connect to the Consul UI

```bash
kubectl port-forward service/consul-server 8500:8500
```

Go to the ACL / Polices page and create a policy for Vault and set it to vaild for the current data centre (dc1)

```hcl
{
  "key_prefix": {
    "vault/": {
      "policy": "write"
    }
  },
  "node_prefix": {
    "": {
      "policy": "write"
    }
  },
  "service": {
    "vault": {
      "policy": "write"
    }
  },
  "agent_prefix": {
    "": {
      "policy": "write"
    }
  },
  "session_prefix": {
    "": {
      "policy": "write"
    }
  }
}
```

Then go to tokens and create a token for vault restricted to this datacenter for the role vault




Create a Vault secret


```bash
echo <<'EOF' >> vault-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: vault-eks-creds
data:
  AWS_ACCESS_KEY_ID: VkFS
  AWS_SECRET_ACCESS_KEY: VkFS
EOF
```

Replace the "VkFS" with the secret and key from AWS in base64 format (without the newline)

```bash
echo -n "VAR" | base64
```

Upload the secret to the cluster

```bash
kubectl create -f vault-secret.yaml
```

Create the vault helm config

```yaml
global:
  enabled: true
  tlsDisable: true
server:
  ingress:
    enabled: false
  extraSecretEnvironmentVars:
    - envName: AWS_ACCESS_KEY_ID
      secretName: vault-eks-creds
      secretKey: AWS_ACCESS_KEY_ID
    - envName: AWS_SECRET_ACCESS_KEY
      secretName: vault-eks-creds
      secretKey: AWS_SECRET_ACCESS_KEY
  ha:
    enabled: true
    config: |
      ui = true
      max_lease_ttl = "8761h"
      listener "tcp" {
        address = "[::]:8200"
        cluster_address = "[::]:8201"
        tls_disable = "true"
      }
      seal "awskms" {
        region     = "eu-west-2"
        kms_key_id = "KMS_KEY_ID"
      }
      storage "consul" {
        path = "vault/"
        address = "HOST_IP:8500"
        scheme = "http"
        token = "CONSUL_TOKEN"
        tls_skip_verify = "true"
      }
```
helm install vault -f vault.yaml hashicorp/vault

Wait for vault to start then forward a connection to the vault server.

```bash
kubectl port-forward service/vault 8200:8200
```

Export the Vault_Address (from the port-forward above) then initialise the vault

```bash
export VAULT_ADDR="http://127.0.0.1:8200"
vault operator init
```

Keep a copy of the Tokens and keys generated offline somewhere safe. (Ideally a safe)


