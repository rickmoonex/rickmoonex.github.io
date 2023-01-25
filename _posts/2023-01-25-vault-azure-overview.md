---
title: "Highly Available HashiCorp Vault on Azure - #1 architectural overview"
date: 2023-01-25
permalink: /posts/2023/01/vault-azure-deploy/
tags:
  - Microsoft Azure
  - HashiCorp Vault
---

This is the first blog of a new series about running a highly available [HashiCorp Vault](https://www.vaultproject.io) cluster on Microsoft Azure. This series will cover a lot, including: Design, Provisioning, Configuration and Maintenance.

In this first installment we are going to discuss the technical design of the Vault cluster that we are building. So let's get started!

## Criteria

Before we begin designing the cluster's infrastructure, let's set some criteria first.

The cluster must meet de following criteria:

- Has a high availability storage backend that supports replication and failover;
- Has a load balancing solution with backend health checks;
- Intra-cluster traffic must be encrypted through mTLS;
- Provides an automatic unsealing method for the Vault, eliminating the need to manually unseal nodes when the cluster scales;
- Allows for Azure AD integrations through Machine Identities.

When these are met we can consider our cluster Highly-Available, Secure and Production-Ready.

## Design

Now that we have set our criteria, let's get into the design of the cluster. I've created a diagram of the infrastructure that we're going to use:

<img src='/images/vault-azure-arch.png'>

As you can see, it's a pretty simple design. I chose for simplicity because it's more manageable when scaling the cluster.

Let's dive into each component of this design:

### Application Gateway

The Application Gateway is the entrypoint into our cluster. It will act as a Layer 7 load balancing device. _"Why not a Layer 2 load balancer?"_ you might ask. Well, because Vault uses client certificate authentication for traffic going to the API. We cannot use a Layer 2 load balancer to do SSL terminations unfortunately. Also, Vault can redirect request between cluster nodes. This can cause instability when using a Layer 2 load balancer.

### Virtual Network and NAT gateway

For security reasons I have chosen to utilize a NAT gateway instead of giving each VM it's own public ip. There will also be a network security group that will be associated to the subnet where the Vault servers are located. This security group will follow the principle of least privilege, only opening the needed ports to the resources that need them.

### Key Vault

The key vault plays a critical role in our Vault cluster. It will hold the key that is used for unsealing the Vault servers. It also holds the root CA and server certificates needed to encrypt intra-cluster traffic. This key vault also contains the certificate that the Application Gateway will use to communicate with the servers, and the certificate that it will attach to it's HTTPS listener.

### Bastion Host

The use of a bastion host is obvious for security reasons. It will ensure that we can access the Vault servers without opening SSH ports to the public internet.

During deployment of the VMs we will generate an SSH key. The public key will be placed on the VMs and the private key will be placed inside Azure Key Vault. Bastion can than retrieve this key from the key vault and authenticate to the machine.

### VM Scale Set

The Virtual Machine Scale Set is where all the magic happens. This is where our actual Vault machines will live. By using a scale set when can easily scale the cluster up and down if needed and even implement auto scaling using the Azure Application Gateway we will be using.

We will provision these machines by creating a bash script that we will pass into the VM using the `user-data` property. This initiates the given script trough cloud-init. The script will authenticate the machine with the Azure CLI through a Managed Identity. It will then be able to use that identity to communicate with the Azure Key Vault to retrieve the needed certificates for mTLS and the unseal key. When this is done it will install and configure Vault with the retrieved secrets from Key Vault.

We will also install an extension into this scale set that is able to poll the Vault API endpoint to see if the machine is up and running. This can be used in monitoring, alerting and troubleshooting.

#### VM Storage

For the disks that will be used I chose Premium SSDs with LRS. This is because the storage replication solution that we are going to use requires very low-latency to operate. Therefor the use of Standard SSDs/HDDs is not recommended.

For storage replication we will use Vault's built-in [raft storage backend](https://developer.hashicorp.com/vault/docs/configuration/storage/raft). This backend will ensure high availability and replication of storage across all nodes in the cluster. Raft uses a 'leader-follower' system for high availability. During cluster operations raft is responsible for electing a 'leader' node. This leader is responsible for handling all API request that are coming into the cluster. The rest of the nodes enter a 'follower' state and do not handle API request, when it does get a incoming request it will automatically redirect the client to the leader node. Raft will constantly sync the storage between all the nodes in the cluster. So that when a node failure occurs, another leader node will be elected and cluster operations resume with only a couple of milliseconds downtime.

#### VM sizes

The size of the VM depends on your personal situation. How big of a load will be on the cluster?
I recommend starting with the `Standard_D2s_v3` or `Standard_D4s_v3` when initially starting your production cluster. Then when the load on the servers grows, upgrading to `Standard_D8s_v3` or `Standard_D16s_v3`

## What's next?"

_"What's next?"_ you may ask. Well, deploying of course!

In the next installment of this series we will cover the provisioning of this cluster using Terraform, so stay tuned!
