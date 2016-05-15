# James' Vagrant Setup
This is my preferred vagrant setup I've used for various projects that
I've decided to share for everyone's enjoyment. Over time I've found
that a very simlified vagrant setup that is completely self-contained
can help teams get going with ansible and experimenting with infra
development locally quickly and painlessly, but getting started with
setting up THAT can sometimes be a huge time sink and possibly take
months to iterate on and get right. Hopefully this will help you
sidestep that. :-)

This whole setup works
by using a nodes.yml to configure a multi-machine vagrant setup and
uses a completely self contained ansible installation to provision roles
that are defined under `ansible/roles`. It also makes use of
ansible-galaxy to install 3rd party role as defined in
`ansible/requirements.yml`.

This also utilizes a preferred configuration management practice of mine
by chopping up provisioning into different stages.

- `install` - typically done when creating images or droplets when
  configuring real machines.
- `post-boot` - some things you just can't know until an actual instance
  boots up. This can include the TLD that should be used, ohter
instances that need to be referenced, hostnames, etc. This could also
include secrets you just don't want living on server images. The idea of
post-boot is it will run once when an instance boots up and configure
those items to the desired state.
- `test` - The test stage basically performs a self test against the
  instance to ensure things are running smoothly. It might check that
instances are running on ports, certain files exist, etc.

## Getting Started
To illustrate how this is used, let's do a quick example by configuring
a server running rabbitmq proxied by nginx.

### Initial Setup
We'll start by editing the nodes.yml and adding the role. We also
specify an IP address that we can use to refer to the server both from
our host machine and from other VMs used in this setup.

```yml
rabbitmq:
  ip: 172.16.32.100
  ram: 512
```

Next, just to get the minimum pieces in place, we'll create an empty
directory at `ansible/roles/rabbitmq`. When we type `vagrant up
rabbitmq` now we should see an initial vagrant run that will do a number
of things:

- boot up the vm
- install ansible
- install third party roles if any are present
- run playbooks for install, `post-boot` and `test` phases

If all goes well the last line of output you see should look something
like

```
PLAY RECAP *********************************************************************
rabbitmq                   : ok=1    changed=0    unreachable=0    failed=0
```

### Actually Install It
Now we have a machine running but there's nothing on it! Let's change
that by installing rabbitmq. Let's create the scaffolding for that under
the `ansible/roles/rabbitmq` directory.

```bash
mkdir ansible/roles/rabbitmq/tasks
touch ansible/roles/rabbitmq/tasks/{main,postboot,install,test}.yml
```

And modify the contents of `ansible/roles/rabbitmq/main.yml` to include
the apporpriate files for each phase.

```yml
---
- include: install.yml

- include: postboot.yml
  tags: postboot

- include: test.yml
  tags: test

```

For the install phase, we'll want to do the following tasks based on a
cursory reading of https://www.rabbitmq.com/install-debian.html to add
the official Pivotal apt repository and install from that.

Breaking these down into tasks, we'll want to do the following:

- add rabbitmq apt key
- add official pivotal repository
- install rabbitmq

Thankfully each of these tasks also have a corresponding module that can
be used to accomplish each: [apt_key](http://docs.ansible.com/ansible/apt_repository_module.html), [apt_repository](http://docs.ansible.com/ansible/apt_key_module.html) and
[package](http://docs.ansible.com/ansible/package_module.html). Feel
free to break now and give that a try yourself and read on for the
complete solution.


```yml
---
- name: add rabbitmq apt key
  apt_key:
    url: https://www.rabbitmq.com/rabbitmq-signing-key-public.asc
    state: present

- name: add official pivotal repository
  apt_repository:
    repo: 'deb http://www.rabbitmq.com/debian/ testing main'
    state: present
    update_cache: yes

- name: install rabbitmq
  package:
    name: rabbitmq-server
    state: present

```

With this in place in `ansible/roles/rabbitmq/install.yml` we run
`vagrant provison` and will see ansible install rabbitmq. If we run
`vagrant ssh rabbitmq` to ssh into the box and type `ps ax |grep rabbitmq-server` we should definitely see rabbitmq up and running.

### Enabling Plugins and Adding Users

Next up we should do what will be a post-boot operation - adding users
to rabbitmq. Therefore we add the following to
`ansible/roles/rabbitmq/tasks/postboot.yml`. Ansible actually has
several rabbitmq modules built-in.

```yaml
---
- name: enable management plugin
  rabbitmq_plugin:
    name: rabbitmq_management
    state: enabled

- name: "Create admin user"
  rabbitmq_user:
    user: administrator
    password: administrator
    vhost: /
    configure_priv: .*
    read_priv: .*
    write_priv: .*
    state: present
    tags: administrator

```

Run `vagrant provision rabbitmq` again and we should be able to
navigate to http://172.16.32.100:15672/ and login as
`administrator`/`administrator` to access the management panel.

### Add a Third Party Role
For the next step we've decided to put a reverse proxy in front of
rabbitmq. This allows us to secure the connection with SSL.

Searching ansible galaxy, it seems that `jdauphant.nginx` is a good rle
to use for installing nginx, so let's go with that. We add the following
to ansible/requirements.yml.

```yaml
---
- src: jdauphant.nginx
  version: v2.1.0

```

This vagrant file will do a hash of the requirements.yml file and run
install when it detects changes so you get the installation for free
when you provision.

Next up we add and configure `jdauphant.nginx` in
`ansible/roles/rabbitmq/meta/main.yml`

```yaml
---
allow_duplicates: yes
dependencies:
  - role: jdauphant.nginx
    nginx_http_params:
      - "tcp_nodelay on"
      - "error_log /var/log/nginx/error.log"
      - "access_log /var/log/nginx/access.log"
      - "client_max_body_size 0"
    nginx_sites:
      docker_app:
        - listen 80
        - server_name _
        - location / {
            proxy_pass http://app_server;
          }
    nginx_configs:
      proxy:
        - proxy_set_header X-Real-IP  $remote_addr
        - proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for
        - proxy_redirect off
        - proxy_set_header Host $http_host
      upstream:
        - upstream app_server {
            server 127.0.0.1:15672 fail_timeout=0;
          }
```

We'll finally run `vagrant provision rabbitmq` one more time and we
should now be able to access the RabbitMQ management page at http://172.16.32.100/.



