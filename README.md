# Ansible Role: Keepalived

[![License: GPL v3](https://img.shields.io/badge/License-GPLv3-blue.svg)](https://www.gnu.org/licenses/gpl-3.0)

An Ansible role to install and configure Keepalived for high availability (HA) virtual IP (VIP) management across Linux distributions.

## Overview

This role installs and configures the Keepalived service, which provides load balancing and high availability through VRRP (Virtual Router Redundancy Protocol). It enables automatic failover of virtual IP addresses between nodes.

The examples demonstrate rolling deployments using the `keepalived_priority_override` variable to temporarily lower node priority, forcing VIPs to fail over to other nodes during maintenance or updates.

## Use cases

### Databases
Use a VIP to connect to Database clusters, the clients use the VIP to connect instead of each specific node name/IP. When a database node is down, the clients are swapped to other node. For an intervention on one of the nodes, the Keepalived priority is changed, forcing the VIP node swap, and again, making the clients swap node gracefully.

### Load Balancers

It can be used to configure a VIP on active-passive, or active-active load balancer configuration.

This pattern can be used on an active-passive layout or also more complicated for example 3 nodes with one or multiple VIPs.

## Requirements

- Ansible 2.9 or higher
- Linux-based systems (Ubuntu, Debian, CentOS, RHEL)
- Root or sudo access on target hosts

## Role Variables

### Required Variables

- `keepalived_vrrp_instances` - List of VRRP instance configurations (see examples below). Each
  `virtual_ip` entry's `dev <interface>` clause is optional -- per keepalived's own behavior, a VIP
  with no `dev` is added to the vrrp_instance's own (auto-detected) interface, so it generally
  doesn't need to be specified at all; only set it if a specific VIP must live on a different NIC
  than the one VRRP itself uses.

### Default Variables

See [defaults/main.yml](defaults/main.yml) for all available variables:

- `keepalived_packages` - Package list to install (default: `["keepalived"]`)
- `keepalived_health` - Health check script configuration
- `keepalived_connected_network_interface` - Kept for backward compatibility only. Each
  `vrrp_instance`'s interface is now auto-detected internally by the role (matching
  `unicast_src_ip`, or `ansible_host` if unset, against each interface's actual bound address), so
  it works no matter how the OS/hypervisor names the NIC (`eth0`, `ensX`, `enpXsY`, ...) and no
  matter how many NICs a host has. Set `interface:` on an individual `keepalived_vrrp_instances`
  entry to override this for that instance. **Never hardcode an interface name** (e.g. `eth0`/`eth1`)
  in a consuming playbook's `virtual_ip`/`interface` values -- that reintroduces the exact bug this
  auto-detection exists to avoid, and silently breaks the next time a host's NIC gets renamed by a
  migration.
- `keepalived_priority_override` - Temporary priority override for rolling deploys 
- `keepalived_global_defs` - Options of keepalived's `global_defs` block, as `option: value`
  (default: `{}`, keepalived's own defaults). See [Gratuitous ARP](#gratuitous-arp).
- `keepalived_vip_guard` - Drop the traffic that reaches a node for a VIP held by another node
  (default: `false`). See [Gratuitous ARP](#gratuitous-arp).
- `keepalived_service_force_restart` - Restart keepalived even if its configuration did not change
  (default: `false`). A changed configuration is applied with a reload, see
  [Applying a configuration change](#applying-a-configuration-change).

### Gratuitous ARP

When a BACKUP takes a VIP, for example because the adverts of the MASTER were lost during a network
problem, it sends gratuitous ARP and the clients learn its MAC for the VIP. When the MASTER wins the
VIP back without having left the MASTER state, keepalived does not send gratuitous ARP by default, so
a client can keep sending the VIP's traffic to the other node. If that node still answers the
connections it had open (for example through Docker's NAT), the client's ARP entry stays confirmed
and never expires.

These options of `keepalived_global_defs` make the MASTER announce the VIP again:

```yaml
keepalived_global_defs:
  vrrp_garp_master_refresh: 60          # every 60 s while MASTER
  vrrp_garp_master_refresh_repeat: 2
  vrrp_garp_lower_prio_repeat: 5        # when a MASTER receives an advert with a lower priority
  vrrp_higher_prio_send_advert: true    # when a MASTER steps down, so the other MASTER sends them
```

They exist in keepalived 2.0 and later.

`keepalived_vip_guard: true` also stops the old node from confirming the wrong ARP entry. It
installs `/usr/local/sbin/keepalived-vip-guard`, which adds one rule per IPv4 VIP of the node and
interface of the VIP's network, in the `raw` table, before Docker's NAT:

```
-A PREROUTING -d <VIP>/32 -i <interface> -m addrtype ! --dst-type LOCAL -m comment --comment keepalived-vip-guard -j DROP
```

- `addrtype ! --dst-type LOCAL` is checked for each packet, so the rule never matches while the node
  holds the VIP, and needs no change when the VIP moves.
- `-i` limits it to the traffic from the network, so local containers can still reach a VIP held by
  another node.
- A client that sends the VIP's traffic to this node gets no answer, its ARP entry expires, and it
  finds the node that holds the VIP, in about a minute with the kernel's defaults.

The script runs before keepalived starts (systemd drop-in `keepalived.service.d/vip-guard.conf`),
so the rules come back after a reboot, and the role runs it again when the VIPs change or the rules
are missing. `keepalived-vip-guard --check` exits with 1 when the rules are not the current ones.
The `filter` table rules of a firewall role are not touched. To remove it, delete the drop-in and
the script, run `systemctl daemon-reload`, and delete the rules with the `keepalived-vip-guard`
comment from the `raw` table.

### Applying a configuration change

A changed configuration is applied with a reload, which keeps each node's state. A restart would
remove the VIPs of the MASTER, a BACKUP would take them, and the MASTER would take them back when it
starts: each VIP would move twice. Lowering the priority with `keepalived_priority_override` still
moves the VIP once, because the BACKUP takes it when it receives the lower priority advert.

## Usage

### Basic Playbook Example

```yaml
- name: Configure Keepalived
  hosts: ha_servers
  serial: 1  # Deploy sequentially to avoid disruption
  become: true
  roles:
    - keepalived
```

### Rolling Deploy Pattern

For zero-downtime deployments, temporarily lower priority to force VIP failover:

```yaml
- name: Deploy with rolling VIP failover
  hosts: service_servers
  serial: 1
  become: true
  tasks:
    - name: Lower keepalived priority to force VIP swap
      import_role:
        name: keepalived
      vars:
        keepalived_priority_override: 1
      when: keepalived_vrrp_instances is defined

    # Deploy your application updates here

    - name: Restore keepalived priority
      import_role:
        name: keepalived
      vars:
        keepalived_priority_override: ""
      when: keepalived_vrrp_instances is defined
```

## Configuration Examples

### Simple Active-Passive Example (2 Nodes)

For a basic Redis failover setup:

```yaml
redis_keepalived_is_primary: "{{ groups['redis_docker_servers'][0] == inventory_hostname }}"
redis_keepalived_state: "{{ 'MASTER' if redis_keepalived_is_primary else 'BACKUP' }}"
redis_keepalived_peer_ipv4: "{{ hostvars[ (groups['redis_docker_servers'] | difference([inventory_hostname]))[0] ].ansible_host }}"
redis_keepalived_priority: "{{ 200 if redis_keepalived_is_primary else 100 }}"

redis_keepalived_vrrp_instances:
  - name: redis_failover_link_ipv4
    ipv6: false
    state: "{{ redis_keepalived_state }}"
    id: 41
    priority: "{{ keepalived_priority_override if ( keepalived_priority_override | default('') | string | length > 0 ) else redis_keepalived_priority }}"
    pass: "{{ redis_keepalived_pass }}"
    peer_ip: "{{ redis_keepalived_peer_ipv4 }}"
    track_script: redis_chk_script
    virtual_ip:
      - "{{ redis_virtual_ipv4 }}/24 label vipRedis"

redis_keepalived_health:
  - name: redis_chk_script
    # Custom health check script
    script: /usr/bin/make -C /nau/ops/redis/ --no-print-directory silent-healthcheck
    timeout: 9
    interval: 10
    
    # Or use a simple TCP port check:
    # script: /usr/bin/nc -z 127.0.0.1 {{ redis_docker_port }}
    # interval: 2
    # timeout: 3
```

### Advanced Active-Active Example (4 VIPs, 2 Nodes)

Configuration for load balancers with dual-stack IPv4/IPv6 and active-active setup:

**Inventory:**

```ini
[balancer_servers]
lb01-qa.nau.fccn.pt  ansible_host=172.XX.1.XX ansible_public_ipv4=193.136.192.175 ansible_public_ipv6=2001:690:a00:4001::175
lb02-qa.nau.fccn.pt  ansible_host=172.XX.1.YY ansible_public_ipv4=193.136.192.176 ansible_public_ipv6=2001:690:a00:4001::176
```

**Variables:**

```yaml
# VIP Configuration
balancer_virtual_ipv4_1: "193.136.192.180"
balancer_virtual_ipv6_1: "2001:690:a00:4001::180"
balancer_virtual_ipv4_2: "193.136.192.183"
balancer_virtual_ipv6_2: "2001:690:a00:4001::183"

# Keepalived VRRP Instances (4 VIPs: 2 IPv4 + 2 IPv6)
# VIP1 is MASTER on node 1, BACKUP on node 2
# VIP2 is MASTER on node 2, BACKUP on node 1
# Dual-NIC example: each instance sets unicast_src_ip to the public IP, so the role resolves
# `interface` to the public NIC automatically (matching that IP), regardless of which NIC has the
# host's default route, and regardless of what either NIC is named.
balancer_keepalived_vrrp_instances:
  - name: failover_link_ipv4_1
    ipv6: false
    state: "{{ balancer_keepalived_state_1 }}"
    id: 41
    priority: "{{ keepalived_priority_override if ( keepalived_priority_override | default('') | string | length > 0 ) else balancer_keepalived_priority_1 }}"
    pass: CHANGEME1
    peer_ip: "{{ balancer_keepalived_peer_ipv4 }}"
    unicast_src_ip: "{{ ansible_public_ipv4 }}"
    track_script: chk_script
    virtual_ip:
      - "{{ balancer_virtual_ipv4_1 }}/24 label vipLB1"
      
  - name: failover_link_ipv6_1
    ipv6: true
    state: "{{ balancer_keepalived_state_1 }}"
    id: 61
    priority: "{{ keepalived_priority_override if ( keepalived_priority_override | default('') | string | length > 0 ) else balancer_keepalived_priority_1 }}"
    pass: CHANGEME2
    peer_ip: "{{ balancer_keepalived_peer_ipv6 }}"
    unicast_src_ip: "{{ ansible_public_ipv6 }}"
    track_script: chk_script
    virtual_ip:
      - "{{ balancer_virtual_ipv6_1 }}/64 label vipLB1"

  - name: failover_link_ipv4_2
    ipv6: false
    state: "{{ balancer_keepalived_state_2 }}"
    id: 44
    priority: "{{ keepalived_priority_override if ( keepalived_priority_override | default('') | string | length > 0 ) else balancer_keepalived_priority_2 }}"
    pass: CHANGEME3
    peer_ip: "{{ balancer_keepalived_peer_ipv4 }}"
    unicast_src_ip: "{{ ansible_public_ipv4 }}"
    track_script: chk_script
    virtual_ip:
      - "{{ balancer_virtual_ipv4_2 }}/24 label vipLB2"
      
  - name: failover_link_ipv6_2
    ipv6: true
    state: "{{ balancer_keepalived_state_2 }}"
    id: 62
    priority: "{{ keepalived_priority_override if ( keepalived_priority_override | default('') | string | length > 0 ) else balancer_keepalived_priority_2 }}"
    pass: CHANGEME4
    peer_ip: "{{ balancer_keepalived_peer_ipv6 }}"
    unicast_src_ip: "{{ ansible_public_ipv6 }}"
    track_script: chk_script
    virtual_ip:
      - "{{ balancer_virtual_ipv6_2 }}/64 label vipLB2"

# State and Priority Logic
balancer_keepalived_state_1: "{{ 'MASTER' if ( groups['balancer_servers'][0] == inventory_hostname ) else 'BACKUP' }}"
balancer_keepalived_state_2: "{{ 'MASTER' if ( groups['balancer_servers'][1] == inventory_hostname ) else 'BACKUP' }}"
balancer_keepalived_peer_ipv4: "{{ hostvars[ (groups['balancer_servers'] | difference([inventory_hostname]))[0] ].ansible_public_ipv4 }}"
balancer_keepalived_peer_ipv6: "{{ hostvars[ (groups['balancer_servers'] | difference([inventory_hostname]))[0] ].ansible_public_ipv6 }}"
balancer_keepalived_priority_1: "{{ 200 if ( groups['balancer_servers'][0] == inventory_hostname ) else 100 }}"
balancer_keepalived_priority_2: "{{ 200 if ( groups['balancer_servers'][1] == inventory_hostname ) else 100 }}"

# Health Check
keepalived_health:
  - name: chk_script
    script: /usr/bin/nc -z -w 5 127.0.0.1 80
    interval: 2
    timeout: 3
```

### Multi-Service VIP Configuration Pattern

For environments with multiple services each requiring their own VIPs, aggregate configurations:

```yaml
# Aggregate all VRRP instances from different services
keepalived_vrrp_instances: "{{
  ( balancer_keepalived_vrrp_instances  | default([]) ) +
  ( redis_keepalived_vrrp_instances  | default([]) ) +
  ( elasticsearch_keepalived_vrrp_instances  | default([]) ) +
  ( richie_mysql_keepalived_vrrp_instances | default([]) ) +
  ( openedx_mysql_read_replica_keepalived_vrrp_instances | default([]) ) +
  ( xtradb_keepalived_vrrp_instances | default([]) ) +
  ( clickhouse_keepalived_vrrp_instances | default([]) ) +
  []
}}"

# Aggregate all health checks
keepalived_health: "{{
  ( balancer_keepalived_health | default([]) ) +
  ( redis_keepalived_health | default([]) ) +
  ( elasticsearch_keepalived_health | default([]) ) +
  ( richie_mysql_keepalived_health | default([]) ) +
  ( openedx_mysql_read_replica_keepalived_health | default([]) ) +
  ( xtradb_keepalived_health | default([]) ) +
  ( clickhouse_keepalived_health | default([]) ) +
  []
}}"
```

Then define service-specific variables in each `group_vars/<service>_servers.yml` file.

## Important Notes

### Serial Deployment

**Critical:** Always use `serial: 1` (or small numbers) when running playbooks that modify Keepalived configuration to prevent simultaneous priority changes that could cause service disruption:

```yaml
- name: Deploy with Keepalived
  hosts: ha_servers
  serial: "{{ serial_number | default(1) }}"
  become: true
  gather_facts: true
  # ...
```

### VRRP Instance IDs

Ensure VRRP instance IDs (`id:` parameter) are unique across all instances on the same network segment to avoid conflicts.

### Health Checks

Health check scripts should exit with:
- `0` - Service is healthy
- Non-zero - Service is unhealthy (triggers failover)

## Dependencies

None.


## Installation

On `requirements.yml` file add:

```yaml
roles:
- src: git+https://github.com/fccn/ansible-keepalived.git
  version: <hash>
```

Then install using:
```bash
ansible-galaxy install -p vendor/roles -r requirements.yml
```

Alternatively install it using git submodules.

git submodule add https://github.com/fccn/ansible-docker-deploy.git

## License

[GNU General Public License v3.0](LICENSE)

## Author Information

Created and maintained by **Ivo Branco** at [FCCN](https://www.fccn.pt/).

## Contributing

Contributions are welcome! Please open an issue or submit a pull request.

---

**Repository**: [fccn/ansible-keepalived](https://github.com/fccn/ansible-keepalived)

