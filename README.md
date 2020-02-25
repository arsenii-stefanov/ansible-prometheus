Ansible-Prometheus
=========

## Components

* `Prometheus` (monitoring server - scrapes mertics from remote exporters/agents exposing metrics via HTTP)

* `Alertmanager` (alerting server created by Prometheus Authors - recieves alerts from Prometheus via an HTTP API according to rules)

* `Consul` (service discovery - usually, used for static servers)

* `Grafana` (visualization - JSON dashboards that query Prometheus periodically)

* `Blackbox Exporter` (pinger, HTTP checker - you might want to have several exporters like this located in different regions)

* `Domain Exporter` (domain expiration checker)

## Intro

> {{ playbook_dir }} is a default Ansible variable. This is where you keep your playbooks, variables, config files and templates

> Ansible roles you install with Ansible-Galaxy can be stored elsewhere. I used this variable to give you a better understanding of where all these files should be stored

## Config example with a minimum set of parameters required to set up a Prometheus server

* `FILE: {{ playbook_dir }}/playbook.yml`

```
- hosts: all
  connection: ssh
  become: true
  become_user: root
  become_method: sudo
  any_errors_fatal: true
  gather_facts: true

  pre_tasks:

    - name: Include Variables
      include_vars: "{{ item }}"
      with_items:
        - "prometheus-secrets.yml"
        - "prometheus-monitoring.yml"

  roles:
     # this one is optional (see the requirements.yml below)
   - { role: "ansible-basic-server-setup", when: skip_basic_server_setup is not defined or ( skip_basic_server_setup is defined and skip_basic_server_setup == false ) }
     # you will need Python modules for Ansible to interact with the Docker API (see the requirements.yml below)
   - { role: "ansible-python", when: skip_basic_server_setup is not defined or ( skip_basic_server_setup is defined and skip_basic_server_setup == false ) }
     # of course, you will need Docker itself (see the requirements.yml below)
   - { role: "ansible-docker", when: skip_basic_server_setup is not defined or ( skip_basic_server_setup is defined and skip_basic_server_setup == false ) }
     # this is the main Prometheus role (see the requirements.yml below)
   - { role: "ansible-prometheus" }
     # You can use a 'skip_basic_server_setup' variable to speed up your Ansible play during further config updates, etc. 
```

* `FILE: {{ playbook_dir }}/requirements.yml`

```
# Basic server setup
- src: https://github.com/arsenii-stefanov/ansible-basic-server-setup.git
  path: roles
  scm: git
# Set up Python
- src: https://github.com/arsenii-stefanov/ansible-python.git
  path: roles
  scm: git
# Set up Docker
- src: https://github.com/arsenii-stefanov/ansible-docker.git
  path: roles
  scm: git
# Set up Prometheus
- src: https://github.com/arsenii-stefanov/ansible-prometheus.git
  path: roles
  scm: git
```

* `FILE: {{ playbook_dir }}/vars/prometheus-secrets.yml`

```
grafana_database_password: <Grafana database password goes here>
grafana_admin_password: <Grafana admin password goes here>
grafana_secret_key: <Grafana secret key goes here>
```

Don't forget to encrypt the file with your secrets:

```
ansible-vault encrypt {{ playbook_dir }}/vars/prometheus-secrets.yml
```

* `FILE: {{ playbook_dir }}/vars/prometheus-monitoring.yml`

