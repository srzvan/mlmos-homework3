# Homework 3

## Swarm nodes
* `node01.mlmos.homework3` - manager
* `node02.mlmos.homework3` - worker

* `docker swarm init --advertise-addr 192.168.95.101` - manager
* `systemctl stop firewalld` - pentru a evita eroare de conectare de la worker la swarm
* setare `--label-add registry=true` pentru manager
* *registry.yml* - pentru configurarea registry-ului
* `docker stack deploy -c registry.yml registry`
