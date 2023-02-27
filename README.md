# Deploy Vault with Helm on Kubernetes
Instructions for the impatient on how to get a vault server up and running on Kubernetes.

> **Warning**
> In this guide the PVC lifecycle is managed by helm so if you undeploy the vault chart your data is GONE!


# Installation

## SSL

Follow SSL guide until you get to the "Deploy the vault cluster via helm with overrides" stage EXCEPT for "Create the Certificate Signing Request (CSR)" - you must update and use the command below to include the DNS names of any loadbalancers you wish to access the vault through: 
https://developer.hashicorp.com/vault/tutorials/kubernetes/kubernetes-minikube-tls#create-the-certificate

```
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
DNS.4 = vault.lan.asio
DNS.5 = dsys.gotdns.com
IP.1 = 127.0.0.1
EOF
```


## Helm

```
helm repo add hashicorp https://helm.releases.hashicorp.com
kubectl create namespace vault
helm upgrade --install -n vault --values values.yaml vault hashicorp/vault
```

You will see a message like this:

```
Release "vault" has been upgraded. Happy Helming!
NAME: vault
LAST DEPLOYED: Mon Feb 27 00:33:19 2023
NAMESPACE: vault
STATUS: deployed
REVISION: 2
NOTES:
Thank you for installing HashiCorp Vault!

Now that you have deployed Vault, you should look over the docs on using
Vault with Kubernetes available here:

https://www.vaultproject.io/docs/


Your release is named vault. To learn more about the release, try:

  $ helm status vault
  $ helm get manifest vault
```

And you will see the vault pod has failed to come up:

```
$ kubectl get pods -n vault
NAME                                    READY   STATUS    RESTARTS   AGE
vault-agent-injector-66b75d668f-tmxnq   1/1     Running   0          6m52s
vault-0                                 0/1     Running   0          6m52s
```

This is because you must first initialize and unseal the vault

# Configuration
## Initialize the vault

> **Warning**
> Secret vaules are only shown once!

* Take note of the `Useal Key` infos - you need these to unseal (turn on) the vault
* Same goes for `Intial Root Token` - this is used for `vault` root level cli access
* You only see these messages once

```
kubectl exec -n vault --stdin=true --tty=true vault-0 -- vault operator init
Unseal Key 1: ny0ksBIXPjRixxh+ljvUdtDeBE/N/GMSS4v7scGuIywY
Unseal Key 2: fI+e9xRjdMT9A97zs3YRbEVRyIgujdnt9KzOl+miVni+
Unseal Key 3: FxEKemn0zqmZDCJx3vyre7DuSIJ4eJSPieYedrAHwwBh
Unseal Key 4: 46zpAvzQOffa6gnVu4UK9Bd5DqQOK7rJmJKjsM9v1K4J
Unseal Key 5: +iSMug2s3L0dd9mnqCqGw+WQNK/4NgydEBzNycnK4nzT

Initial Root Token: hvs.vqMQfdsPNfyBU9CVQWZnZBQV

Vault initialized with 5 key shares and a key threshold of 3. Please securely
distribute the key shares printed above. When the Vault is re-sealed,
restarted, or stopped, you must supply at least 3 of these keys to unseal it
before it can start servicing requests.

Vault does not store the generated root key. Without at least 3 keys to
reconstruct the root key, Vault will remain permanently sealed!

It is possible to generate new unseal keys, provided you have a quorum of
existing unseal keys shares. See "vault operator rekey" for more information.
```

## Unseal the vault

* Repeat with each of the unseal keys until seal threshold is met and vault pod comes up. 3 is the default.

```
kubectl exec -n vault --stdin=true --tty=true vault-0 -- vault operator unseal ny0ksBIXPjRixxh+ljvUdtDeBE/N/GMSS4v7scGuIywY
kubectl exec -n vault --stdin=true --tty=true vault-0 -- vault operator unseal fI+e9xRjdMT9A97zs3YRbEVRyIgujdnt9KzOl+miVni+
kubectl exec -n vault --stdin=true --tty=true vault-0 -- vault operator unseal FxEKemn0zqmZDCJx3vyre7DuSIJ4eJSPieYedrAHwwBh
```

## Create and read a secret

Into the vault pod first:

```
kubectl exec -n vault -ti vault-0 -- sh
```

Now use the token to setup the `secret` path and read + write a secret

```
export VAULT_TOKEN=hvs.vqMQfdsPNfyBU9CVQWZnZBQV

vault secrets enable -path=secret kv-v2
vault kv put -mount=secret hello foo=world
vault kv get -mount secret hello
```

## Vault policy - readonly access to /secret

```
vault policy write ro-secret-policy - << EOF
path "secret/data/*" {
  capabilities = ["read"]
}
EOF
```

## Create token - using the policy

> **Warning**
> Token is shown only once!

```
vault token create -policy=ro-secret-policy -display-name="RO: /secret"
token                hvs.CAESIKyA_nC9oAOE8e6Ndnq091enk81bNPIu_U4VSOAhzdHOGh4KHGh2cy55ZWhXWG94Vm5mM2xPV1ZyTTZaV0ViUE4
token_accessor       QcllvL065DNqkSeUrJ1HE0y9
```



# Enable external access

```
kubectl apply -f vault-lb.yaml
```

* take note of the load balancer IP address: `kubectl -n vault get service`

# Client setup

Grab a copy of `$WORKDIR/vault.ca` and copy to `~/.vault/vault.ca`

Then run:

```
export VAULT_CACERT=~/.vault/vault.ca
export VAULT_TOKEN=hvs.CAESIKyA_nC9oAOE8e6Ndnq091enk81bNPIu_U4VSOAhzdHOGh4KHGh2cy55ZWhXWG94Vm5mM2xPV1ZyTTZaV0ViUE4

# try to read the secret created earlier
vault kv get -mount secret -address https://vault.lan.asio:8200 hello
```

# Troubleshooting

## Server up?

```
vault status
```

## List all secret paths

```
vault secrets list -detailed
```

## Restart the vault

Normally you have to do:

```
kubectl -n vault rollout restart statefulset vault
```
But this doesn't work for me - probably because only one replica...


Instead go nuclear:

```
kubectl delete pod -n vault vault-0
```

The vault needs to be unsealed following instructions above to come back up when killed this way

## vault pod not starting

Check the logs may need unsealing

