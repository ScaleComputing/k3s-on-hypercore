# k3s-on-hypercore

Ansible playbooks and roles to deploy and configure multi-node K3S clusters on Scale Computing // HyperCore from scratch.

This can be used to:
- setup K3s on ScaleComputing HyperCore using Ansible playbook
- setup demo app on K3s

## Prepare

Steps:

```
python3 -m venv .venv  # python>=3.12 prefered
source .venv/bin/activate
pip install -r requirements.txt
ansible-galaxy install -r requirements.yml

# We need to clone k3s-ansible until their packaging get fixed.
# See https://github.com/k3s-io/k3s-ansible/issues/321
git clone https://github.com/k3s-io/k3s-ansible
ansible-galaxy collection install k3s-ansible/
```

Setup access to HyperCore cluster:

```
export SC_HOST=https://IP
export SC_USERNAME=admin_user
export SC_PASSWORD=admin_pass
```

Decide on prefix to your VM names, IP addresses etc.
```
# set vm_group variable
nano vars.yml

# adjust to be aligned with vm_group variable
nano inventory/k3s.yml
```

## Setup K3s - one command

Now a K3s cluster can be setup with a single command:

```
ansible-playbook -i inventory -e@vars.yml playbooks/all.yml -e token=mytoken
```

## Setup K3s - multiple command

Alternative to single command is to run individual steps one by one.
This allows debugging, and makes retrying a failed step easier.

### Create VMs on HyperCore

```
ansible-playbook -e@vars.yml playbooks/hypercore_template_vm.yml
ansible-playbook -i inventory -e@vars.yml playbooks/nfs-server.yml
ansible-playbook -i inventory -e@vars.yml playbooks/hypercore-cluster.yml
```

### Setup K3s

```
# ansible-playbook -i inventory -e@vars.yml k3s.orchestration.site
ansible-playbook -i inventory -e@vars.yml k3s-ansible/playbooks/site.yml -e token=mytoken

ansible-playbook -i inventory -e@vars.yml playbooks/k3s-tools.yml
ansible-playbook -i inventory -e@vars.yml playbooks/k3s-nfs-utils.yml
ansible-playbook -i inventory -e@vars.yml playbooks/k3s-nfs-provisioner.yml
ansible-playbook -i inventory -e@vars.yml playbooks/k3s-dns.yml
ansible-playbook -i inventory -e@vars.yml playbooks/k3s-metallb.yml
```

## Use this K3S cluster

Configure `kubectl` on your laptop/workstation to use this cluster.

```
ls $HOME/.kube
cp $HOME/.kube/config.new $HOME/.kube/config_k3s_scale
export KUBECONFIG=$HOME/.kube/config_k3s_scale
kubectl config use-context k3s-ansible
kubectl config get-contexts
```

## Add new K3s nodes

Add new K3s servers and/or agents to `vars.yml`, then apply needed changes.

```
kubectl get no
nano vars.yml

ansible-playbook -i inventory -e@vars.yml playbooks/hypercore-cluster.yml
ansible-playbook -i inventory -e@vars.yml playbooks/k3s-nfs-utils.yml
ansible-playbook -i inventory -e@vars.yml k3s-ansible/playbooks/site.yml -e token=mytoken

kubectl get no
```

## Upgrade K3s

Read release notes (https://docs.k3s.io/release-notes/v1.29.X).

In `vars.yml` change `k3s_version` to more recent version, then run:

```
ansible-playbook -i inventory -e@vars.yml k3s-ansible/playbooks/upgrade.yml -e token=mytoken
```

## Use K3s

### Single pod webapp with external volume

```
kubectl apply -f kube/webapp.yml
# ask exdns-k8s-gateway for IP of the just-deployed webapp demo
kubectl get svc -n default -l run=web-nfs  # EXTERNAL-IP=172.31.6.52 was returned
```

Visit http://EXTERNAL_IP:8080/ URL.
The pod is showing its filesystem.

The web-nfs service also has a DNS name assigned.
We can ask the exdns about this.

```
kubectl get svc -A -l app.kubernetes.io/instance=exdns  # 172.31.6.51 was returned
dig nfs.demo1.example.com @172.31.6.51
```

If 172.31.6.51 would be configured as DNS resolver, then we could open
http://nfs.demo1.xyz.si:8080/ URL.

```
# Add a test file directly on NFS server
root@b-k3s-nfs-server:~# date >> /nfsdata/nfs-retain/default-nfs-web-pvc/aa.txt
```

### Multi pod app

Try a more complex app, say https://github.com/GoogleCloudPlatform/microservices-demo

```
cd kube
git clone https://github.com/GoogleCloudPlatform/microservices-demo
cd microservices-demo

kubectl create ns demo2
kubectl config set-context --current --namespace=demo2
kubectl apply -f ./release/kubernetes-manifests.yaml
kubectl get po
kubectl get service frontend-external
```

Visit http://EXTERNAL_IP of service frontend-external.
