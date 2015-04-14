pulp
====

Ansible role which installs and confiures the Pulp Project. The role was tested
in single host installation but it should be possible to spread the Pulp service
across multiple hosts and even use different messaging tool than Qpid (e.g.
RabbitMQ).

The configuraton of the role is done in such way that it should not be necessary
to change the role for any kind of configuration. All can be done either by
changing role parameters or by declaring completely new configuration as a
variable. That makes this role absolutely universal. See the examples below for
more details.

Please report any issues or send PR.


Example
-------

This role was tested with single-host setup only and this is the configuration
which worked for me:

```
---

# Example of a single host installation
- hosts: myhost
  roles:
    - role: mongodb
      mongodb_net_bindIp: 127.0.0.1
      mongodb_net_wireObjectCheck: false
      mongodb_net_unixDomainSocket_enabled: true
      mongodb_processManagement_fork: true
      mongodb_systemLog_logAppend: true
      mongodb_systemLog_timeStampFormat: iso8601-utc
    - role: pulp
      pulp_install_qpid: true
      pulp_install_server: true
      pulp_install_admin: true
      pulp_install_consumer: true
      pulp_run_celerybeat: true
      pulp_run_resource_manager: true
```

After everything is installed, you can use the following commands to create a clone of a repo:

```
pulp-admin login -u admin -p admin
pulp-admin rpm repo create --repo-id=base-el6-64-extras-20150103 --display-name=base-el6-64-extras-20150103 --relative-url=base/centos6/x86_64/extras/20150103 --serve-http=true --feed=http://mirror.centos.org/centos/6/extras/x86_64/
pulp-admin rpm repo sync run --repo-id=base-el6-64-extras-20150103
pulp-admin rpm repo list
```

The repo should then be accessible via URL `http://myhost/pulp/repos/`.

It should also be possible to use it in multi-host setup, although I did not test
it. The setup should look something like this:

```
---

# Mongo DB server
- hosts: pulp-mongodb
  roles:
    - role: mongodb
      mongodb_net_bindIp: 0.0.0.0
      mongodb_net_wireObjectCheck: false
      mongodb_net_unixDomainSocket_enabled: true
      mongodb_processManagement_fork: true
      mongodb_systemLog_logAppend: true
      mongodb_systemLog_timeStampFormat: iso8601-utc

# Messaging server
- hosts: pulp-messaging
  roles:
    # We can use the qpid role directly
    - role: qpid_cpp_server
      # Configure Qpid for multi-host setup
      qpid_cpp_server_qpidd_config:
        ...

# First Pulp server
- hosts: pulp-server1
  roles:
    - role: pulp
      pulp_install_server: true
      # Only the first Pulp server will runt these two services
      pulp_run_celerybeat: true
      pulp_run_resource_manager: true
      # Configure Pulp server for multi-host setup
      pulp_server_config:
        ...
      # Configure repo_auth
      pulp_server_repo_auth_config:
        ...

# Second Pulp server
- hosts: pulp-server2
  roles:
    - role: pulp
      pulp_install_server: true
      # Configure Pulp server for multi-host setup
      pulp_server_config:
        ...
      # Configure repo_auth
      pulp_server_repo_auth_config:
        ...

# Pulp admin
- hosts: pulp-admin
  roles:
    - role: pulp
      pulp_install_admin: true
      # Configure Pulp admin for multi-host setup
      pulp_admin_config:
        ...

# Pulp customer
- hosts: pulp-consumer
  roles:
    - role: pulp
      pulp_install_consumer: true
      # Configure Pulp consumer for multi-host setup
      pulp_consumer_config:
        ...
```

It should also be possible to use RabbitMQ instead of the default Qpid messaging
system. Again, I did not test it but the setup should look something like this:

```
---

# Example how to setup Pulp with RabbitMQ on a single host
- hosts: myhost
  roles:
    - role: mongodb
      mongodb_net_bindIp: 127.0.0.1
      mongodb_net_wireObjectCheck: false
      mongodb_net_unixDomainSocket_enabled: true
      mongodb_processManagement_fork: true
      mongodb_systemLog_logAppend: true
      mongodb_systemLog_timeStampFormat: iso8601-utc
    # Get some rabbitmq role and configure it
    - role: rabbitmq
       ...
    - role: pulp
      pulp_install_server: true
      pulp_install_admin: true
      pulp_install_consumer: true
      pulp_run_celerybeat: true
      pulp_run_resource_manager: true
      # The server and customer YUM groups are different for RabbitMQ
      pulp_server_pkgs:
        - "@pulp-server"
        - python-gofer-amqplib
      pulp_consumer_pkgs:
        - "@pulp-consumer"
        - python-gofer-amqplib
      # You will probably need to change some configuration
      pulp_server_config:
        messaging:
          transport: rabbitmq
          broker_url: amqp://guest:guest@localhost/foo
          ...
        ...
      pulp_consumer_config:
        messaging:
          transport: rabbitmq
          broker_url: amqp://guest:guest@localhost/foo
          ...
        ...
```

