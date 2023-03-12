---
layout: post
title: Ansible dynamic inventory for Apache Guacamole
author: Kenneth KOFFI
categories: [ansible, "apache guacamole"]
tags: [ansible, "apache guacamole", inventory]
image: https://images.unsplash.com/photo-1518112390430-f4ab02e9c2c8?ixlib=rb-4.0.3&ixid=MnwxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8&auto=format&fit=crop&w=881&q=80
---

**Ansible and Guacamole, a match made in heaven.**

If you already use [Ansible](https://www.ansible.com/) and [Apache Guacamole](https://guacamole.apache.org/) on a daily basis, what would you say about combining them? Using Guacamole as a source for your Ansible inventory? I mean, fetching your hosts' connection details from Guacamole and passing them to Ansible as inventory.

> Wow, really ? That would be insane ! How do you do that ? Can you please show me ?

Sure buddy, that's the whole point of this article. But first, let me explain (give a small context) to people in the back, what's going on.

![](https://cdn-images-1.medium.com/max/800/1*vVrc3cAGg336lVpCzZIjDg.png)

## Context

[Inventory](https://docs.ansible.com/ansible/latest/inventory_guide/intro_inventory.html?extIdCarryOver=true&sc_cid=701f2000001OH7EAAW) is one of the fundamental pieces of Ansible. It's a list of machines you want to run your Ansible playbooks against. An inventory can be static or dynamic. While used by most Ansible users, static inventory isn't entirely practical in certain situations. For example, when you use Guacamole to connect to the hosts in your infrastructure, you may have already encountered the following issues:

* Manually keeping your hosts connection details stored in Guacamole synced with your static inventory file is tedious. Admit it; you have better things to do than edit YAML files, right?
* Having your hosts (servers) passwords stored in plain text isn't secure.
* Your host inventory is extensive.

Ansible dynamic inventory has been introduced in order to solve these problems. Ansible can pull inventory information directly from dynamic sources like Azure, docker, AWS, Guacamole, etc. To achieve this, a **dynamic inventory plugin** specific to each source is used. This article is an outline of how to use the **guacamole_inventory** plugin.

Now that the stage is set, let's eat !!

## Requirements

To complete this tutorial, you will need the following things:

* Ansible installed on your laptop/server
* A running Apache Guacamole instance

## Installation

To tie your Ansible inventory to Guacamole, you have to install the **guacamole_inventory** plugin through one of the installation methods below.

### Installation from Collection

The **guacamole_inventory** plugin is part of the [scicore.guacamole](https://galaxy.ansible.com/scicore/guacamole) collection on Ansible galaxy. That collection contains all the stuffs needed to communicate with your Guacamole instance via Ansible. It's also the easiest way to obtain the plugin. Just open your terminal and run:

```bash
ansible-galaxy collection install scicore.guacamole
```

To check if the plugin has been successfully installed, execute:

```bash
ansible-doc -t inventory scicore.guacamole.guacamole_inventory
```

You should have an output similar to this :
![](https://cdn-images-1.medium.com/max/800/1*JxuFeilBrU9Ox2KM08ricg.png)

### Installation as a standalone plugin (out of collection)

Download the `guacamole_inventory.py` file from [this repository](https://github.com/theko2fi/ansible-guacamole-dynamic-inventory), to the `DEFAULT_INVENTORY_PLUGIN_PATH` path on your machine.

> What is `DEFAULT_INVENTORY_PLUGIN_PATH` ?

It's a colon separated paths list in which Ansible will search for Inventory Plugins by default.

Execute `ansible-config dump | grep DEFAULT_INVENTORY_PLUGIN_PATH` to view your current configuration settings for Inventory Plugins. After the plugin file is added to one of these locations, Ansible loads it and you can use it in any local module, task, playbook, or role.

The execution of the above command on my machine, give me this:
![](https://cdn-images-1.medium.com/max/800/1*SkqVv5h6UT_e0ITXvTyI8w.png)

My Ansible installation is then expecting to have inventory plugins in one of these locations:

* `/home/kenneth/.ansible/plugins/inventory`
* `/usr/share/ansible/plugins/inventory`

I decided to install it to the `/home/kenneth/.ansible/plugins/inventory` directory.

Since it's my first plugin installation, this location doesn't exist. I will create it manually and download the plugin there:

```bash
mkdir -p ~/.ansible/plugins/inventory
cd ~/.ansible/plugins/inventory/
wget https://raw.githubusercontent.com/theko2fi/ansible-guacamole-dynamic-inventory/main/guacamole_inventory.py
```

To check if the plugin has been successfully installed, execute:

```bash
ansible-doc -t inventory guacamole_inventory
```

You should have an output similar to this :
![](https://cdn-images-1.medium.com/max/800/1*Sn32wEa1KQj1Z9WqdzeEIg.png)

## Usage

Since I'm the author of this plugin (yes I'm bragging about it haha), I'm going to paste parts of the documentation here. Below, you have a list of all the options supported by the plugin with their default values, a small description, and the type of data expected.

```yaml
    plugin:
        description: Token that ensures this is a source file for the 'guacamole_inventory' plugin.
        type: string
        required: true
        choices: [ guacamole_inventory ]
    base_url:
        description:
          - URL of the Apache Guacamole instance.
          - It is recommended to use HTTPS so that the username/password are not
            transferred over the network unencrypted.
        required: true
        type: string
    auth_username:
        description: the username to authenticate against the Apache Guacamole API
        type: string
        default: guacadmin
        env:
          - name: GUACAMOLE_USER
    auth_password:
        description: the password to authenticate against the Apache Guacamole API
        type: string
        default: guacadmin
        env:
          - name: GUACAMOLE_PASSWORD
    selected_connection_groups:
        description:
          - A list of connection group names to search for connections.
          - 'ROOT' will include all connections from Guacamole instance.
        type: list
        elements: str
        default: ["ROOT"]
    validate_certs:
        description:
            - Validate ssl certs?
        default: true
        type: bool
```

The mandatory options are:

* **plugin**: the value should always be either `scicore.guacamole.guacuacamole_inventory`or `guacamole_inventory`, depending on the installation method you have chosen.
* **base_url**: the URL of your guacamole instance API
* **auth_username**: your admin username to authenticate to the API
* **auth_password**: your admin password to authenticate to the API

I will cover later in this article, the usage of the other options with some examples. But first, let's see the most basic usage of this plugin.

Create a folder for all the examples in this article:

```bash
mkdir medium-article
cd medium-artcile
```

Now let's create our dynamic inventory file, which will use the plugin. The name of the file **SHOULD end** with either **guacamole.yml** or **guacamole.yml**. This is required for the plugin to detect and read the inventory file. I'll call mine, `myhosts_guacamole.yml`.

```bash
touch myhosts_guacamole.yml
```

Paste the content below into the `myhosts_guacamole.yml` file.

```yaml
plugin: guacamole_inventory
base_url: http://localhost:8080/guacamole
auth_username: guacadmin
auth_password: guacadmin
```

> Note: **If you have chosen the collection installation method, you should use** `scicore.guacamole.guacamole_inventory` **as the plugin value like this :**

```yaml
plugin: scicore.guacamole.guacamole_inventory
base_url: http://localhost:8080/guacamole
auth_username: guacadmin
auth_password: guacadmin
```

Now, let's test it:

```bash
ansible-inventory -i myhosts_guacamole.yml --list
```

This should return a JSON output of all your SSH connections details stored in your Apache Guacamole instance. Something like this:
![](https://cdn-images-1.medium.com/max/800/1*5DIounZg8HRnafe0sFnYcA.png)

From Guacamole web UI, you can see that I have two hosts: vm1 et vm2.
![](https://cdn-images-1.medium.com/max/800/1*3TPMZA-v29kKbAC9A52Mrw.png)

If you would rather get a well formatted YAML output in Ansible way, try this:

```bash
ansible-inventory -i myhosts_guacamole.yml --list --yaml
```

And that would return a similar output:
![](https://cdn-images-1.medium.com/max/800/1*m5kxYi-bzU5XesbnrZYdbg.png)

You could also keep the output minimal as a graph containing just the groups and hosts with:

```bash
ansible-inventory -i myhosts_guacamole.yml --graph
```

![](https://cdn-images-1.medium.com/max/800/1*eL92CU2GXVZmMJ9grWQ1SA.png)

Both VMs belong to the `ungrouped` group, which is a child of the `all`group in the Ansible inventory.

This means the plugin is working. Ansible is now able to target your hosts based on the connection details stored in Guacamole.

You can also test the connectivity to these hosts with the ping module:

```bash
ansible all -i myhosts_guacamole.yml -m ansible.builtin.ping
```

That's the quickest way to get started with this plugin. Now let's dive into all the good ways to customize our inventory with a few examples.

> **Note**: For the hosts using ssh private key as authentication method, the **`ansible_ssh_private_key_file`** variable points by default to **`~/.ssh/id_rsa`** file. We will see in the next section, how we could override this.

## Examples

Now let's explore the other available options of this plugin with some examples.

### 1. Compose

_Add variables to each host found by this inventory plugin, whose values are the result of the associated expression._

**Scenario:** Let's say we want to add a particular variable and value to all the hosts listed by this plugin. We should use the **`compose`** option. Update the content of the `myhosts_guacamole.yml` file like this:

```yaml
plugin: guacamole_inventory
base_url: http://localhost:8080/guacamole
auth_username: guacadmin
auth_password: guacadmin
compose:
    we_come_from_guacamole: "'yes'"
```

We execute again:

```bash
ansible-inventory -i myhosts_guacamole.yml --list --yaml
```

**Result:**
![](https://cdn-images-1.medium.com/max/800/1*Vn711Y_CYx4rKv-WL05mug.png)

This option could also be used to override some existing variables used by Ansible to connect to the hosts. For example, if I would rather use a different ssh key file than the default one, the inventory file will look like this:

```yaml
plugin: guacamole_inventory
base_url: http://localhost:8080/guacamole
auth_username: guacadmin
auth_password: guacadmin
compose:
    ansible_ssh_private_key_file: "/path/to/my/guacamole_key"
```

All the hosts with _private-key_ defined and listed by this plugin, will use the `guacamole_key`.

### 2. Groups

_Places a host in the named group if the associated condition evaluates to true._

**Scenario:** Let's assume we have the following connections in Guacamole.
![](https://cdn-images-1.medium.com/max/800/1*mdoGuI7yTlv-TvkYzjMC0Q.png)

And we want to group together in our inventory, all the connections using the name pattern `DatabaseServer-*`.

```yaml
plugin: guacamole_inventory
base_url: http://localhost:8080/guacamole
auth_username: guacadmin
auth_password: guacadmin
groups:
    Guacamole_Databases: "'DatabaseServer' in name"
```

**Result:**
![](https://cdn-images-1.medium.com/max/800/1*PPqwGwY3AFP0KKfVzc2wuA.png)

All the connections containing the name `DatabaseServer`, are placed inside the group `Guacamole_Databases`.

### 3. Keyed_groups

_Places hosts in dynamically-created groups based on a variable value._

**Scenario:** We want to group together, hosts listening on the same ssh port number.

Sample of `myhosts_guacamole.yml` file:

```yaml
plugin: guacamole_inventory
base_url: http://localhost:8080/guacamole
auth_username: guacadmin
auth_password: guacadmin
keyed_groups:
- prefix: group-based-on-ssh-port
  key: ansible_port
```

**Result:**
![](https://cdn-images-1.medium.com/max/800/1*ydXkK7EsRxEtLyEcj5RXwQ.png)

### 4. Selected_connection_groups

_Fetches connections from an explicit list of connection groups instead of default all (- 'ROOT')_

**Scenario:** We have the connections groups shown below in Guacamole. And we want to target the hosts in `VMware-Servers` group only.
![](https://cdn-images-1.medium.com/max/800/1*J_4MLhFY26NNX2_FHRionQ.png)

The content of the `myhosts_guacamole.yml` file could look like this:

```yaml
plugin: guacamole_inventory
base_url: http://localhost:8080/guacamole
auth_username: guacadmin
auth_password: guacadmin
selected_connection_groups:
    - VMware-Servers
```

**Result:**
![](https://cdn-images-1.medium.com/max/800/1*Kq4s0G-r7Iw7o7AcmQeiLQ.png)

## Conclusion

While primarily used as remote desktop gateway to connect to computers/servers from anywhere, Guacamole can also serve as 'vault' to keep servers login credentials. In this article, we have seen how it can be used an inventory source for Ansible, in order to get the best out of each tool. Thanks for reading to the end. Feedbacks are welcome, here in the comments section or on the project GitHub repository:

[theko2fi/ansible-guacamole-dynamic-inventory: Ansible dynamic inventory plugin for Apache Guacamole (github.com)](https://github.com/theko2fi/ansible-guacamole-dynamic-inventory)

