
# Ansible REST API
a crossplane composition for ansible

a step-by-step guide is available at [tutorial.md](tutorial/readme.md)

## local setup

```bash
kind create cluster --image kindest/node:v1.29.0

helm install crossplane \
crossplane-stable/crossplane \
--version 1.14.4 \
--create-namespace \
--namespace crossplane-system \
--set configuration.packages='{xpkg.upbound.io/jcw/ansible-api:v0.2.0}'
```

### gitlab setup
create a private gitlab repo and a corresponding token with `read_repository` permission and store it in a kubernetes secret:
```bash
kubectl -n crossplane-system create secret generic gitlab-token --from-literal=url=https://foo:glpat-rNTFau47Hhzteqoxvptb@gitlab.com/janwillies/ansible-automation-private.git
```
a `~/.gitconfig` is missing in the `provider-ansible` container image, use sth like this to fix
```bash
git config --global credential.helper 'store --file /tmp/ansibleDir/8ba94f8d-78a2-4075-ac13-56cb9b61e0c6/.git-credentials'
```
The directory is only available after the first run, so you might want to come back here later.
ToDo: This seems to be missing: https://github.com/crossplane-contrib/provider-ansible/blob/5dba0f13852a098f31dfbfb86b1775e274a0cfd7/internal/controller/ansibleRun/ansibleRun.go#L232

### vm test
Build and load a vm image with ssh daemon into the cluster, so that we have something we can configure:
```bash
docker build -t ubuntu-23-10:ssh-2 vm/
kind load docker-image docker.io/library/ubuntu-23-10:ssh-2
```
create vm
```bash
kubectl apply -f vm/
```
test
```bash
kubectl -n crossplane-system create secret generic ssh-key --from-file=key=./ssh.key
```
### first test
```bash
kubectl apply -f examples/providerconfig.yaml
kubectl apply -f examples/xrc.yaml
```
this creates a couple of resource:
- namespaced `Ansible`: example-run
- cluster scoped `XAnsible`: example-run-f00bar
- cluster scoped `AnsibleRun`: example-run

### simulate client
Now that we have everything in place, we can create user accounts and JWTs

create `user-1` and corresponding oidc jwt
```bash
kubectl apply -f user/
TOKEN=$(kubectl create token user-1 --namespace user-1)
```
cluster address
```bash
ADDRESS=$(kubectl config view -o jsonpath='{.clusters[?(@.name == "kind-kind")].cluster.server}')
```

finally apply the request as `user-1`:
```bash
curl -k -X POST -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' -d @examples/xrc.json $ADDRESS/apis/crossplane.accenture.com/v1alpha1/namespaces/user-1/ansibles
```

debug with admin permissions:
```bash
kubectl proxy
curl -s -X POST -H 'Content-Type: application/json' -d @examples/xrc.json http://localhost:8001/apis/crossplane.accenture.com/v1alpha1/namespaces/user-1/ansibles
```

## notes
- [design.md](https://github.com/crossplane-contrib/provider-ansible/blob/main/docs/design.md)
- `ProviderConfigUsage` was not cleaned up: https://github.com/crossplane-contrib/provider-ansible/issues/260
- TODO: propagate `status` 