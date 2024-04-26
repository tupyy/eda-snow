# How to connect ServiceNow instance to Event-Driven Ansible

## Setup ServiceNow

The [EDA Notification app]( https://store.servicenow.com/sn_appstore_store.do#!/store/application/cb4182e69767611049b0b9dfe153af15/1.0.0) needs to installed in the snow instance. 

## Setup reverse proxy on Event-Driven Ansible

This rulebook uses the default event source plugin [webhook](https://github.com/ansible/event-driven-ansible/blob/main/extensions/eda/plugins/event_source/webhook.py) provided by `ansible.eda` collection.

This plugin setup a server listen on the provided port. 

To expose this port on `https`, you need to install a reverse-proxy such as `caddy` listen on the provided port.
For example, if you use `podman`, the following yml installed `caddy` on port `8088` and passing traffic to port `5000` of `ansible.eda.webhook`:
```yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: caddy-file
data:
  Caddyfile: |
    {
        http_port 80
        https_port 8088
        email someone@example.com
        auto_https disable_redirects
    }
    <eda_host_fqdn>:8088 {
        reverse_proxy localhost:5000 {
             lb_policy first
        }
    }
---
apiVersion: v1
kind: Pod
metadata:
  name: eda-snow
spec:
  containers:
    - name: caddy
      image: docker.io/caddy:latest
      volumeMounts:
      - name: config-volume
        mountPath: /etc/caddy/
  volumes:
    - name: config-volume
      configMap:
        name: caddy-file
```

> You need to open port `80` because Let's Encrypt will send the challages on this port.

To create the pod run:
```bash
podman kube play caddy.yml
```

## Setup Event-Driven Ansible

How to setup a rulebook is beyond this documentation. You can find more information in the official documentation or by visiting [Colin McNaughton github](https://github.com/cloin/).

## Events

An event sent by SNow app looks like:
```json
{
  "payload": {
    "state": "1",
    "active": "1",
    "impact": "3",
    "notify": "1",
    "number": "INC0014332",
    "sys_id": "b75d67151bb9ca10ed59419ead4bcb5c",
    "urgency": "3",
    "approval": "not requested",
    "category": "inquiry",
    "made_sla": "1",
    "priority": "5",
    "severity": "3",
    "caller_id": "6816f79cc0a8016401c5a33be04be441",
    "knowledge": "0",
    "opened_at": "2024-04-25 18:47:04",
    "opened_by": "6816f79cc0a8016401c5a33be04be441",
    "escalation": "0",
    "sys_domain": "global",
    "upon_reject": "cancel",
    "reopen_count": "0",
    "sys_mod_count": "1",
    "upon_approval": "proceed",
    "incident_state": "1",
    "sys_class_name": "incident",
    "sys_created_by": "admin",
    "sys_created_on": "2024-04-25 18:47:04",
    "sys_updated_by": "admin",
    "sys_updated_on": "2024-04-25 18:47:05",
    "child_incidents": "0",
    "sys_domain_path": "/",
    "short_description": "Test incident",
    "reassignment_count": "0",
    "task_effective_number": "INC0014332",
    "hierarchical_variables": "variable_pool"
  }
}
```
