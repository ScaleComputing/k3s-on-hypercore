# k3s-on-hypercore

Ansible playbooks and roles to deploy and configure multi-node K3S clusters on Scale Computing // HyperCore from scratch.

This can be used to:
- setup K3s on ScaleComputing HyperCore using Ansible playbook
- setup demo app on K3s

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

Create VMs on HyperCore

```
ansible-playbook -e@vars.yml playbooks/hypercore_template_vm.yml
ansible-playbook -i inventory -e@vars.yml playbooks/nfs-server.yml
ansible-playbook -i inventory -e@vars.yml playbooks/k3s-cluster.yml
```
