---
layout: post
author: Breno Cesar
title:  "My personal good practices when filtering environment in Ansible"
subtitle: "So, would you ever needed to run a role in ansible and you face some troubbles with different environment ? Here a quick tip for a fast and reliable whay to filter an enviroment"
categories: ansible
date: 2020-08-06 12:15:55
---
Sometimes when we are automating some process, its common to face a scenario where a command/script needed to be runned in a vast and different environments, and than we discovered that some key command in the process have a different behaviour on every environment.
<br>
Let's supose a scenario where we want to execute ```my_ansible_script.yml``` ,where:
- All hosts is in the same group in the inventory and they came from different types of OS environments.
- For some reason, this playbook get the OS environment information exectuting a script on each host.
- And based on that output another script is executed. 
<br>
The common and fast response to that problem is:
- Grab some information about the environment with a shell command and register that in a variable;

```yaml
- name: Grabbing OS type of my environment
  shell: /some/shell/or/script/to/get/the/info/command
  register: my_var
```
- Use the output of that variable as statement when running the desired command, providing specific path or extra information:
```yaml
- name: Running my command on CentOS
  shell: command
  args:
    chdir: /my_path/in/CentOS/
  when: '"CentOS" in my_var.stdout'

- name: Running my command on SLES
  environment:
    ENV_SLES: "/my_personal_env"
  shell: /my_path/in/SLES/command
  when: '"SLES" in my_var.stdout'

- name: Running my command on Solaris
  environment:
    ENV_SLES: "/my_personal_env"
  shell: /my_path/in/Solaris/command
  args:
    chdir: /my_path/in/Solaris/
  when: '"Solaris" in my_var.stdout'  
```
There is no problem for a couple of simple task like these executing in this way.

But in cases were your automation have alot of task , probably you will face some issues like:
- All 3 tasks will be executed, but skipped in case the "my_var" contains only "Solaris".
- If each task take a large amount of time to be executed, that will affect your playbook performance
- If the script that get the OS information have some performance issue or for some reason stop to work, these playbook will end-up with a error or failed result.
- If a manutention in the script is needed, you will need to check if every when statement work as spected for every environment.

