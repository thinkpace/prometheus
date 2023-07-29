# Prometheus

This repository covers my personal [Prometheus](https://prometheus.io) instance. At the time of writing, it's mainly used to collect data from my [Sonnen battery](https://sonnen.de) in combination with [thinkpace/sonnen_exporter](https://github.com/thinkpace/sonnen_exporter).

It covers setup, upgrade and backup of this instance. I publish it because it might help anybody to setup a similar solution.

# Drawbacks

There are some drawbacks in this scenario which I would like to point out:

* Retention time of prometheus is configured for several years. Default Prometheus retention time is 15 days, so there is huge difference. You definitly need to observe storage usage and adapt retention time if necessary.
* There is no encryption or authentication/authorization included, so I suggest to run this only in a protected environment (like your LAN ... hopefully 😉) and don't publish this service to the Internet.
* Admin API is enabled and accessible without authentication/authorization.
* docker-compose.yml is referencing to the latest Prometheus available in Docker Registry. In a commercial environment, this is often seen as an anti pattern because it could lead to unintended updates and broken systems. However, in my specific scenario, I want to refer to the latest available image. If something fails unintended, I need to fix it.

# How to use

Requirements:

* Have a server with Docker installed

## Setup

There are two ways to setup this repository:

1. Ansible setup
1. Manual setup

Please choose one of them.

### Ansible setup

To setup Prometheus using Ansible please ensure to have at least Ansible 2.7 installed at your system. If that's the case, run following commands:

```
cd /YOUR/REPO/CHECKOUT/PATH/prometheus.git/ansible
cp roles/install-prometheus/vars/main-sample.yml roles/install-prometheus/vars/main.yml
cp roles/install-prometheus/files/prometheus-sample.yml roles/install-prometheus/files/prometheus.yml
```

Open `roles/install-prometheus/vars/main.yml` in your prefered editor and adapt settings. If you would like to use backup, please make sure configured backup path is available.

Open `roles/install-prometheus/files/prometheus.yml` in your prefered editor and adapt prometheus config according to your requirements.

To run this role and install Prometheus at your server, you can use following command:

```
cd /YOUR/REPO/CHECKOUT/PATH/prometheus.git/ansible
ansible -i /PATH/TO/INVENTORY -b YOURSERVERNAME -m ansible.builtin.include_role --args name=install-prometheus
```

### Manual setup

To setup Prometheus the manual way, clone this repository and adapt [prometheus.yml](/prometheus.yml) and [docker-compose.yml](/docker-compose.yml) if necessary.

## Start

After setup, Prometheus can be started using

```
docker compose up
```

or 

```
docker compose up -d
```

in setup directory.

This will start Prometheus at port 9090.

## Upgrade

Ansible role is creating a cronjob which updates base image every night. If you installed manually, you can use following cronjob:

```
0 4 * * * docker compose -f /opt/prometheus/docker-compose.yml down && docker compose -f /opt/prometheus/docker-compose.yml pull && docker compose -f /opt/prometheus/docker-compose.yml up -d
```

# Backup

Backup script triggers a snapshot using Prometheus Admin API. A tgz containing this snapshot will be created at configured backup_path (see [main-sample.yml](/ansible/roles/install-prometheus/vars/main-sample.yml)).
