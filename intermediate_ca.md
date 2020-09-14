# TLS enable Vault / Consul and Cert Manager


Firstly export your VAULT_TOKEN for the kubernetes hosted vault

```bash
export VAULT_TOKEN=VAULT_ROOT_TOKEN
```

Create an intermediate PKI store and CSR for the cluster

```bash
vault secrets enable -path=pki_int pki
vault secrets tune -max-lease-ttl=43800h pki_int
vault write -format=json pki_int/intermediate/generate/internal \
        common_name="default.cluster.local  Talos Production Intermediate Authority" \
        | jq -r '.data.csr' > pki_intermediate.csr
```


Sign the certificate using the RootCA against the Vault stored on the Raspberry PI 
(ideally use a different terminal window for this next set of commands)

```bash
export VAULT_ADDR="https://PI_IP_ADDRESS:8200"
export VAULT_TOKEN=PI_ROOT_TOKEN
vault write -tls-skip-verify -format=json pki_root/root/sign-intermediate csr=@pki_intermediate.csr \
        format=pem_bundle ttl="43800h" \
        | jq -r '.data.certificate' > intermediate.cert.pem
```


Upload the signed certificate back to the kubernetes cluster vault
(use the original terminal window again)

```bash
vault write pki_int/intermediate/set-signed certificate=@intermediate.cert.pem
```


Create role to allow one for Consul and one for Cert-Manager to be able to request certificates

```bash
vault write pki_int/roles/default-svc-cluster-local \
        allow_any_name=true \
        allow_bare_domains=true \
        allow_subdomains=true \
        allow_glob_domains=true \
        allow_localhost=true \
        allow_ip_sans=true \
        allowed_other_sans="*" \
        use_csr_common_name=true \
        use_csr_sans=true \
        require_cn=false \
        max_ttl="8760h"

```


Create Vault a TLS certiifcate valid for one year

```bash
vault write pki_int/issue/default-svc-cluster-local common_name="vault.default.svc.cluster.local" ttl="8760h"> out.txt
```

Manually copy the key (including the headers) to a file called *tls.key*, copy the certificate to a file called *tls.crt*.
Then copy the certificates left into a file called *ca.crt* with the Root CA second and the issuing CA first.

Next append the ca.crt to the tls.crt file.
```bash
cat ca.crt >> tls.crt
```


## Upgrade Vault to service via TLS

Create a kubernetes secret that includes the ca with the tls cert and key.

```bash
kubectl create secret generic vault-default-cert --from-file=./tls.crt --from-file=./tls.key --from-file=./ca.crt
```

Update the helm config with the following:
```yaml
global:
  enabled: true
  tlsDisable: false
server:
  ingress:
    enabled: false
  extraEnvironmentVars:
    VAULT_CACERT: "/certificates/ca.crt"
  extraSecretEnvironmentVars:
    - envName: AWS_ACCESS_KEY_ID
      secretName: vault-eks-creds
      secretKey: AWS_ACCESS_KEY_ID
    - envName: AWS_SECRET_ACCESS_KEY
      secretName: vault-eks-creds
      secretKey: AWS_SECRET_ACCESS_KEY
  volumes:
    - name: vault-pod-cert
      secret:
        secretName: vault-default-cert
  volumeMounts:
    - name: vault-default-cert
      mountPath: /certificates
  ha:
    enabled: true
    config: |
      ui = true
      max_lease_ttl = "8761h"
      listener "tcp" {
        address = "[::]:8200"
        cluster_address = "[::]:8201"
        tls_cert_file = "/certificates/tls.crt"
        tls_key_file  = "/certificates/tls.key"
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

```bash
helm upgrade vault -f vault.yaml hashicorp/vault
kubectl delete pod vault-0 vault-1 vault-2
```

As we have restarted vault we will need to reset the port-forward to the kubernetes cluster and update the VAULT_ADDR environment token now we are using TLS

```bash
kubectl port-forward service/vault 8200:8200
```

```bash
export VAULT_ADDR="https://172.0.0.1:8200"
```

Create a policy for cert-manager for Vault in a file called */tmp/cert-manager.hcl*

```hcl
path "pki_int/issue/default-svc-cluster-local" {
  capabilities = ["read", "list", "create", "update"]
}

path "pki_int/sign/default-svc-cluster-local" {
  capabilities = ["read", "list", "create", "update"]
}
```

Upload the policy to Vault

```bash
vault policy write -tls-skip-verify default-svc-cluster-local /tmp/cert-manager.hcl
```


```bash
vault token create -tls-skip-verify -period=8760h -policy=default-svc-cluster-local -explicit-max-ttl=8760h
```

Create a kubernetes secret containing the secret for cert-manager to access vault as *cert-manager-secret.yaml*

```bash
echo -n "$TOKEN" | base64
```


```yaml 
apiVersion: v1
kind: Secret
type: Opaque
metadata:
  name: cert-manager-vault-token
  namespace: default
data:
  token: "<base-64-encoded>"
```


Create the Issuer yaml for cert-manager as *vault-issuer.yaml*

```yaml
apiVersion: cert-manager.io/v1alpha3
kind: Issuer
metadata:
  name: vault-issuer
  namespace: default
spec:
  vault:
    path: pki_int/sign/default-svc-cluster-local
    server: https://vault.default.svc.cluster.local:8200
    caBundle: <base64 encoded caBundle PEM file>
    auth:
      tokenSecretRef:
        name: cert-manager-vault-token
        key: token
