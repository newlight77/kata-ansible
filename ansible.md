# Ansible

## Introduction

### What is Ansible ?

- Configuration Management and Orchestration tool.
- Repeatably put systems into a desired state
  - files
  - packages
  - services

### Why use Ansible ?

- Easy to install
  - control machine needs ssh and python
  - target machine needs ssh (and preferably python)
  - no agents other than ssh required
- No complicated access control
  - if you can ssh to a target, you can run ansible against it with your privileges
  - you only get root power if you already have root access
- Easy to do easy things with
- Not much harder to do hard things with
- Usually easy for others to understand what is going on

- Getting started with Ansible

  - Install Ansible
  - RHEL/CentOS/Fedora: yum install ansible
  - OS X: brew install ansible
  - Windows: doesn't install directly — use a VM or cygwin
  - Most platforms: pip install ansible

- Set up target hosts
Ansible is most easily used if you can ssh as a normal user to a host without a password, and then sudo to root without a password.

- Set up ssh keys

```sh
ssh-keygen -f ansible
ssh-add ~/.ssh/ansible
ssh-copy-id -f ~/.ssh/ansible $targethost
```

Demo: Test ping

- Test with the ping module:

```sh
ansible -m ping target
```

- Setting up sudo

```sh
echo "ansible_user (ALL) NOPASSWD: ALL" > \
  /etc/sudoers.d/ansible
```

In some cases you will want to lock this down further, perhaps with a password (you can enter a sudo password if you add -K to your command line).

Demo: Test sudo setup

```sh
ansible -a whoami target
ansible -a whoami -b target
```

(Note that ansible runs the command module by default so no -m is needed)

### Modules

A single module allows the execution of a self-contained task.

Modules are designed to provide an abstraction around simple and complex tasks to allow them to be repeatable and handle error conditions nicely

We've already briefly seen the ping and command modules.

- Built-in modules

There are modules for an awful lot of things — e.g.:

- configuring services in AWS, Google, Azure, Openstack etc.
- installing OS packages
- writing to files
- updating network devices
- configuring databases
- and many others...

See the Ansible Module Index for a full list of categories.

- Demo: ansible

Using the ansible command line utility, it's easy to run a simple module to get all of the facts from a repo

```sh
ansible -m setup target
```

or run an ad-hoc task

```sh
ansible -m file -a "state=directory path=~/throwaway" target
```

Demo: ansible-doc

ansible-doc is very useful for finding the syntax of a module without having to look it up online

e.g. ansible-doc mysql_user

## Playbooks

A playbook is, at its simplest, a list of tasks to run in sequence against a list of hosts. The setup task is run first.

```yml
- hosts: target

  tasks:
  - name: ensure ~/throwaway doesn't exist
    file:
      path: ~/throwaway
      state: absent
      
```

- Structuring playbooks

There are a number of elements to a playbook, that can be separated into the following classes of elements:

- Connection specification: hosts, forks, serial
- User specification: remote_user, become,  become_user etc.
- Variable inclusion: vars, vars_files,  vars_prompt etc.
- Logic before tasks: pre_tasks, roles
- Tasks and handlers: tasks, handlers

Playbooks can also include other playbooks, using  include at the play level, or include task files by using the  include task

```yml
- hosts: all

  include: another_playbook.yml

  tasks:
  - include: setup.yml
  - include: configure.yml
```

- Check Mode

Ansible can be run in test mode using the -C or  --check-mode flags.

```sh
ansible-playbook -C examples/playbook.yml
````

Because ansible doesn't know if a command will have an effect in dry run mode, tasks that don't support dry run (particularly command and shell tasks) are skipped by default. You can force them to be run using the  check_mode flag (previously the always_run flag)

Example: 

```yml
- name: get version of coreutils rpm
  command: rpm -q coreutils
  args:
    warn: no
  changed_when: False
  check_mode: yes
```

You can also do things differently when in check mode by changing behaviour based upon  when: ansible_check_mode.

- Diff Mode

Diff mode is incredibly useful in conjunction with check mode to see how a file would change, before the change is made. Diff mode can be run by passing -D or  --diff-mode to ansible-playbook

- Repeatability

If a playbook runs twice, the expected result is that nothing should change on the second run — ansible should report zero changes.

This should be true whether 10 seconds later, or 10 months later.

- Organising playbooks

For any self-contained set of playbooks (this might be all of your playbooks, or playbooks just for a particular application), the following is a reasonable directory structure

```
.
├── ansible.cfg
├── inventory
│   └── group_vars
│       └── web
└── playbooks
    ├── simple
    │   ├── deploy-web-service.yml
    │   └── templates
    │       ├── index.html.j2
    │       └── python-web.service.j2
    └── with_roles
