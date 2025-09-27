# On-prem Kubernetes Cluster (k0s) for OpenSearch Stack

## Requirements

- VPS: Ubuntu 24.04 LTS - 1x controller (2CPU/2GB/50GB), 3x workers (4CPU/4GB/100GB), 1x loadbalancer (1CPU/1GB/25GB)
- Network: Local network connectivity between VPS

## Kubernetes Cluster

- Built on VPS with local networking
- Based on **k0s**
- Deployed with **k0sctl**
- One controller and three worker nodes
- Load balancing with **HAProxy** in front of the cluster

## Log Observability Stack

Cluster components:

- **OpenSearch** – log aggregation and search
- **OpenSearch Dashboards** – visualization
- **cert-manager** – certificate and TLS management
- **ingress-nginx** – ingress controller
- **longhorn** - distributed storage management

External agents:

- **Fluent Bit** – parses and ships logs into OpenSearch

Fluent Bit handles:

- Apache logs
- Nginx logs
- Postfix logs
- Dovecot logs
- SSH activity logs

## Deployment

### 1. Provision Kubernetes Nodes

Run [prepare_k0s_node](https://github.com/HossBigft/ansible_playbooks/tree/main/prepare_k0s_node)
This playbook:

- Updates system.
- Disables swap.
- Limits size of system logs.

### 2. Set Up Load Balancer

Deploy HAProxy with [prepare_k0s_loadbalancer](https://github.com/HossBigft/ansible_playbooks/tree/main/prepare_k0s_loadbalancer)

This playbook automates:

- Updating the system.
- Ensures HAProxy is installed.
- Deploying ACL IP whitelist to control access.
- Deploying HAProxy templated configuration .
- Enables HAProxy as a service.

### 3. Deploy k0s Cluster

Generate config and bootstrap:

```bash
cd k0sctl
bash -c 'set -a; source .env; envsubst < k0sctl.yaml.template > k0sctl.yaml'
k0sctl apply
```

### 4. Deploy Observability Stack

#### Create longhorn GUI password

```bash
htpasswd -c logpass USERNAME
kubectl create secret generic longhorn-ui-password --from-file=logpass -n longhorn-system
```

#### Set initial opensearch admin password

```bash
read -s ADMIN_PASS && kubectl create namespace opensearch &&  kubectl create secret generic opensearch-admin-secret --from-literal=OPENSEARCH_INITIAL_ADMIN_PASSWORD="$ADMIN_PASS" -n opensearch
```

#### Use Helmfile to deploy cluster components:

```bash
cd helmfile
helmfile apply
```

This installs:

- OpenSearch
- OpenSearch Dashboards
- cert-manager
- ingress-nginx
- longhorn

### 5. Create OpenSearch user for logshipper

1.  Create `fluent_bit` role with rights to create and write to indices:

```bash
xh PUT https://$OPENSEARCH_HOST/_plugins/_security/api/roles/$USERNAME  \
  --auth $ADMIN_USER:$ADMIN_PASS \
  --verify=no \
  Content-Type:application/json \
  index_permissions:='[
    {
      "index_patterns": ["*"],
      "allowed_actions": [
        "write",
        "create_index"
      ]
    }
  ]'
```

2. Create user

```bash
xh --verify=no --auth $ADMIN_USER:$ADMIN_PASS PUT https:///_plugins/_security/api/internalusers/$USERNAME  password=$USER_PASS
```

3. Map user to role

```bash
xh --auth $ADMIN_USER:$ADMIN_PASS --verify=no PUT https://$OPENSEARCH_HOST/_plugins/_security/api/rolesmapping/fluent_bit  Content-Type:application/json users:='["$USERNAME"]'
```

### 6. Deploy Fluent Bit logshipper

Run [playbook](https://github.com/HossBigft/ansible_playbooks/tree/main/setup_fluent_bit_plesk) on monitored servers.
This playbook:

- Installs Fluent Bit across RedHat/Debian hosts using custom repo setup.
- Deploys parsers and configs from templates (with inventory- and vars-driven values).
- Sets systemd overrides for logging, firewall rules, and ensures OpenSearch accessibility.
- Ensures the service is started, enabled, and working.