<br>
To avoid this problemas, i came with a simple and fast solution, that is to grab all environment informantion trought [ansible setup](https://docs.ansible.com/ansible/latest/modules/setup_module.html) module.
<br>
> *"But wait a minute, this module take too much time to be executed, and will flood the memory of my ansible execution machine with unusefull information..."* ( User, Random - 2020).<br>

You are correct **Random User**, but only if you don't use it properly.
<br>
In my daily work, i pass to use roles, if you don't know what is is you can checkout the official documentation [here](https://galaxy.ansible.com/docs/contributing/creating_role.html). 
<br>
Using this simple role structure:
<br>
```
.
├── my_ansible_script.yml
└── roles
    └── my_role
        └── tasks
            ├── main.yml
            ├── check_environment.yml
            ├── centos_play.yml
            ├── sles_play.yml
            └── solaris_play.yml
```
Was possible turn ```my_ansible_script.yml``` in to a fast an easy to maintaining and [idempotent](https://docs.ansible.com/ansible/latest/reference_appendices/glossary.html) playbook.
<br>
For that, first we will need to change the ```./my_ansible_script.yml``` as bellow, where we point to out role, nothing special about it.
```yaml
---
- hosts: all
  become: yes
  become_method: su
  gather_facts: False
  roles:
    - my_role
```
Here in the ```./roles/my_role/tasks/main.yml``` is were the magic beging:

```yaml
---
- name: Check the Environment
  include_tasks: check_environment.yml
  
- name: Running my command on CentOS
  include_tasks: centos_play.yml
  when: '"CentOS" in ansible_distribution'

- name: Running my command on SLES 
  include_tasks: sles_play.yml
  when: '"SLES" in ansible_distribution'

- name: Running my command on Solaris 
  include_tasks: solaris_play.yml
  when: '"Solaris" in ansible_distribution'
```
Note that here we are telling to the role to only execute a play when a certain match occurs, in ou case, a desired OS.

So in the ```centos_play.yml```, it will only run the command bellow if the string "**CentOS**" is in the **ansible_distribution** variable.
```yaml
- name: Running my command on CentOS
  shell: command
  args:
    chdir: /my_path/in/CentOS/
```
And in the ```sles_play.yml```, it will only run the command bellow if the string "**SLES**" is in the **ansible_distribution** variable.
```yaml
- name: Running my command on SLES
  environment:
    ENV_SLES: "/my_personal_env"
  shell: /my_path/in/SLES/command
```
And finaly in the ```solaris_play.yml```, it will only run the command bellow if the string "**Solaris**" is in the **ansible_distribution** variable.
```yaml
- name: Running my command on Solaris
  environment:
    ENV_SLES: "/my_personal_env"
  shell: /my_path/in/Solaris/command
  args:
    chdir: /my_path/in/Solaris/
```

With this minor change, we will have only 4 global "*when*" statement, and if a manintainence is needed in a play for a specific environment, the change will not affect the behavior on other plays for other environments as well.
<br>
>*"There is noting new until here Breno..."* ( User, Random - 2020).<br>

Chill out **Random User**, and please be patiente, because here is were the magic realy happen.
<br>
On ```./roles/my_role/tasks/check_environment.yml``` we will use the [ansible setup](https://docs.ansible.com/ansible/latest/modules/setup_module.html) module as i mentioned before, but with some spices.
<br>
Using the [ansible setup](https://docs.ansible.com/ansible/latest/modules/setup_module.html) in his pure form, will get all information about a host, and it a very time consuming task, but here as we only need the OS information, we will provide the option **gather_subset:**, this option as mentioned in the [official documentation](https://docs.ansible.com/ansible/latest/modules/setup_module.html):
<br>
>*If supplied, restrict the additional facts collected to the given subset (...)*<br>

and

>*Values can also be used with an initial ! to specify that that specific subset should not be collected.(...)*

With this information, we will exclude the execution of what we don't need because is time consuming (the "**!all**" that get everthing in the environment and the "**!min**" that is the "**!all**" with less information) and execute the setup for only what we need, in our case here, is the **distribution**.
<br>
 ```yaml
---
- name: Check Enviroment
  setup:
    gather_subset:
    - '!all'
    - '!min'
    - 'distribution'
```

Under he hood, the [ansible setup](https://docs.ansible.com/ansible/latest/modules/setup_module.html) will fastly execute only the distribution part, and provide us a simple and fast distribution information about a host under the "ansible_facts" default varible, as the example bellow:


```json
    "ansible_facts": {
        "ansible_distribution": "Solaris",
        "ansible_distribution_major_version": "10",
        "ansible_distribution_release": "Oracle Solaris 10 1/13 s10s_u11wos_24a SPARC",
        "ansible_distribution_version": "10",
        "ansible_os_family": "Solaris",
        "discovered_interpreter_python": "/usr/bin/python",
        "gather_subset": [
            "!all",
            "!min",
            "distribution"
        ],
        "module_setup": true
    },
    "changed": false
}
```

Using this process, no script,specific command or extra variables is needed, because, all that we need to filter and redirect our commands according to our enviroment is in the defaults ansible variables and the setup module.

And If you have a large amount of host, and want to reduce the data stored on memory during the execution, you can use the **environment filter**, and store only the variable that you need:
```yaml
- name: Check Enviroment
  setup:
    gather_subset:
    - '!all'
    - '!min'
    - 'distribution'
    filter: ansible_distribution
      
```

Under the hood, ansible will excetude everything under the **distribution** option, but will only store the **ansible_distribution** provided before.

```json
"ansible_facts": {
        "ansible_distribution": "Solaris",
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": false
}
```
This will work for every host option mentionated in the [setup documentatin](https://docs.ansible.com/ansible/latest/modules/setup_module.html), with and exception for the **facter**, that need's the [puppet](https://puppet.com/docs/puppet/latest/man/agent.html) be installed in the host.

I hope that you enjoyed this small tutorial, and be usefull for you as well.

See you !!

[]'s