This role requires [Config Encoder
Macros](https://github.com/picotrading/config-encoder-macros) which must be
placed into the same directory as the playbook:

```
$ ls -1 *.yaml
site.yaml
$ git clone https://github.com/picotrading/config-encoder-macros.git ./templates/encoder
```


Role variables
--------------

List of variables used by the role:

```
# Whether to install Qpid server as a dependency of this role
pulp_install_qpid: false

# Whether to install Pulp server
pulp_install_server: false

# Whether to install Pulp admin client
pulp_install_admin: false

# Whether to install Pulp consumer
pulp_install_consumer: false

# Whether to run pulp_celerybeat service
# (should run only on one Pulp server)
pulp_run_celerybeat: false

# Whether to run pulp_resource_manager service
# (should run only on one Pulp server)
pulp_run_resource_manager: false

# Default YUM repository
pulp_yum_repo:
  pulp:
    name: Pulp Project repository
    baseurl: https://repos.fedorapeople.org/repos/pulp/pulp/stable/latest/$releasever/$basearch/
    gpgcheck: 0

# Default YUM group for Pulp server
pulp_server_pkgs:
  - "@pulp-server-qpid"
  # For RabbitMQ use this instead:
  #- "@pulp-server"
  #- python-gofer-amqplib

# Default YUM group for Pulp consumer
pulp_consumer_pkgs:
  - "@pulp-consumer-qpid"
  # For RabbitMQ use these instead:
  #- "@pulp-consumer"
  #- python-gofer-amqplib

# Whether to enable or disable SSL verification
pulp_verify_ssl: false

# Default FQDN of the host (only for single machine installation)
pulp_host: localhost.localdomain

# Default Qpid daemon configuration
pulp_qpidd_config:
  auth: 'no'

# Default Pulp server configuration
pulp_server_config:
  database: {}
  server: {}
  authentication: {}
  security: {}
  consumer_history: {}
  data_reaping: {}
  oauth: {}
  messaging: {}
  tasks: {}
  email: {}

# Default parameters of the Pulp workers configuration
pulp_workers_celeryd_log_file: /var/log/pulp/%n.log
pulp_workers_celeryd_log_level: INFO
pulp_workers_celeryd_pid_file: /var/run/pulp/%n.pid
pulp_workers_pythonencoding: UTF-8
pulp_workers_pulp_concurrency: "{{ ansible_processor_cores }}"

# Default Pulp workers configuration
pulp_workers_config:
  # Workers variables which can be changed
  celeryd_log_file: "{{ pulp_workers_celeryd_log_file }}"
  celeryd_log_level: "{{ pulp_workers_celeryd_log_level }}"
  celeryd_pid_file: "{{ pulp_workers_celeryd_pid_file }}"
  pythonioencoding: "{{ pulp_workers_pythonencoding }}"
  pulp_concurrency: "{{ pulp_workers_pulp_concurrency }}"
  # Workers variables which should not be changed
  celery_app: pulp.server.async.app
  celery_create_dirs: 1
  celeryd_nodes: ""
  celeryd_opts: "-c 1 --events --umask=18"
  celeryd_user: apache
  default_nodes: ""

# Default Pulp admin config file path
pulp_admin_config_file: /etc/pulp/admin/admin.conf

# Default Pulp admin configuration
pulp_admin_config:
  server:
    host: "{{ pulp_host }}"
    verify_ssl: "{{ pulp_verify_ssl }}"
  client: {}
  filesystem: {}
  logging: {}
  output: {}

# Default Pulp consumer configuration
pulp_consumer_config:
  server:
    host: "{{ pulp_host }}"
    verify_ssl: "{{ pulp_verify_ssl }}"
  authentication: {}
  client: {}
  filesystem: {}
  reboot: {}
  logging: {}
  output: {}
  messaging: {}
  profile: {}

# Default Pulp repo_auth configuration
pulp_server_repo_auth_config:
  main:
    enabled: false
    log_failed_cert: true
    log_failed_cert_verbose: false
    max_num_certs_in_chain: 100
    verify_ssl: "{{ pulp_verify_ssl }}"
  repos:
    cert_location: /etc/pki/pulp/content
    global_cert_location: /etc/pki/pulp/content
    protected_repo_listing_file: /etc/pki/pulp/content/pulp-protected-repos
```


Dependencies
------------

* [`mongodb`](https://github.com/picotrading/ansible-mongodb) role
* [`yumrepo`](https://github.com/picotrading/ansible-yumrepo) role
* [`qpid_cpp_server`](https://github.com/picotrading/ansible-qpid_cpp_server) role (optional)
* [Config Encoder Macros](https://github.com/picotrading/config-encoder-macros)


License
-------

MIT


Author
------

Jiri Tyr
