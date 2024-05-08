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
## ansible-galaxy install -r requirements.yml
```

```
TEMP
(cd ../ansible_collections/scale_computing/hypercore/; ansible-galaxy collection build -f;)
## ansible-galaxy collection install ../ansible_collections/scale_computing/hypercore/scale_computing-hypercore-1.5.0-dev.tar.gz
```

Decide on prefix to your VM names
```
# set vm_group variable
nano vars.yml

# adjust to be aligned with vm_group variable
nano inventory/k3s.yml
```

## Create VMs on HyperCore

```
ansible-playbook -e@vars.yml playbooks/hypercore_template_vm.yml
ansible-playbook -i inventory -e@vars.yml playbooks/nfs-server.yml
ansible-playbook -i inventory -e@vars.yml playbooks/hypercore-cluster.yml
```

## Setup K3s

```
# ansible-playbook -i inventory -e@vars.yml k3s.orchestration.site

git clone https://github.com/k3s-io/k3s-ansible
ansible-galaxy collection install k3s-ansible/
ansible-playbook -i inventory -e@vars.yml k3s-ansible/playbook/site.yml -e token=mytoken

ansible-playbook -i inventory -e@vars.yml playbooks/k3s-tools.yml
ansible-playbook -i inventory -e@vars.yml playbooks/k3s-nfs-utils.yml
ansible-playbook -i inventory -e@vars.yml playbooks/k3s-nfs-provisioner.yml
ansible-playbook -i inventory -e@vars.yml playbooks/k3s-dns.yml
ansible-playbook -i inventory -e@vars.yml playbooks/k3s-metallb.yml
```

Use this K3S cluster

```
ls $HOME/.kube
cp $HOME/.kube/config.new $HOME/.kube/config_k3s_scale
export KUBECONFIG=$HOME/.kube/config_k3s_scale
kubectl config use-context k3s-ansible
kubectl config get-contexts
```

### Add new K3s nodes

Add new K3s servers and/or agents to `vars.yml`, then apply needed changes.

```
kubectl get no
nano vars.yml

ansible-playbook -i inventory -e@vars.yml playbooks/hypercore-cluster.yml
ansible-playbook -i inventory -e@vars.yml playbooks/k3s-nfs-utils.yml
ansible-playbook -i inventory -e@vars.yml k3s-ansible/playbook/site.yml -e token=mytoken

k get no
```

## Use K3s

```
# k == kubectl
k apply -f kube/webapp.yml
# ask exdns-k8s-gateway for IP of the just-deployed webapp demo
k get svc -A -l app.kubernetes.io/instance=exdns  # 172.31.6.51 was returned
dig nfs.demo1.xyz.si @172.31.6.51
```

```
# to setup "split DNS" on local PC
sudo nano /etc/systemd/resolved.conf
  [Resolve]
  DNS=172.31.6.51
  Domains=~xyz.si

sudo systemctl daemon-reload
sudo systemctl restart systemd-resolved
dig nfs.demo1.xyz.si
```

Open http://nfs.demo1.xyz.si:8080/ URL.

```
# Add a test file directly on NFS server
root@b-k3s-nfs-server:~# date >> /nfsdata/nfs-retain/default-nfs-web-pvc/aa.txt
```

Try a more complex app, say https://github.com/GoogleCloudPlatform/microservices-demo

```
cd kube
git clone https://github.com/GoogleCloudPlatform/microservices-demo
cd microservices-demo

k create ns demo2
kubectl config set-context --current --namespace=demo2
kubectl apply -f ./release/kubernetes-manifests.yaml
kubectl get po
kubectl get service frontend-external
```

Visit http://EXTERNAL_IP of service frontend-external.
