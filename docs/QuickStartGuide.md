# Quickstart Installation Guide

## Prepare your instances for the deployment

You need (at least) one openstack account that has all the permissions to create/update/delete all those services you want to be tested and you need access to an openstack account with swift for storing the logs of the executed test szenarios.

Recommendation is to use one project for deploying the instances which are firing the tests and one project in which all services of interest can be used (and appropriate users).

Lets start with one deploy instance for the deployment and one for all the services. The size of the instance(s) needed depends highly on your use case/used services. There are no recommendations yet.

We used a small instance (SCS-2V:4:20) for the deployment (cloudmon "orchestrator") and a SCS-4V:8:100 running Ubuntu 22.04 (tested in our case) for the services for testing. Any newer Linux distro capable running Docker (and Python 3.10?) should be fine.

## Security Groups

When you want to access the cloudmon services (Grafana + Graphite UI) you will have to allow access to the following ports:

* Port 80 (Graphite UI)
* Port 3000 (Grafana UI)

## Reverse proxy for TLS termination

The installation method described in this document does not deploy a SSL certificate for these services. Just install a nginx or caddy as reverse proxy.

## Prepare the deploy host with tox + ansible

Before the installation you will need to install ansible and tox.

```
ubuntu@deploy:~$ sudo apt install ansible tox -y
```

**NOTE:** You can install tox via pip install also, but then you will have to take care of all other required Python modules of your own. Ansible needs(?) to be in your system path.

## Install cloudmon

Next check out the cloudmon repository via git clone.

```
ubuntu@deploy:~$ git clone https://github.com/stackmon/cloudmon.git
ubuntu@deploy:~$ cd cloudmon/
```

Install cloudmon with tox:

```
ubuntu@deploy:~/cloudmon$ tox -epy310 --notest && source .tox/py310/bin/activate
py310 create: /home/ubuntu/cloudmon/.tox/py310
py310 installdeps: -r/home/ubuntu/cloudmon/test-requirements.txt
py310 develop-inst: /home/ubuntu/cloudmon
py310 installed: 
[..]
  py310: skipped tests
  congratulations :)
(py310) ubuntu@deploy:~/cloudmon$
```

## Configuration

Next you will need a configuration and your inventory file.

You can copy the sample_config.yaml and tailor it to your own needs. You need at least on monitoring zone. Let's start with one internal.

### etc/config.yaml

These group names correspond to the inventory group names in the ansible repository you will define later.

```
monitoring_zones: # Defining from where we are monitoring
  internal:
    graphite_group_name: graphite
    statsd_group_name: statsd
```

see also https://github.com/stackmon/cloudmon/blob/main/etc/sample_config.yaml#L3

Currently there has to be one cloud named _production_. This part refers to the clouds.yaml configs which are specified at the end of the config file.

```
environments: # What we monitor and which credentials to use from every monitoring location (zone)
  # NOTE: not allowed to use same cloud names in same zone for different envs
  production:
    env:
      OS_CLOUD: production
    monitoring_zones:
      internal:
        clouds:
          - name: production
            ref: p1
          - name: swift
            ref: swift1
```

see also https://github.com/stackmon/cloudmon/blob/main/etc/sample_config.yaml#L11

p1 is the project which shall be used for tests. swift1 is the project where the swift bucket will be created which is used for uploading the logs of the tests (output of the ansible playbooks of apimon for example).

In the next section you will have to define the plugins which shall be used for testing you cloud enviroment. There currently three plugins available:

- apimon
- epmon
- lb

```
# Known CloudMon plugins with their basic settings
cloudmon_plugins:
  apimon:
    type: apimon
    scheduler_image: quay.io/opentelekomcloud/apimon:change_35_latest
    executor_image: quay.io/opentelekomcloud/apimon:change_35_latest
    epmon_image: quay.io/opentelekomcloud/apimon:change_35_latest
    tests_projects:
      - name: apimon
        repo_url: https://github.com/stackmon/apimon-tests
        repo_ref: main
        exec_cmd: ansible-playbook -i inventory/production %s -vvv
        scenarios_location: playbooks
        grafana_dashboards_location: dashboards
```

see also https://github.com/stackmon/cloudmon/blob/main/etc/sample_config.yaml#L30

We will start with apimon only at first. For all plugins see sample_config.yaml.

Fork the apimon-tests repository for defining you own scenarios and change the repo_url accordingly.

Set your passwords for the database users (root, grafana, apimon).

**NOTE:** Replace ONLY the `ChangeMe123$` parts. The `&apimon_database...` statements are used for referral.

```
graphite:
  host: localhost

database:
  root_password: ChangeMe123$
  databases:
    - name: grafana
      users:
        - name: grafana
          password: &grafana_database_password ChangeMe!123$
    - name: apimon
      users:
        - name: apimon
          password: &apimon_database_password ChangeMe!123$
```

see also https://github.com/stackmon/cloudmon/blob/main/etc/sample_config.yaml#L52

In this section you will have to change the initial grafana _admin_ password (grafana_security_admin_password) for UI.

