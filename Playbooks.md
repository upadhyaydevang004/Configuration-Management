[Virtual Machine](VM.md) | [Ansible](Ansible.md) | [Playbooks](Playbooks.md)

Although you can run ad-hoc commands in ansible, in practice, you'll largely be expected to create ansible playbooks. Ansible playbooks are essentially files formatted as [yaml](http://docs.ansible.com/ansible/YAMLSyntax.html).

### Hello Playbooks

Let's confirm our ansible/node0 VM setup still works.

```bash
ansible all -i inventory -m ping
```

Instead of running this on the command line, we can run this within an ansible playbook (see [ping.yml](ping.yml)). 

```yaml
---
- hosts: all
  gather_facts: no
  tasks:
  - name: ping all hosts
    ping:
```

You can run this playbook by running:

```bash
ansible-playbook ping.yml -i inventory
```

### Commands/Modules

The simpliest way to get started is to try executing some basic tasks inside of a playbook.
In examples folder, execute the [commands.yml](examples/commands.yml) playbook.

```bash
ansible-playbook commands.yml
```

This will ensure a .ssh directory exists and creates a ssh key. Inspect the directory and ensure it exists. Notice that this runs on your ansible server and not your nodes. That's because the hosts in the playbook is specified as localhost.

* Run the command again. You should see changes=0.
* Manually delete the ssk key that was generated. Run the command again.

### Variables/Loops

In examples folder, execute the [loops.yml](examples/loops.yml) playbook.
This will install git and print out a list of packages. Instead of running on your localhost, this will run on all servers in the `[nodes]` group. The `-s` option will allow the playbook to sudo as root if necessary.

```bash
ansible-playbook -i inventory loops.yml -s
```

Extend the example to run now install all packages defined in the packages variable.

### Output/Register/Conditions

When a module is executed, you can save the results of that command using `register`.
Further, you can use conditions to control whether a task is executed with `when`.
You can control whether a command is considered changed, using `changed_when`.

For example, this command will save the output of `echo` into the variable `out`.
Its changed status will change depending on whether there is a list item with z.

```yaml
- action: command echo {{item}}
  register: out
  changed_when: "'z' in out.stdout"
  with_items:
    - hello
    - foo
    - bye
```

Mixing `register`, `changed_when`, and `with_items` can get tricky. Based on context, sometimes the variable saved in register will change per item during task execution. Othertimes, when all done contain a list of all items and their results. For some more details:
[see this example](http://stackoverflow.com/a/41292571/547112).

Ok. Check out [results.yml](examples/results.yml). There is a task that downloads a file. But since it uses the `shell` command it doesn't know that the task has already been done. It isn't idempotent and it will wastefully download the file over and over again!

* Change the command so that there is a task that [first checks if the file already exists](http://docs.ansible.com/ansible/stat_module.html) in /tmp. Only download when it doesn't exist by adding a `when` condition. Note that the it is possible to pass this option to the shell command, however, it there are some limits to this approach: `creates={{dest_dir}}/{{exchange_item}}"`.

### Templates

Templates are powerful ways to setup basic configuration settings without hard coding values.
When you use a template, it will get the template, fill in any parameters, and then copy the file over to the destination. 

For example, a template file, called `.my.cnf.j2` containing the following:

```ini
[client]
user=root
password={{root_db_password}}
local_infile=1
```

Templates can be instantiated and copied to a server with the following task.

```yaml
- name: copy .my.cnf file with mysql root password credentials
  template: src=templates/root/.my.cnf dest={{ ansible_env.HOME}}/.my.cnf owner={{mysql_account}} mode=0600
```

This can be useful for setting up complex configuration files such as apache, mysql, or jenkins. 

**See if you can create a new task using a template**.

### Vaults

It is possible to store secrets encrypted using [ansible-vault](http://docs.ansible.com/ansible/playbooks_vault.html). This is recommended if you need to store tokens, passwords, or ssh keys.

### Content Layout/Roles

Finally, as your playbooks grow more complex in size, you will start to think about ways to organize and seperate out tasks.

Ansible describes some recommended way to organize your playbooks:
http://docs.ansible.com/ansible/playbooks_best_practices.html#content-organization

An important part of organization is [roles](http://docs.ansible.com/ansible/playbooks_roles.html). Roles allow for you to essentially "include" in other playbooks. However, they use a particular layout to organize content.

Folder layout:
```
- roles/
  - setup/
    - tasks/
      - main.yml
    - templates/
      .my.cnf
- playbook.yml
```

An example use of roles:

```yaml
---
- hosts: dbservers
  vars_prompt:
    - name: run_schema_import
      prompt: "Do import of data (schema/import)? Y: Will destory existing data. N: Will skip"
      default: "N"
  # This loads sudo password from encrypted vault
  vars_files:
    - vault/dbservers.vars
  vars: 
    - VersionBotDB: VersionsDB

  roles: 
    - { role: schema, when: run_schema_import == "Y" or run_schema_import == "y" }
    - indexes
    - import
    - users
```

### Everything else

The ansible community supports a wide collection of modules to do amazing things. It is possible to build custom modules or programmatically interface with ansible. Learning ansible will be tricky, but the investiment can be worth it!
