# Kansible

Kansible lets you orchestrate _processes_ in the same way as you orchestrate your Docker _containers_ with [Kubernetes](http://kubernetes.io/). 

<p align="center">
  <a href="http://github.com/fabric8io/kansible/">
  	<img src="https://raw.githubusercontent.com/fabric8io/kansible/master/docs/images/logo.png" height="200" width="200" alt="kansible logo"/>
  </a>
</p>

Kansible uses:

* [Ansible](https://www.ansible.com/) to install, configure and provision your software onto machines using [playbooks](http://docs.ansible.com/ansible/playbooks.html) 
* [Kubernetes](http://kubernetes.io/) to run and manage the processes and perform service discovery, scaling and load balancing.

Kansible provides a single pane of glass, CLI and REST API to all your processes whether they are inside docker containers or running as vanilla proccesses on Windows, AIX, Solaris or HP-UX or an old Linux distros that predate docker. 

Kansible lets you slowly migrate to a pure container based Docker world while using Kubernetes to manage all your processes.

## Features

* All your processes appear as Pods inside Kubernetes namespaces so you can visualise, query and watch the status of your processes and containers in a canonical way
* Each kind of process has its own [Replication Controller](http://kubernetes.io/v1.1/docs/user-guide/replication-controller.html) to ensure processes keep running and so you can [manually or automatically scale](http://kubernetes.io/v1.1/docs/user-guide/replication-controller.html#scaling) the number of processes up or down; up to the limit in the number of hosts in your [Ansible inventory](http://docs.ansible.com/ansible/intro_inventory.html)
* Reuse Kubernetes [liveness checks](http://kubernetes.io/v1.1/docs/user-guide/liveness/README.html) so that Kubernetes can monitor the state of your process and restart if it goes bad  
* Reuse Kubernetes [readiness checks](http://kubernetes.io/v1.1/docs/user-guide/production-pods.html#liveness-and-readiness-probes-aka-health-checks) so that Kubernetes can know when your process can be included into the [internal or external service load balancer](http://kubernetes.io/v1.1/docs/user-guide/services.html) 
* You can view the logs of all your processes in the canonical kubernetes way via the CLI, REST API or web console
* You can open a shell into the remote process machine via the CLI, REST API or web console
* Port forwarding works from the pods to the remote processes so that you can reuse [Kubernetes Services](http://kubernetes.io/v1.1/docs/user-guide/services.html) to load balance across your processes automatically
* [Centralised logging](http://fabric8.io/guide/logging.html) and [metrics and alerting](http://fabric8.io/guide/metrics.html) works equally across your containers and processes

## How it works

You use kansible as follows:

* create an [Ansible playbook](http://docs.ansible.com/ansible/playbooks.html) to _install and provision_ the software you wish to run on a number of machines defined by the [Ansible inventory](http://docs.ansible.com/ansible/intro_inventory.html)
* run the [Ansible playbook](http://docs.ansible.com/ansible/playbooks.html) either as part of a [CI / CD build pipeline](http://fabric8.io/guide/cdelivery.html) when there's a change to the git repo of the Playbook, or using a command line tool, cron or [Ansible Tower](https://www.ansible.com/tower)
* define a Replication Controller YAML file at `kubernetes/$HOSTS/rc.yml` for running the command for your process [like this example](https://github.com/fabric8io/fabric8-ansible-spring-boot/blob/master/kubernetes/appservers/rc.yml#L15-L16). 
* the RC YAML file contains the command you need to run remotely to execute your process via [`$KANSIBLE_COMMAND`](https://github.com/fabric8io/fabric8-ansible-spring-boot/blob/master/kubernetes/appservers/rc.yml#L15-L16)
. You can use the `{{ foo_bar }}` ansible variable expressions to refer to variables from your [global ansible variables file](https://github.com/fabric8io/fabric8-ansible-spring-boot/blob/master/group_vars/appservers)
* whenever the playbook git repo changes, run the **kansible rc** command inside a clone of the playbook git repository:

```bash
    kansible rc myhosts
```    
    
where `myhosts` is the name of the hosts you wish to use in the [Ansible inventory](http://docs.ansible.com/ansible/intro_inventory.html).    

Then **kansible** will then create/update [Secrets](http://kubernetes.io/v1.1/docs/user-guide/secrets.html) for any SSH private keys in your [Ansible inventory](http://docs.ansible.com/ansible/intro_inventory.html) and create a [Replication Controller](http://kubernetes.io/v1.1/docs/user-guide/replication-controller.html) of kansible pods which will start and supervise your processes.  

So for each remote process on Windows, Linux, Solaris, AIX, HPUX kansible will create a kansible pod in Kubernetes which starts the command and tails the log to stdout/stderr. You can then use the [Replication Controller scaling](http://kubernetes.io/v1.1/docs/user-guide/replication-controller.html#scaling) to start/stop your remote processes!

### Using kansible

* As processes start and stop, you'll see the processes appear or disappear inside kubernetes, the CLI, REST API or the console.
* You can scale up and down the Replication Controller via CLI, REST API or console.
* You can then view the logs of any process in the usual kubernetes way via the command line, REST API or web console. 
* [Centralised logging](http://fabric8.io/guide/logging.html) then works great on all your processes (providing the command you run outputs logs to `stdout` / `stderr`)

### Exposing ports

Any ports defined in the Replication Controller YAML file will be automatically forwarded to the remote process.

This means you can take advantage of things like [centralised metrics and alerting](http://fabric8.io/guide/metrics.html) or Kubernetes Services and the built in service discovery and load balancing inside Kubernetes!

### Opening a shell on the remote process

You can open a shell directly on the remote machine via the web console or by running 

    oc exec -p mypodname bash


## Examples
  
To try out running one of the example Ansible provisioned apps try the following:

* [Download a release](https://github.com/fabric8io/kansible/releases) and add `kansible` to your `$PATH` 
* Or [Build kansible](https://github.com/fabric8io/kansible/blob/master/BUILDING.md) then add the `$PWD/bin` folder to your `$PATH` so that you can type in `kansible` on the command line

These examples assume you have a working [Kubernetes](http://kubernetes.io/) or [OpenShift](https://www.openshift.org/) cluster running.

To get started quickly try using the [fabric8 vagrant image that includes OpenShift Origin](http://fabric8.io/guide/getStarted/vagrant.html) as the kubernetes cluster.

### [fabric8-ansible-spring-boot](https://github.com/fabric8io/fabric8-ansible-spring-boot)

type the following to setup the VMs and provision things with Ansible

```bash
    git clone https://github.com/fabric8io/fabric8-ansible-spring-boot.git
    cd fabric8-ansible-spring-boot
    vagrant up
    ansible-playbook -i inventory provisioning/site.yml -vv
```

You now should have 2 sample VMs (app1 and app2) with a Spring Boot based Java application provisioned onto the machines in the `/opt` folder, but with nothing actually running yet.
    
Now to setup the kansible Replication Controller run the following, where `appservers` is the hosts from the [Ansible inventory](http://docs.ansible.com/ansible/intro_inventory.html)
    
```bash
    kansible rc appservers
```      

This should now create a Replication Controller called `springboot-demo` along with 2 pods for each host in the `appservers` inventory file. 

You should be able to look at the logs of those 2 pods in the usual Kubernetes / OpenShift way.

e.g.

```bash
    oc get pods 
    oc logs -f springboot-demo-81ryw 
```

where `springboot-demo-81ryw` is the name of the pod you wish to view the logs.

You can now scale down / up the number of pods using the web console or the command line:

```bash
    oc scale rc --replicas=2 springboot-demo
```

#### Important files

The examples use the following files:

* [inventory](https://github.com/fabric8io/fabric8-ansible-spring-boot/blob/master/inventory) is the Ansible inventory file to define the [hosts](https://github.com/fabric8io/fabric8-ansible-spring-boot/blob/master/inventory#L1-L3) to run the processes
* [kubernetes/$HOSTS/rc.yml](https://github.com/fabric8io/fabric8-ansible-spring-boot/blob/master/kubernetes/appservers/rc.yml) is the Replication Controller configuration used to [define the command `$KANSIBLE_COMMAND`](https://github.com/fabric8io/fabric8-ansible-spring-boot/blob/master/kubernetes/appservers/rc.yml#L15-L16) which kansible uses to run the process remotely
 
### [fabric8-ansible-hawtapp](https://github.com/fabric8io/fabric8-ansible-hawtapp)

type the following to setup the VMs and provision things with Ansible

```
    git clone https://github.com/fabric8io/fabric8-ansible-hawtapp.git
    cd fabric8-ansible-hawtapp
    vagrant up
    ansible-playbook -i inventory provisioning/site.yml -vv
```
    
Now to setup the Replication Controller for the supervisors run the following, where `appservers` is the hosts from the inventory

```    
    kansible rc appservers
```      

The pods should now start up for each host in the inventory!

### Using Windows

To use windows you need to first make sure you've installed **pywinrm**:

    sudo pip install pywinrm
    
To try using windows machines, replace `appservers` with `winboxes` in the above commands; assuming you have created the [Windows vagrant machine](https://github.com/fabric8io/fabric8-ansible-hawtapp/tree/master/windows) locally

Or you can add the windows machine into the `appservers` hosts section in the `inventory` file.


## Configuration

To configure kansible you need to configure a [Replication Controller](http://kubernetes.io/v1.1/docs/user-guide/replication-controller.html) in a file called [kubernetes/$HOSTS/rc.yml](https://github.com/fabric8io/fabric8-ansible-spring-boot/blob/master/kubernetes/appservers/rc.yml).
 
Specify a name and optionally some labels for the replication controller inside the `metadata` object. There's no need to specify the `spec.selector` or `spec.template.containers[0].metadata.labels` values as those are inherited by default from the `metadata.labels`.

You can specify the following environment variables in the `spec.template.spec.containers[0].env` array like the use of `KANSIBLE_COMMAND` below. 

These values can use Ansible variable expressions too. 

### KANSIBLE_COMMAND 
Then you must specify a command to run via the [`$KANSIBLE_COMMAND`](https://github.com/fabric8io/fabric8-ansible-spring-boot/blob/master/kubernetes/appservers/rc.yml#L15-L16) environment variable:

```yaml
---
apiVersion: "v1"
kind: "ReplicationController"
metadata:
  name: "myapp"
  labels:
    project: "myapp"
    version: "{{ app_version }}"
spec:
  template:
    spec:
      containers:
      - env:
        - name: "KANSIBLE_COMMAND"
          value: "/opt/foo-{{ app_version }}/bin/run.sh"
      serviceAccountName: "fabric8"
```

### KANSIBLE_COMMAND_WINRM 

This environment variable lets you provide a Windows specific command. It works the same as the `KANSIBLE_COMMAND` environment variable above, but this value is only used for Ansible connections of the form `winrm`. i.e. to supply a windows only command to execute.

Its quite common to have a `foo.sh` script to run sh/bash scripts on unix and then a `foo.bat` or `foo.cmd` file for Windows. 


#### KANSIBLE_EXPORT_ENV_VARS

Specify a space separated list of environment variable names which should be exported into the remote shell when running the remote command.

Note that typically your [sshd_config](http://linux.die.net/man/5/sshd_config) will disable the use of most environment variables being exported that don't start with `LC_*` so you may need to [configure your sshd](http://linux.die.net/man/5/sshd_config) in `/etc/ssh/sshd_config` to enable this.


#### KANSIBLE_BASH

This defines the path where the bash script will be generated for running a remote bash shell. This allows running the command `bash` inside the kansible pod to remotely execute either `/bin/bash` or `cmd.exe` for Windows machines on the remote machine when you try to open a shell inside the Web Console or via:

```bash
    oc exec -p mypodname bash
```

#### KANSIBLE_PORT_FORWARD

Allows port forwarding to be disabled. 

```
export KANSIBLE_PORT_FORWARD=false
```

This is mostly useful to allow the `bash` command within a pod to not also try to port forward as this will fail ;)

### SSH or WinRM

The best way to configure if you want to connect via SSH for unix machines or WinRM for windows machines is via the Ansible Inventory.

By default SSH is used on port 22 unless you specify `ansible_port` in the inventory or specify `--port` on the command line.

You can configure Windows machines using the `ansible_connection=winrm` property in the inventory:


```yaml
[winboxes]
windows1 ansible_host=localhost ansible_port=5985 ansible_user=foo ansible_pass=somepasswd! ansible_connection=winrm

[unixes]
app1 ansible_host=10.10.3.20 ansible_user=vagrant ansible_ssh_private_key_file=.vagrant/machines/app1/virtualbox/private_key
app2 ansible_host=10.10.3.21 ansible_user=vagrant ansible_ssh_private_key_file=.vagrant/machines/app2/virtualbox/private_key
```

You can also enable WinRM via the `--winrm` command line flag: 

```
export KANSIBLE_WINRM=true
kansible pod --winrm somehosts somecommand

```

or by setting the environment variable `KANSIBLE_WINRM` which is a little easier to configure on the RC YAML:

```
export KANSIBLE_WINRM=true
kansible pod somehosts somecommand

```
### Checking the runtime status of the supervisors
 
To see which pods own which hosts run the following command:
 
```
    oc export rc hawtapp-demo | grep ansible.fabric8  | sort
```

Where `hawtapp-demo` is the name of the RC for the supervisors.

The output is of the format:

```
    pod.ansible.fabric8.io/app1: supervisor-znuj5
    pod.ansible.fabric8.io/app2: supervisor-1same
```

Where the output is of the form ` pod.ansible.fabric8.io/$HOSTNAME: $PODNAME`



 
