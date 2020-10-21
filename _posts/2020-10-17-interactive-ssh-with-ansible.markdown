---
layout: post
title:  "Interactive SSH with Ansible"
date:   2020-10-17 16:27:34 +0200
categories: ansible ssh
---

I use Ansible daily in my work. I started to learn it when I installed Kubernetes in 2015 using Ansible. After getting to know it I quickly switched to using it for all administrative tasks like software installation. Why? For the person who is familiar with the secure shell (ssh), Ansible offers the most advantages, I would even say Ansible is ssh on steroids. You can do the same tasks as with ssh, but do them automatically. And it is much easier than writing shell scripts that use ssh.

However, I quickly found out one single disadvantage. Ansible is not designed to be used as an interactive shell. And even if you do all tasks automatically, it will always be necessary to manually log in to the host to perform some tasks manually. For a while I simply kept both configurations side by side, Ansible inventory for accessing hosts with Ansible commands and playbooks and ssh scripts for manual operation. To maintain this, of course, meant a double effort that seemed unproductive in the long run. So I had the idea to use Ansible itself to login interactively with SSH. Since Ansible itself uses SSH for login, you have to describe the necessary information (like username, path to the SSH private key and other options) in the inventory. The only thing missing is a suitable tool to extract this information and start ssh directly with it. Unfortunately, Ansible currently does not ship with anything suitable. One possibility is to use the tool [ansible-inventory](https://docs.ansible.com/ansible/latest/cli/ansible-inventory.html). With it you can output all ansible variables of a host. Variables that control the ssh configuration are `ansible_user`, `ansible_ssh_common_args`, `ansible_ssh_private_key_file`, etc. ([see here for details](https://docs.ansible.com/ansible/latest/user_guide/intro_inventory.html)). But in Ansible variables with the [Jinja2 template syntax](https://jinja.palletsprojects.com/en/2.11.x/templates/) can refer to other variables as described [here](https://docs.ansible.com/ansible/latest/user_guide/playbooks_variables.html#referencing-simple-variables) and `ansible-inventory` returns them unevaluated.

My idea to solve the problem was pretty simple, because when you run Ansible Playbook, Ansible evaluates all variables and you can even print them with the [debug module](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/debug_module.html), so why not just let Ansible itself generate an SSH configuration with all hosts in the inventory?

**TL;DR** My full solution to this problem is an Ansible playbook in this repository [https://github.com/dmrub/ansible-ssh-scripts-creator](https://github.com/dmrub/ansible-ssh-scripts-creator).

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

## Writing to a file

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

By using the hostvars variable we can access variables that are stored on other hosts with the `set_fact` module (see [here](https://docs.ansible.com/ansible/latest/reference_appendices/special_variables.html)).
The `loop` keyword allows us to iterate over a list of elements, in our case the expression `groups['all']`, which is the list of all hosts defined in the inventory. For each iteration, the variable `item` is set to the element of the list. Thus, the task `Print vars` prints evaluated variables for all hosts,
but is executed on a local host.

But what we need is to create an ssh configuration file and not just print values to stdout. For this task we can use an Ansible
[`copy` module](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/copy_module.html).
Although it is primarily intended for copying files, it has a content field that allows to embed the file content directly into the playbook.
And since Jinja templates are evaluated for all values, we can easily output our variables into the file:

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
  tasks:

    - name: Create output.txt file
      copy:
        content: |
          {% for item in target %}

          Host {{ item }}
          ansible_ssh_common_args: {{ hostvars[item]['eval_ansible_ssh_common_args'] | default('UNDEFINED') }}
          ansible_connection: {{ hostvars[item]['eval_ansible_connection'] | default('UNDEFINED') }}
          ansible_ssh_private_key_file: {{ hostvars[item]['eval_ansible_ssh_private_key_file'] | default('UNDEFINED') }}
          ansible_host: {{ hostvars[item]['eval_ansible_host'] | default('UNDEFINED')}}
          ansible_user: {{ hostvars[item]['eval_ansible_user'] | default('UNDEFINED')}}
          ansible_port: {{ hostvars[item]['eval_ansible_port'] | default('UNDEFINED') }}
          {% endfor %}
        dest: "output.txt"
        mode: 0600
{% endraw %}
{% endhighlight %}

Here we use a different way of iteration than in the previous example. We cannot use a `loop` because it would execute the module as many times as we have hosts, but we want to execute it only once. But in the generated text, i.e. in the template, we want to iterate over all hosts and use the variables for each host.
To achieve this, we use the jinjas [`for loop`](https://jinja.palletsprojects.com/en/2.11.x/templates/#for)
using the same list as in the previous example.

Let's run our playbook with the following inventory and see what we get in the file `output.txt`:

{% highlight yaml %}
{% raw %}
---
all:
  children:
    my_group:
      hosts:
        my_host_1:
          ansible_ssh_common_args: '{{ hostvars[node_ref].ansible_ssh_common_args | default(omit) }}'
        my_host_2:
          ansible_ssh_common_args: >-
            -o ProxyCommand='ssh -o UserKnownHostsFile=/dev/null
            -o StrictHostKeyChecking=no -W %h:%p user@example.org -p 2222
            -i my_id_rsa '
      vars:
        node_ref: my_host_2
        ansible_user: my_user
        ansible_ssh_private_key_file: my_id_rsa
{% endraw %}
{% endhighlight %}

If we run our playbook with this inventory, we get the following text in the `output.txt`:
```
Host my_host_1
ansible_ssh_common_args: -o ProxyCommand='ssh -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no -W %h:%p user@example.org -p 2222 -i my_id_rsa '
ansible_connection: ssh
ansible_ssh_private_key_file: my_id_rsa
ansible_host: my_host_1
ansible_user: my_user
ansible_port: UNDEFINED

Host my_host_2
ansible_ssh_common_args: -o ProxyCommand='ssh -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no -W %h:%p user@example.org -p 2222 -i my_id_rsa '
ansible_connection: ssh
ansible_ssh_private_key_file: my_id_rsa
ansible_host: my_host_2
ansible_user: my_user
ansible_port: UNDEFINED
```

## Generating the SSH configuration

Now the next step is to use the SSH configuration file syntax, but this is where the next problem comes up: There is no option in SSH configuration file to define command line options. That means we have to convert the [SSH command line options](https://man7.org/linux/man-pages/man1/ssh.1.html) to SSH configuration file syntax.
For example, the `-i identity_file` option to specify the private key for public key authentication should be converted to the `IdentityFile identity_file` statement in the configuration file. Doing this task with the Ansible alone would simply be too complicated, so I decided to use a simple Python script that accepts all command line arguments of the ssh command and outputs the corresponding SSH configuration statements as standard output: [ssh-args-to-config.py](https://github.com/dmrub/ansible-ssh-scripts-creator/blob/main/ssh-args-to-config.py).

To execute a Python script we use [script module](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/script_module.html):

{% highlight yaml %}
{% raw %}
- hosts: 127.0.0.1
  connection: local
  gather_facts: false
  vars:
    dest_dir: "."
    target: "{{ groups['all'] }}"
    ansible_python_interpreter: "{{ansible_playbook_python}}"
  tasks:

    - name: Run python script with ansible interpreter
      script: >-
        {{ playbook_dir }}/ssh-args-to-config.py
        --dest-dir {{ dest_dir | quote }}
        {%if hostvars[item]['eval_ansible_ssh_common_args'] is defined %}
        {{ hostvars[item]['eval_ansible_ssh_common_args'] }}
        {% endif %}
        {%if hostvars[item]['eval_ansible_ssh_private_key_file'] is defined %}
        -i {{ hostvars[item]['eval_ansible_ssh_private_key_file'] | quote }}
        {% endif %}
      args:
        executable: "{{ansible_python_interpreter}}"
      register: ssh_config_r
      loop: "{{ target }}"
      ignore_errors: true


    - name: Debug: Print ssh_config_r variable
      debug:
        var: ssh_config_r
{% endraw %}
{% endhighlight %}

We set the variable `ansible_python_interpreter` to the value of `ansible_playbook_python` to use the same interpreter
for the script that was used to execute the playbook
(see [here](https://docs.ansible.com/ansible/latest/inventory/implicit_localhost.html)).
The both {%raw%}`{%if ... is defined ... %} ... {% endif %}` {%endraw%}
template expressions use  test to output value of the variables only
if they are defined.
The both template expressions use the [`is defined`](https://jinja.palletsprojects.com/en/2.11.x/templates/#tests) test to output the value
of the variable only if it is defined.
Since `eval_ansible_ssh_common_args` contains several command line arguments, we do not quote this variable when passing it to the script,
but `eval_ansible_ssh_private_key_file` is a file name and a single parameter that can contain spaces, so we apply a quote filter to it.
Since SSH options use file names, we provide the variable `dest_dir` as the directory where the configuration file should be created and
to resolve all paths relative to it.
We execute the script for all hosts specified in the `target` variable using the `loop` keyword and store the results in
the variable `ssh_config_r`. If errors occur during execution, we ignore them with [`ignore_errors: true`](https://docs.ansible.com/ansible/latest/user_guide/playbooks_error_handling.html#ignoring-failed-commands) statement.

If we run playbook with the above mentioned inventory, the last debug task should be output afterwards:

```
ok: [127.0.0.1] => {
    "ssh_config_r": {
        "changed": true,
        "msg": "All items completed",
        "results": [
            {
                "ansible_loop_var": "item",
                "changed": true,
                "failed": false,
                "item": "my_host_1",
                "rc": 0,
                "stderr": "",
                "stderr_lines": [],
                "stdout": "ProxyCommand=ssh -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no -W %h:%p user@example.org -p 2222 -i my_id_rsa \nIdentityFile /home/rubinste/Kubernetes/ClusterManager/ansible-test/my_id_rsa\n",
                "stdout_lines": [
                    "ProxyCommand=ssh -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no -W %h:%p user@example.org -p 2222 -i my_id_rsa ",
                    "IdentityFile /home/rubinste/Kubernetes/ClusterManager/ansible-test/my_id_rsa"
                ]
            },
            {
                "ansible_loop_var": "item",
                "changed": true,
                "failed": false,
                "item": "my_host_2",
                "rc": 0,
                "stderr": "",
                "stderr_lines": [],
                "stdout": "ProxyCommand=ssh -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no -W %h:%p user@example.org -p 2222 -i my_id_rsa \nIdentityFile /home/rubinste/Kubernetes/ClusterManager/ansible-test/my_id_rsa\n",
                "stdout_lines": [
                    "ProxyCommand=ssh -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no -W %h:%p user@example.org -p 2222 -i my_id_rsa ",
                    "IdentityFile /home/rubinste/Kubernetes/ClusterManager/ansible-test/my_id_rsa"
                ]
            }
        ]
    }
}
```

## Reorganization of data

The next complication is that we iterate over a list of hostnames when generating our output file,
but the results of the script are organized as a `results` list in the variable `ssh_config_r` and are indexable by a numeric index.
To keep the file generation simple, we should reorganize the data so that we can keep our iteration.
We just have to create a new variable with the module `set_fact` that maps the hostname to the data,
in our case stdout, that we have to create:

{% highlight yaml %}
{% raw %}
    - name: Populate ssh config
      set_fact:
        ssh_config: >-
          {{ ssh_config | default({}) |
             combine( {item.item:
                         item.stdout if item.rc == 0 else
                         '# Script could not generate configuration for host %s, check ansible_ssh_common_args variable' | format(item.item)
                      }
                    )
          }}
      loop: "{{ ssh_config_r.results }}"
{% endraw %}
{% endhighlight %}

In the task `Populate ssh config` we create `ssh_config` dictionary by adding mappings from each hostname (`item.item`) to stdout (`item.stdout`) if the script for the host was executed successfully (`item.rc == 0`). If the script was not successful, we add an error message as comment `#`.
The expression `ssh_config | default({})` creates an empty dictionary in the first step because the variable ssh_config is not defined at
the beginning of the loop via the result list `ssh_config_r.results`.

## Output of ssh-config file

Now we are finally able to create an SSH configuration file:

{% highlight yaml %}
{% raw %}
    - name: Create ssh config file in ssh-config
      copy:
        content: |
          {% for host in target %}
          {% if hostvars[host]['eval_ansible_connection'] | default('ssh', true) in ['ssh', 'network_cli'] %}
          {% set ansible_host = hostvars[host]['eval_ansible_host'] %}
          {% set ansible_user = hostvars[host]['eval_ansible_user'] %}
          {% set ansible_port = hostvars[host]['eval_ansible_port'] | default(22, true) %}

          Host {{ host }}
            HostName {{ ansible_host }}
            User {{ ansible_user }}
            Port {{ ansible_port }}
            UserKnownHostsFile /dev/null
            StrictHostKeyChecking no
            PasswordAuthentication yes
            {# Note: IdentityFile option is output by ssh-args-to-config.py script #}
            {% if ssh_config[host] is defined %}

            # Options extracted from ssh command line arguments :
            {{ ssh_config[host] | indent(width=2) }}
            {% endif %}
            {% endif %}
          {% endfor %}
        dest: "{{ dest_dir }}/ssh-config"
        mode: 0600
{% endraw %}
{% endhighlight %}

We use the `set` statement to reduce the writing effort.  Since `ansible_port` can be undefined, we use 22 as default value.
The [`indent` filter](https://jinja.palletsprojects.com/en/2.11.x/templates/#indent) is used for a nicer formatting of the output.

*TO BE CONTINUED ...*
