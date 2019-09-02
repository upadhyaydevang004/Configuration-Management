Configuration Management Workshop
----------------------------------

In this workshop, we'll cover the basics of setting up a simple configuration server, enabling ssh access, and using ansible to configure a simple web server.

![image](img/ansible-setup.png)


## Workshop

### Before you get started

* [opunit and node.js should be installed](https://github.com/CSC-DevOps/profile#opunit), and you pass the course profile check:  
   `opunit profile CSC-DevOps/profile:519.yml`
* You can create a local VM with VirtualBox/Baker. If you can only get vagrant to work, then you will need to perform [some extra steps](VM.md).
* Clone this repo with: `git clone https://github.com/CSC-DevOps/CM` 
* Make sure you have a shell open in the right directory: `cd CM`.

### Checking progress on workshop

To check the configuration of the configuration and web server, we will use `opunit` to run checks on the virtual machines listed in the inventory file. We can run checks from the top-level directory as follows: 

```bash
$ opunit verify -i opunit_inventory.yml
```

If you're using vagrant VMs, you'll have to use `-i vagrant_inventory.yml`, instead.

## Creating your servers 

### The configuration server

Let's create a configuration server. This server will be using a "push-based model", where we will be sending configuration commands to other external servers. It also needs to be configured with ansible.

Create the Virtual Machine.

```bash
$ cd servers/ansible-srv
$ cat baker.yml
$ baker bake
```

You should see baker create the virtual machine.

```
✔ Running apt-get update on VM
✔ Preparing ansible
✔ Installing ansible
```

Verify that ansible was installed by running opunit `cd ../..`, then `opunit verify -i opunit_inventory.yml`.

### The web server

Let's create another virtual machine for the web server. 

```bash
$ cd servers/web-srv
$ cat baker.yml
$ baker bake
```

You should see baker create the virtual machine. Verify `baker ssh` works, then `exit` back to your host machine.

## Creating a connection between your servers

You need a way to automatically connect to your server without having to manually authenicate each connection. We will create a pair of public/private keys for [authentication through ssh](https://www.ssh.com/ssh/public-key-authentication#sec-Asymmetric-Cryptography-Algorithms).

From your host machine, create a new public/private key pair, running the following command, and hitting enter for the default prompts:

    ssh-keygen -t rsa -b 4096 -C "web-srv" -f web-srv

![ssh connection](img/connection.png)

After generating the keys, you need to copy them onto the servers. The private key (*secret*, tell know one 🤐), needs to be sent over to the configuration server. The public key (🌐), needs to be sent over to the web-server.

#### Setting up the private key

One nice trick is to use a copy utility to copy a file into your copy/paste buffer:

* Mac: `pbcopy < web-srv`
* Windows: `clip < web-srv`
* Linux: [Check out one of these options](https://superuser.com/a/288333/862331).

Let's go to the ansible-srv (`cd servers/ansible-srv`, then `baker ssh`).

Using a file editor, paste and store your private key in a file:

```bash
ansible-srv $ vim ~/.ssh/web-srv
# Make sure key is not readable by others.
ansible-srv $ chmod 600 ~/.ssh/web-srv
# We're done here, go back to host
ansible-srv $ exit
```

#### Setting up the public key

Copy the web-srv.pub file from your host.

Go inside the web-srv.

Using a file editor, add the public key to the list of authorized keys:

```bash
web-srv $ vim ~/.ssh/authorized_keys`
web-srv $ exit
```

Take care not to delete other entries, which may affect your ability to use vagrant/Baker to ssh into the machine. You also do not need to change the permissions of the file.

#### Testing your connection/Errors

Inside the ansible-srv, test your connection between the servers:

    ssh -i ~/.ssh/web-srv vagrant@192.168.33.100

Note: You should also be able to make this same connection from your host machine, since you also have private key, locally.

If you see an error or prompt for a password, you have a problem with your key setup. Double check have pasted in the content in correctly (where you in insert mode in vim before pasting?). 

If you see this warning, you need to remember to perform the `chmod 600` in order to fix the permissions of the private key file:

```
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@         WARNING: UNPROTECTED PRIVATE KEY FILE!          @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
Permissions 0664 for '/home/vagrant/.ssh/web-srv' are too open.
```

## Performing commands over ssh

Once you have established a ssh connection between two servers, you have achieved an important milestone in configuration management.

You can now perform ad-hoc commands and even execute scripts on machines without having to manually log-in.

For example, when you run this command.

    ssh -i ~/.ssh/web-srv vagrant@192.168.33.100 ls /

You can see the directory of the web-srv.

Practice: From your ansible-srv, create a file by executing `touch ssh_test.txt` on the remote server. Verify it exists.

By this point, you should be able to pass the ssh setup checks in opunit: `opunit verify -i opunit_inventory.yml`.

## Ansible

While being able to run ad-hoc commands and scripts provides a useful capability, there are several constraints that make this impractical. 

Writing bash scripts can be error-prone. Most commands are not idempotent, meaning they may cause errors or unexpected behaviors if run multiple times on the same servers. Finally, configuration of servers is an inherently noisy problem, due to network issues and random service and hardware failures. This means, you often need to resume a configuration operation after experiencing partial failure.

Ansible is a tool for performing configuration changes on multiple machines. Ansible uses a push-based model for configuration management, performing idempotent commands over ssh, without requiring any agent running. The implementation is rather straightforward: _Ansible commands are translated into python snippets, and then copied over to the target machine, and executed. This requires that python is installed on the target machine_.

## Inventory

An inventory file allows ansible to define, group, and coordinate configuration management of multiple machines. At the most basic level, it basically lists the names of an asset and details about how to connect to it.

Inside the ansible-srv, edit the `inventory` file to include the ip address, user, and path to the private key:

```ini    
[web]
192.168.33.100 ansible_ssh_user=vagrant ansible_ssh_private_key_file=~/.ssh/web-srv 
```

Now, run the ping test to verify ansible is able to talk to the web-srv!

    ansible all -m ping -i inventory

Note: this will fail because python is not available on the target machine. However, python3 is! We can adjust the connection to account for this. Inside the inventory, we can add a new variable for all entries in our `web` group:

```ini
[web:vars]
ansible_python_interpreter=/usr/bin/python3
```

Now, we should see a successful connection!

```
192.168.33.100 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

## Ansible commands

We can test our ability to use ansible for a simple configuration management task.

#### Installing nginx

Let's install a web server, called nginx (say like engine-X), on the web-srv VM. The web server will automatically start as a service.

    ansible all -b -m apt -i inventory -a 'pkg=nginx state=installed update_cache=true'

Open a browser and enter in your node's ip address, e.g. http://192.168.33.100

#### Completing workshop checks

You should be able to verify all checks pass:

    opunit verify -i opunit_inventory.yml

Great work!

#### Removing nginx.

    ansible all -s -m apt -i inventory -a 'pkg=nginx state=absent update_cache=true'

Since `nginx` is a metapackage, you may also need to run this to stop and remove the services:

    ansible all -s -m shell -i inventory -a 'sudo apt-get -y autoremove'
    
Webserver should be dead.

## Extra fun

Can you provision a remote VM (from digitalocean/etc.) and then add its details to the inventory? Can you run your nginx command on it too?

## Next steps

Run ansible commands can be useful for exploration and debugging. However, we want to be able to organize these commands into reusable configuration scripts.

Next workshop, we will learn about creating and running ansible playbooks.
