# ANSIBLE

## Introduction
This introduction will go over the basics of __ansible__. It will use __vagrant__ to
to host several machines that will act as a basic web application behind a
load balancer. It will go over the following:

1. basics of __ansible__
1. using __ansible__ as a provisioner
1. using __ansible__ to do a zero down time deployment

Here are a list of resources that were used to create this example project.  
* http://docs.ansible.com/ansible/
* https://sysadmincasts.com/episodes/43-19-minutes-with-ansible-part-1-4
* http://hakunin.com/six-ansible-practices#make-it-easy-to-sync-your-hosts-file-with-your-vms

### Vagrant setup
The __vagrant__ environment is composed of the following:

* one management server (mgmt)
  * this will be the box we run all of our __ansible__ commands from
* one load balancer (lb)
  * this __HAProxy__ will be used to balance traffic across the web servers
  * we will use __HAProxy__ s web ui to monitor the web application
* four web servers (web1-4)
  * these are very basic web servers running __nginx__

### How Ansible is different from Puppet (and other similar tools)
* __ansible__ is completely agentless, everything is run through ssh
  * other tools have options to run like this, but __ansible__ does this by default

## Requirements
1. [virtualbox](https://www.virtualbox.org/wiki/Downloads)
1. [vagrant](https://www.vagrantup.com/downloads.html)

## Ansible walk through
The first thing we want to do is get our vagrant environment up and running. We
can do this by simply running `vagrant up` in the project directory. After the
command finishes we should be left with six vms up and running. We can confirm
this by running `vagrant status` and should see
```
Current machine states:

mgmt                      running (virtualbox)
lb                        running (virtualbox)
web1                      running (virtualbox)
web2                      running (virtualbox)
web3                      running (virtualbox)
web4                      running (virtualbox)
```
We can now ssh into our mgmt box by running `vagrant ssh mgmt`. This will be the
vm that we run all of our commands from.

#### Connection setup
The first thing to notice is our `inventory.ini` file. This is file that tells
__ansible__ where all of our servers are. You can see that we have two groups.
One for our load balancer and one for our web servers, with their corresponding
servers below them. __ansible__ can run commands on these servers by calling a
host (`web1`) or a group (`web`). There is also a global group `all` that all
hosts listed in the `inventory.ini` are automatically a part of.

The next file to take note of is `ansible.cfg`. This is the configuration file
for __ansible__. All kinds of settings can be put [here](http://docs.ansible.com/ansible/intro_configuration.html).
You can see that we are telling __ansible__ where to find our inventory file.
Alternatively we could just add the `-i` flag to any __ansible__ command to tell
it where to look.

The first thing we can do is try to make sure we can connect to all of our
servers. We can do this by running `ansible all -m ping` from our home
directory. We should get the following error on every vm
```
web2 | UNREACHABLE! => {
    "changed": false,
    "msg": "ERROR! SSH encountered an unknown error during the connection. We recommend you re-run the command using -vvvv, which will enable SSH debugging output to help diagnose the issue",
    "unreachable": true
}
```
This is because we have not established an ssh connection between any of the
vms. We need to add our ssh key to all of the vms. We can do this by running
`ansible-playbook ssh-addkey.yml --ask-pass`. This will add our ssh key to all
of the boxes after we enter in the default ssh password `vagrant`. We should see
an output that lists all of the servers and what tasks were successful. We can
see that each server had one item changed and one item okay. If we run the same
command again without the `--ask-pass` flag we will see that everything just
says okay. This is because __ansible__ is idempotent, so because we have already
added the ssh key it will not add it again. It just confirms that it is there.
Now we can successfully run the command from earlier `ansible all -m ping`.

We have just run two types of commands. The `ansible` command runs a single task
(module) on a specific host of group. Another example would be
`ansible lb -m shell -a "uptime"`. This command uses the shell module to run the
`uptime` command on the lb group.

The `ansible-playbook` command runs a playbook which is essentially a list of
tasks. If we take a look at `ssh-addkey.yml` we can see that it is composed of
one play which has one task. We can control what vms this play is executed
against with the `hosts` parameter; in this case `all`. Next we see
`become: yes`. This tells __ansible__ that this play should be run with `sudo`.
After that we see `gather_facts: no`. By default __ansible__ will gather all
kinds of information about the machine it is about to run on. Many playbooks
use this kind of information to conditionally run tasks. However, this process
is slow and is unneeded for this play so we have disabled it. Next is the list
of tasks that this play will run. The `name` field is completely arbitrary and
just describes what the task is for, but it must be unique. This will be the
name that shows up in the task report when the playbook is done. The
`authorized_key`field is simply a module that we are using for this task. The
keys following it are parameters specific to the module. You can see that we are
looking up the public key on the current machine and passing it to the module.
The `state` key is common to almost all modules. This tells __ansible__ that the
public key should be `present` in the authorized_keys file in this case.

#### Provisioning with ansible
Now that we have the keys setup, we can now set up the website. We can run the
playbook that will set up our mock website with `ansible—playbook site.yml`.
This should configure our loadbalancer and the four web servers. If we take a
look at `site.yml` we can see that there are several plays targeting different
groups using roles. Roles are the construct that __ansible__ uses to create
modular and reusable groups of tasks. In this case they are organized around
what type of server they are setting up, but other examples could include a role
for __nginx__ or a role for __mysql__.

If we take a look at the `web` role we can see that it is made up of three
sub—folders. These folders follow the __ansible__ naming conventions and
structure for a role. You can get a better sense of roles by checking out [ansible—galaxy](https://galaxy.ansible.comn which is where people can share
roles or by creating an empty role by running 'ansible—galaxy init <role_name>'.

The main folder in the role is the `tasks` folder. This is where we define the
tasks that we want __ansible__ to execute. __ansible__ will look for `main.yml`
in this folder. This file could include other task files as well. Within this
file we can see two knew keys on some of the tasks; `notify` and `template`.
`notify` is used to call a handler. When the task finishes in a `changed` state
the handler will be called. This pattern is typically used for things that might
be called by initiated by multiple tasks but only need to be called once. All
handlers are called after all of the tasks within that play have finished. You
can see this if you review the output in the terminal. We can see that two tasks
are calling the `restart nginx` handler (which is in `handlers/main.yml`), but
it only gets called once during execution. The `template` module uses jinja 2
templating. The `src` of the template is relative to the template folder within
the current role. The template uses a few __ansible__ variables to render the
final file. These variables are calculated during the `setup` task and are
accessible globally This task runs by default and is controlled by the
`gather_facts` key we talked about earlier. If you look at our output you can
see that this task is skipped on the `common` play, which is the only play that
does not run the `setup` task.

The template in the `lb` role probably has the most interesting example of using
variables. On lines 38 - 44 we can see the template referencing variables from
other hosts. This is done by using one of "magic variables" __ansible__,
`hostvars`. We can see the values that __ansible__ populates this mapped
variable with by running `ansible all -m setup`. The template uses this variable
to add all of our web servers to the __HAProxy__ configuration. You can read
more about variables [here](http://docs.ansible.com/ansible/playbooks_variables.html).

Another thing you will notice is that all of the template files have
`{{ ansible_managed }}` at the top of them. This will add a line with some info
about when the file was created and lets anyone that comes across it that it is
managed by __ansible__ and any changes made to it manually will probably be
overwritten.

We should be able to use the website [here](http://localhost:8080/) and see our
__HAProxy__ status page [here](http://localhost:8080/haproxy?stats).

#### Deploying with ansible
Now that we have our website setup, we can go over how we can use __ansible__ to
deploy changes with zero downtime. We can run
`ansible-playbook deploy-rolling.yml --extra-vars "web_title='THE SITE'"`. If we
check our site we should see the title has been replaced with `THE SITE`. We can
set this to whatever we want by running this command and setting `web_title` to
any value we want. This should run through each web server, removing it from the
load balancer, running our `web` role, adding the web server back to the
loadbalancer, and then running our `lb` role. This playbook is almost identical
to the `site.yml` playbook except for a few key things in the second play. In
this case we are reprovisioning our web servers, but in a specific way.

The first difference is the `serial` key. This controls how the play is
executed. By default __ansible__ will run plays in parallel across the different
hosts. The `serial` key tells __ansible__ to run the whole play on `x` amount of
hosts before running the play on the next group of `x` hosts. In this case we
will be running the whole play on each server before moving on to the next one.
You can play with this concept by comparing the `parallel.yml` and `serial.yml`
playbooks.

The only other difference are the `pre_tasks` and `post_tasks`. These are run
before and after (respectfully) any other tasks and roles. So in this case we
iterating through each server. Removing it from the loadbalancer (`pre_tasks`).
Deploying the new changes. Then adding the that server back to our loadbalancer
(`post_tasks`). This allows us to update our web servers with zero downtime
only one server is down at any given time.
