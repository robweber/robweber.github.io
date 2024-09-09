---
layout: post
title:  "Updating Kodi Repository with Ansible"
author: robweber
categories: coding automation
tags: kodi cli python ansible
---

I'm not an [Ansible][ansible] guru but I do use it to automate some tasks in my home lab. It is a real time saver to issue commands from a single interface and not rely on a bunch of hacked together Bash scripts all over the place. Recently I was doing some updates to a custom [Kodi repository][kodi-addon-repo] that I use for testing addons in development. The process was pretty cumbersome and after a while I decided it was time to convert the whole process to an Ansible Playbook.

Below I'll go through the process of modifying a few scripts and setting up the Ansible playbook. This process can be used as a template for other system automations where a legacy script is being adapted for Ansible.

<!--more-->

{% include toc %}

## Overview

Before digging in to the whole process here is a high level overview of the pieces at play. [Kodi][kodi] is a media center platform that runs on a variety of hardware platforms. One of it's core features is the ability to install [community created addons](https://kodi.tv/addons/). These can be anything from UI improvements to streaming sources to PVR integrations. Kodi provides an official repository bundled with the software but it is possible to add additional [3rd party repositories][kodi-addon-repo] as well.

I run a local repository on my home network. This allows me to pull in unofficial addons, or test beta version of my own addons, on my home Kodi systems. The exact process for setting up a repository I won't detail here but it's essentially an HTTP server that hosts ZIP files of all the addons available. A special file, the `addons.xml` file must exist in the root directory of the repository and contain a manifest of all addons and their versions. Kodi reads in this file to present a list of downloadable content.

### Addon Python Script

While the structure and files for an addon repository can be created manually, it's a real pain. Luckily [Chad Perry](https://github.com/chadparry) has created and shared a helpful Python script to automate this process.

His script, [create_repository.py][create-script], takes in a list of addon sources and packages everything up in the format a Kodi repository requires. This includes downloading the addons, creating the zip folders, and generating an addons.xml file. It's pretty slick. I used this as the base for my Ansible playbook.

## Script Modifications

Normally I would run the script from the command line to generate the repository. This is pretty straightforward but did have some quirks. The command is essentially the local file path where you want the repo to live, followed by a list of addons to include.

```
python3 create_repository.py -datadir=/html/software/kodi https://github.com/chadparry/kodi-repository.chad.parry.org.git:repository.chad.parry.org
```

You can specify more than one addon and also include some additional information via a few delimiters. These are in the format `URL#BRANCH:PATH`. The branch identifier is if the addon is in a different branch than the main one, and the path identifier is if the addon doesn't live in the root directory of the git repository.

### Modification 1

In order to better use this with Ansible I decided passing in the list of addons via a text file would be easier than directly via the command line. This way I could keep the list separate from the playbook and update it as needed. This was fairly simple as I modified the argument list to take one entry, the filename, and then read in the file contents prior to running the rest of the script.

```
# read in the file, one addon per line
with open(args.addon) as f:
  addons = f.readlines()
```

### Modification 2

The second modification stemmed from an issue I discovered with repo URLs. If the Git URL contained a colon to designate a port number (like _http://localhost:8080/repo.git_) the expression parsing would get messed up. The script expected the colon to be used as a delimiter for the PATH part of the URL expression. I decided to change this delimiter to a pipe `|` instead as this wouldn't be part of a normal URL. This was pretty straightforward as well, just a modification to one line:

```
# original regex
match = re.match(r'((?:[A-Za-z0-9+.-]+://)?.*?)(?:#([^#]*?))?(?::([^:]*))?$', addon_location)
# new regex
match = re.match(r'((?:[A-Za-z0-9+.-]+://)?.*?)(?:#([^#]*?))?(?:\|([^\|]*))?$', addon_location)

```

## Converting to Ansible

With the Python script working as expected the next piece was writing the [Ansible playbook](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_intro.html). The playbook is a set of instructions executed on the host (server), or a group of hosts. For this playbook I leveraged the [ansible.builtin.script module][script-module]. This is a built in task to run a script and also specify arguments.

{% raw %}
```
# Updates the Kodi Addon Repo
- name: Update Kodi Repo
  hosts:  hostname
  gather_facts: true
  tasks:
    - name: Run Build Script
      ansible.builtin.script: "/home/path/to/create_repository.py --datadir /var/www/html/repository --addon {{ addons_list }}"
      args:
        executable: python3
```

The above will run the repository script on the host, using the variable `{{ addons_list }}` to pass in the name of the text file. This variable can be passed in at runtime, or specified in the playbook. Since the task will fail if the text file doesn't exist it's useful to add a few other checks to make sure the file is there, and also output some information so you know the whole thing worked.

```
- name: Update Kodi Repo
  hosts:  hostname
  gather_facts: true
  vars:
    addons_list: "/path/to/kodi_addons_list.txt"
  tasks:
    - name: Addons List File Exists
      ansible.builtin.stat:
        path: "{{ addons_list }}"
      register: addon_list_stat

    - name: Run Build Script
      ansible.builtin.script: "/home/rob/Git/home-scripts/create_repository.py --datadir /var/www/html/repository --addon {{ addons_list }}"
      args:
        executable: python3
      register: python_output
      when: addon_list_stat.stat.exists

    - name: Print Addon List
      ansible.builtin.debug:
        msg: "{{python_output.stdout}}"
```


This is my final version of the playbook. The first task checks that the `{{ addons_list }}` file path exists on the server utilzing the [ansible.builtin.stat][stat-module] command. If it does the Python script will run. The `when` piece of the script command acts as a true/false statement to only execute the command when the check returns true. Finally the output of the script is output so you can see which addons were updated. This is achived using the `register` keyword to save the output to a variable and then printing it in the final task.
{% endraw %}

## Final Thoughts

This is a very basic example of how Ansible can be leveraged as a wrapper around an existing command line process or script. Even in this simple example you get some quality of life improvements and error checking. Ansible has some other great built in functions that allow for pulling in Git repositories, installing binary files, and deploying application configs. I'm far from an Ansible expert but it's already helped my workflow when managing my home network and hobby projects. I can spend more time on the fun parts and less time in the update/deploy pipeline.

## Links

* [Ansible][ansible] - Open source provisioning, configuration management, and app deployment tool
* [Kodi][kodi] - Open source media center software.
* [Ansible Script Module][script-module] - built-in Ansible module for running a script
* [Ansible Stat Module][stat-module] - built-in Ansible module for getting file info
* Chad Perry [create_repository.py script][create-script] - Original Chad Perry script for creating a Kodi repository

[ansible]: https://www.ansible.com/
[script-module]: https://docs.ansible.com/ansible/latest/collections/ansible/builtin/script_module.html
[stat-module]: https://docs.ansible.com/ansible/latest/collections/ansible/builtin/stat_module.html
[kodi]: https://kodi.tv/
[kodi-addon-repo]: https://kodi.wiki/view/Add-on_repositories
[create-script]: https://github.com/chadparry/kodi-repository.chad.parry.org/blob/master/tools/create_repository.py