```


Upload the secret and issuer to kubernetes

```bash
kubectl create -f cert-manager-secret.yaml
kubectl create -f vault-issuer.yaml
```


Install Cert-Manager

```bash
kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v1.0.1/cert-manager.yaml
```
[https://cert-manager.io/docs/installation/kubernetes/ Cert-Manager Docs](https://cert-manager.io/docs/installation/kubernetes/)


Create vault cert issued by cert-manager

```bash
echo <<'EOF' >> vault-cert.yaml
---
apiVersion: cert-manager.io/v1alpha2
kind: Certificate
metadata:
  name: vault-pod-cert
spec:
  secretName: vault-pod-cert
  duration: 2160h # 90d
  renewBefore: 360h # 15d
  isCA: false
  keySize: 2048
  keyAlgorithm: rsa
  keyEncoding: pkcs1
  issuerRef:
    name: vault-issuer
    kind: Issuer
  usages:
    - server auth
    - client auth
  commonName: "*.default.svc.cluster.local"
  dnsNames:
    - vault.default
    - vault-0.default
    - vault-1.default
    - vault-2.default
    - vault.default.svc
    - vault-0.default.svc
    - vault-1.default.svc
    - vault-2.default.svc
    - vault.default.svc.cluster.local
    - vault-0.default.svc.cluster.local
    - vault-1.default.svc.cluster.local
    - vault-2.default.svc.cluster.local
    - localhost
  ipAddresses:
    - 127.0.0.1
EOF
kubectl create -f vault-cert.yaml
```

Check the certificate wss issued

```bash
kubectl get certificates | grep vault
```

Update the helm Vault config with the following to use the new TLS settings:
 
```yaml
global:
  enabled: true
  tlsDisable: false
server:
  ingress:
    enabled: false
  extraEnvironmentVars:
    VAULT_CACERT: "/certificates/ca.crt"
  extraSecretEnvironmentVars:
    - envName: AWS_ACCESS_KEY_ID
      secretName: vault-eks-creds
      secretKey: AWS_ACCESS_KEY_ID
    - envName: AWS_SECRET_ACCESS_KEY
      secretName: vault-eks-creds
      secretKey: AWS_SECRET_ACCESS_KEY
  volumes:
    - name: vault-pod-cert
      secret:
        secretName: vault-pod-cert
  volumeMounts:
    - name: vault-default-cert
      mountPath: /certificates
  ha:
    enabled: true
    config: |
      ui = true
      max_lease_ttl = "8761h"
      listener "tcp" {
        address = "[::]:8200"
        cluster_address = "[::]:8201"
        tls_cert_file = "/certificates/tls.crt"
        tls_key_file  = "/certificates/tls.key"
        tls_client_ca_file = "/certificates/ca.crt"
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

Restart Vault to use the new settings

```bash
helm upgrade vault -f vault.yaml hashicorp/vault
kubectl delete pod vault-0 vault-1 vault-2
```


## Upgrade Consul to TLS


Create an intermediate PKI store and CSR for the consul cluster

```bash
vault secrets enable -path=pki_int_consul pki
vault secrets tune -max-lease-ttl=43800h pki_int_consul
```

```bash
vault write -tls-skip-verify -format=json pki_int_consul/intermediate/generate/exported \
    add_basic_constraints=true\
    common_name="cluster Talos Production Intermediate Authority" \
    format=pem \
    key_type=ec \
    key_bits=384 \
    >pki_intermediate
jq -r '.data.private_key' < pki_intermediate > intermediate.key.pem
jq -r '.data.csr' < pki_intermediate > pki_intermediate.csr
```

Copy the csr into a file called *pki_intermediate_consul.csr*

vault write -tls-skip-verify -format=json pki_root/root/sign-intermediate \
        csr=@pki_intermediate.csr \
        format=pem_bundle \
        ttl="43800h"
        | jq -r '.data.certificate' > intermediate.cert.pem
        
vault write -tls-skip-verify pki_int_consul/intermediate/set-signed certificate=@intermediate.cert.pem 


kubectl create secret generic consul-ca-key --from-file='tls.key=./intermediate.key.pem'
kubectl create secret generic consul-ca-cert --from-file='tls.crt=./intermediate.cert.pem'

helm upgrade consul -f consul.yaml hashicorp/consul

```yaml
global:
  tls:
    verify: true
    httpsOnly: true
```



Update the helm Vault config with the following to use Consul via TLS:
 
```yaml
global:
  enabled: true
  tlsDisable: false
server:
  ingress:
    enabled: false
  extraEnvironmentVars:
    VAULT_CACERT: "/certificates/ca.crt"
  extraSecretEnvironmentVars:
    - envName: AWS_ACCESS_KEY_ID
      secretName: vault-eks-creds
      secretKey: AWS_ACCESS_KEY_ID
    - envName: AWS_SECRET_ACCESS_KEY
      secretName: vault-eks-creds
      secretKey: AWS_SECRET_ACCESS_KEY
  volumes:
    - name: vault-pod-cert
      secret:
        secretName: vault-pod-cert
  volumeMounts:
    - name: vault-default-cert
      mountPath: /certificates
  ha:
    enabled: true
    config: |
      ui = true
      max_lease_ttl = "8761h"
      listener "tcp" {
        address = "[::]:8200"
        cluster_address = "[::]:8201"
        tls_cert_file = "/certificates/tls.crt"
        tls_key_file  = "/certificates/tls.key"
        tls_client_ca_file = "/certificates/ca.crt"
      }
      seal "awskms" {
        region     = "eu-west-2"
        kms_key_id = "KMS_KEY_ID"
      }
      storage "consul" {
        path = "vault/"
        address = "HOST_IP:8501"
        scheme = "https"
        token = "CONSUL_TOKEN"
        tls_skip_verify = "true"
      }
```

Restart Vault to use the new settings

```bash
helm upgrade vault -f vault.yaml hashicorp/vault
kubectl delete pod vault-0 vault-1 vault-2
```
