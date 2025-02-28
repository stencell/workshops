# Exercise 1.7 - Roles: Making your playbooks reusable

**Read this in other languages**: ![uk](../../../images/uk.png) [English](README.md),  ![japan](../../../images/japan.png) [日本語](README.ja.md).

While it is possible to write a playbook in one file as we've done throughout this workshop, eventually you’ll want to reuse files and start to organize things.

Ansible Roles is the way we do this.  When you create a role, you deconstruct your playbook into parts and those parts sit in a directory structure.  This is explained in more detail in the [best practice](http://docs.ansible.com/ansible/playbooks_best_practices.html) already mentioned in exercise 3.

## Step 7.1 - Understanding the Ansible Role Structure

Roles are basically automation around *include* directives as described above, and really don’t contain much additional magic beyond some improvements to search path handling for referenced files.

Roles follow a defined directory structure, a role is named by the top level directory. Some of the subdirectories contain YAML files, named `main.yml`. The files and templates subdirectories can contain objects referenced by the YAML files.

An example project structure could look like this, the name of the role would be "apache":

```text
apache/
├── defaults
│   └── main.yml
├── files
├── handlers
│   └── main.yml
├── meta
│   └── main.yml
├── README.md
├── tasks
│   └── main.yml
├── templates
├── tests
│   ├── inventory
│   └── test.yml
└── vars
    └── main.yml
```

The `main.yml` files contain content depending on the corresponding directory:  `vars/main.yml` references variables, `handlers/main.yaml` describes handlers, and so on. Note that in contrast to playbooks, the `main.yml` files only contain the specific content and not additional playbook information like hosts, `become` or other keywords.

> **Tip**
>
> There are actually two directories for variables: `vars` and `default`: Default variables have the lowest precedence and usually contain default values set by the role authors and are often used when it is intended that their values will be overridden.. Variables can be set in either `vars/main.yml` or `defaults/main.yml`, but not in both places.

Using roles in a Playbook is straight forward:

```yaml
---
- name: launch roles
  hosts: web
  roles:
    - role1
    - role2
```

For each role, the tasks, handlers and variables of that role will be included in the Playbook, in that order. Any copy, script, template, or include tasks in the role can reference the relevant files, templates, or tasks *without absolute or relative path names*. Ansible will look for them in the role's files, templates, or tasks respectively, based on their
use.

## Step 7.2 - Create a Basic Role Directory Structure

Ansible looks for roles in a subdirectory called `roles` in the project directory. This can be overridden in the Ansible configuration. Each role has its own directory. To ease creation of a new role the tool `ansible-galaxy` can be used.

> **Tip**
>
> Ansible Galaxy is your hub for finding, reusing and sharing the best Ansible content. `ansible-galaxy` helps to interact with Ansible Galaxy. For now we'll just using it as a helper to build the directory structure.

Okay, lets start to build a role. We'll build a role that installs and configures Apache to serve a virtual host. Run these commands in your `~/ansible-files` directory:

```bash
[student<X>@ansible ansible-files]$ mkdir roles
[student<X>@ansible ansible-files]$ ansible-galaxy init --offline roles/apache_vhost
```

Have a look at the role directories and their content:

```bash
[student<X>@ansible ansible-files]$ tree roles
```

## Step 7.3 - Create the Tasks File

The `main.yml` file in the tasks subdirectory of the role should do the following:

  - Make sure httpd is installed

  - Make sure httpd is started and enabled

  - Put HTML content into the Apache document root

  - Install the template provided to configure the vhost

> **WARNING**
>
> **The `main.yml` (and other files possibly included by main.yml) can only contain tasks, *not* complete Playbooks!**

Change into the `roles/apache_vhost` directory. Edit the `tasks/main.yml` file:

```yaml
---
- name: install httpd
  yum:
    name: httpd
    state: latest

- name: start and enable httpd service
  service:
    name: httpd
    state: started
    enabled: true
```

Note that here just tasks were added. The details of a playbook are not present.

The tasks added so far do:

  - Install the httpd package using the yum module

  - Use the service module to enable and start httpd

Next we add two more tasks to ensure a vhost directory structure and copy html content:

<!-- {% raw %} -->
```yaml
- name: ensure vhost directory is present
  file:
    path: "/var/www/vhosts/{{ ansible_hostname }}"
    state: directory

- name: deliver html content
  copy:
    src: index.html
    dest: "/var/www/vhosts/{{ ansible_hostname }}"
```
<!-- {% endraw %} -->

Note that the vhost directory is created/ensured using the `file` module.

The last task we add uses the template module to create the vhost configuration file from a j2-template:

```yaml
- name: template vhost file
  template:
    src: vhost.conf.j2
    dest: /etc/httpd/conf.d/vhost.conf
    owner: root
    group: root
    mode: 0644
  notify:
    - restart_httpd
```
Note it is using a handler to restart httpd after an configuration update.

The full `tasks/main.yml` file is:

<!-- {% raw %} -->
```yaml
---
- name: install httpd
  yum:
    name: httpd
    state: latest

- name: start and enable httpd service
  service:
    name: httpd
    state: started
    enabled: true

- name: ensure vhost directory is present
  file:
    path: "/var/www/vhosts/{{ ansible_hostname }}"
    state: directory

- name: deliver html content
  copy:
    src: index.html
    dest: "/var/www/vhosts/{{ ansible_hostname }}"

- name: template vhost file
  template:
    src: vhost.conf.j2
    dest: /etc/httpd/conf.d/vhost.conf
    owner: root
    group: root
    mode: 0644
  notify:
    - restart_httpd
```
<!-- {% endraw %} -->


## Step 7.4 - Create the handler

Create the handler in the file `handlers/main.yml` to restart httpd when notified by the template task:

```yaml
---
# handlers file for roles/apache_vhost
- name: restart_httpd
  service:
    name: httpd
    state: restarted
```

## Step 7.5 - Create the index.html and vhost configuration file template

Create the HTML content that will be served by the webserver.

  - Create an index.html file in the "src" directory of the role, `files`:

```bash
[student<X>@ansible ansible-files]$ echo 'simple vhost index' > files/index.html
```

  - Create the `vhost.conf.j2` template file in the role's `templates` subdirectory.

<!-- {% raw %} -->
```html
# {{ ansible_managed }}

<VirtualHost *:8080>
    ServerAdmin webmaster@{{ ansible_fqdn }}
    ServerName {{ ansible_fqdn }}
    ErrorLog logs/{{ ansible_hostname }}-error.log
    CustomLog logs/{{ ansible_hostname }}-common.log common
    DocumentRoot /var/www/vhosts/{{ ansible_hostname }}/

    <Directory /var/www/vhosts/{{ ansible_hostname }}/>
  Options +Indexes +FollowSymlinks +Includes
  Order allow,deny
  Allow from all
    </Directory>
</VirtualHost>
```
<!-- {% endraw %} -->

## Step 7.6 - Test the role

You are ready to test the role against `node2`. But since a role cannot be assigned to a node directly, first create a playbook which connects the role and the host. Create the file `test_apache_role.yml` in the directory `~/ansible-files`:

```yaml
---
- name: use apache_vhost role playbook
  hosts: node2
  become: yes

  pre_tasks:
    - debug:
        msg: 'Beginning web server configuration.'

  roles:
    - apache_vhost

  post_tasks:
    - debug:
        msg: 'Web server has been configured.'
```

Note the `pre_tasks` and `post_tasks` keywords. Normally, the tasks of roles execute before the tasks of a playbook. To control order of execution `pre_tasks` are performed before any roles are applied. The `post_tasks` are performed after all the roles have completed. Here we just use them to better highlight when the actual role is executed.

Now you are ready to run your playbook:

```bash
[student<X>@ansible ansible-files]$ ansible-playbook test_apache_role.yml
```

Run a curl command against `node2` to confirm that the role worked:

```bash
[student<X>@ansible ansible-files]$ curl -s http://22.33.44.55:8080
simple vhost index
```

All looking good? Congratulations! You have successfully completed the Ansible Engine Workshop Exercises!

----

[Click here to return to the Ansible for Red Hat Enterprise Linux Workshop](../README.md#section-1---ansible-engine-exercises)