```

If you need modules or plugins, they can also be installed in the top level directory. Something like

```sh
.
├── ansible.cfg
├── inventory
├── library
├── playbooks
└── plugins
    ├── filter
    └── lookup
```

is likely a reasonable setup.

## Tasks

A task comprises the module to run and the arguments with which to run the task.

There are also several modifiers to the task, determining whether it is run, who it is run by, where it is run, and others.

- Naming tasks

Best practices suggest always naming tasks — it's easier to follow what is happening if the tasks are well named.

Naming tasks also allows the use of --start-at-task to allow ansible-playbook to start at a later point in the playbook

- Naming tasks in action

The task in the previous playbook looks like this when named:

```
TASK [ensure ~/throwaway doesn't exist] ****************************************
ok: [target]
```
and when not named:

```
TASK [file] ********************************************************************
ok: [target]
```

- Task modifiers: when

You can tell a task to run only if certain conditions are met.

```yml
- name: install httpd (Red Hat)
  yum:
    name: httpd
  when: 'ansible_distribution == "RedHat"'

- name: install apache (Debian)
  apt:
    name: apache2
  when: 'ansible_distribution == "Debian"'
```

See more: [Ansible Conditionals](http://docs.ansible.com/ansible/playbooks_conditionals.html)

- Task modifiers:  with_items

A task can loop over a list of items (or indeed many other kinds of data structures). For example, you can provide a list of packages for apt or yum to install, or a list of definitions of database users (username, password, privileges etc).

See more: [Ansible Loops](http://docs.ansible.com/ansible/playbooks_loops.html)

Example:

```
- name: add database users
  mysql_user:
  - name: "{{ item.name }}"
    password: "{{ item.password }}"
  with_items:
  - name: bob
    password: abc123
  - name: sys
    password: superSekrit!
```

- Task modifiers:  become

The become modifiers are useful when you need a task to run as a different user to the rest of the playbook. Security best practices suggest using a standard user for most tasks and elevating privilege when required — but even if you're running the playbook as root, you might need to become e.g. the postgres user to run a DB related task.

See more: [Become](http://docs.ansible.com/ansible/become.html)

Example:

```yml
- name: install package
  yum:
    name: httpd
    state: present
  become: yes

- name: connect to the database
  command: psql -U postgres 'select * from pg_stat_activity()'
  become_user: postgres
```

- Best practices: YAML

Note that the previous task could be written

```yml
    yum: name=httpd state=present
```

Ansible have said that the preferred format is the YAML dictionary format, as the key=value form can lead to strange errors (we'll see this later with the --syntax-checker)

- Repeatability — command tasks

In particular, shell and command will always return  changed: True. Use of changed_when: False when running read-only commands is encouraged to minimise false alarms:

```yml
- name: get list of files in a directory
      command: ls /path/to/directory
      register: directory_contents
      changed_when: false
      check_mode: true
```

- Command tasks

Other means of reducing the amount of unnecessary changes:
use creates or removes with commands to prevent an action happening if it's already happened
use when with a pre-check read-only task (with  changed_when: False) to see if an action needs to happen before doing it

- Command pre-check example

```yml
- name: check tuned profile
  command: tuned-adm active
  register: tuned_adm
  changed_when: False

- name: set tuned profile
  command: tuned-adm profile virtual-guest
  when: "'Current active profile: virtual-guest' \
         not in tuned-adm.stdout"
```

## Templates

Templates allow you to generate configuration files from values set in various inventory properties. This means that you can store one template in source control that applies to many different environments.

Ansible uses the Jinja templating language. The language offers control structures ({% for %},  {% if %} etc.) and filters that process variables (e.g.{{ "hello"|upper }} would print HELLO.

Example :

An example might be a file specifying database connection information that would have the same structure but different values for dev, test and prod environments

```yml
$db_host = '{{ database_host }}';
$db_name = '{{ database_name }}';
$db_user = '{{ database_username }}';
$db_pass = '{{ database_password }}';
$db_port = {{ database_port}};
```

- Using templates

Templates are populated using

```yml
  - template:
      src: path/to/template.j2
      dest: /path/to/result
```

- Template directory structure

Because configuration files for an application can end up with similar names in different directories, reflect the target destination in the source repository

```yml
- name: configure logrotate
  template:
    src: etc/logrotate.d/httpd.conf.j2
    dest: /etc/logrotate.d/httpd.conf

- name: configure apache
  template:
    src: etc/httpd/conf/httpd.conf.j2
    dest: /etc/httpd/conf/httpd.conf
    owner: apache
```

## Handlers

Handlers are tasks that are 'fired' when a task makes a change

```yml
    tasks:
    - name: install httpd configuration file
      template:
        src: etc/httpd/conf/httpd.conf.j2
        dest: /etc/httpd/conf/httpd.conf
      notify: reload apache

    handlers:
    - name: reload apache
      service:
        name: apache
        state: reloaded
```

This is another mechanism of ensuring a change is made only when it needs to be (as if no change is made to the configuration, the handler will not fire)

Handlers are run at the end of all tasks — if you want to run them earlier, you can use the meta task:

```yml
- meta: flush_handlers
```

## Verbose Mode

- Adding extra -v arguments will increase the verbosity of ansible's output.

No -v — just output tasks and plays and whether or not it's successful
-v — shows the results of modules
-vv — shows the files from which tasks come
-vvv — shows what commands are being executed under the hood on the target machine
-vvvv — shows what callbacks have been loaded
-vvvvv — shows some additional ssh configuration information

- Best practice: debug messages
You can (and should) configure your debug messages to appear only at certain verbosities

```yml
  debug:
    msg: "This will appear with -vv but not before"
    verbosity: 2
```

## Configuration file

```
[defaults]
hostfile = ./inventory
roles_path = ./roles
library = ./library
filter_plugins = ./plugins/filter
lookup_plugins = ./plugins/lookup
callback_whitelist = profile_tasks,timer

[ssh_connection]
pipelining = True
control_path = %(directory)s/ssh-%%h-%%p-%%r
```

hostfile = ./inventory allows inventory to be stored next to the playbooks it is for
library = ./library allows the easy installation of custom modules near the code (useful if a module is not available from Ansible or a bug has been fixed for a newer/unreleased version)
filter_plugins = ./plugins/filter — allows for addition of new plugins for Jinja templating
callback_whitelist = profile_tasks,timer turns on timing information for individual tasks and for the playbook run as a whole
pipelining = True speeds up the execution of modules as ansible only has to run one ssh command, not three.
control_path reduces the length of the default setting, which is useful if you have long hostnames (as the setting can only be 106 characters).

#### Variables (Part 2)
#### Inventory (Part 2)
#### Roles (Part 3)



## Variables

In general logic should be the same (or similar) for all environments.
Variables fill in the contents of template files, can be used for the source of files, and to choose whether or not to perform a task (to name some reasons)
Fewer variations in configuration sources reduces likelihood of errors (Don't Repeat Yourself)

roles can set default values for variables, and these are available to the rest of the playbook.

## Inventory

Inventory is used to define the hierarchy of hosts and the groups to which they belong.
The inventory source can be a file, or a script, or a directory containing such files and scripts.
Ansible also sources host variables and group variables from host_vars and group_vars stored in the inventory directory

- Inheritance

Inventory variables take precedence the closer they are to the host.
Host variables override group variables
Child group variables override their parents
This means you can set defaults in top level groups, and override them lower down (e.g. the default log level for an application might be WARN, but in development it should be DEBUG).

- ansible-inventory-grapher

Use ansible-inventory-grapher (pip install ansible-inventory-grapher) along with graphviz to help visualize inventory hierarchies:

```sh
ansible-inventory-grapher -q target | \
  dot -Tpng | display png:-
  ```

- Best practices: inventory

  - Set hosts_file in the ansible configuration file to a directory
  - Each file in that directory can be an independent part of the inventory
  - Inventory scripts can also live in that directory
  - That directory can contain host_vars and  group_vars

- Best practices:  host_vars

  - Host variables should be used only for things that will only be true for a single host. An example of this might be caching of a UUID of a host, or setting kerberos keytabs

  - This means that SSL certificates and keys, kerberos keytabs, server uuids etc. might be candidates, but most other inventory variables will be properties of groups.

- Anti-pattern: variables in host files

  - Variables should not be stored in inventory host files (using [group:vars] or [host:vars] mechanism) — the inventory files should be used for group contents and hierarchy definitions (using  [group:children]).
  - Use group_vars instead, or host_vars at a push.

- Playbook vars and vars_prompt

In general playbooks shouldn't need to define vars, but the capability exists.

vars_prompt is useful if you need to provide a variable at run time — e.g. a password for a service and don't want to source it from a vaulted file.

```yml
- hosts: certificate_authority

  vars_prompt:
  - name: ca_password
    prompt: "Please enter your CA password"

  tasks:
  - name: sign certificate
    command: openssl ca -in req.pem \
      -out newcert.pem -passin env:CA_PASSWD
    environment:
      CA_PASSWD: "{{ ca_password }}"
```

- registered variables

registered variables used to store the results of a task in a playbook.

```yml
   - name: get stat data for file
     stat:
       path: /path/to/file
     register: stat_file

   - name: fail if path doesn't exist
     fail:
       msg: "File does not exist"
     when: not stat_file.stat.exists
```

- Facts

  - Information about a host sourced at runtime, e.g. IP address or OS version.
  - You don't need to run the setup module directly to gather facts — it is always run in playbook mode, unless gather_facts is set to False
  - If you ran the previous lab, you should be able to see the facts for target at http://192.168.33.11:8000/

- set_fact module

  - The set_fact module is used to derive new facts from existing facts to produce more useful ones.

```yml
 - name: set timezone fact
    set_fact:
    args:
      timezone: "{{ ansible_date_time.tz }}"
```

  If os_version is the fact obtained by joining  ansible_distribution with  ansible_distribution_major_version then:

  - The following will look under the vars directory of a role for a file called e.g. CentOS7.yml

```yml
- name: include variables based on OS version
    include_vars: "{{ os_version }}.yml"
```

  - The following will look under the tasks directory of a role for a file called e.g. CentOS7.yml

```yml
- name: run tasks based on OS version
    include: "{{ os_version }}.yml"
```

- include_vars and  vars_files

  - Use include_vars to include a variables file as a task in a playbook run. You can use no_log to ensure vars aren't logged.

  - You can also use vars_files in a playbook to include one or more variables files. vars_files can't be used in a role.

- Role vars/main.yml

  - Use roles vars/main.yml when you want to override another role's defaults.

- Extra vars -e

  - Command line extra vars are useful for setting configuration at run-time.
  - Set lots of variables at once by including a variables file using -e @filename.yml — can be useful for overriding defaults during an outage.

- Variable precedence
The order of variables presented has been in increasing order. There are more variable types than presented here — others aren't widely used or highly recommended

See more: [Ansible Variable precedence](http://docs.ansible.com/ansible/playbooks_variables.html#variable-precedence-where-should-i-put-a-variable)

- Secret variables

ansible provides a tool called ansible-vault for encrypting secret variables. while other tools are available, the vault is usefully integrated.

- ansible-vault

  - create: ansible-vault create secrets.yml
  - edit: ansible-vault edit secrets.yml
  - view: ansible-vault view secrets.yml
  - encrypt existing file:  ansible-vault encrypt secrets.yml
  - decrypt existing file:  ansible-vault encrypt secrets.yml
  - change password:  ansible-vault rekey secrets.yml

see more: [ansible vault](http://docs.ansible.com/ansible/playbooks_vault.html#running-a-playbook-with-vault)

- Using vaulted secrets

```sh
ansible-playbook playbook.yml --ask-vault-pass
```

you can also set the password in a file (e.g.  ~/.ansible/vault_pass) and use:

```sh
ansible-playbook playbook.yml --vault-password-file ~/.ansible/vault_pass
```

or set the ANSIBLE_VAULT_PASSWORD_FILE environment variable.

## Roles

- What is a role

  - Roles are reusable packages of configuration included by playbooks
  - Roles can do most of the things that playbooks do — run tasks and handlers, install files or write templates, set variables
  - Examples include roles that install database engines (mysql, postgresql etc.), web servers (apache, nginx) and many more

- Why do we have roles
  - Roles should implement best practices (e.g. an apache role that enforced secure SSL ciphers)
  - If more than one playbook might do something in the same way, that should be abstracted to a role

- Where do roles come from
  - Ansible Galaxy (http://galaxy.ansible.com/) — there are 1000s of roles suitable for most operating systems
  - ansible-galaxy init new-role creates a role skeleton

- Ansible role skeleton

```sh
testrole/
├── defaults         ├── tasks
│   └── main.yml     │   └── main.yml
├── files            ├── templates
├── handlers         ├── tests
│   └── main.yml     │   ├── inventory
├── meta             │   └── test.yml
│   └── main.yml     └── vars
└── README.md            └── main.yml
```

- Installing a role

```sh
Best practice is to use a requirements.yml file containing the specification of the role you wish to use. This can be from Ansible Galaxy or from github or your own internal source repository.
```

- requirements.yml

The following are effectively equivalent:

```yml
- src: geerlingguy.mysql

- src: https://github.com/geerlingguy/ansible-role-mysql
  version: 2.4.0
  name: mysql
  scm: git
```

The latter mechanism is more useful for roles that don't come from galaxy.

The roles are then installed using

```sh
ansible-galaxy install -p playbooks/application/env/roles \
  -r playbooks/application/env/requirements.yml -f
```

- Using a role

  - Read the README file to see what variables are expected, and then set them appropriately in inventory.

  - Rather than a bunch of tasks, your playbook might then look like

```yml
- hosts: appserver

    roles:
    - mysql
    - nginx
    - application
```

where application might be your role that installs your own application

- Creating new roles

  - The ansible-galaxy init rolename (http://docs.ansible.com/galaxy.html) command can be use to create new roles
  - Each application-environment combination gets its own roles file used to provide roles for the playbooks

- Roles, playbooks, source control

  - We keep each role in its own source control repo for ease of software development and version management.
  - Recommend naming the repo rolename-role or even ansible-role-rolename to avoid clashes (you can choose what name a role gets in the requirements file
  - Add the roles directories for the playbooks to the  .gitignore file (or similar for your source control)

- Versioning roles

There are some good reasons to version roles:

  - Allows safe development of roles without enforcing strict backward compatibility — older playbooks can continue to use older versions of a role until it is ready to upgrade.
  - Versioning allows repeatability — if the same playbook with the same rolesfile is used in 6 months time, it should work in exactly the same way. This would not be the case if a playbook just used the latest versions of the role.

- How we version roles

  - Production playbooks must use versioned roles
  - Production roles must have versions.
  - Once a change to a role has passed code review and is accepted on the master branch:

```sh
git pull upstream master
git tag v2.3
git push upstream v2.3
```


## Tools

- syntax check

Ansible can check the syntax of a playbook using  --syntax-check argument.

```sh
$ ansible-playbook --syntax-check playbooks/simple/broken-syntax.yml
ERROR! Syntax Error while loading YAML.


The error appears to have been in '/home/vagrant/playbooks/simple/broken-syntax.yml': line 5, column 22, but may
be elsewhere in the file depending on the exact syntax problem.

The offending line appears to be:

  - name: this task will fail syntax check
    debug: msg="hello: world"
                     ^ here
```

- ansible-lint

ansible-lint was developed to find various style and usage issues with playbooks and roles, and suggest improvements.

There are several categories of rules:

  - Using command/shell module instead of Ansible modules
  - Deprecated syntax that is being removed by Ansible
  - Incorrect formatting

Every rule is treated as an error — there is no way to mark rules as warnings. You can choose to run specific rules or to exclude specific rules

- Code review

Code reviews are essential to maintaining codebase quality. We use code reviews to check for standards compliance and best practices, and also to ensure that the change is the right thing to do for future maintenance.

Reviews should be objective - there should be a documented reason for comments made in a review.

Code reviews can lead to less code in the codebase. This can be a good thing. By spotting common patterns, functionality that is in a common role can be removed from an application role (roles do not need to install apache themselves when they can rely on the apache role, for example).

Conversely, code that is specific to an application should be kept away from the more general roles.

- Standards and Best Practices

  - We keep our standards, best practices, guide to code reviews etc. in version control
  - Our standards documentation goes through a similar review process to our code. This is so that standards are agreed, and not imposed.
  - Standards have a version number and a changelog, so that it's obvious what standards have been added, improved or even removed over time.

- ansible-review

ansible-review is designed to apply an automated code review process to work with an organisation's standards.

As you add new standards to your style guide, you can say what version of standards a playbook or role is designed to meet. This allows older code to not fail against newer standards.

- Running a review

After following the instructions on the ansible-review github repo you can run a review tool on just the changes made against master

This can be run in any ansible repo, against roles or playbooks. To run against all files in a repo (e.g. for a new role), just use

```sh
git ls-files | xargs ansible-review
```

or

```sh
git diff master | ansible-review
```

## Reference

- [Ansible Docs](http://docs.ansible.com/ansible/)
- [Ansible source](https://github.com/ansible/ansible)
- [Ansible Mailing list](https://groups.google.com/forum/!ansible-project)
- [Ansible Galaxy](http://galaxy.ansible.com/)
