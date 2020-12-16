# NFS Server Setup Playbook

This playbook configures a simple NFS server, available over a node's Private IP address, for use by a Kubernetes cluster.

## Cloud Server

You should set up a standalone VPS that can be reached over private network by your Kubernetes cluster. For example, in Linode, create a simple Linode with a private IP address.

I chose to create a 'Dedicated 4GB' linode ($30/month), which comes with 80 GB of storage.

## Ansible Playbook

To run the playbook:

  1. Make sure Ansible is installed
  2. Edit the `inventory` file and make sure your server's public IP address is entered.
  3. Make sure you can log into the server.

Then run the Ansible commands necessary to configure the server:

    ansible-galaxy install geerlingguy.firewall geerlingguy.nfs
    ansible-playbook main.yml

> **NOTE**: There is _minimal_ security configuration on this server. This setup is for testing only. Don't use this configuration for production. It's also not setup for HA/failover. No redundancy... basically this was set up for a demo and this playbook should not be used in a production environment.
