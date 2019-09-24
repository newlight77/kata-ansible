# kata-ansible

This kata will guide you to learn Ansible, with code snippets and demonstration.

## Requirements

- One control node : this would be a VM (Vagrant)
- One target node : this would be a VM (Vagrant)
- Ansible
- Git
- [Bootstrap Ansible for CentOS](https://github.com/newlight77/centos) : run that playbook on the target node. You should have a ssh access without password (SSH public key exchanged), and that acount should be able to sudo as root without password.

## Prepare

Create the 2 VMs for control and target nodes.

1. get this repo

```sh
git clone https://github.com/newlight77/kata-ansible.git

cd kata-ansible
```

2. Create the VMs

```sh
vagrant up --provision target
vagrant up --provision control
vagrant reload --provision target
```

3. Verify that control node can access to target node without password prompt

```sh
vagrant ssh control
```

Then,

```sh
ansible -m ping target
ansible -m command -a whoami -b target
```

4. Now let's setup the the bootstrap playbook for the following kata

```sh
cd /vagrant
git clone https://github.com/newlight77/centos
cd centos

mkdir /vagrant/centos/inventories/kata

echo > /vagrant/centos/inventories/kata/hosts << EOF
[web]
target ansible_host=192.168.111.2 ansible_user=vagrant
EOF
```

5. Finally check again that control node can access to target node without password prompt

```sh

ansible -m ping target
ansible -m command -a whoami -b target
```

***Note*** : Here we can ping the target because it's defined in the inventory.

## Workshop

- [Introduction to Ansible](#Introduction)
- [Working variables](#variables)
- Exercise : inventories, group_vars, host_vars
- [Working with roles](#roles)
- Exercise :
- [Tools](#tools)
- Exercise : Find top 5 accounts sorted by balance

Duration : 3 hours
