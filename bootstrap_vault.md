# Bootstrap Vault

ssh on to your vault raspberry pi.

```bash
ssh pi@vault.local
```

Hashicorp currently do not offer an Apt repository for Arm based images, as such you need to manually download and install using the following steps.

```bash
wget https://releases.hashicorp.com/vault/1.5.3/vault_1.5.3_linux_arm.zip
unzip vault_1.5.3_linux_arm.zip
sudo chown root:root vault
sudo mv vault /usr/local/bin/
vault -autocomplete-install
complete -C /usr/local/bin/vault vault
sudo setcap cap_ipc_lock=+ep /usr/local/bin/vault
```

Create a non privileged vault user to run vault

```bash
sudo useradd --system --home /etc/vault.d --shell /bin/false vault
```


Configure systemd - to start the vault server

```bash
sudo su -
cat <<'EOF' >> /etc/systemd/system/vault.service
[Unit]
Description="HashiCorp Vault - A tool for managing secrets"
Documentation=https://www.vaultproject.io/docs/
Requires=network-online.target
After=network-online.target
ConditionFileNotEmpty=/etc/vault.d/vault.hcl
StartLimitIntervalSec=60
StartLimitBurst=3

[Service]
User=vault
Group=vault
ProtectSystem=full
ProtectHome=read-only
PrivateTmp=yes
PrivateDevices=yes
SecureBits=keep-caps
AmbientCapabilities=CAP_IPC_LOCK
Capabilities=CAP_IPC_LOCK+ep
CapabilityBoundingSet=CAP_SYSLOG CAP_IPC_LOCK
NoNewPrivileges=yes
ExecStart=/usr/local/bin/vault server -config=/etc/vault.d/vault.hcl
ExecReload=/bin/kill --signal HUP $MAINPID
KillMode=process
KillSignal=SIGINT
Restart=on-failure
RestartSec=5
TimeoutStopSec=30
StartLimitInterval=60
StartLimitIntervalSec=60
StartLimitBurst=3
LimitNOFILE=65536
LimitMEMLOCK=infinity

[Install]
WantedBy=multi-user.target
EOF
```

Configure Vault

```bash
sudo su -
mkdir /etc/vault.d/ /certificates /vault
touch /etc/vault.d/vault.hcl
chown --recursive vault:vault /etc/vault.d /certificates /vault
chmod 640 /etc/vault.d/vault.hcl

cat <<'EOF' >> /etc/vault.d/vault.hcl
listener "tcp" {
  address       = "127.0.0.1:8200"
  tls_disable = "true"
}
storage "file" {
  path = "/vault/data"
}
ui = true
EOF

sudo systemctl enable vault
sudo systemctl start vault
```

Check the status of vault by running

```bash
sudo systemctl status vault
```


Initialise Vault

Make sure you copy the keys and tokens somewhere secure

```bash
export VAULT_ADDR="http://127.0.0.1:8200"
vault operator init
```


Unseal Vault

The following command needs running three times, each time you need to give it a different unseal key from the initial setup.
Once completed the vault will be unsealed and ready for use.

```bash
vault operator unseal
```


## Creating the Root CA

Token is taken from initial vault setup (root token)

```bash
export VAULT_TOKEN=xxx
vault secrets enable -path=pki_root pki
vault secrets tune -max-lease-ttl=87600h pki_root

vault write -field=certificate pki_root/root/generate/internal common_name="example.com" ttl=87600h > ca.crt

vault write pki_root/config/urls \
        issuing_certificates="http://127.0.0.1:8200/v1/pki_root/ca" \
        crl_distribution_points="http://127.0.0.1:8200/v1/pki_root/crl"
```

Create a role to allow issuing of certificates based on the Root CA (for the vault server) and then create a signed server cert.

```bash
vault write pki_root/roles/local \
        allowed_domains="local" \
        allow_subdomains=true \
        max_ttl="87600h"
vault write pki_root/issue/local common_name="vault.local" ttl="87599h
```

Copy both the certificate and ca into a file called tls.crt (ensuring the ca is second in the file)
Copy the key into a file called tls.key
```bash
chmod 400 tls.*
sudo mv tls* /certificates/
sudo chown vault:vault /certificates/*
rm tls*
```


If you wish to access the vault ui on your local network securly you can now edit the config and enable TLS
```bash
sudo su -
cat <<'EOF' > /etc/vault.d/vault.hcl
listener "tcp" {
  address       = "0.0.0.0:8200"
  tls_disable = "false"
  tls_cert_file = "/certificates/tls.crt"
  tls_key_file  = "/certificates/tls.key"
  tls_min_version = "tls13"
}
storage "file" {
  path = "/vault/data"
}
ui = true
EOF
systemctl stop vault
systemctl start vault
```


To copy the CA back to your machine you can use the following command:

```bash
scp pi@vault.local:~/ca.crt .
```


For security, you should now seal the vault and ideally shutdown the Raspberry PI (storing it in a safe) until you need to issue intermediate CA's
```bash
sudo /sbin/shutdown -HP --no-wall 0
```


