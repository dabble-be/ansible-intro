# Ansible intro

> When stopping the exercise (or extended pauses), consider removing (Cloud) resources that might incur costs

- [Exercise Source code](https://github.com/dabble-be/ansible-intro)
- [Documentation](https://docs.ansible.com/ansible/latest/)

![Ansible-logo](.assets/ansible/Ansible_logo.svg)

### System Requirements

This exercise has been constructed using the following software versions. These are not hard requirements.

| Software | Version | Description |
|----------|---------|-------------|
| [BASH](https://www.gnu.org/software/bash/) | `5.0.17(1)-release` | Bash is the GNU Project's shell |
| [Ansible](https://docs.ansible.com/?extIdCarryOver=true&sc_cid=701f2000001OH7YAAW)  | `2.9.6`   | Configuration Management and orchestration tool. Lightweight, powerful, portable |
| [jq](https://stedolan.github.io/jq/) | `jq-1.6` | Command line JSON processor |
| [Caddy](https://caddyserver.com/) | `v2.4.6` | Caddy 2 is a powerful, enterprise-ready, open source web server with automatic HTTPS written in Go |
| [tee](https://www.gnu.org/software/coreutils/manual/html_node/Introduction.html) | `8.30` | The tee command copies standard input to standard output and also to any files given as arguments |

## 1. Intro

"Ansible intro" is a quick guided tour into using Configuration Management to provision a single machine. You will use common Ansible `modules` to achieve some simple tasks.

By the end of this exercise, you'll have used Ansible to download, install and run a web server called [`Caddy`](https://caddyserver.com/). This web server will display a custom HTML page.

## 2. Introduction: `Ansible`

### a. Creating an `Ansible` inventory

`Ansible` is a simple configuration management and orchestration system.

We are going to execute our exercises against `localhost`, specifying it explicitly when running `Ansible`. Like:

```sh
ansible localhost <further instructions>
```

> To identify the machines we want to operate on, `Ansible` expects us to create an "inventory". As we are only working on 1 machine, we won't be using an inventory in this exercise.

### b. Creating an `Ansible` playbook

To express demands for our infrastructure's configuration settings, `Ansible` allows us to create a `YAML` file called a: `playbook`.

Let's start off with a simple demand for the instance we made in the previous chapter. We'll start off our `playbook.yml` file like so:

```sh
cat <<EOF | tee playbook.yml
- hosts: localhost
  tasks:

EOF
```

This first part acts as a filter; The task list will be applied to all hosts in the `task`s `hosts` group (eg: ` localhost` in the example above).

```sh
cat <<EOF | tee --append playbook.yml
    - name: create a marker in tmp filesystem
      command: "touch /tmp/a_rockstar_was_here"
EOF
```

Second part contains a single task. The task is named `"create a marker in tmp filesystem"` and it uses the `command` module to execute `touch /tmp/a_rockstar_was_here` on the remote file system.

### c. Running the `playbook`

Let's see what happens when we execute the `playbook` now:

```sh
ansible-playbook playbook.yml -v
```

_output:_

```log
Using /etc/ansible/ansible.cfg as config file
[WARNING]: provided hosts list is empty, only localhost is available. Note that the implicit localhost does not match 'all'

PLAY [localhost] ******************************************************************************************************************

TASK [Gathering Facts] ************************************************************************************************************
ok: [localhost]

TASK [create a marker in tmp filesystem] ******************************************************************************************
[WARNING]: Consider using the file module with state=touch rather than running 'touch'.  If you need to use command because file
is insufficient you can add 'warn: false' to this command task or set 'command_warnings=False' in ansible.cfg to get rid of this
message.
changed: [localhost] => {"changed": true, "cmd": ["touch", "/tmp/a_rockstar_was_here"], "delta": "0:00:00.002655", "end": "2021-11-23 14:05:06.428940", "rc": 0, "start": "2021-11-23 14:05:06.426285", "stderr": "", "stderr_lines": [], "stdout": "", "stdout_lines": []}

PLAY RECAP ************************************************************************************************************************
localhost                  : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

There we go. Sweet sweet provisioning. If we login to the instance, the file has been created.

### e. Idempotent

Let's run the `Ansible` `playbook` again, see what happens:

```sh
ansible-playbook playbook.yml -v
```

_output:_

```log
Using /etc/ansible/ansible.cfg as config file
[WARNING]: provided hosts list is empty, only localhost is available. Note that the implicit localhost does not match 'all'

PLAY [localhost] ******************************************************************************************************************

TASK [Gathering Facts] ************************************************************************************************************
ok: [localhost]

TASK [create a marker in tmp filesystem] ******************************************************************************************
[WARNING]: Consider using the file module with state=touch rather than running 'touch'.  If you need to use command because file
is insufficient you can add 'warn: false' to this command task or set 'command_warnings=False' in ansible.cfg to get rid of this
message.
changed: [localhost] => {"changed": true, "cmd": ["touch", "/tmp/a_rockstar_was_here"], "delta": "0:00:00.002757", "end": "2021-11-23 14:06:28.683052", "rc": 0, "start": "2021-11-23 14:06:28.680295", "stderr": "", "stderr_lines": [], "stdout": "", "stdout_lines": []}

PLAY RECAP ************************************************************************************************************************
localhost                  : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

I don't like it. The file has been _updated_, with new metadata no less. We should make our task's demands more specific by requiring it to only execute the command if the file doesn't already exist.

`Ansible`'s command module allows to define extra arguments you can toggle to configure its behavior. To create the file only if it is missing, perform the following:

```sh
cat <<EOF | tee --append playbook.yml
      args:
        creates: "/tmp/a_rockstar_was_here"
EOF
```

The `creates` option allows the command module to check if a file is already present and if so, skip executing the target command. Let's validate.

```sh
ansible-playbook playbook.yml -v
```

_output:_

```log
Using /etc/ansible/ansible.cfg as config file
[WARNING]: provided hosts list is empty, only localhost is available. Note that the implicit localhost does not match 'all'

PLAY [localhost] ******************************************************************************************************************

TASK [Gathering Facts] ************************************************************************************************************
ok: [localhost]

TASK [create a marker in tmp filesystem] ******************************************************************************************
ok: [localhost] => {"changed": false, "cmd": ["touch", "/tmp/a_rockstar_was_here"], "rc": 0, "stdout": "skipped, since /tmp/a_rockstar_was_here exists", "stdout_lines": ["skipped, since /tmp/a_rockstar_was_here exists"]}

PLAY RECAP ************************************************************************************************************************
localhost                  : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

As we can see in the output: `"msg": "Did not run command since '/tmp/a_rockstar_was_here' exists",`. The `PLAY RECAP` confirms that all tasks where OK, and no resources where changed.

### f. Installing Caddy

If we want to serve a page generated using `Ansible`, we'll need a web server. This example shall use `caddy`: A powerful open source web server.

We could use `apt update` and `apt install caddy` in this case, but lets assume we have to do this on many, many nodes and we don't have any help from pesky package managers!

Let's add another task to our `playbook`:

```sh
cat <<EOF | tee --append playbook.yml

  - name: Create required directory structure
    file:
      path: ~/.local/bin
      state: directory
EOF
```

As it's name implies: We are creating a directory structure to home our Caddy binary. Before we can start downloading the required files, we want a directory to place our service in.

The above code block uses a module. The module is called [`file`](https://docs.ansible.com/ansible/2.9/modules/list_of_files_modules.html). The file module allows us to create a file of `state:`: `directory`, at path: `$HOME/.local/bin`.

```
ansible-playbook playbook.yml -v
```

_output:_

```sh
Using /etc/ansible/ansible.cfg as config file
[WARNING]: provided hosts list is empty, only localhost is available. Note that the implicit localhost does not match 'all'

PLAY [localhost] ***************************************************************************************************************************************************

TASK [Gathering Facts] *********************************************************************************************************************************************
ok: [localhost]

TASK [create a marker in tmp filesystem] ***************************************************************************************************************************
ok: [localhost] => {"changed": false, "cmd": ["touch", "/tmp/a_rockstar_was_here"], "rc": 0, "stdout": "skipped, since /tmp/a_rockstar_was_here exists", "stdout_lin
es": ["skipped, since /tmp/a_rockstar_was_here exists"]}

TASK [Create required directory structure] *************************************************************************************************************************
ok: [localhost] => {"changed": false, "gid": 1000, "group": "dabbler", "mode": "0755", "owner": "dabbler", "path": "/home/dabbler/.local/bin", "size": 4096, "state"
: "directory", "uid": 1000}

PLAY RECAP *********************************************************************************************************************************************************
localhost                  : ok=3    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

Let's download a Caddy release into the `/tmp/` directory somewhere:

```sh
cat <<EOF | tee --append playbook.yml
  - name: Download caddy binary
    get_url:
      url: https://github.com/caddyserver/caddy/releases/download/v2.4.6/caddy_2.4.6_linux_amd64.tar.gz
      dest: /tmp/caddy.tar.gz
      mode: 0600
      checksum: sha256:690ad64538a39d555294cd09b26bb22ade36abc0e3212342f0ed151de51ec128
EOF
```

The [`get_url` module](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/get_url_module.html) is built into Ansible. It offers several (optional) useful arguments we can pass to optimize our downloads.

Parameters:
- `url`: HTTP, HTTPS, or FTP URL.
- `dest`: Absolute path of where to download the file to.
- `mode`: Permissions the resulting file system object should have.
- `checksum`: The digest of the destination file will be calculated after it is downloaded to ensure its integrity and verify that the transfer completed successfully.

```
ansible-playbook playbook.yml -v
```

_output:_

```sh
Using /etc/ansible/ansible.cfg as config file
[WARNING]: provided hosts list is empty, only localhost is available. Note that the implicit localhost does not match 'all'

PLAY [localhost] ***************************************************************************************************************************************************

TASK [Gathering Facts] *********************************************************************************************************************************************
ok: [localhost]

TASK [create a marker in tmp filesystem] ***************************************************************************************************************************
ok: [localhost] => {"changed": false, "cmd": ["touch", "/tmp/a_rockstar_was_here"], "rc": 0, "stdout": "skipped, since /tmp/a_rockstar_was_here exists", "stdout_lin
es": ["skipped, since /tmp/a_rockstar_was_here exists"]}

TASK [Create required directory structure] *************************************************************************************************************************
ok: [localhost] => {"changed": false, "gid": 1000, "group": "dabbler", "mode": "0755", "owner": "dabbler", "path": "/home/dabbler/.local/bin", "size": 4096, "state"
: "directory", "uid": 1000}

TASK [Download caddy binary] ***************************************************************************************************************************************
changed: [localhost] => {"changed": true, "checksum_dest": null, "checksum_src": "a3d300f7191ed0be0ce16be237f2c4da83932585", "dest": "/tmp/caddy.tar.gz", "elapsed":
 1, "gid": 1000, "group": "dabbler", "md5sum": "8d26c237b3ae582478d599d0acd8c710", "mode": "0600", "msg": "OK (11697750 bytes)", "owner": "dabbler", "size": 1169775
0, "src": "/home/dabbler/.ansible/tmp/ansible-tmp-1650272992.235394-196237038360445/tmp02vru33k", "state": "file", "status_code": 200, "uid": 1000, "url": "https://
github.com/caddyserver/caddy/releases/download/v2.4.6/caddy_2.4.6_linux_amd64.tar.gz"}

PLAY RECAP *********************************************************************************************************************************************************
localhost                  : ok=4    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

Nice. Now that we've successfully downloaded the required file, we can move on to extracting the archive onto our file system:

```sh
cat <<EOF | tee --append playbook.yml
  - name: Extract archive
    unarchive:
      src: /tmp/caddy.tar.gz
      dest: ~/.local/bin/
EOF
```

The [`unarchive module`](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/unarchive_module.html) helps us extracting archives. In this case, we are extracting all files in the archive at `src` to `dest` `$HOME/.local/bin/`.

```
ansible-playbook playbook.yml -v
```

_output:_

```sh
Using /etc/ansible/ansible.cfg as config file
[WARNING]: provided hosts list is empty, only localhost is available. Note that the implicit localhost does not match 'all'

PLAY [localhost] ***************************************************************************************************************************************************

TASK [Gathering Facts] *********************************************************************************************************************************************
ok: [localhost]

TASK [create a marker in tmp filesystem] ***************************************************************************************************************************
ok: [localhost] => {"changed": false, "cmd": ["touch", "/tmp/a_rockstar_was_here"], "rc": 0, "stdout": "skipped, since /tmp/a_rockstar_was_here exists", "stdout_lin
es": ["skipped, since /tmp/a_rockstar_was_here exists"]}

TASK [Create required directory structure] *************************************************************************************************************************
ok: [localhost] => {"changed": false, "gid": 1000, "group": "dabbler", "mode": "0755", "owner": "dabbler", "path": "/home/dabbler/.local/bin", "size": 4096, "state"
: "directory", "uid": 1000}

TASK [Download caddy binary] ***************************************************************************************************************************************
ok: [localhost] => {"changed": false, "checksum_dest": null, "checksum_src": null, "dest": "/tmp/caddy.tar.gz", "elapsed": 0, "gid": 1000, "group": "dabbler", "mode
": "0600", "msg": "file already exists", "owner": "dabbler", "size": 11697750, "state": "file", "uid": 1000, "url": "https://github.com/caddyserver/caddy/releases/d
ownload/v2.4.6/caddy_2.4.6_linux_amd64.tar.gz"}

TASK [Extract archive] *********************************************************************************************************************************************
changed: [localhost] => {"changed": true, "dest": "/home/dabbler/.local/bin/", "extract_results": {"cmd": ["/usr/bin/tar", "--extract", "-C", "/home/dabbler/.local/
bin/", "-z", "-f", "/home/dabbler/.ansible/tmp/ansible-tmp-1650273263.6491778-244802995360804/source"], "err": "", "out": "", "rc": 0}, "gid": 1000, "group": "dabbl
er", "handler": "TgzArchive", "mode": "0755", "owner": "dabbler", "size": 4096, "src": "/home/dabbler/.ansible/tmp/ansible-tmp-1650273263.6491778-244802995360804/so
urce", "state": "directory", "uid": 1000}

PLAY RECAP *********************************************************************************************************************************************************
localhost                  : ok=5    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

Lastly, we want to test the binary, to see if it is functioning properly:

```sh

cat <<EOF | tee --append playbook.yml
  - name: Run version command
    command:
      cmd: ~/.local/bin/caddy version
EOF
```

```
ansible-playbook playbook.yml -v
```

_output:_

```sh
Using /etc/ansible/ansible.cfg as config file
[WARNING]: provided hosts list is empty, only localhost is available. Note that the implicit localhost does not match 'all'

PLAY [localhost] ***************************************************************************************************************************************************

TASK [Gathering Facts] *********************************************************************************************************************************************
ok: [localhost]

TASK [create a marker in tmp filesystem] ***************************************************************************************************************************
ok: [localhost] => {"changed": false, "cmd": ["touch", "/tmp/a_rockstar_was_here"], "rc": 0, "stdout": "skipped, since /tmp/a_rockstar_was_here exists", "stdout_lin
es": ["skipped, since /tmp/a_rockstar_was_here exists"]}

TASK [Create required directory structure] *************************************************************************************************************************
ok: [localhost] => {"changed": false, "gid": 1000, "group": "dabbler", "mode": "0755", "owner": "dabbler", "path": "/home/dabbler/.local/bin", "size": 4096, "state"
: "directory", "uid": 1000}

TASK [Download caddy binary] ***************************************************************************************************************************************
ok: [localhost] => {"changed": false, "checksum_dest": null, "checksum_src": null, "dest": "/tmp/caddy.tar.gz", "elapsed": 0, "gid": 1000, "group": "dabbler", "mode
": "0600", "msg": "file already exists", "owner": "dabbler", "size": 11697750, "state": "file", "uid": 1000, "url": "https://github.com/caddyserver/caddy/releases/d
ownload/v2.4.6/caddy_2.4.6_linux_amd64.tar.gz"}

TASK [Extract archive] *********************************************************************************************************************************************
ok: [localhost] => {"changed": false, "dest": "/home/dabbler/.local/bin/", "gid": 1000, "group": "dabbler", "handler": "TgzArchive", "mode": "0755", "owner": "dabbl
er", "size": 4096, "src": "/home/dabbler/.ansible/tmp/ansible-tmp-1650273454.549988-134392872419476/source", "state": "directory", "uid": 1000}

TASK [Run version command] *****************************************************************************************************************************************
changed: [localhost] => {"changed": true, "cmd": ["~/.local/bin/caddy", "version"], "delta": "0:00:00.023996", "end": "2022-04-18 09:17:36.309183", "rc": 0, "start"
: "2022-04-18 09:17:36.285187", "stderr": "", "stderr_lines": [], "stdout": "v2.4.6 h1:HGkGICFGvyrodcqOOclHKfvJC0qTU7vny/7FhYp9hNw=", "stdout_lines": ["v2.4.6 h1:HG
kGICFGvyrodcqOOclHKfvJC0qTU7vny/7FhYp9hNw="]}

PLAY RECAP *********************************************************************************************************************************************************
localhost                  : ok=6    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

Seems to be working just fine as is; However, when we execute the same playbook again we run into the following issue:

```
ansible-playbook playbook.yml -v
```

_output:_

```sh
[...]
PLAY RECAP *********************************************************************************************************************************************************
localhost                  : ok=6    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

In our play recap, we can see that the last operation we added is always "changed". When we do run our several nodes, having many of these "false positives" would make us blind to actual changes.
Let's add some conditionals that allow us to control when this resource is actually run. Apply the following patch:

```
cat <<EOF | tee register.patch
11a12
>     register: caddy
17a19
>     register: caddy
21a24
>     register: caddy
24a28
>     when: caddy.changed
EOF
```

```
patch playbook.yml < register.patch
```

_output:_

```sh
patching file playbook.yml
```

These changes register several of our jobs using the `register` keyword.
This names (or groups) tasks. In this case, we are giving each resource related to the Caddy install the name: `"caddy"`.

In our last resource we add a condition:

```yml
  - name: Run version command
    command:
      cmd: ~/.local/bin/caddy version
    when: caddy.changed
```

`    when: caddy.changed` Tells Ansible that whenever the task(s) registered under name `"caddy"` are `.changed`, this resource needs to be triggered.

```
ansible-playbook playbook.yml -v
```

_output:_

```sh
Using /etc/ansible/ansible.cfg as config file
[WARNING]: provided hosts list is empty, only localhost is available. Note that the implicit localhost does not match 'all'

PLAY [localhost] ***************************************************************************************************************************************************

TASK [Gathering Facts] *********************************************************************************************************************************************
ok: [localhost]

TASK [create a marker in tmp filesystem] ***************************************************************************************************************************
ok: [localhost] => {"changed": false, "cmd": ["touch", "/tmp/a_rockstar_was_here"], "rc": 0, "stdout": "skipped, since /tmp/a_rockstar_was_here exists", "stdout_lin
es": ["skipped, since /tmp/a_rockstar_was_here exists"]}

TASK [Create required directory structure] *************************************************************************************************************************
ok: [localhost] => {"changed": false, "gid": 1000, "group": "dabbler", "mode": "0755", "owner": "dabbler", "path": "/home/dabbler/.local/bin", "size": 4096, "state"
: "directory", "uid": 1000}

TASK [Download caddy binary] ***************************************************************************************************************************************
ok: [localhost] => {"changed": false, "checksum_dest": null, "checksum_src": null, "dest": "/tmp/caddy.tar.gz", "elapsed": 0, "gid": 1000, "group": "dabbler", "mode
": "0600", "msg": "file already exists", "owner": "dabbler", "size": 11697750, "state": "file", "uid": 1000, "url": "https://github.com/caddyserver/caddy/releases/d
ownload/v2.4.6/caddy_2.4.6_linux_amd64.tar.gz"}

TASK [Extract archive] *********************************************************************************************************************************************
ok: [localhost] => {"changed": false, "dest": "/home/dabbler/.local/bin/", "gid": 1000, "group": "dabbler", "handler": "TgzArchive", "mode": "0755", "owner": "dabbl
er", "size": 4096, "src": "/home/dabbler/.ansible/tmp/ansible-tmp-1650274416.5801642-170224085336612/source", "state": "directory", "uid": 1000}

TASK [Run version command] *****************************************************************************************************************************************
skipping: [localhost] => {"changed": false, "skip_reason": "Conditional result was False"}

PLAY RECAP *********************************************************************************************************************************************************
localhost                  : ok=5    changed=0    unreachable=0    failed=0    skipped=1    rescued=0    ignored=0
```

Now our resource gets `skipped` instead of `changed`: That's more transparent.

### g. Using the template module

Last chapter, we downloaded the requirements to run the Caddy web server. Let's create some content for it to serve using Ansible; To test our skills.

Before we can add a custom file we need to make sure target path exists:

```sh
cat <<EOF | tee --append playbook.yml

    - name: Ensures ~/.local/html/shell-http/{{ name }} dir exists
      file:
        path=~/.local/html/shell-http/{{ name }}
        state=directory
EOF
```

This task uses the `file`-module to make sure that the directory we specified exists.

Time to customize the default landing page. We can do this by adding yet another task:

```sh
cat <<EOF | tee --append playbook.yml

    - name: customize greeting
      template:
        src: caddy-greeting.html
        dest: ~/.local/html/shell-http/{{ name }}/index.html
        mode: 0644
EOF
```

Taking a closer look at the task we added here, we are, once more, using an administrator account.

The `template` module has the following arguments:

- `src`: Set to `caddy-greeting.html`. `Ansible` will look for a source _template_ file. Behind the scenes, `Ansible` will use a library called [`jinja`](https://jinja.palletsprojects.com/en/3.0.x/templates/) to parse the given source file; Like it does for both its inventory files and `playbook`s.
- `dest`: Set to `/var/www/html/shell-http/{{ name }}/index.html`. This is the target file to create using the template file. Notice the `{{ name }}` in the destination parameter. It is a variable. (more on variables later)
   `mode`: Set permissions for the target file (octal notation)

Let us generate an example template landing page. Behold:

```sh
cat <<EOF | tee caddy-greeting.html
<html>
	<head>
		<title>Learn Ansible</title>
	</head>
	<body>
		<div id="home">
			<h1>Home</h1>
			<p>This is the default landing page, for more details, contact: <span class="h2">{{ name }}</span>!</p>
		</section>
	</body>
</html>
EOF
```

Will create the `caddy-greeting.html` file; With rudimentary contents and (again) something eye catching: `{{ name }}`. This is a placeholder for the `name` variable that will be set when generating the template.

Now all we need is a way to set that `name` variable. Variables can be defined at all levels of your inventory and plays.

```sh
cat <<EOF | tee --append playbook.yml

  vars:
    name: $STUDENT_ID
EOF
```

Lastly, we want a simple configuration file for our web server. Generate the `Caddyfile` using the following command:

```sh
cat <<EOF | tee --append playbook

  - name: Configure Caddy
    copy:
      dest: ~/.local/html/Caddyfile
      content: |
              :8002 {
                        file_server browse
              }
EOF
```

Normally, if we'd want to add a file using `Ansible`, we'd prefer using the (builtin) `file`-module. In this case, we want to add content _inline_. This is not possible using the `file`-module. We could add a `blockinfile`-module to add content to a new file created by the `file`-module, but having just a single `copy`-module in this case seems more reasonable, it conveniently has a `content` parameter.

Now to configure our `localhost`:

```sh
ansible-playbook playbook.yml -v
```

_output:_

```log
<omitted>
```

You can run the Caddy server using the following command:

```sh
cd ~/.local/html/
~/.local/bin/caddy start -config Caddyfile
cd -
```

_output:_

Your output might look similar to:

```log
2022/04/22 13:39:12.474 INFO    using provided configuration    {"config_file": "Caddyfile", "config_adapter": ""}
2022/04/22 13:39:12.476 WARN    input is not formatted with 'caddy fmt' {"adapter": "caddyfile", "file": "Caddyfile", "line": 2}
2022/04/22 13:39:12.477 INFO    admin   admin endpoint started  {"address": "tcp/localhost:2019", "enforce_origin": false, "origins": ["localhost:2019", "[::1]:2019
", "127.0.0.1:2019"]}
2022/04/22 13:39:12.477 INFO    tls.cache.maintenance   started background certificate maintenance      {"cache": "0xc000472620"}
2022/04/22 13:39:12.478 INFO    tls     cleaning storage unit   {"description": "FileStorage:/home/dabbler/.local/share/caddy"}
2022/04/22 13:39:12.478 INFO    tls     finished cleaning storage units
2022/04/22 13:39:12.478 INFO    autosaved config (load with --resume flag)      {"file": "/home/dabbler/.config/caddy/autosave.json"}
2022/04/22 13:39:12.478 INFO    serving initial configuration
Successfully started Caddy (pid=8462) - Caddy is running in the background
[...]
```

To see this page in all it's glory, you can find your shell's public HTTP address for your `caddy`'s port by performing:

```sh
echo -e "$STUDENT_ADDRESSES"
```

_output:_

```log
This shells http port: https://lab.dabble.be/shell-http/demo/
Kubernetes http port: https://lab.dabble.be/http/
```

> The last line in the output is provided for the [Kubernetes exercise](https://github.com/dabble-be/kubernetes-intro).

## Conclusion

In this exercise, we've reaped the fruits of a configuration automation tool. This was only an introduction to the subject, there's a lot more to learn!

Check the references below for more information and documentation.

###### References

- [Exercise Source code](https://github.com/dabble-be/ansible-intro)
- [Official Ansible Documentation](https://docs.ansible.com/ansible/latest/)
- [Jinja Template Engine (used by Ansible)](https://jinja.palletsprojects.com/en/3.0.x/templates/)
- [Caddy Server](https://caddyserver.com/)

- [Asset attribution](.assets/README.md)
