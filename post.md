# Managing CoreOS with Ansible

This post will cover basic techniques for managing [CoreOS][coreos] machines using [Ansible][ansible]. Familiarity with Ansible and basic understanding of CoreOS are helpful in following along with this post.

## What is Ansible?

From the [Ansible Documetation](http://docs.ansible.com/)

>
Ansible is an IT automation tool. It can configure systems, deploy software, and orchestrate more advanced IT tasks such as continuous deployments or zero downtime rolling updates.

At the most basic level, Ansible is tool that will run sets of commands (typically over SSH) on remote boxes (called the **inventory**). These commands can be as simple as one line shell statements, or use any of the built-in ansible [**modules**](http://docs.ansible.com/modules_by_category.html) for common tasks like file copying, package management, system information, and more. Ansible ships with many useful modules and you can easily create your own.

## Why Ansible for CoreOS?

Ansible does not require a remote agent running on the target machine. It can perform all of its functions over a basic SSH connection.

CoreOS is a minimal linux distribution meant for running containers. It does not ship with a package manager or many of the other common system elements one might expect coming from other more desktop oriented distributions.

Because Andible does not require a remote agent, and because CoreOS design favors running containers over system software directly on the machine, the two make a good fit.

## Getting Started

Before continuing, make sure that you have [Ansible installed](http://docs.ansible.com/intro_installation.html). If ansible is properly installed, you should be able to run the following command in a shell.

```shell
$ ansible --version
ansible 1.8
```

We have prepared a sample repository with a [Vagrant][vagrant] file to demostrate using Ansible with CoreOS locally. This is also a good way to test your **playbooks** (sets of Ansible commands) to make sure they are working as you expect. You will need to [install vagrant](https://docs.vagrantup.com/v2/installation/) to use these samples.

If you are already running Ansible and are familiar with launching CoreOS machines in existing cloud providers, you can ignore these steps and jump to the [next section](#first_run)

```
$ git clone https://github.com/defunctzombie/coreos-ansible-example.git
$ cd coreos-ansible-example
$ vagrant up
-- wait for vagrant to finish booting the machine(s) --
$ ./bin/generate_ssh_config
```

This will boot a CoreOS machine and configure it with some basic networking. After the machine has booted, we will run a local script *generate\_ssh\_config* to create a configuration file for Ansible so that it knows how to access our machine over SSH.

## Inventory Setup [](#first_run)

The inventory file defines the hosts and groups. Our vagrant example has an inventory file in `inventory/vagrant` which we will use when configuring our example CoreOS hosts with ansible.

```ini
## inventory file for vagrant machines
core-01 ansible_ssh_host=172.12.8.101

[web]
core-01
```

We only have 1 host called `core-01` and we have created a `web` group and listed our host under this group. You can have any number of hosts and groups. Ansible even support [dynamic inventory files](http://docs.ansible.com/intro_dynamic_inventory.html) which is what you are likely to use in large scale production environments.

To run a test ping against our vagrant inventory, execute the following command in your shell (from the project folder).

```shell
$ ansible -i inventory/vagrant all -m setup
```

If everything worked as expected you should see the following output.

```
core-01 | FAILED >> {
    "failed": true,
    "msg": "/bin/sh: /usr/bin/python: No such file or directory\r\n",
    "parsed": false
}
```

The command has failed. To understand why this happened and how to fix it, lets take a closer look at how Ansible runs commands on remote machines.

## Getting Ansible Running

When you run Ansible command or playbooks, Ansible will ssh into the remote machine, copy over the module code (python) and run the module using the arguments specified in the playbook.

The target machine must have a python interpreter for Ansible to be able to execute these modules and thus configure your machine.

CoreOS is designed for running containers and does not ship with a python intepreter. Additionaly, it has no package manager to install python. This presents a small chicken-and-egg problem.

Luckily Ansible has a `raw` execution mode which bypasses python modules and runs shell commands directly on a remote system. We will leverage this feature to bootstrap a lightweight python interpreter onto our CoreOS hosts. Once the hosts are bootstrapped with python, playbooks can leverage the myriad of provided Ansible modules to perform system tasks like start services, install python libraries, and manage docker containers.

Edit the `inventory/vagrant` file and add the following items at the end

```
[coreos]
core-01

[coreos:vars]
ansible_python_interpreter="PATH=/home/core/bin:$PATH python"
```

This will configure Ansible to look for `python` and `pip` in `/home/core/bin` and use it for all hosts in the `coreos` group. Without this, Ansible will try to use `/usr/bin/python` which does not exists on our CoreOS hosts.

To bootstrap our CoreOS hosts, we will use the [coreos-bootstrap][ansible-coreos-bootstrap] role.

Install the role using the following command
```
$ ansible-galaxy install defunctzombie.coreos-bootstrap -p ./roles
```

Now we can run the provided `bootstrap.yml` file using ansible.
```
$ ansible-playbook -i inventory/vagrant bootstrap.yml
```

Once this command has completed, we can run our original ansible setup command and see a list of the gathered facts.

```shell
$ ansible -i inventory/vagrant all -m setup
core-01 | success >> {
    "ansible_facts": {
        "ansible_all_ipv4_addresses": [
            "172.17.42.1",
            "10.0.2.15",
            "172.12.8.101"
        ],
    ...
```

Take a moment to look at `bootstrap.yml` and `site.yml`. Notice that `bootstrap.yml` is included first. Your own Ansible scripts will need to similaly have a `bootstrap.yml` or playbook which configures CoreOS hosts before running other playbooks.

## Example Playbook

Once Ansible can run successfully on your CoreOS hosts, we can do things like start system services or launch docker containers.

The `website.yml` file shows an small example. It starts the `etcd` service. Then it installs the `docker-py` library using `pip` and finally uses the Ansible [docker module](http://docs.ansible.com/docker_module.html) to launch a container on the `web` host group.

```yml
- name: example nginx website
  hosts: web
  sudo: true
  tasks:
    - name: Start etcd
      service: name=etcd.service state=started

    - name: Install docker-py
      pip: name=docker-py

    - name: pull container
      raw: docker pull nginx:1.7.1

    - name: launch nginx container
      docker:
        image="nginx:1.7.1"
        name="example-nginx"
        ports="8080:80"
        state=running
```

Run this playbook with the following command

```shell
$ ansible-playbook -i inventory/vagrant website.yml
```

You can now open [http://172.12.8.101:8080](http://172.12.8.101:8080) and see the default nginx landing page.

You are now ready to create more plays to configure your CoreOS hosts. Plays can be leveraged for many tasks like automated app deployment, custer management, and more.

## Tips

### Local Docker Registry

Downloading imagines from remote registries can be time and bandwidth consuming.  You can speed up deployments to a set of machines by running a local registry for a cluster and having other machines pull from the registry. This can be easily automated with roles and playbooks.

### Install pip modules in a common playbook

If you will be using the docker module, consider installing docker-py via the pip module in a playbook that runs on all hosts and comes after the bootstrap.yml playbook.

### Prefer containers over local binaries

Avoid over-configuring the CoreOS hosts with too many locally installed tools and tweaks. In many cases, containerized services will do the job just as well and can be granted limited access to the underlying host.

You can even monitor host processes by bind mounting the relevant paths into a container.

<!-- links -->
[vagrant]: https://www.vagrantup.com/
[coreos]: https://coreos.com/
[ansible]: http://www.ansible.com/home
[ansible-coreos-bootstrap]: https://github.com/defunctzombie/ansible-coreos-bootstrap
[toolbox]: https://github.com/coreos/toolbox
