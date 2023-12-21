
# Ansible REST API
implemented with crossplane provider-ansible

## setup

```bash
kind create cluster --image kindest/node:v1.29.0

helm install crossplane \
--version 1.14.4 \
--create-namespace \
--namespace crossplane-system \
crossplane-stable/crossplane
```

### install `crossplane-cli`
[https://docs.crossplane.io/latest/cli/](https://docs.crossplane.io/latest/cli/)
```bash
curl -sL "https://raw.githubusercontent.com/crossplane/crossplane/master/install.sh" | sh
```

### install `provider-ansible`
[https://marketplace.upbound.io/providers/crossplane-contrib/provider-ansible](https://marketplace.upbound.io/providers/crossplane-contrib/provider-ansible)
```bash
crossplane xpkg install provider xpkg.upbound.io/jcw/provider-ansible:v0.5.0-jcw2-ansible-8.7.0
```


## First test: local debug
```bash
kubectl apply -f 1-demo.yaml
```
should result in `hello world` printed in the ansible pod:
```bash
kubectl logs -n crossplane-system -l pkg.crossplane.io/provider=provider-ansible
```
```bash
TASK [Gathering Facts] *********************************************************
ok: [localhost]

TASK [ansibleplaybook-simple] **************************************************
ok: [localhost] => {
    "msg": "Hello world!"
}

PLAY RECAP *********************************************************************
localhost                  : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```



## remote vm with ssh
create a container with ssh access, so that we can simulate a remote vm to configure

create ssh key
```bash
ssh-keygen -t ed25519 -C "your_email@example.com" -f ssh.key -N ""
```
create vm
```bash
kubectl apply -f vm/vm-1.yaml
```
test if ssh works locally
```bash
kubectl -n vm port-forward service/vm-1 2222:22
ssh -p 2222 ansible@localhost -i ./ssh.key
```
add the ssh key to `provider-ansible` and apply the playbook
```bash
kubectl -n crossplane-system create secret generic ssh-key --from-file=key=./ssh.key
kubectl apply -f 2-inventory.yaml
```

```bash
│ PLAY [all] *********************************************************************
│ TASK [Gathering Facts] *********************************************************
│ ok: [vm-1.default]                                                              
│                                                                                 
│ TASK [Harden the system] *******************************************************
│ changed: [vm-1.default]                                                         
│                                                                                 
│ PLAY RECAP *********************************************************************
│ vm-1.default               : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```

a file should have been created in the vm at `/home/ansible/hardened`

## inventory & roles on public gitlab repo
create a public gitlab repo which hosts the ansible role, e.g. https://gitlab.com/janwillies/ansible-automation
```bash
kubectl apply -f 3-gitlab.yaml
```

debug:
```bash
cd /ansibleDir/ea000..
ansible-galaxy role install --roles-path . git+https://gitlab.com/janwillies/ansible-automation.git
ansible-galaxy role install --roles-path . --role-file requirements.yml
```

## add private gitlab credentials
create a private gitlab repo and a corresponding token with `read_repository` permission
```bash
kubectl -n crossplane-system create secret generic gitlab-token --from-literal=url=https://foo:glpat-rNTFau47Hhzteqoxvptb@gitlab.com/janwillies/ansible-automation-private.git
kubectl apply -f 4-gitlab-private.yaml
```

a `~/.gitconfig` is missing in the container image, use sth like this to fix
```bash
git config --global credential.helper 'store --file /tmp/ansibleDir/8ba94f8d-78a2-4075-ac13-56cb9b61e0c6/.git-credentials'
```
This seems to be missing: https://github.com/crossplane-contrib/provider-ansible/blob/5dba0f13852a098f31dfbfb86b1775e274a0cfd7/internal/controller/ansibleRun/ansibleRun.go#L232


## ansible collection on gitlab
It's possible to have multiple collections in a single git repo. They can be executed individually as well.
```bash
kubectl apply -f 5-gitlab-collection.yaml
```

## client api via composition

### install p&t function
The function is only needed if you start with compositions.
```bash
crossplane xpkg install function xpkg.upbound.io/crossplane-contrib/function-patch-and-transform:v0.2.1 function-patch-and-transform
```

create the client api
```bash
kubectl apply -f crossplane/apis/defintion.yaml
kubectl apply -f crossplane/apis/composition.yaml
```

test
```bash
kubectl apply -f examples/xrc.yaml
```
an `AnsibleRun` MR should have been created

### simulate client
Now that we have everything in place, we can create some user accounts

create `user-1` and corresponding oidc jwt
```bash
kubectl apply -f user/
TOKEN=$(kubectl -n user-1 create token user-1)
```

finally apply the request as `user-1`:
```bash
curl -k -X POST -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' -d @examples/xrc.json https://127.0.0.1:64746/apis/crossplane.accenture.com/v1alpha1/namespaces/default/ansibles
```

debug:
```bash
kubectl proxy
curl -s -X POST -H 'Content-Type: application/json' -d @examples/xrc.json http://localhost:8001/apis/crossplane.accenture.com/v1alpha1/namespaces/default/ansibles
```

## notes
- [design.md](https://github.com/crossplane-contrib/provider-ansible/blob/main/docs/design.md)
- `ProviderConfigUsage` was not cleaned up: https://github.com/crossplane-contrib/provider-ansible/issues/260
- TODO: propagate `status` 

### build composition
```bash
crossplane xpkg build --package-root=crossplane/ --ignore=apis/composition.yaml --examples-root=./examples
up xpkg push -f crossplane/configuration-ansible-3276fe8cc29b.xpkg jcw/ansible-api:v0.2.0

```