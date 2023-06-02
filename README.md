# estuary-postgres-playbook

🚧 Under construction 🚧
🚧 DO NOT USE, especially on Production EstuaryV1 PostgreSQL! 🚧
🚧 This playbook is transitioning to be a generally useful playbook instead, for deploying arbitrary PostgreSQL clusters 🚧

Builds a highly available PostgreSQL setup for Estuary using [Patroni](https://github.com/zalando/patroni) and [HAProxy](https://www.haproxy.org/).

Depends on having a working HAProxy deployment via estuary-haproxy-playbook. In future, this playbook will actually allow you to call that one as appropriate.


## Quick Start

1) Install the Ansible requirements:

`$ ansible-galaxy install -r requirements.yml`


2) This playbook requires a 'tls_bastion' server to be defined in the inventory, this is a server you're using exclusively for certificates/challenges etc. Using localhost is also an option if you don't have a tls_bastion server:
```
# add tls_bastion as localhost
[tls_bastion]
localhost ansible_connection=local
```


3) You must define a seperate inventory group for the {{ postgresql_cluster_name }} variable, for example if it's:
`postgresql_cluster_name: pchq_perthchat`

Means you'll need to configure a group for it in your inventory/hosts file like so:
```
[pchq_perthchat:children]
etcd_master
```


4) Define these extra variables in your ./inventories/pchq/group_vars/all.yml file:
```
internal_subnet: "10.1.1.0/16"                 # The internal subnet of your network
postgresql_cluster_name: pchq_perthchat        
patroni_postgresql_version: 15
patroni_install_haproxy: false
patroni_system_group: postgres
patroni_install_watchdog_loader: false         # This is needed as it disables a currently broken section
patroni_etcd_hosts: "{% for host in groups[postgresql_cluster_name] %}{{ hostvars[host]['inventory_hostname'] }}:2379{% if not loop.last %},{% endif %}{% endfor %}"  # yamllint disable-line rule:line-length
patroni_etcd_protocol: https
patroni_etcd_cacert: "/var/lib/patroni.pki/ca.pem"
patroni_etcd_cert: "/var/lib/patroni.pki/{{ inventory_hostname }}.pem"
patroni_etcd_key: "/var/lib/patroni.pki/{{ inventory_hostname }}-key.pem"
patroni_postgresql_connect_address: "{{ ansible_ens18.ipv4.address }}:5432"
patroni_restapi_connect_address: "{{ ansible_ens18.ipv4.address }}:{{ patroni_restapi_port }}"

patroni_postgresql_pg_hba:                                              # This section defines connection permissions between the postgresql hosts
  - { type: "local", database: "all", user: "all", method: "peer" }
  #- { type: "host", database: "all", user: "postgres", address: "{{ internal_subnet }}", method: "scram-sha-256" }
  - { type: "host", database: "replication", user: "{{ patroni_replication_username }}", address: "{{ internal_subnet }}", method: "scram-sha-256" } # Allow Patroni replication
  - { type: "host", database: "replication", user: "{{ patroni_replication_username }}", address: "127.0.0.1/32", method: "scram-sha-256" } # Allow local Patroni access
```


5) Run the playbook against your chosen inventory:
`$ ansible-playbook -v -i ./inventories/pchq site.yml`


## Troubleshooting Guide

Note: The certificates this script generates MUST have the IP as a 'Subject Alternative Name', test the pubkey at this URL to verify it: https://www.sslshopper.com/certificate-decoder.html


Once all that is done, you should have working PostgreSQL via the loadbalancer:
```
root@postgres02:~# patronictl -c /etc/patroni/postgres02.penholder.xyz.yml list
+ Cluster: main -----------+-------------+---------+---------+----+-----------+
| Member                   | Host        | Role    | State   | TL | Lag in MB |
+--------------------------+-------------+---------+---------+----+-----------+
| postgres01.penholder.xyz | 10.1.11.119 | Replica | running |  1 |         0 |
| postgres02.penholder.xyz | 10.1.11.118 | Leader  | running |  1 |           |
| postgres03.penholder.xyz | 10.1.11.117 | Replica | running |  1 |         0 |
+--------------------------+-------------+---------+---------+----+-----------+
```

And should be able to login to PostgreSQL.
```
root@estuary-pg01:~# psql -h 10.1.3.1 -U postgres -W
Password:
psql (14.5 (Debian 14.5-2.pgdg110+2))
Type "help" for help.

postgres=#
```

You can also visit the loadbalancer's floating IP on port 8080 to see a stats page:

[EXAMPLE LINK NEEDED]

This stats page shows you which node is the current leader (shown in green).

Keep a backup of the certificate material from the secure box, and (if needed) shut down and destroy the secure box once the material has been safely backed up and verified.