```
grafana:
  datasources:
    - name: cloudmon
      type: graphite
    - name: apimon_db
      type: postgres
      database: apimon
      user: apimon
      jsonData:
        postgresVersion: 14
        sslmode: disable
      secureJsonData:
        password: *apimon_database_password
  config:
    grafana_image: quay.io/opentelekomcloud/grafana:9.1.5
    grafana_renderer_image: quay.io/opentelekomcloud/grafana-image-renderer:3.6.1
    grafana_security_admin_password: fake_password_change_me
    grafana_enable_renderer: 'true'
    grafana_grafana_host: grafana
    grafana_renderer_host: renderer
    grafana_database_type: postgres
    grafana_database_name: grafana
    grafana_database_user: grafana
    grafana_database_password: *grafana_database_password
  dashboards:
    - name: main
      repo_url: https://github.com/stackmon/apimon-tests.git
      repo_ref: main
      path: dashboards/grafana
```

see also https://github.com/stackmon/cloudmon/blob/main/etc/sample_config.yaml#L67

Change the repo_url here also when you forked your own apimon-test repository with your customized tests.

In the next section the parameters for the plugins will be set.

Set the environment (production in our case), the plugins to use, the inventory group names, test_project name and the tasks.

```
matrix:
  # Mapping of environments to test projects
  # Regular apimon project in env ext
  - env: production
    monitoring_zone: internal
    # TODO: placing db_url here feels questionable
    # db_url or db_entry as ref to database.databases
    db_entry: apimon.apimon
    plugins:
      - name: apimon
        schedulers_inventory_group_name: schedulers
        executors_inventory_group_name: executors
        #epmons_inventory_group_name: epmons
        tests_project: apimon
        tasks:
          - scenario1_token.yaml
```

This example for a task is a very simple test for the identity service which requests token and deletes them right after successful creation.

The corresponding playbook:

https://github.com/stackmon/apimon-tests/blob/main/playbooks/scenario1_token.yaml

It calls a phyton script which does the job.

The last part of the config holds the cloud credentials as they will be written into the clouds.yaml of the test executor containers.

```
clouds_credentials:
  p1:
    auth:
      auth_url: https://fake.com
      username: fake_user
      password: fake_pass
      project_name: fake_project
      user_domain_name: fake_domain
  swift1:
    profile: otc
    auth:
      auth_url: https://fake.com
      username: fake_user
      password: fake_pass
      project_name: fake_project
      user_domain_name: fake_domain
```

see also https://github.com/stackmon/cloudmon/blob/main/etc/sample_config.yaml#L126

This will look familiar to you.

### ansible/inventory/hosts

In the next step you will have to build your ansible inventory file. We will need hosts for

- schedulers
- executors
- statsd
- graphite
- postgres

in our example.

```
---
all:
  hosts:
    cloudmon:
      ansible_host: 192.168.0.123
      internal_address: 192.168.0.123
  children:
    statsd:
      hosts:
        cloudmon
    grafana:
      hosts:
        cloudmon
    graphite:
      hosts:
        cloudmon
    schedulers:
      hosts:
        cloudmon
    executors:
      hosts:
        cloudmon
    postgres:
      hosts:
        cloudmon
```

We use the hosts file located in ansible/inventory/. Take care of proper dns name resolution (you local dns, /etc/hosts) or - like in our example - use ansible_host and internal_address. Change the ansible_host and internal_address accordingly to your instance ips!

**NOTE:** Without internal address grafana fails to start (GF_DATABASE_HOST in /etc/grafana/env)

Test your inventory. Are all hosts up and running? And accessable via ssh? Mind the key forwarding! ;-)

```
(py310) ubuntu@deploy-test:~/cloudmon$ ansible -i ansible/inventory/hosts -m ping all
```

## Deployment

The deployment is done via cloudmon command:

```
(py310) ubuntu@deploy:~/cloudmon$ cloudmon --config etc/config.yaml --inventory /home/ubuntu/cloudmon/ansible/inventory/ provision
```

This will deploy the basic containers as graphite, postgres, grafana, statsd and grafana-renderer.

**NOTE:** The path to the inventory must be an _absolute_ path.

In the next step will install the apimon plugin.

```
(py310) ubuntu@deploy:~/cloudmon$ cloudmon --config etc/config.yaml --inventory /home/ubuntu/cloudmon/ansible/inventory/ provision --plugin apimon
```

## Verify proper operation

After a few minutes you should see the first graphs in your Grafana Dashboard. Login with admin and the password you just in your config.yaml.

You should find a dashboard Identity Service Statistics in a Folder CloudMon when you browse your available dashboards. When everything is fine you should see your first graph with the title _API calls duration_. The second graph titled _Highest API calls duration_ should be showing a "No data". The second graph will show only data when the call duration exceeds 1000 ms.

The lower section _Scenario results_ will list a timestamp, the name of the playbook (scenario1_token.yaml), a job id, the result code and the URL to the log written to your swift bucket. The log contains the output of the playbook execution.

If anything goes wrong you should be able to see errors here.
