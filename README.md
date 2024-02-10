# Prometheus

This repository covers my personal [Prometheus](https://prometheus.io) instances, [Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/), [node_exporter](https://github.com/prometheus/node_exporter) and [snmp_Exporter](https://github.com/prometheus/snmp_exporter) (which connects to multiple [Synology](https://www.synology.com) devices). It's used to observe parts of my network infrastructure and collect data from my [Sonnen battery](https://sonnen.de) in combination with [thinkpace/sonnen_exporter](https://github.com/thinkpace/sonnen_exporter).

It covers setup, upgrade and backup of this instance. I publish it because it might help anybody to setup a similar solution.

# Drawbacks

There are some drawbacks in this scenario which I would like to point out:

* Retention time of Sonnen prometheus is configured for 1 year. Default Prometheus retention time is 15 days, so there is huge difference. You definitly need to observe storage usage and adapt retention time if necessary.
* There is no encryption or authentication/authorization included, so I suggest to run this only in a protected environment (like your LAN ... hopefully 😉) and don't publish this service to the Internet.
* [Admin API](https://prometheus.io/docs/prometheus/latest/querying/api/) is enabled and accessible without authentication/authorization.
* [Management API](https://prometheus.io/docs/prometheus/latest/management_api/) is enabled and accessible without authentication/authorization.
* docker-compose.yml is referencing to the latest images available in Docker Registry. In a commercial environment, this is often seen as an anti pattern because it could lead to unintended updates and broken systems. However, in my specific scenario, I want to refer to the latest available image. If something fails unintended, I need to fix it.

# Requirements

* Ansible 2.7 or higher
* Docker runtime environment with docker compose
* Please have a look at [ansible_playbook.yml](ansible/ansible_playbook.yml) and understand what's happening.

Install Prometheus on hosts of group "prometheus":

```
- hosts: prometheus
  become: true
  roles:
    - install-prometheus
```

Install Alertmanager on hosts of group "prometheus":

```
- hosts: prometheus
  become: true
  roles:
    - install-prometheus-alertmanager
```

Install node_exporter on hosts of group "prometheus_node_exporter" (containing the clients you want to observe):

```
- hosts: prometheus_node_exporter
  become: true
  roles:
    - prometheus.prometheus.node_exporter
  vars:
    node_exporter_enabled_collectors: ["systemd", "processes", {"textfile": {"directory": "{{ node_exporter_textfile_dir }}"}}]
```

Install snmp_exporter on hosts of group "prometheus_snmp_exporter" (please note that this is the host running snmp_exporter, not the hosts to observe!):

```
- hosts: prometheus_snmp_exporter
  become: true
  roles:
    - install-snmp_exporter
```

# Setup

First of all, install Prometheus collection from Ansible Galaxy:

```
ansible-galaxy collection install prometheus.prometheus
```

After that, please instantiate Ansible var samples:

```
cd /YOUR/REPO/CHECKOUT/PATH/ansible
cp roles/install-prometheus/vars/main-sample.yml roles/install-prometheus/vars/main.yml
cp roles/install-prometheus-alertmanager/vars/main-sample.yml roles/install-prometheus-alertmanager/vars/main.yml
```

Open `roles/install-prometheus/vars/main.yml` in your prefered editor and adapt settings. If you would like to use backup, please make sure configured backup path is available.

Open `roles/install-prometheus-alertmanager/vars/main.yml` in your prefered editor and adapt prometheus config according to your requirements. In my setup, Alertmanager is using [Pushover](https://pushover.net/) to send notifications.

[main.yml](ansible/roles/install-prometheus/tasks/main.yml) is refering to multiple files, which do not exist in this repo:

* `ansible/roles/install-prometheus/files/scrape_configs.yml`
* `ansible/roles/install-prometheus/files/scrape_configs_sonnen.yml`
* `ansible/roles/install-prometheus/files/alertrules.yml`
* `ansible/roles/install-prometheus/files/alertrules_sonnen.yml`

Please create them according to your needs, find some sample content below:

`scrape_configs.yml`:
```
scrape_configs:
- job_name: prometheus
  honor_timestamps: true
  metrics_path: /metrics
  scheme: http
  static_configs:
  - targets:
    - localhost:9090
    - localhost:9091
- job_name: node
  honor_timestamps: true
  metrics_path: /metrics
  scheme: http
  static_configs:
  - targets:
    - node_target1.internal:9100
    - node_target2.internal:9100
- job_name: 'snmp-exporter'
  static_configs:
  - targets:
    - snmp_target1.internal
    - snmp_target2.internal
  metrics_path: /snmp
  params:
    module: [synology]
  relabel_configs:
    - source_labels: [__address__]
      target_label: __param_target
    - source_labels: [__param_target]
      target_label: instance
    - source_labels: [__param_target]
      regex: (.*)
      replacement: snmp_exporter.internal:9116
      target_label: __address__
```

`scrape_configs_sonnen.yml`:
```
scrape_configs:
- job_name: sonnen_v2_status
  honor_timestamps: true
  scheme: http
  metrics_path: /probe
  params:
    module: ['sonnen_v2_status']
    target: ['http://SONNEN_BATTERY.internal/api/v2/status']
  static_configs:
  - targets:
    - SONNEN_EXPORTER.internal:7979
- job_name: sonnen_v2_battery
  honor_timestamps: true
  scheme: http
  metrics_path: /probe
  params:
    module: ['sonnen_v2_battery']
    target: ['http://SONNEN_BATTERY.internal/api/v2/battery']
  static_configs:
  - targets:
    - SONNEN_EXPORTER.internal:7979
- job_name: sonnen_v2_inverter
  honor_timestamps: true
  scheme: http
  metrics_path: /probe
  params:
    module: ['sonnen_v2_inverter']
    target: ['http://SONNEN_BATTERY.internal/api/v2/inverter']
  static_configs:
  - targets:
    - SONNEN_EXPORTER.internal:7979
- job_name: sonnen_v2_configurations
  honor_timestamps: true
  scrape_interval: 1d
  scheme: http
  metrics_path: /probe
  params:
    module: ['sonnen_v2_configurations']
    target: ['http://SONNEN_BATTERY.internal/api/v2/configurations']
  static_configs:
  - targets:
    - SONNEN_EXPORTER.internal:7979
```

`alertrules.yml` and `alertrules_sonnen.yml`:
```
groups:
- name: PV Notifications
  rules:
  - alert: battery_l_15
    expr: sonnen_v2_battery_relativestateofcharge_percent < 15
    labels:
      category: photovoltaik
      severity: info
      frequency: daily
    annotations:
      summary: 'Battery {{ $value  }}, less than 15%'
```

To execute this Ansible playbook, you can use following command:

```
ansible -i /PATH/TO/INVENTORY /YOUR/REPO/CHECKOUT/PATH/ansible/ansible_playbook.yml
```

## Start

After setup, all containers needs to get started executing

```
docker compose up -d
```

in setup directory (for Prometheus, Alertmanager and snmp_exporter).

## Upgrade

Ansible role is creating a cronjob which updates base images every night.

# Backup

Backup script triggers a snapshot using Prometheus Admin API. A tgz containing this snapshot will be created at configured backup_path (see [main-sample.yml](/ansible/roles/install-prometheus/vars/main-sample.yml)).
