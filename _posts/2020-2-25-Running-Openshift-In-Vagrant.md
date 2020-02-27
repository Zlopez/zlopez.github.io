---
layout: post
title: Running Openshift Origin on Silverblue
---

I spent last few weeks trying to get Openshift Origin running on Fedora Silverblue, so I said to myself, that I will share my experience with others. I was making it run for this project https://github.com/fedora-infra/mbbox, so everything could be find here right after https://github.com/fedora-infra/mbbox/pull/1 will be merged. I will add links to files where applicable. The links are currently only for the PR.

# Prerequisites

* *Vagrant* - Although vagrant is not part of the Silverblue ostree, I'm using it for various things everytime I need a light VM.

* *Ansible* - Ansible is used for vagrant VM provisioning and is also helpful for provisioning various containers.

To install vagrant and ansible on Silverblue, unfortunately you need to layer it:
`rpm-ostree install libvirt vagrant vagrant-sshfs ansible`

# Setting the environment
Vagrant environment is defined in [*Vagrantfile*](https://github.com/fedora-infra/mbbox/blob/08ee9ca747b455b894a576a40e4c042ac7bd08b8/Vagrantfile) and then provisioned by [ansible role](https://github.com/fedora-infra/mbbox/blob/08ee9ca747b455b894a576a40e4c042ac7bd08b8/ansible/roles/mbbox-dev/tasks/openshift.yml).

## Vagrantfile
There are few things in the Vagrantfile that I want to explain related to Openshift.

1. `config.vm.box = "fedora/30-cloud-base"`
Why Fedora 30 and not newer? The reason is simple, because cgroups v2 are incompatible with docker and the Openshift Origin is using docker for everything.

2. `config.vm.network "forwarded_port", guest: 8443, host: 8443`
This should allow you to access web console, but I didn't have any luck here. So I just used the `oc` command.

3. `domain.cpus = Etc.nprocessors`
This allows the VM to get as many processors as available on the host, because building of the containers is rather CPU heavy.

## Ansible role
I will go through every task in ansible provisioning file for OpenShift except the ones that are project specific, so everybody could understand, why the tasks is needed.

1.  Install openshift dependencies
This task will install the only dependency you need on Fedora and that is the `origin-clients`, it installs everything you need to have working Openshift instance in your vagrant.
```
 dnf:
    name: [
           origin-clients
          ]
    state: present
```

2. Add insecure-registries entry
This task will enable the use of local registry for Openshift, without it you are unable to use registry module in OpenShift and thus build from any local buildconfig.
```
  replace:
    path: /etc/containers/registries.conf
    after: 'registries.insecure'
    before: 'registries.block'
    regexp: '^(registries =).*$'
    replace: '\1 ["172.30.0.0/16"]'
```

3. Restart registries
After previous task you need to restart registries for changes to take effect.
```
 systemd:
    state: restarted
    name: registries
```

4. Add cgroups to docker systemd service
This one blocked me for some time, you need to change the docker cgroupdriver from systemd (default) to cgroupfs. I found this advice in https://bugzilla.redhat.com/show_bug.cgi?id=1558425.
```
replace:
    path: /usr/lib/systemd/system/docker.service
    regexp: '(native.cgroupdriver=)systemd'
    replace: '\1cgroupfs'
```

5. Start docker
You need to make sure docker service is actually running before doing anything with OpenShift itself.
```
systemd:
    state: restarted
    daemon_reload: yes
    name: docker
```

6. Start cluster
You can start cluster now, this will take few minutes before it starts and sometimes this failed on timeout in my case, but most of the time the cluster was successfuly started. I needed to enable only few modules, but others should work too.
```
command: oc cluster up --enable=[router, registry, web-console]
```

7. Add registry
Even if you enabled the registry in previous step you still need to add it manually. I didn't figure out why this is needed, but otherwise the registry doesn't work as it should.
```
command: oc cluster add registry
```

8. (Optional) Create project
In this step you could create any project you want, I worked on the setup script for mbox, so I used the mbox as project name.
```
command: oc new-project mbox
```

9. (Optional) Copy template
If you want to work with the database it is good to use template. The templates could be obtained from [openshift-ansible](https://github.com/openshift/openshift-ansible/tree/release-3.11/roles/openshift_examples/files/examples/x86_64/db-templates), but for the PostgreSQL I needed few changes to make it work, so the updated template could be found [here](https://github.com/fedora-infra/mbbox/blob/08ee9ca747b455b894a576a40e4c042ac7bd08b8/ansible/roles/mbbox-dev/files/postgresql-ephemeral-template.json). I also recommend to use ephemeral templates in vagrant, otherwise you need persistent volume, which is one thing I didn't figure out how to setup.
```
copy: src=postgresql-ephemeral-template.json dest=/tmp/postgresql-ephemeral-template.json
```

10. (Optional) Switch to admin user
This step is related to the database setup, because you need admin rights to install the templates in OpenShift.
```
command: oc login -u system:admin
```

11. (Optional) Install PostgreSQL templates
The installation of the template(s) is done using `oc create` command, this will install the template into `openshift` namespace and it's ready to be used in any project you create.
```
command: oc create -f "/tmp/postgresql-ephemeral-template.json" -n openshift
```

12. (Optional) Use centos registry for postgress imagestream
I didn't find any better PostgreSQL imagestream for OpenShift than the one in registry.centos.org. The default in the original template didn't worked, so I used this one. As you can see in command bellow, I'm just tagging the registry.centos.org as `potgresql:latest`.
```
command: oc tag --scheduled=true registry.centos.org/postgresql/postgresql postgresql:latest
```

13. (Optional) Grant router access to host network
This is just needed if you want to deploy router in your Openshift. This is adding a default HAProxy router. You need admin rights to do this, look at step 10.
```
command: oc adm policy add-scc-to-user hostnetwork -z router
```

14. (Optional) Deploy router
The second step for the router is deployment. I deployed the router with only port 8443 allowed, because I didn't need anything else in my setup.
```
command: oc adm router --ports='8443'
```

# Using the vagrant VM

* *Start vagrant VM*
To start vagrant environment simply use `vagrant up`, this will setup the VM and do the provisioning.

* *Log into vagrant VM*
To log inside our vagrant VM use `vagrant ssh`.

* *Restart vagrant VM*
If there is any issue that needs restart of the VM, you need to do `vagrant destroy` and then `vagrant up` (because of the Openshift setup you can't just use `vagrant provision`).

* *Working with Openshift inside vagrant VM*
To work with Openshift use `oc` command and you can do whatewer you want. Just remember because of the docker, you must use `sudo oc` or create an alias. If you want to do any admin work using `oc adm`, you need to login as system admin `oc login -u system:admin`.

I hope this will help to someone, who is struggling with OpenShift on Silverblue as I was.
