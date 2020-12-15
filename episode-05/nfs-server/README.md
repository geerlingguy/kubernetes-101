# NFS Server Setup Playbook

This playbook configures a simple NFS server, available over a node's Private IP address, for use by a Kubernetes cluster.

To run the playbook, make sure Ansible is installed, edit the `inventory` file and make sure your server's IP address is entered, and run:

    ansible-galaxy install geerlingguy.nfs
    ansible-playbook main.yml

Note: There is NO SECURITY on this server. This is for testing only. Don't use this configuration for production. It's also not setup for HA/failover. No redundancy... basically this was set up for a demo and this playbook should not be used in a production environment.
