# Ansible intro

[Overview](https://github.com/dabble-be/ansible-intro/tree/main)


*Note*
> When stopping (or extended pauses) the exercise, consider removing resources that might incur costs.

- [Exercise Source code](https://github.com/dabble-be/ansible-intro)
- [Documentation](https://docs.ansible.com/ansible/latest/)

![ansible-logo](.assets/ansible/Ansible_logo.svg)

### System Requirements

This exercise has been constructed using the following software versions. These are not hard requirements.

| Software | Version | Description |
|----------|---------|-------------|
| [BASH](https://www.gnu.org/software/bash/) | `5.0.17(1)-release` | Bash is the GNU Project's shell |
| [ansible](https://docs.ansible.com/?extIdCarryOver=true&sc_cid=701f2000001OH7YAAW)  | `2.9.6`   | Configuration Management and orchestration tool. Lightweight, powerful, portable |
| [jq](https://stedolan.github.io/jq/) | `jq-1.6` | Command line JSON processor |

### Example details

This example is run with specific details appearing many times. If in any example you are wondering what these values represent, you can refer to the following table:

| Variable | Value |
|----------|-------|
| Workspace | `$HOME/ansible-intro` |
| Inventory Address (ansible target) | `localhost` |

## 2. Introduction: `Ansible`

### a. Creating an `Ansible` inventory

`Ansible` is a simple configuration management and orchestration system. To identify the nodes we want to influence with tasks we'll build, `Ansible` expects us to create an "inventory". We won't be using an inventory in this exercise;

We are going to execute our exercises against `localhost`, specifying it explicitly when running `Ansible`. Like:

```sh
ansible localhost <further instructions>
```

### b. Creating an `Ansible` playbook

To express demands for our infrastructure's configuration settings, `Ansible` allows us to create a `YAML` file called a: `playbook`.

Let's start off with a simple demand for the instance we made in the previous chapter:

```sh
cat <<EOF | tee playbook.yml
- hosts: localhost
  tasks:

EOF
```

This first part acts as a filter; The task list will be applied to all hosts in the `task`s group (`aka: localhost`).

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

There we go. Sweet sweet provisioning. If we log into the instance, the file has been created.

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

I don't like it. The file has been updated, with new metadata no less. We should make our task's demands more specific by requiring it to only execute the command if the file doesn't already exist.

`Ansible`'s command module has extra arguments you can set to configure its behavior. To create the file only if it is missing, perform:

```sh
cat <<EOF | tee --append playbook.yml
      args:
        creates: "/tmp/a_rockstar_was_here"
EOF
```

The `creates` option allows the command module to check if a file is already present and if so, skip executing the target command. It looks like so:

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

### f. Installing NginX

If we want to serve a page generated using `Ansible`, we'll need a web server. This example shall use `nginx`: A powerful open source web server.

We could use `sudo apt update` and `sudo apt install nginx` in this case, but lets assume we have to do this on many, many nodes! To demonstrate the portability of `Ansible` Modules we'll be using the `package` module.
The module is aimed to be used transparently across multiple platforms and multiple package managers.

Let's add another task to our `playbook`:

```sh
cat <<EOF | tee --append playbook.yml

    - name: install nginx
      become: yes
      package:
      args:
        update_cache: yes
        name: nginx
        state: present
EOF
```

As it's name implies, we'll be installing NginX in this task. As we are connecting to our instance using a non-root-user, we'll want to become an administrator to execute the module's instructions. We do here: `      become: yes`, in this case, it will perform our `apt-get` commands preceded by `sudo`. Using this keyword, we don't have to run `ansible` using `sudo` or as user root.

The package module is fed several arguments in this case:

- `update_cache` asks the module to update package source lists before performing the next operations.
- `name` sets the name of the package to install. Can also be passed a list of strings for multiple packages.
- `state` indicates if we want the package to be `present` (yes!), `absent` or in specific cases `latest`.

```
ansible-playbook playbook.yml -v
```

_output:_

```sh
# Omitted (too long)
```

That went awesome. Start the `nginx` process (it will fork for you) and perform a request to the default `nginx` page:

```sh
sudo nginx
{
  echo "Waiting"
    for i in $( seq 3 )
    do
      sleep 1
      printf "."
    done
    echo;
}
curl -i http://localhost:80
```

_output:_

```log
Waiting
...
HTTP/1.1 200 OK
Server: nginx/1.18.0 (Ubuntu)
Date: Thu, 02 Dec 2021 14:16:42 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Thu, 02 Dec 2021 13:19:33 GMT
Connection: keep-alive
ETag: "61a8c7e5-264"
Accept-Ranges: bytes

<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

That's looking good so far; Our web service seems to be running and responding to requests.

### g. Using the template module to create dynamic content

Before we can add a custom file in the correct path (for the `lab.dabble.be` proxy) we need to make sure the path exists:

```sh
cat <<EOF | tee --append playbook.yml

    - name: Ensures /var/www/html/shell-http/{{ name }} dir exists
      become: yes
      file:
        path=/var/www/html/shell-http/{{ name }}
        state=directory
EOF
```

This task uses the file module to make sure that the directory we specified exists.


It's time to customize the default landing page. We can do this by adding yet another task.

```sh
cat <<EOF | tee --append playbook.yml

    - name: customize greeting
      become: yes
      template:
        src: nginx-greeting.html
        dest: /var/www/html/shell-http/{{ name }}/index.html
        owner: root
        group: root
        mode: 0644
EOF
```

Taking a closer look at the task we added here, we are, once more, using an administrator account.

The `template` module has the following arguments:

- `src`: Set to `nginx-greeting.html`. `Ansible` will look for a source _template_ file. Behind the scenes, `Ansible` will use a library called [`jinja`](https://jinja.palletsprojects.com/en/3.0.x/templates/) to parse the given source file; Like it does for both it's inventory files and `playbook`s.
- `dest`: Set to `/var/www/html/shell-http/{{ name }}/index.html`. This is the target file to create using the template file. Notice the `{{ name }}` template. We'll be add more details on that later; It is a variable.
- `owner`, `group`, `mode`: These values set ownership and permissions for the target file.

Let us generate an example template for our default landing page. Behold:

```sh
cat <<EOF | tee nginx-greeting.html
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

This will create the `nginx-greeting.html` file; With rudimentary contents and (again) something eye catching: `{{ name }}`. This is a placeholder for the `name` variable that will be set when generating the template.

Now all we need is a way to set that `name` variable. Variables can be defined at all levels of your inventory and plays.

```sh
cat <<EOF | tee --append playbook.yml
  vars:
    name: $STUDENT_ID
EOF
```

Run to see the profits of our trials:

```sh
ansible-playbook playbook.yml -v
```

_out:_

```log
<omitted>
```

To see this page in all it's glory, you can find your shell's public HTTP address for your `nginx` port by performing:

```sh
echo -e "$STUDENT_ADDRESSES"
```

_output:_

```log
This shells http port: https://lab.dabble.be/shell-http/demo/
Kubernetes http port: https://lab.dabble.be/http/
```

The last line in the output is provided for the [kubernetes exercise](https://github.com/dabble-be/kubernetes-intro).

## Conclusion

In this exercise, we've reaped the fruits of a configuration automation tool. This was only an introduction to the subject, there's a lot more to learn!

Check the references below for more information and documentation.

###### References

- [Exercise Source code](https://github.com/dabble-be/ansible-intro)
- [Official Ansible Documentation](https://docs.ansible.com/ansible/latest/)
- [Jinja Template Engine (used by Ansible)](https://jinja.palletsprojects.com/en/3.0.x/templates/)
- [AWS Free Tier Documentation](https://aws.amazon.com/free/)

- [Asset attribution](.assets/README.md)
