---
layout: post
title: "Command Line Backups"
author: robweber
categories: coding automation
tags: python cli linux
---

In most cases there is no reason to re-invent the wheel, and there are already tons of great backup solutions. [Just Google it.](https://www.google.com/search?q=open+source+linux+backup). These run from full bare metal backups to file based backups that include rotating incrementals. Most include awesome client/server architecture or some type of web based GUI.

As seems to often be the case with my projects, I didn't need any of this. I wanted a solution that on the surface was so simple it was the kind of thing others flew past on their way to a real solution. Something simple enough it wasn't worth maintaining a project for it but yet complex enough that it wasn't just boilerplate code.

<!--more-->

To skip the methodology and jump straight to the code go to the [links area](#links) and view the Gist.

{% include toc %}

## The Problem

In addition to [using ghettoVCB](/automation/smarthome/random/home-lab-backups/); for years I've relied on a simple bash file to round up important files on different servers and send them off to Dropbox. This worked well for me and was simple to implement. Mostly this was to capture files that had minor changes day-to-day or collect things like SQLite database files and save them.

The downside to the bash script approach is that it's pretty low level. Each server needed it's own script and I had to duplicate a lot of code between scripts. It also had very little in the way of real error checking and I just sort of assumed things were working. The plus side was that it was easy. I really only need the most recent version of a handful of files. The full VM backup covers full system failure but occasionally one particular app would give me trouble and it was nice to have the latest config available quickly.

### Existing Solutions

I started looking at some existing solutions. All of these were great in their own way. After digging into a few of them though I couldn't _quite_ find the list of requirements I was looking for.

1. Linux CLI driven - didn't need a gui
2. No database or external system needed - no special client/server setup, rsync server, etc.
3. Don't need incremental or advanced backup management
4. No special archive - a zip or TAR is ok but no special formats
5. Hooks for pre and post backup operations - this is where stuff like MySQL dumps could happen
6. One central codebase with config files per server
7. Can push to remote storage - SMB, NFS, etc
8. Some record of last run status - either a log file, notification, or web hook

In a nutshell what I needed some kind of a wrapper around basic Linux commands like `tar` and `mv` but also needed it to be flexible enough I wasn't copying/pasting a lot of boilerplate code for each server.

## The Solution

I decided to lean on Python and YAML. This seemed like a pretty good idea as it would be easy to prototype, give me access to low level commands via [subprocess][subprocess], and allow a simple configuration file format. Using the above list of requirements I was able to knock off 1-4 plus 6 just with this decision alone. The other requirements could be done in the software.

As far as the backup process itself this was easily done using the `tar` command. This accepts a list of files on the system. As long as you have read access to them you can bundle them in an archive for easy storage. On the command line it would look like:

```bash
tar czf /path/to/archive.tar.gz file1 file2 folder1
```

The `c` option creates the archive, the `z` option compresses it with gzip and the `f` option is where you specify the file. More info on `tar` specifically can be [found here][tar].

Linking the `tar` command with some Python `subprocess` code and a simple YAML file the basics of a backup system could be done in just a few lines. There is a lot that can be improved here but a working version was pretty easy to get going.

__YAML__
```yaml
archive_name: name_of_archive
destination: /backups
files:
  - /path/to/file1.txt
  - /path/to/file2.tx
  - /path/to/folder
```

__Python__
```python
import os.path
import subprocess
import yaml

def run_process(program, args):
    """
    Kicks off a subprocess to run the defined program
    with the given arguments. Returns subprocess output.
    """
    command = program + args
    # run process, pipe all output
    output = subprocess.run(command, encoding="utf-8", stdout=subprocess.PIPE, stderr=subprocess.PIPE)

    return output

file = '/path/to/config.yaml'
with open(file, 'r') as f:
  config_file = yaml.safe_load(f)

archive_name = f"{config_file['archive_name']}.tar.gz"

run_process(['tar', 'czf', os.path.join(config_file['destination'], archive_name)], config_file['files'])

print("Complete!")
```

### Pre and Post Hooks

One of the requirements (#5) was that there be a mechanism for pre and post backup hooks. This was so that other commands can be run to gather files or do things like MySQL database dumps prior to the tar file being created. I very basic way to do this would be to lean on the subprocess function above and just pass in the path to a pre and post bash file. This would work but violated requirement 6 - I wanted one config file per server without extra scripts and other nonsense.

After some thought I decided to add a bit of complexity using [Jinja][jinja]. Jinja is a templating engine that allows you to process a string with placeholders. It's used in a lot of other projects (like Jekyll which runs this blog) to generate stuff like HTML. In this case I wanted to use it to process variables from my YAML config file and have them expanded during the execution of the program. Consider the following:

{% raw %}
```yaml
config:
  mysql_username: username
  mysql_password: password
pre_backup_hook: "mysqldump -U {{ mysql_username }} -p{{ mysql_password }} mysql_database > /path/to/mysql_database.sql"
```
{%endraw %}

The above defines two items, the `mysql_username` and the `mysql_password` and puts them as placeholders in the `mysqldump` command string. Using Jinja, at runtime, this can be expanded to the full command and passed along to the subprocess system for execution. Adding this type of expansion at runtime was a powerful component to keeping a unified configuration syntax across servers.

To make sure I could define commands or configuration variables a single time across multiple devices I also added support for the `!include` syntax. This is something lots of YAML implementations allow for but isn't part of `pyyaml` by default. It allows you to load the contents of another YAML file at runtime from whithin another at runtime. Adding it is pretty simple.



```yaml
config: !include config_file.yaml
pre_backup_hook: "......."
```

```python
import os.path
import yaml

DIR_PATH = os.path.dirname(os.path.realpath(__file__))

def custom_yaml_loader(loader, node):
    """loads another yaml file in the same directory as this one using !include syntax"""
    yaml_file = loader.construct_scalar(node)
    return read_yaml(os.path.join(DIR_PATH, yaml_file))

def read_yaml(file):
   """reads a yaml file from the given full path"""
    result = {}

    try:
        with open(file, 'r') as f:
            result = yaml.safe_load(f)
    except Exception:
        print(f"Error parsing YAML file {file}")

# add custom processor for external loading
yaml.add_constructor('!include', custom_yaml_loader, Loader=yaml.SafeLoader)
```

## Final Result

To round out the requirements list I had to add a few more bells and whistles but these were pretty easy. For sending to storage (requirement 7) I could utilize the same syntax as I did for the pre and post backup hooks. Simply define a command and pass in the right arguments. In my case I used `smbclient` to send the archive file to my home NAS, which is also linked to my Dropbox account.

For the final requirement (8), a record of the status I got a little lazy. Rather than implement a full notification system I decided that simply creating a file at the end of the script was good enough. This file would include the date/time of the last successful run. Since the program will exit without updating this file on any error it's a quick way to check that it was completed. I can use Home Assistant, or some other external checker, to check the contents or timestamp on the file at a later date.

The final result clocked in at under 200 lines of code and can quickly archive files on the system - exactly what I wanted. For full disclosure I did make a private Git project with the contents so I can quickly clone it to machines I want to back up. The full Python file with some example configuration files are found in a [GitHub Gist][full_gist], rather than dumping it all here. While definitely not as feature rich as a real backup solution this was the quick and dirty solution I needed.

## Links

* [Full Code][full_gist] - complete backup script with config file examples
* [Python subprocess module][subprocess] - launching system processes in Python
* [Jinja][jinja] - Python templating engine
* [TAR Man Page][tar] - command line archive creation

[full_gist]: https://gist.github.com/robweber/fe1d908c918ec1eaa6e08d43c50ce0d5
[subprocess]: https://docs.python.org/3/library/subprocess.html
[jinja]: https://jinja.palletsprojects.com/en/3.1.x/
[tar]: https://man7.org/linux/man-pages/man1/tar.1.html
