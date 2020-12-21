---
layout: post
title: Running Minikube in Vagrant
tags: fedora vagrant minikube
---

I wrote before about [Running OpenShift Origin on Silverblue](https://zlopez.github.io/Running-Openshift-In-Vagrant/) and it started to be an issue lately because of the Fedora 30. So I decided to dedicate some time to switch it to Fedora 33 with [minikube](https://minikube.sigs.k8s.io/docs/). We choose minikube for your project, because it allows you to run light weight [Kubernetes](https://kubernetes.io/) cluster relatively easy and because [minishift](https://www.okd.io/minishift/) (Light weight OpenShift cluster) doesn't support [OpenShift 4](https://www.openshift.com/) yet. The ansible role that does the provisioning of the Vagrant machine could be found [here](https://github.com/fedora-infra/mbbox/tree/master/ci/roles/openshift-osdk).

## Prerequisites

* *Vagrant* - [Vagrant](https://www.vagrantup.com/) is a virtual machine provider, which allows you to create light weight virtual machines quickly.

* *Ansible* - [Ansible](https://www.ansible.com/) is used for vagrant VM provisioning and is also helpful for provisioning various containers.

## Setting the environment
Vagrant environment is defined in [*Vagrantfile*](https://github.com/fedora-infra/mbbox/blob/master/Vagrantfile) and then provisioned by [ansible role](https://github.com/fedora-infra/mbbox/tree/master/ci/roles/openshift-osdk).

### Vagrantfile
There is one thing related to Minikube in the Vagrantfile that I want to explain.

1. `domain.cpus = Etc.nprocessors`

   This allows the VM to get as many processors as available on the host, because building of the containers is rather CPU heavy.

### Ansible role
I will go through every task related to Minikube in ansible provisioning file so everybody could understand, why the tasks is needed.

1. Configure Cgroups v1

   Because we are using docker driver for Minikube (podman driver is still marked as experimental) we need to change the Cgroups to v1, because v2 is not supported by docker. This was the main reason we stayed on Fedora 30 for so long, but the change is really easy.
   
   ```
   - name: Configure Cgroups v1 for docker in grub.cfg
     replace:
       path: /etc/default/grub
       regexp: '^(GRUB_CMDLINE_LINUX="[^"]+)'
       replace: '\1 systemd.unified_cgroup_hierarchy=0'
   ```
   
   This will modify the GRUB to use Cgroups v1. The next thing that needs to be done is to regenerate the GRUB configuration.
   
   ```
   - name: Generate the new grub configuration with cgroups v1
     command: grub2-mkconfig -o /boot/grub2/grub.cfg
   ```

   For this change to apply, we need to reboot the machine. This is really easy to do in ansible.
   
   ```
   - name: Reboot the machine
     reboot:
   ```
   
   Because of the reboot the vagrant machine needs to be reloaded after provisioning otherwise it doesn't have any additional synced folders. The reload is done by `vagrant reload`.
   
2. Install Minikube dependencies

   There are few dependencies that needs to be installed for Minikube. Docker is needed for Minikube to run inside it and kubernetes-client package provides `kubectl` command which allows you to interact with Minikube cluster. In the project we created this for we have plenty of other dependencies, but the Docker and kubernetes-client should be enough.
   
   ```
   - name: Install Operator SDK system dependencies
     dnf:
       name: [
               ansible,
               ansible-lint,
               docker,
               kubernetes,
               kubernetes-client,
               python-flake8,
               python-molecule,
               python-molecule-docker,
               python-openshift
             ]
       state: present 
   ```
3. Add user to docker group
   
   The next thing that needs to be done is to add user to docker group, this will allow you to use minikube as vagrant user without need to use `sudo` for everything.
   
   ```
   - name: Create docker group
     group:
       name: docker
       state: present
     become: yes
   ```
   
   This will ensure that the docker group really exists. If not, it's created.
   
   
   {% raw %}
   ```
   - name: Ensure user is added to docker group
     user:
       name: "{{ dev_username }}"
       groups: ['docker']
       append: yes
     become: yes
   ```
   {% endraw %}

   This step will add user to docker group. In our case `dev_username` variable is set to `vagrant`, which is the default user for Vagrant created virtual machines. To allow the vagrant user to benefit from this group ssh connection needs to be reset.
   
   ```
   - name: Reset ssh connection to allow user changes to affect 'current login user'
     meta: reset_connection
   ```
   
   And now the user is able to use docker without the need to use `sudo`.
   
4. Minikube preparations

   There are a few things we need to prepare before downloading and installing Minikube.
   
   ```
   - name: Start docker
     systemd:
       state: restarted
       daemon_reload: yes
       name: docker
   ```

   Docker needs to be running before Minikube could be started.

   ```
   - name: Set SELinux to permissive
     selinux:
       policy: targeted
       state: permissive
   ```
   
   I really don't like if I need to disable SELinux, but in this case it is much easier than trying to add all the SELinux rules for Minikube.
   
5. Installation of Minikube

   Unfortunately Minikube isn't packaged in Fedora and it needs to be downloaded from official site, but at least it's distributed as RPM package.
   
   {% raw %}
   ```
   - name: Download minikube
     get_url:
       url: https://storage.googleapis.com/minikube/releases/latest/minikube-{{minikube_version}}.x86_64.rpm
       dest: /tmp/
   ```
   {% endraw %}

   This task will download the specific minikube version (in our case the `minikube_version` is set in [defaults](https://github.com/fedora-infra/mbbox/blob/master/ci/roles/openshift-osdk/defaults/main.yml) to `1.15.1-0`).
   
   {% raw %}
   ```
   - name: Install minikube
     dnf:
       name: /tmp/minikube-{{minikube_version}}.x86_64.rpm
       state: present
       disable_gpg_check: true
   ```
   {% endraw %}
   
   Installation is done using the dnf ansible module with disabled GPG check (RPM isn't signed by Fedora, so it's understandable that the signature is not recognized).
   
   {% raw %}
   ```
   - name: Set minikube default driver to docker
     command: minikube config set vm-driver docker
     become: yes
     become_user: "{{ dev_username }}"
   ```
   {% endraw %}
   
   The last thing that needs to be done before you can run the Minikube is to set the driver to docker, this is done as the specifically as a `vagrant` user. 

## Using the vagrant VM

* *Start vagrant VM*

  To start vagrant environment simply use `vagrant up && vagrant reload`, this will setup the VM and do the provisioning. The reload is needed to remount any synced folders after the restart of the machine when changing the Cgroups.

* *Log into vagrant VM*

  To log inside your vagrant VM use `vagrant ssh`.

* *Working with Minikube inside vagrant VM*

  To start the Minikube cluster inside vagrant VM just type `minikube start`. To interact with Minikube use `kubectl` command. If you are familiar with Kubernetes you wouldn't have any problems to work with it. To get direct access to docker container running minikube just use `docker ps` and connect to it using `docker -it <container> bash`.

I hope this blog post could help someone trying to run their own minikube cluster in Vagrant VM.
