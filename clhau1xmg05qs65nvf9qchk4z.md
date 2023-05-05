---
title: "Deploy and Run Ansible with Docker"
datePublished: Tue Apr 11 2023 00:00:00 GMT+0000 (Coordinated Universal Time)
cuid: clhau1xmg05qs65nvf9qchk4z
slug: deploy-and-run-ansible-with-docker
canonical: https://brandont.dev/blog/deploy-and-run-ansible-with-docker/
ogImage: https://cdn.hashnode.com/res/hashnode/image/upload/v1683308314247/52ee57dd-ac3f-4eb4-ad12-a703ec1aa020.png
tags: docker, ansible, devops, podman

---

## Deploy and Run Ansible with Docker

How to deploy and run your Ansible instance in a scalable way with Docker.

My current position involves monitoring large enterprise environments for numerous clients. As the company grows, the need to scale our operations grows with it. Recently, I've been experimenting with [Ansible](https://ansible.com/) to scale our agent deployment to large environments, and run other large-scale operations.

This posed some interesting questions, and turned out to be a fun challenge. So I'd like to jump in to the solutions I came up with to deploy Ansible to a control node using Docker.

### The Container

First, we'll get a container set up. The continer itself is really just a wrapper around Ansible that we can adjust to fit our needs. First, we'll set up a `Containerfile`.

```plaintext
# Containerfile FROM ubuntu:latest AS ansible
ENV DEBIAN_FRONTEND=noninteractive

RUN apt update && 
apt upgrade -y && 
apt install -y gcc libkrb5-dev && 
apt install python3-pip -y && 
apt install openssh-client -y && 
apt install python3.10-venv -y &&
pip3 install --upgrade pip && 
pip3 install --upgrade virtualenv && 
pip3 install ansible

COPY scripts/run-entrypoint.sh /usr/local/bin/entrypoint

VOLUME ["/etc/ansible", "/root/.ssh"]

WORKDIR /etc/ansible

ENTRYPOINT ["/usr/local/bin/entrypoint"]
```

We'll be using an entrypoint script to call various Ansible CLI tools and pass arguments to them, so we'll need to `COPY` the script to the container. Our wrapper script is going to be mounting our Ansible directory containing our playbooks and inventories (and anything else you need) at `/etc/ansible`; this is the default location, but you can adjust it in the `ansible.cfg` file if you want it somewhere else. We'll also be mounting the key Ansible will use to ssh to managed node, so we'll need a volume at `/root/.ssh` in the container.

### Running the Container

Next, let's take a look at the script we'll be using to run Ansible from the container, `ansible-runner.sh`.

```bash
#!/bin/bash # ansible-runner.sh
set -euf

docker run 
    -it 
    -e user=$USER 
    --rm 
    -v $HOME/.ssh:/root/.ssh 
    -v /etc/ansible:/etc/ansible 
    ansible "$@"
```

Ok, so first we run `set -euf` for some error handling with bash. I'd like to dive deeper into scripting down the road, but for now, you can check out what these flags do [here](https://www.gnu.org/software/bash/manual/html_node/The-Set-Builtin.html) in the `set` bulitin documentation.

The next line runs the container; we'll be running with the `-it` flag so the container has an interactive terminal, which is needed for us to execute the enterypoint script and pass arguments to Ansible's CLI. We'll also need the `--rm` flag to kill the container when all of the processes inside of it are done, so we don't have any hanging or duplicate containers.

Now here's the fancy stuff. We want Ansible to connect to the managed nodes as the user who's running the container, so we'll take that `env` variable, and pass it as the container user with `-e user:$USER`. User escilation in Ansible should be carried out via the playbooks, not from the CLI, and we also want to control which users can access the managed nodes.

Next, we'll mount the key that Ansible will use to `ssh` to the managed node with `-v $HOME/.ssh:/root/ssh`, which will also use an `env` variable to mount the current user's `~/.ssh` directory. Now, we mount the global `/etc/ansible` directory, which contains our playbooks and inventory files that Ansible will use.

Lastly, we call the container, in this case I've named it `ansible`, and pass the arguments that will be used by the Ansible instance. A call using this script will look like this (from the script's directory):

./ansible-runner.sh ansible -m ping all

### The Container Entrypoint

Now we'll get our entrypoint script together. We're using `ENTRYPOINT` on our container because it's [more difficult to override](https://docs.podman.io/en/latest/markdown/podman-run.1.html#entrypoint-command-command-arg1); we want to control the containers default behavior as much as possible for security purposes. We'll call this script `run-entrypoint.sh`

```bash
#!/bin/sh # run-entrypoint.sh
set -euf

case "${1:-}" in 
ansible) 
    shift ansible -u "$user" "$@" 
    ;;

ansible-playbook) 
    shift ansible-playbook -u "$user" "$@" 
    ;;

ansible-vault) 
    shift ansible-vault "$@" 
    ;; 
esac
```

The first thing you'll notice is our shebang is calling `sh` rather than `bash`. We want to keep things as simple as possible within the container, and minimize any potential undesired or unexpected behavior, so calling a plain old `sh` over `bash` is preferred.

Next, we call our `set -euf` again, for the same reasons as last time, then we handle the arguments passed in from the runner script using `case`. Here, the `case "${1:-}"` will account for the very first argument passed.

Let's take a look back at `ansible-runner.sh` for a minute. The arguments we gave it were `ansible -m ping all`. This is an `ansible` CLI call to have ansible reach and ping all managed nodes in the inventory file. Our `run-entrypoint.sh` script will take our *first* argument here (`ansible`) and will run the `case` pattern match against it, shift to remove the argument from the argument array, set the user from the environment variable we gave to the runner script earlier, and then pass the rest of the arguments to the call to our container's Ansible instance (in this case, that would be `-m ping all`).

We've distinguished our `-u "$user"` argument here because other Ansible CLI tools will complain if given a user argument, so we account for them inside of the container on the entrypoint. This will also streamline the user connection, so our end-users will not have to account for their remote user, minimizing error and creating another layer of security.

Now that our scripts are done, you can deploy your playbooks, inventory, and config files to the control node, get Docker or Podman installed, and build your image.

Once it's all said and done, the container's Ansible instance will execute our commands on the managed nodes, and you'll be managing environments in no time.