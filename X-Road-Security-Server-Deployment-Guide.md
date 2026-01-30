<h1 align="center">X-Road Security Server Deployment Guide</h1>

## Introduction

NISS (Nordic Institute for Interoperability Solutions) provides two deployment options:

1. On virtual machines using the [Ansible Playbook](https://github.com/nordic-institute/X-Road/tree/develop/ansible)
2. In a Kubernetes cluster using [X-tee Helm Chart](https://koodivaramu.eesti.ee/x-tee/x-road-helm)

Supported operating systems for VM deployments are Ubuntu 22.04 and Red Hat 9. The recommended database for the current version of the Security Server is PostgreSQL 16.

The X-Road Security Server uses several ports for data exchange between members and for server administration. These ports are:
- 5500 (HTTPS) - Public (must be accessible from the internet), used for data exchange between security servers.
- 5577 (OCSP) - Public (must be accessible from the internet), used for data exchange between security servers.
- 8080 (HTTP) - Internal only, used by services to access the X-Road network.
- 8443 (HTTPS) - Internal only, used by services to access the X-Road network.
- 4000 (HTTPS) - Internal only, used for the Administration UI.

Before deployment, please contact ISAA at x-road@isaa.am to get the configuration anchor and your member code. This information is required for the post-deployment Security Server configuration.

## Virtual Machine deployment using Ansible
To deploy the Security Server using Ansible, first clone the X-Road Git repository (https://github.com/nordic-institute/X-Road) and go to the ansible folder. In that folder, we need to create the following hosts file:

../ansible/hosts/hosts.txt
```ini
[ss_servers]
${VM-IP-ADDRESS} ansible_ssh_user=${VM-USER}

[example:children]
ss_servers

[ss_servers:vars]
variant=vanilla
```

Where:
- VM-IP-ADDRESS - the virtual machine’s IP address and SSH port if it’s not the default.
- VM-USER - the user on the virtual machine with sudo access, or root.

X-Road distributions support a setup where the database is installed on the same machine as the server, <span style="color: red;">which we do not recommend</span>, even for testing environments. Instead, specify a remote database instance in the Security Server variable file. Here is an example configuration:

> [!Note] 
> Please make sure that PostgreSQL listens on the interface connected to the network (the default is localhost), and that pg_hba.conf is configured to accept requests from your network for all users.

../ansible/vars_files/ss_database.yaml
```shell
database_host: "127.0.0.1:5432"
database_admin_password: "superuser-password" ## password of postgres user
```

The ``database_admin_password`` should always be the password of the ``postgres`` user. The superuser is only used when setting up the database for the server and is not used by the server afterward.

After that, enable operational monitoring if it’s not already enabled:

../ansible/vars_files/base.yaml
```shel
xroad_install_opmonitoring: true
```

Lastly, deploy the Security Server using the following command:

```shell
ansible-playbook -i hosts/hosts.txt xroad_init.yml
```

## Kubernetes deployment using Helm Chart
To deploy the Security Server using the Helm chart, clone the X-Road-Helm Git repository (https://koodivaramu.eesti.ee/x-tee/x-road-helm) and go to the helm/security-server folder.

In that folder, we need to edit the values.yaml file of the chart to:
#### 1. Set the Security Server version to 7.7.0.

```yaml
image:
  ...
  primaryTag: "7.7.0-primary"
  secondaryTag: "7.7.0-secondary"
```

#### 2. Set the Security Server PIN and create the first admin user (from the Kubernetes Secret).

```yaml
envSecretName: ${XROAD-USER-SECRET}
```

Where XROAD-USER-SECRET is a pre-created Kubernetes Secret that must contain the following keys:
- ``XROAD_TOKEN_PIN``
- ``XROAD_ADMIN_USER``
- ``XROAD_ADMIN_PASSWORD``

The ``XROAD_TOKEN_PIN`` must be a passphrase with upper and lower case letters, symbols, and numbers (minimum length is 10).

#### 3. Provide the database secret with the superuser password (from the Kubernetes Secret).

```yaml
dbHost: "your-db-hostname"
dbSecretName: {XROAD-DB-SECRET}
```

Where XROAD-DB-SECRET is a pre-created Kubernetes Secret that must contain the ``password`` key with the ``postgres`` user’s password as the value.

#### 4. Provide the SSH keys so the secondary pods can synchronize their configuration from the primary pod.

```
sshSecretName: ${XROAD-SSH-SECRET}
```

Where XROAD-SSH-SECRET is a pre-created Kubernetes Secret that must contain the following keys:
- ``private-key``
- ``public-key``


#### 5. Apply Helm chart 

At the end, adjust the PVC volume sizes and service type if necessary. The defaults are:

```yaml
serviceType: NodePort
...
etcVolumeSize: 100Mi
libVolumeSize: 1000Mi
```

And deploy to the cluster using the following command:

```shell
helm install security-server -n test-ss -f helm/security-server/values.yaml helm/security-server 
```

Once deployed, follow the instructions in the [X-Road Security Server Configuration Guide](./X-Road-Security-Server-Configuration-Guide.md) to continue the installation.