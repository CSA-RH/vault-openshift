# Vault and Openshift
Demo of Vault inside Openshift.

The goal is to use a git repository to install `Vault` by using `Helm` and then to configure it so we can create new secrets inside the UI and inject them into pods.

This demo is based on [this article](https://www.hashicorp.com/blog/vault-secrets-operator-for-kubernetes-now-ga?utm_source=YouTube&utm_medium=video&utm_campaign=24Q3_NewReleases&utm_content=VSOk8s&utm_offer=blog).

## Prerequisites
- helm 3
- oc
- Project *vault* exists
- Project *vso-example* exists

## Vault's installation
First of all, here is how you gather the latest version of `Vault`:

``` bash
helm repo add hashicorp https://helm.releases.hashicorp.com
helm search repo hashicorp/vault -l
# The next command will give you the latest version as a tarball - not commited
helm fetch hashicorp/vault
# The next command will give you the root vault repository - commited, in my case, it was verison 0-27-0
tar -xvf vault-0.27.0.tgz
# The only modification made is the addition of the resource "vault/templates/ui-route.yaml"
# /!\/!\/!\ Note that this is a very bad practice, the UI should not be exposed to the internet /!\/!\/!\ 
# The goal was to give an internet access to the demo platform so the participants could play around and test it. 
# In production, if you need to access to the ui, please use 
# the port-forwarding approach instead: 'oc port-forward pod/vault-0 8200:8200` and never expose the UI to the internet!
```

Note that there is no operator for Vault available for now so the recommended approach is to install it though `Helm`.

To install `Vault`, we will go for the easy way by using the cli. However, you can use whichever automation way to deploy it such as `ArgoCD` or automate it in any CD pipeline.

```bash
TODO: do this
oc new-project vault
# helm install [flags] [NAME] [CHART]
helm install -f vault/values.openshift.yaml vault ./vault 
```

The application will still stay in a non ready state (probe will fail) until you unseal the vault. Let's do this:

``` bash
# Exec to the vault pod
oc exec -it vault-0 -- /bin/sh
# Initialise the configuration of the vault, please refer to Vault documentation
# /!\ We decided to have 1 key and only 1 is mandatory to unseal the vault - not fine for Production /!\
vault operator init --tls-skip-verify -key-shares=1 -key-threshold=1
...
Unseal Key 1: <key-1>
Initial Root Token: <root-token>
# Please backup this root token because it will be useful in the next steps
# Export variables and unseal the vault ussing the key you get
export KEYS=<key-1>
export ROOT_TOKEN=<root-token>
export VAULT_TOKEN=${ROOT_TOKEN}
vault operator unseal -tls-skip-verify ${KEYS}
```

## Vault's configuration

To make things possible, we need to configure the link between `Kubernetes` and `Vault`.
All the commands must be run on the vault pod.
Run `oc rsh vault-0` to get a shell or you can still use the terminal tab inside the console.

``` bash
# First, we need to login to the Vault, you will be asked for a token, provide the root one
vault login
# Then, we need to enable the `Kubernetes` authentification method:
vault auth enable kubernetes

vault write auth/kubernetes/config \
    kubernetes_host=https://kubernetes.default.svc

# To read the configuration we just write, we can use
vault read auth/kubernetes/config
```

We now need to create a key/value store secret engine.
``` bash
# Enable the secret engine called kv-v2 (key value store 2nd generation)
# And we need to specify the path. This path will be the root of all your secrets
vault secrets enable -path=secretstorename kv-v2
# Let's populate our secret store with a few secrets
vault kv put secretstorename/foo key1=value1 secretpasswordfoo=12345
vault kv put secretstorename/bar key2=value2 secretpasswordbar=abcde
vault kv put secretstorename/database username=root password=P@SsW0rD

# You can read the data in many different ways
vault read secretstorename/data/foo
vault kv get -mount secretstorename foo
vault kv get secretstorename/foo
``` 

Then, we need to create a kind of "link" to allow a specific `ServiceAccount` from somewhere to access something. 
Here is a simple policy to grant access to any secret under the "secretstorename" path we just created and populated.

/!\ Note that this is for a demo purpose, please adapt the policies to your own strategies. A common strategy is to give only access to what a specific microservice and/or environement needs. Always try to apply the security principle of least privilege. /!\

``` bash
# Create the policy to read everything inside a store
cat <<EOT > /tmp/policy.hcl
path "secretstorename/*" {
  capabilities = ["read"]
}
EOT

# Create the policy
vault policy write demo /tmp/policy.hcl
# Now we need to say that this role is linked with a specific ServiceAccount inside a specific namespace
vault write auth/kubernetes/role/demo \
    bound_service_account_names=default \
    bound_service_account_namespaces=vso-example \
    policies=demo \
    ttl=1h

vault read auth/kubernetes/role/demo 

# Optional: for the demo purpose, let's create a user with the policy to see what we can do/can not do
# Enable the userpass auth method
vault auth enable userpass
# Create the user and link the policy to it
vault write auth/userpass/users/jeremy password="jeremy" policies="demo"
vault read auth/userpass/users/jeremy
```

At this point, let's go to the UI and see a bit the different things we created.

## Vault Secrets Operator's installation
The operator we will install is called [vault-secrets-operator](https://github.com/hashicorp/vault-secrets-operator).
This operator will allow us to synchronise `Kubernetes` secrets with `Vault`. 

You can install this operator though the UI, or use the `Subscription` file available in `vso/subscription.yaml`. Note that as this is a global namespace operator, you do not need to create a specific `Project` and a specific `OperatorGroup` as these are already available.

```bash
oc apply -f vso/subscription.yaml
```

## VSO and Vault synchronization

Now that `Vault` is configured, we need to create a few CRD to enable the synchronisation inside Kubernetes.

The CRD objects you need to create are:
1. A `VaultConnection` object which will be in charge of connecting to a specific `Vault` instance. This is a namespaced resource.
2. A `VaultAuth` object which will use a `VaultConnection` and specify the required credentials to log into `Vault`. In this case, we configured the 'kubernetes' auth method, mounted under the 'kubernetes' path. We created a policy called 'demo' and we want to link it with the 'default' service account of the vso-example namespace
3. Two `VaultStaticSecret` which will synchronise the content of Vault into two native `Secret` Kubernetes objects. The goal is to:
    - Maintain them up to date wherever you use the secret
    - Have a single source of truth for the secrets (inside Vault)
    - Use native Secret object so we do not have to migrate all our existing `Deployment` yaml file
The spec requires: a `VaultAuth` name, the 'mount' of the secret engine, the 'type' of the secret engine, the 'path' where the secret is located inside the secret engine, the 'refresh time' (basically how often do we need to check if the secret has been updated inside `Vault`) and, of course, the destination of the `Secret`.
4. Two nginx `Pod` which will mount each secret into a specific path. This is a pretty classic usage of secrets. Note that we could also have used environement variables, poiting directly to a specific key inside the `Secret`.

```bash
oc create -f demo/
```

## Demo 1 - secret synchronisation

1. We can list the secret and read them though the UI or the cli. You should find two secrets available for you, called `foo` and `bar`. The content should be the same as what we wrote inside Vault: `oc get secret {foo,bar} -o yaml`
2. You should find two pods as well, called `foo` and `bar`. If you exec on each of them you should find your secrets as files under the path `/etc/secrets`: `oc rsh foo`.
3. Let's jump to vault and edit the secret `foo` from `key1=value1` and  `secretpasswordfoo=12345` to `key1=value2`, `secretpassword=12345` and `newsecret=newvalue`. Basically we want to: modify a secret key, modify a secret value and insert a new one. Before validate the changes inside the Vault UI, run `watch oc get pod,secret`.
4. Note that the secret have been synced but the pod did not restart. This is the behaviour we expect. Delete and re create the pod, the new values are now injected to the pod.

## Demo 2 - secret injection

This demo is based on the [official documentation](https://developer.hashicorp.com/vault/tutorials/kubernetes/kubernetes-sidecar#kubernetes-sidecar).

1. You can deploy the Deployment `deployment-orgchart-inject-demo.yaml` after checking the annotations and the content inside Vault.
2. Once deployed, you can check the logs from the init-container and also exec to the pod to check the paths `/vault/secrets`
3. You can deploy the Deployment `deployment-orgchart-inject-demo-template.yaml` which does pretty much the same thing, but instead it allows you to templatise a bit your file. Note that in both scenarios, no `Secret` are created, the secret are "injected" inside the container. 
4. /!\ If the `Pods` run with a `ServiceAccount` different from the ones defined in the `Vault Kubernetes` authentication role, then they will *NOT* be able to access the secrets defined at that path. /!\ To demonstrate this, you can deploy the `orgchart-inject-demo-failing.yaml` resources and check the logs: `oc logs -f <pod-name> -c vault-agent-init`
5. /!\ If the `Pods` run in a `Namespace/Project` different from the ones defined in the `Vault Kubernetes` authentication role, then they will not be *NOT* be able to access the secrets defined at that path. /!\ To demonstrate this, you can try to deploy almost the same resource in another `Namespace/Project` without doing any modification on the Vault side. This will result in a failure. Trying to fix it might be a good exercise, same as restrict a few rights about the SA, etc.
6. Last but not least, again, modifying the secret inside `Vault` will not result in re deploying the `Pods`. If you want to achieve this goal, please consider to visit [this link regarding Reloader](https://github.com/stakater/Reloader) as a possible alternative solution (not supported). Note that you need to be aware of the consequences that such a configuration might do. My personnal suggestion would be to run a bunch of tests to be able to understand any scenarios in case of a failure (vault restart, vault unavailable, sync of the secrets failed, pods restart with wrong configuration, rollback the secret, etc.)

