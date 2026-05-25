# IPv6 Dual-Stack Deployments on Azure

## Overview

The Azure CPI supports **dual-stack (IPv4 + IPv6)** VMs with both address families on the same NIC.

> **Azure limitation:** IPv6-only VMs are not supported. Every NIC must include at least one IPv4 `ipConfiguration`. There is no single-stack IPv6 mode on Azure. See [IPv6 overview — Limitations](https://learn.microsoft.com/en-us/azure/virtual-network/ip-services/ipv6-overview#limitations).

## How It Works

### Azure Side

A dual-stack Azure subnet carries both an IPv4 CIDR and an IPv6 CIDR. For each grouped NIC, the CPI creates two `ipConfigurations`:

| ipConfiguration | IP version | Primary | Eligible attachments |
|-----------------|------------|---------|----------------------|
| `ipconfig<N>-0` | IPv4       | Yes     | IPv4 public IP, IPv4 load balancer pool, application gateway pool |
| `ipconfig<N>-1` | IPv6       | No      | IPv6 public IP and IPv6 load balancer pool |

### BOSH Side

The BOSH director uses `nic_group` to place network definitions on the same NIC. Grouped networks must reference the same Azure resource group, virtual network, and subnet; the CPI rejects mismatches and orders IPv4 first.

The CPI also sets an `alias` field (e.g. `eth0`) on the agent network settings so the bosh-agent can map both networks to the same OS interface without relying on MAC addresses (which Azure does not provide at NIC creation time).

Use manual BOSH networks for both families. Put the default DNS and gateway on IPv4, and keep NIC-level cloud properties consistent because the first network in each group supplies them.

## Prerequisites

1. **Azure virtual network with a dual-stack subnet.** The VNet needs IPv4 and IPv6 address spaces, and the IPv6 subnet prefix must be `/64`.

   ```bash
   # Create a dual-stack VNet
   az network vnet create \
     --resource-group my-rg \
     --name boshvnet \
     --address-prefixes 10.0.0.0/16 fd00::/48

   # Create a dual-stack subnet
   az network vnet subnet create \
     --resource-group my-rg \
     --vnet-name boshvnet \
     --name dual-stack-subnet \
     --address-prefixes 10.0.0.0/24 fd00::/64
   ```

2. **BOSH director with `nic_group` support.** The director must support RFC-0038 dual-stack networking ([`v282.1.0+`](https://github.com/cloudfoundry/bosh/releases/tag/v282.1.0)).

## Cloud Config

Define two separate networks (one IPv4, one IPv6) that reference the same Azure subnet:

```yaml
networks:
- name: default-ipv4
  type: manual
  subnets:
  - range: 10.0.0.0/24
    gateway: 10.0.0.1
    dns: [168.63.129.16]
    reserved: [10.0.0.1-10.0.0.3]
    cloud_properties:
      virtual_network_name: boshvnet
      subnet_name: dual-stack-subnet

- name: default-ipv6
  type: manual
  subnets:
  - range: fd00::/64
    gateway: fd00::1
    dns: [168.63.129.16]
    reserved: [fd00::1-fd00::3]
    cloud_properties:
      virtual_network_name: boshvnet
      subnet_name: dual-stack-subnet
```

## Deployment Manifest

Reference both networks in your instance group and assign them the same `nic_group`:

```yaml
instance_groups:
- name: my-instance-group
  networks:
  - name: default-ipv4
    nic_group: "1"
    default: [dns, gateway]
  - name: default-ipv6
    nic_group: "1"
```

## Load Balancers

For dual-stack load balancers, specify separate backend pools for IPv4 and IPv6 in your `vm_type`:

```yaml
vm_types:
- name: dual-stack-with-lb
  cloud_properties:
    instance_type: Standard_D2s_v3
    load_balancer:
      name: my-dual-stack-lb
      backend_pool_name: pool-v4
      backend_pool_name_v6: pool-v6
```

The CPI attaches `backend_pool_name` to IPv4 and `backend_pool_name_v6` to IPv6. Application gateway pools are IPv4-only. Load balancer and application gateway attachments apply only to the primary NIC group.

## Multi-NIC Dual-Stack

You can combine dual-stack with multiple NICs. Each NIC group gets its own `nic_group` value:

```yaml
instance_groups:
- name: multi-nic-instance
  networks:
  - name: net-v4-primary
    nic_group: "1"
    default: [dns, gateway]
  - name: net-v6-primary
    nic_group: "1"
  - name: net-v4-secondary
    nic_group: "2"
  - name: net-v6-secondary
    nic_group: "2"
```

This creates two dual-stack NICs. The VM size must support the required NIC count.