```
# Use the modern Python interpreter :)
ansible_python_interpreter: /usr/bin/python3

####################################
###     BASIC SERVER CONFIGS     ###
####################################

python_modules_general: [
  { name: "pip", state: "latest", extra_args: "--user", executable: "pip3" }
]
python_modules_docker: [
  { name: "docker", state: "present", extra_args: "--user", executable: "pip3" }
]

# If you include the `ansible-docker` role, don't forget to specify the packages, the basic one is 'docker-ce'
docker_packages_ubuntu: [ "docker-ce" ]

# Currently only Docker installation type is available
prometheus_installation_type: "docker"

# All your config files and Docker mounts will be stored here
prometheus_main_mount_dir: "/srv"

# Additional disks (it is recommended to have at least one disk for Prometheus and Grafana data)
prometheus_disk_mounts: [
    {
      src: "/dev/sdb",
      dest: "{{ prometheus_main_mount_dir }}",
      fstype: "ext4",
      fs_opts: "-m 0 -F -E lazy_itable_init=0,lazy_journal_init=0,discard",
      mount_opts: "discard,defaults,nofail",
      state: mounted
    }
]

# If you don't want to use an additional disk (not recommended), just set 'prometheus_disk_mounts' to an empty value
prometheus_disk_mounts: ""

# How long to store Prometheus metrics for
prometheus_retention_policy: "180d"

# Software versions (there are Docker image tags which you can find on the corresponding pages on https://hub.docker.com/ )
consul_version_docker: "1.6.2"
prometheus_version_docker: "v2.15.2"
prometheus_alertmanager_version_docker: "v0.20.0"
blackbox_exporter_version_docker: "v0.16.0"
domain_exporter_version_docker: "v1.4.0"
grafana_version_docker: "6.6.0"

# Currently, the following versions/docker-tags have been tested
# Consul: "1.6.2"
# Prometheus: "v2.15.2"
# Prometheus_alertmanager: "v0.20.0"
# Blackbox_exporter: "v0.16.0"
# Domain_exporter: "v1.4.0"
# Grafana: "6.6.0"

# Optionally, you can add pre-configured Datasources and Dashboards for Grafana. See the example below
# (config_templates_path, config_templates_path and other variables are defined in 'defaults/main.yml')

grafana_templates: [
    {
      src: "{{ config_templates_path }}/grafana/provisioning/dashboards/default.yml.j2",
      dest: "{{ grafana_provis_dashboards_host_dir }}",
      owner: "{{ grafana_owner }}",
      group: "{{ grafana_group }}",
      mode: "0755"
    },
    {
      src: "{{ config_templates_path }}/grafana/provisioning/datasources/default.yml.j2",
      dest: "{{ grafana_provis_datasources_host_dir }}",
      owner: "{{ grafana_owner }}",
      group: "{{ grafana_group }}",
      mode: "0755"
    }
]

grafana_files: [
    {
      src: "{{ config_files_path }}/grafana/dashboards/",
      dest: "{{ grafana_dashboards_host_dir }}",
      owner: "{{ grafana_owner }}",
      group: "{{ grafana_group }}",
      mode: "0755"
    }
]

# NGINX reverse proxy (with NGINX you can access your services by a DNS name and use the same port)
install_nginx_reverse_proxy: true

# Usually, internal services, such as Prometheus, Alertmanager and Consul should not be accessible from the Internel
# It is recommended to use a local domain for such services. As for Grafana, I would suggest setting a public DNS for Grafana
# but limiting access by IP (team VPN or office IP)
prometheus_domain: prometheus.example.local # If you prefer not to install NGINX, make sure you add the {{ prometheus_http_port }} variable to the URL
prometheus_url: "http://{{ prometheus_domain }}"
prometheus_alertmanager_domain: alertmanager.example.local
prometheus_alertmanager_url: "http://{{ prometheus_alertmanager_domain }}" # If you prefer not to install NGINX, make sure you add the {{ prometheus_alertmanager_http_port }} variable to the URL
consul_domain: consul.example.local
grafana_domain: grafana.example.com

# In case you decided to hide your real IP, or if your Prometheus instance is in an internal network (which is even better), you put
# Grafana (or the other services if you decided to make them public) behind a Load Balancer, or another reverse proxy, or CloudFlare DNS proxy
# You need to expose the real client IP to your NGINX reverse proxy. In mose cases you will use an 'X-Forwarded-For' header, although CloudFlare
# uses 'CF-Connecting-IP', there may be others

nginx_real_ip:
  set_real_ip_from:
    - ip: "35.35.25.25/32"
      comment: "My GCP Load Balancer"
  real_ip_header: "X-Forwarded-For"
  real_ip_recursive: "on"

nginx_allow_ips:
  - ip: "35.35.45.45/32"
    comment: "VPN"
  - ip: "130.211.0.0/22"
    comment: "GCP healthcheckers"
  - ip: "35.191.0.0/16"
    comment: "GCP healthcheckers"
  - ip: "10.128.0.0/9"
    comment: "GCP Internal network"
```

By default, you can put your regular config files in `{{ playbook_dir }}/config-files` and Jinja2 templates - in `{{ playbook_dir }}/config-templates`. Although, it is recommended to use the default file/diretcory structure and names, youc ou can override these vaules with corresponding variables (see the `defaults/main.yml` file for details)

Here is an example of how your directory structure would look:

```
./playbook_dir
├── config-files
│   ├── grafana
│   │   ├── dashboards
│   │   │   ├── Domains_Overview.json    # optional (Grafana dashboard)
│   │   │   ├── MySQL_Overview.json      # optional (Grafana dashboard)
│   │   │   └── Servers_Overview.json    # optional (Grafana dashboard)
│   │   └── provisioning
│   │       ├── dashboards
│   │       │   └── default.yml.j2       # optional (Grafana provisioning config, it'a a Jinja2 template because you pass the dashboards directory to it)
│   │       └── datasources
│   │           └── default.yml.j2       # optional (Grafana provisioning config, it'a a Jinja2 template because you pass datasource details to it)
│   └── prometheus
│       └── rules
│           ├── alerts-kubernetes.yml    # optional (Prometheus alert rules)
│           └── alerts.yml               # optional (Prometheus alert rules)
├── config-templates
│   ├── blackbox-exporter
│   │   └── blackbox.yml.j2              # mandatory (Blackbox Exporter main config file, it'a a Jinja2 template in case you need to pass some variables from an Ansible vars file)
│   ├── prometheus
│   │   └── prometheus.yml.j2            # mandatory (Prometheus main config file, it'a a Jinja2 template because you pass container names, IPs, AWS/GCP creds for discovery, etc. )
│   └── prometheus-alertmanager
│       └── alertmanager.yml.j2          # mandatory (Alertmanager main config file, it'a a Jinja2 template because you pass Slack or email creds to it for alerting)
```

`config-files/prometheus/rules` - put as many files with Prometheus rules as you wish; usually, alert rules for different projects differ from each other, so I would not use Jinja2 templates here

`config-templates/prometheus`, `config-templates/prometheus-alertmanager` - these files contain data that may be defined in other variables, such as Consul address, Slack API token, etc, so Jinja2 templates would work best here

#### You can override any variable defined in the 'defaults/main.yml'

## CLI Example

```
ansible-galaxy -vvvv install -r requirements.yml
ansible-playbook prometheus.yml -i inventory --limit="prometheus-server" --vault-password-file ./vault.py
```
