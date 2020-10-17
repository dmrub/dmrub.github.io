---
layout: post
title:  "Interactive SSH with Ansible"
date:   2020-10-17 16:27:34 +0200
categories: ansible ssh
---

I use Ansible daily in my work. I started to learn it when I installed Kubernetes in 2015 using Ansible. After getting to know it I quickly switched to using it for all administrative tasks like software installation. Why? For the person who is familiar with the secure shell (ssh), Ansible offers the most advantages, I would even say Ansible is ssh on steroids. You can do the same tasks as with ssh, but do them automatically. And it is much easier than writing shell scripts that use ssh.

However, I quickly found out one single disadvantage. Ansible is not designed to be used as an interactive shell. And even if you do all tasks automatically, it will always be necessary to manually log in to the host to perform some tasks manually. For a while I simply kept both configurations side by side, Ansible inventory for accessing hosts with Ansible commands and playbooks and ssh scripts for manual operation. To maintain this, of course, meant a double effort that seemed unproductive in the long run. So I had the idea to use Ansible itself to login interactively with SSH. Since Ansible itself uses SSH for login, you have to describe the necessary information (like username, path to the SSH private key and other options) in the inventory. The only thing missing is a suitable tool to extract this information and start ssh directly with it. Unfortunately, Ansible currently does not ship with anything suitable. One possibility is to use the tool [ansible-inventory](https://docs.ansible.com/ansible/latest/cli/ansible-inventory.html). With it you can output all ansible variables of a host. Variables that control the ssh configuration are `ansible_user`, `ansible_ssh_common_args`, `ansible_ssh_private_key_file`, etc. ([see here for details](https://docs.ansible.com/ansible/latest/user_guide/intro_inventory.html)). But in Ansible variables with the [Jinja2 template syntax](https://jinja.palletsprojects.com/en/2.11.x/templates/) can refer to other variables as described [here](https://docs.ansible.com/ansible/latest/user_guide/playbooks_variables.html#referencing-simple-variables) and `ansible-inventory` returns them unevaluated.

My idea to solve the problem was pretty simple, because when you run Ansible Playbook, Ansible evaluates all variables and you can even print them with the [debug module](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/debug_module.html), so why not just let Ansible itself generate an SSH configuration with all hosts in the inventory?

## Evaluating Variables

We will write an Ansible playbook that creates an SSH configuration as described in the Linux manpage [ssh_config](https://man7.org/linux/man-pages/man5/ssh_config.5.html). Such a configuration file can be used with the ssh command by specifying the `-F` option.
The first step is to evaluate all variables we need to create SSH configuration.
As already mentioned, there are `ansible_user`, `ansible_ssh_common_args`, `ansible_ssh_private_key_file`, etc. Since our playbook is supposed to create a local file, we do not need to run on all hosts for which we want to create the SSH configuration. But to evaluate all needed variables correctly, we have to evaluate them in the context of all hosts. We will use the [`set_fact` module](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/set_fact_module.html) to store the evaluated results in new variables with the `eval_` prefix.

{% highlight yaml %}
{% raw %}
- hosts: all
  gather_facts: false
  tasks:

    - name: Force evaluation of used host variables
      set_fact:
        eval_ansible_ssh_common_args: "{{ ansible_ssh_common_args | default(omit) }}"
        eval_ansible_connection: "{{ ansible_connection | default(omit) }}"
        eval_ansible_ssh_private_key_file: "{{ ansible_ssh_private_key_file | default(omit) }}"
        eval_ansible_host: "{{ ansible_host | default(omit) }}"
        eval_ansible_user: "{{ ansible_user | default(omit) }}"
        eval_ansible_port: "{{ ansible_port | default(omit) }}"
{% endraw %}
{% endhighlight %}

Since we are not interested in performing any tasks on hosts and only want to evaluate variables, we [disable the fact gathering](https://docs.ansible.com/ansible/latest/user_guide/playbooks_vars_facts.html#disabling-facts) with `gather_facts: false`.
The expression `... | default(omit)` is a filter that uses a special 
[`omit`](https://docs.ansible.com/ansible/latest/user_guide/playbooks_filters.html#making-variables-optional) variable 
and causes that if `ansible_` variable is not defined, the corresponding `eval_` variable is not defined either.

When we create such a playbook, it is always useful to check after each step to make sure that we get what we expect. In this case, we can print out all evaluated variables and check if all templates have been evaluated:

{% highlight yaml %}
{% raw %}
    - name: Print vars
      debug:
        msg:
          - "eval_ansible_ssh_common_args: {{ eval_ansible_ssh_common_args | default('UNDEFINED') }}"
          - "eval_ansible_connection: {{ eval_ansible_connection | default('UNDEFINED') }}"
          - "eval_ansible_ssh_private_key_file: {{ eval_ansible_ssh_private_key_file | default('UNDEFINED') }}"
          - "eval_ansible_host: {{ eval_ansible_host | default('UNDEFINED')}}"
          - "eval_ansible_user: {{ eval_ansible_user | default('UNDEFINED')}}"
          - "eval_ansible_port: {{ eval_ansible_port | default('UNDEFINED') }}"
{% endraw %}
{% endhighlight %}

## Generating the SSH configuration

All next steps should be done on the local host without using SSH, for this purpose Ansible supports the [local connection](https://docs.ansible.com/ansible/latest/user_guide/playbooks_delegation.html#local-playbooks):

{% highlight yaml %}
{% raw %}
- hosts: "all"
  gather_facts: false
  tasks:

    - name: Force evaluation of used host variables
      set_fact:
        eval_ansible_ssh_common_args: "{{ ansible_ssh_common_args | default(omit) }}"
        eval_ansible_connection: "{{ ansible_connection | default(omit) }}"
        eval_ansible_ssh_private_key_file: "{{ ansible_ssh_private_key_file | default(omit) }}"
        eval_ansible_host: "{{ ansible_host | default(omit) }}"
        eval_ansible_user: "{{ ansible_user | default(omit) }}"
        eval_ansible_port: "{{ ansible_port | default(omit) }}"

- hosts: 127.0.0.1
  connection: local
  gather_facts: false
  vars:
    target: "{{ groups['all'] }}"
    ansible_python_interpreter: "{{ansible_playbook_python}}"
  tasks:

    - name: Print vars
      debug:
        msg:
          - "eval_ansible_ssh_common_args: {{ hostvars[item]['eval_ansible_ssh_common_args'] | default('UNDEFINED') }}"
          - "eval_ansible_connection: {{ hostvars[item]['eval_ansible_connection'] | default('UNDEFINED') }}"
          - "eval_ansible_ssh_private_key_file: {{ hostvars[item]['eval_ansible_ssh_private_key_file'] | default('UNDEFINED') }}"
          - "eval_ansible_host: {{ hostvars[item]['eval_ansible_host'] | default('UNDEFINED')}}"
          - "eval_ansible_user: {{ hostvars[item]['eval_ansible_user'] | default('UNDEFINED')}}"
          - "eval_ansible_port: {{ hostvars[item]['eval_ansible_port'] | default('UNDEFINED') }}"
      loop: "{{ target }}"
{% endraw %}
{% endhighlight %}

*TO BE CONTINUED ...*
