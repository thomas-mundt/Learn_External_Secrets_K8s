# Learn External Secrets in K8s

## Setup Vault 

```
k create ns vault
k -n vault get all
helm repo add hashicorp https://helm.releases.hashicorp.com
helm search repo hashicorp/vault --versions
helm install vault hashicorp/vault -n vault --version 0.23.0

helm install vault -n vault --set='ui.enabled=true' \
--set='ui.serviceType=NodePort' \
--set='server.dataStorage.enabled=false' hashicorp/vault --version 0.23.0

edit vault svc vault-internal to NodePort
3 3
copy keys and unseal

Token: hvs.yVXZ9vxtEbGHM7ihEOReGYI3
eLB1R5bv/qi4rpaxo75GYLU4hlopJnkQ8b+mJRW2jKWJ
MqdWrcyPOFScxEBZTjK6tqwV+jZQOfX7q/68OG1zFNJT
SsBeRbJD9n8esK2d0GdCWCDBvcW7o+dxpPjc+lAM1C0F
```



enable engine simple key value

```
helm repo add external-secrets https://charts.external-secrets.io

helm install external-secrets \
  external-secrets/external-secrets \
  -n external-secrets \
  --create-namespace \
  --set installCRDs=true

```

```
k -n external-secrets get po
k get crd | grep externalsecrets
```


https://github.com/saiyam1814/external-secrets-operatos



vi SecretStore.yaml
```
apiVersion: external-secrets.io/v1alpha1
kind: SecretStore
metadata:
  name: vault-backend
spec:
  provider:
    vault:
      server: "http://172.16.0.107:30696/" # your vault address
      path: "kv"
      # Version is the Vault KV secret engine version.
      # This can be either "v1" or "v2", defaults to "v2"
      version: "v2"
      auth:
        # points to a secret that contains a vault token
        # https://www.vaultproject.io/docs/auth/token
        tokenSecretRef:
          name: "vault-token"
          key: "token"
---
apiVersion: v1
kind: Secret
metadata:
  name: vault-token
data:
  token: aHZzLnlWWFo5dnh0RWJHSE03aWhFT1JlR1lJMwo= # "your vault root token" but encoded: echo "" | base64
```
k apply -f ddd

k get secretstore
vault-backend   32s   Valid    ReadWrite      True

Create the external secret

vi ExternalSecret.yml
```
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: vault-example
spec:
  refreshInterval: "15s"
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: example-sync
  data:
  - secretKey: foobar
    remoteRef:
      key: kv/demo #make sure this matches with what you create in the vault UI
      property: name #this is the secret you create under the path
```
k apply -f ExternalSecret.yml

Push secret so secret store



vi PushSecret.yaml
```
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
  namespace: default
stringData:
  source-key: "{\"pushed\":\"saiyam\"}"
---
apiVersion: external-secrets.io/v1alpha1
kind: PushSecret
metadata:
  name: pushsecret-example # Customisable
  namespace: default # Same of the SecretStores
spec:
  refreshInterval: 10s 
  secretStoreRefs: 
    - name: vault-backend
      kind: SecretStore
  selector:
    secret:
      name: my-secret 
  data:
    - match:
        secretKey: source-key
        remoteRef:
          remoteKey: pushed
```
k apply -f PushSecret.yaml
