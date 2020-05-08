# Wordpress Server Full Setup With Monitoring

## Table of Contents

  - [What is this even?](#what-is-this-even)
  - [What can I expect this thing to even do?](#what-can-i-expect-this-thing-to-even-do)
    - [Monitoring Stack](#monitoring-stack)
    - [Services](#services)
  - [How do I customize it for me?](#how-do-i-customize-it-for-me)
    - [Configuration Variables](#configuration-variables)
  - [How Do I Work This Thing?](#how-do-i-work-this-thing)
    - [List of Requirements](#list-of-requirements)
    - [The Actual Steps](#the-actual-steps)
        - [Some Notes From Me](#some-notes-from-me)
  - [Future To-Dos](#future-to-dos)

## What is this even?

This deployment is designed to be used on CentOS 7. It is a full server configuration designed for running and monitoring/managing a wordpress server. It will be securely serving traffic with automatic ssl renewal. There will be long time performance metrics gathered ( 1month ) so you can see how the servers performance compares to the past. This is helpful in knowing when you may be exceeding the capabilities of your server and need to scale up the infrastructure. In future updates there will be alerting and monitoring configured to alert you before things become a problem via various methods such as slack or email. This way you can provide the best available infrastructure to fit you and your users needs in a way that saves you some time. Not me so much cause I wrote this. lol.

## What can I expect this thing to even do?

When you deploy this playbook using the steps in the section [How Do I Work This Thing?](#how-do-i-work-this-thing) you can expect the following things to accessible. In the below example we'll consider the following variables to be set in the host variables:

  ```yaml
  traefik_domain: "badasswebsite.example"
  wordpress_dns_name: "www.{{ traefik_domain }}"
  ```

Service | Endpoint
-- | --
Grafana | `grafana.badasswebsite.example`
Traefik | `monitor.badasswebsite.example`
Wordpress | `www.badasswebsite.example` or `badasswebsite.example`
Wordpress Login | `www.badasswebsite.example/wp-login.php`
Portainer | `port.badasswebsite.example`
Prometheus | `prom.badasswebsite.example`

You will set the grafana and traefik passwords via the ansible variables, but the portainer login will be created by the first user to access it. So be sure to get that configured before sending out to other people or they may set your password and you'll lose access. If this happens, simply kill the container, remove the `{{container_data_dir}}/portainer` directory and re-run the playbook.

### Monitoring Stack

Service | Info | Purpose
--- | --- | ---
cAdvisor | [docs](https://github.com/google/cadvisor) | cAdvisor reads the data from the docker sock to gather and expose performance information about the running containers to Prometheus
Prometheus | [docs](https://github.com/prometheus/prometheus) | Prometheus scrapes the targets cAdvisor, NodeExporter, VictoriaMetrics, and itself for exposed metrics to store for 5 days maximum. Data is also written simultaneously to VictoriaMetrics by Prometheus
Grafana | [docs](https://github.com/grafana/grafana) | Grafana is configured to query the metric data from VictoriaMetrics in order to graph and display it in an easily consumable format.
VictoriaMetrics | [docs](https://github.com/VictoriaMetrics/VictoriaMetrics) | VictoriaMetrics is configured as the long term TSDB Backend. Retention period of data is 1 month. It also is the queryy point for Grafana to request data to display as it returns information faster than Prometheus.
NodeExporter | [docs](https://github.com/prometheus/node_exporter) | Monitors the host system and exposes performance metrics to prometheus to retrieve via it's configured scrape job.

### Services

Service | Info | Purpose
--- | --- | ---
Wordpress | [docs](https://github.com/WordPress/WordPress) | Open Source Blog and website platform. Use this to design and manage your website.
Mysql | [docs](https://hub.docker.com/_/mysql) | SQL DB for Wordpress Deployment
Traefik | [docs](https://github.com/containous/traefik) | Traefik uses the labels on the docker containers to get valid ssl certificates for the configured domains. It also acts as a router for the docker containers and entrypoiont for the dns requests.
Portainer | [docs](https://github.com/portainer/portainer) | Portainer is used to manage the running docker services via a remotely accessible dashboard. View logs, run commands, and manage all without connecting in Terminal.

---

## How do I customize it for me?

### Configuration Variables

Variable Name | Type | Default Value |  Description
--- | --- | --- | ---
container_data_dir | String | `/data` | Default Data directory to store all container data with in.
monitoring_docker_network | String | `monitoring` | The Docker network name for the monitoring containers.
traefik_docker_network | String | `web` | The Docker network name for traefik to watch.
traefik_ssl_email | String | ❌ | Email address for SSL LETSENCRYPT Certificate.
traefik_htpasswd | String | ❌ | Admin username and password for Traefik access. Generate this in the command line with: `htpasswd -nb admin <secure password here>` copy and paste the whole line.
traefik_domain | String | ❌ | Your domain name.
grafana_admin_password | String | ❌ | Plaintext password to use for your grafana admin user access.
server_hostname | String | `wordpress-server` | The server hostname that will be set as part of installing the required resources.
server_software | List | `['@development', 'nano', 'docker', 'python3', 'wget', 'epel-release']`| List of software to install to ensure that all required packages are installed you can add to this list or simply use the below default if necessary. If you add to this list, copy and paste the below list into your host_vars file and add to it to avoid any issues. Accepts YAML Formatted lists also.
service_control | Dictionary | `{ state: "started", enabled: "yes", name: "docker" }` | List of services and if you are enabling or disabling them. You can  add to this list or use the default as below, if you add to this list, copy and paste the below list into your host_vars file and add to.
mysql_db_name | String | `wordpressdb` | Wordpress MySQL DB Name. If you don't want to change it, leave it undefined.
mysql_db_user | String | `wordpressdbuser` | Wordpress MySQL DB Username. If you don't want to change it, leave it undefined.
mysql_db_password | String | `wordpressdbpass` | Wordpress MySQL DB Password. If I were you I'd put it into an ansible Vault variable. Don't use the vault_pass.txt in this repo tho, change the password in it first.
wordpress_dns_name | String | `wordpress.{{ traefik_domain }}` | DNS Name if you want to direct your site like blah.com instead of a subdomain to the main domain.

---

## How Do I Work This Thing?

### List of Requirements

I hate to break it to ya, but this does require some things to use this, but no worries. It won't be anything more than the normal needs to host your own website.

Thing | Why You Need It
--- | ---
Your Own Domain Name with DNS Control | You will need to configure a few DNS records in order for all of this to work properly.
A Server to run this on with a public IP | I recommend Digital Ocean. This can run on the smallest droplet size just fine for most needs. Can't beat 5$ a month amiright?

### The Actual Steps

Well, to start with clone this repo. Then proceed to the below steps.

*NOTE: There is a vault_pass.txt and ansible.cfg file in this repo already. the values can be modified to fit your needs, but I often found when I was beginning to use things like this that there were no good examples for how someone might actully configure a common full deployment and thought this could serve as a good example.*

1. Modify the file `host_vars/wordpress-host.yml` and adjust the variables to fit your needs. This includes at least updating the following variables
    - `traefik_ssl_email`
    - `traefik_htpasswd`
    - `grafana_admin_password`
    - `traefik_domain`

2. Modify the file `inventory/hosts` and put in the public IP address of your server.
    - Example: replace `wordpress-host ansible_host=IP GOES HERE` with `wordpress-host ansible_host=123.45.67.789`

3. Add the following records to your DNS, the exact steps vary on providers so here's the records needed:
    - URL REDIRECT RECORD for @ to `http://www.<yourdomainhere>/`
    - A RECORD for `www` to `<YOUR SERVER IP>` TTL 1M
    - A RECORD for `monitor` to `<YOUR SERVER IP>` TTL 1M
    - A RECORD for `grafana` to `<YOUR SERVER IP>` TTL 1M
    - A RECORD for `port` to `<YOUR SERVER IP>` TTL 1M
    - A RECORD for `prom` to `<YOUR SERVER IP>` TTL 1M

4. Wait for 5-10 min for the dns records to propogate. That way you can avoid any timeouts or issues related to the automatic ssl creation.

5. Run the command: `ansible-playbook -i inventory/hosts main.yml --tags="init"`
    - This will run a full configuration to install all dependencies and then deploy the dockerized services.

6. Access the `port.*` domain endpoint and create your login. Confirm all of the containers are running under the containers tab, all should be green. if not, then you can check the logs by clicking the icon that looks like a log. Use the [above grid](#monitoring-stack) to read the docs to fix any startup errors. 

### Available Ansible Tags

Tag Name | Description
--- | ---
init | Run full playbook.
setup | Installs docker and service dependencies.
portainer | deploys all portainer dependencies and container.
portainer_dir | creates the portainer data folder.
portainer_container | deploys the portainer container.
traefik | deploys all traefik dependencies and container.
traefik_dir | creates the traefik  data folder.
acme | checks if the acme.json file exists or not, if it doesn't it'll be created in the traefik data directory.
traefik_config | deploys just the traefik config rendered from the template file `traefik.toml.j2`.
traefik_container | deploys the traefik container.
monitoring | deploys all the monitoring resources and dependencies.
cadvisor | deploys all cadvisor dependencies and container.
cadvisor_permissions | cadvisor requires some special permissions to access the data it needs to gather docker metrics and this handles their creation.
cadvisor_container | deploys the cadvisor container.
victoriametrics | deploys victoriametrics dependencies and container.
victoriametrics_dir | creates the data directory for victoriametrics.
victoriametrics_container | deploys the victoriametrics container only
nodeexporter | deploys nodeexporter container.
prometheus | deploys prometheus dependencies and container.
prometheus_dir | creates the data directory for prometheus.
prometheus_config | deploys prometheus configuration files.
prometheus_container | deploys prometheus container.
grafana | deploys all grafana dependencies and container.
grafana_dir | creates the data directories for grafana.
grafana_container | deploys the grafana container.
wordpress | deploys all wordpress and mysql dependencies and containers
wordpress_dir | creates the wordpress data directories.
wordpress_container | deploys the wordpress container.
mysql | deploys only mysql dependencies and containers.
mysql_dir | creates mysql data directories.
mysql_container | deploys the mysql container.

#### Some Notes From Me

If the dns records aren't propogated before traefik and the other containers are up then the ssl certs will fail their authentication. this means you'll need to wwait til they propogate and try again. I recommend using https://www.whatsmydns.net/ to check this.

I tested this repeatedly on my own before uploading but if you find any problems please submit an issue and I can get them sorted out. If you have a feature request feel free to submit as well

## Future To-Dos

- Additional Grafana Dashboards customized for this stack
- Alerting Rules\Configuration
- Add Images to Documentation
