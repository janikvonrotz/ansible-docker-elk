
# Ansible Docker ELK

Deploy (B)ELK stack with Ansible Docker and monitor the Docker deployment itself.

This Ansible project deploys a fully configured ELK stack secured by an Nginx proxy. For each service there is an Ansible role. Services can be deployed independently.

Roles:

* clean - Cleanup task for all docker containers
* docker - Checks if docker daemon is up and running
* elasticsearch - Deploys elasticsearch container
* kibana - Deploys Kibana container
* logstash - Deploys Logstash container
* nginx - Deploys Nginx proxy with LetsEncrypt certificates
* heartbeat - Deploys uptime metric container
* heartbeat-watcher - Custom node app that watches the heartbeat index
* metricbeat - Deploys container that collects host system metrics
* filebeat - Deploys container that forwards log files

## Requirement

The target Docker host requires the following packages:

* Docker
* Pip `sudo apt-get install python-pip`
* Ansible Docker module `sudo su; pip install docker`
* Passwordless sudo for ansible user

## Setup

Create a password file.

`echo "$VAULTPASSWORD" > .vault_pass`

Make it executable.

`chmod 600 .vault_pass`

### Localhost

Deploying to localhost requires local ssh access.

Install ssh server.

`sudo apt install openssh-server`

Copy the public key.

`echo $SSHKEY >> ~/.ssh/authorized_keys`

Enable passwordless sudo login.

`sudo /bin/bash -c "echo \"$USERNAME     ALL=(ALL) NOPASSWD:ALL\" >> /etc/sudoers"`

Test ssh access.

`ssh $USERNAME@localhost`

## Deployment

Test connection

`ansible all -m ping -i inventory`

Deploy elk stack

`ansible-playbook -i inventory elk.yml`

Deploy role only

`ansible-playbook -i inventory elk.yml --tags logstash`

Deploy role to localhost

`ansible-playbook -i inventory elk.yml --tags nginx --extra-vars "ehosts=local"`

Deploy role to localhost with non-default user

`ansible-playbook -i inventory elk.yml --tags nginx --extra-vars "ehosts=local" -u username`

Clean elk stak

`ansible-playbook -i inventory elk-clean.yml`

Clean role only

`ansible-playbook -i inventory elk-clean.yml --tags logstash`

## Development

Lint the project using Ansible lint.

`ansible-lint elk.yml elk-clean.yml`

## Manual Setup

If the ELK stack has been successfully deployed, you need to make manual configurations.

Create these index pattern:

```
filebeat-*
metricbeat-*
heartbeat-*
syslog-*
```

Ensure maximum index size with by configuring the defualt lifecycle policies:

```
filebeat:
Maxium index size: 5 GB
Maxium documents: 15'000'000
Maxium age: 7 days

heartbeat:
Maxium index size: 1 GB
Maxium documents: 3'000'000
Maxium age: 7 days


metricbeat:
Maxium index size: 1 GB
Maxium documents: 3'000'000
Maxium age: 7 days
```

Enable Delete phase for all indexes with 1 hour from rollover.

Create lifecycle policy for syslog:

```
Policy name: syslog
Maximum index size: 5 GB
Maxium documents: 15'000'000
Maxium age: 7 days
Enable delete phase: 1 day from rollover
```

### TODO

- [ ] Create and set the password of the elastic and kibana user automatically
- [ ] Encrypt connection to logstash beat and syslog input
- [ ] Ensure localhost deployment works with generated certificates
- [ ] Document role variables
- [x] Check if the setup index template task is necessary for every beats role

### Architecture

List of connections:

External

* Logdrain → syslog://$HOSTNAME:5000 (!) this connection is not secured.
* Kibana dashboard → https://$HOSTNAME
* Elasticsearch → https://$HOSTNAME:9200
* Logstash beats → tcp://$HOSTNAME:5044 (!) this connection is not secured.

Kibana

* Elasticsearch → https://$HOSTNAME:9200

Nginx

* Kibana dashboard → http://kb01:5601
* Elasticsearch → http://es01:9200

Logstash

* Elasticearch → http://es01:9200

Heartbeat

* Logstash → tcp://$HOSTNAME:5044

Heartbeat Watcher

* Elasticsearch → https://$HOSTNAME:9200
* SMTP → smtp://$SMTPHOST:587

Metricbeat

* Logstash → tcp://$HOSTNAME:5044

Filebeat

* Logstash → tcp://$HOSTNAME:5044

# Troubleshooting

Collection of problems and soltions.

### unsupported loc

During the installation of the python docker package the following error occurs:

```
locale.Error: unsupported locale setting
```

We need to change the locale settings. Set it to us english in bash `vim ~/.bashrc`.

```
export LC_ALL="en_US.UTF-8"
export LC_CTYPE="en_US.UTF-8"
```

Then run the installation again.