# MLMOS
## Homework 3
### Docker swarm

* Infrastructură
* Docker swarm
* Compose files
   * Registry
   * Gitlab
   * Gitlab Runner
* Runner registration

## 1. Infrastructură
Crearea a trei mașini virtuale, identificate prin hostname-urile **manager.swarm**, **gitlab.swarm**, **runner.swarm**, folosind [tutorialul acesta](https://courses.tss-yonder.com/resurse/tutorial/virtualbox/instalare-centos-7/).

**IMPORTANT**: pentru mașina cu hostname-ul de **gitlab.swarm** trebuie setat 2GB de RAM în loc de 1GB.

## 1.1 Pachete
* Instalare **yum-utils**.
  * `yum install -y yum-utils`
* Instalare **docker**
  1. `yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo`
  2. `yum install -y docker-ce`
  3. `systemctl enable docker`, `systemctl start docker`

## 1.2 Set hostname
În cadrul procesului de instalare sau folosind comanda `hostnamectl`.

### VM 1 - manager.swarm
* `hostnamectl set-hostname manager.swarm`

### VM 2 - gitlab.swarm
* `hostnamectl set-hostname gitlab.swarm`

### VM 3 - runner.swarm
* `hostnamectl set-hostname runner.swarm`

## 1.3 Crearea conexiunii către interfața Host-Only (enp0s8)
* `ifup enp0s8` - trebuie executată pe toate mașinile
  * reținerea adreselor IP asignate pentru fiecare mașină virtuală (adresa poate fi aflată prin comanda `ip a`)
## 2. Docker swarm
### 2.1 Inițializarea swarm-ului
* `docker swarm init --advertise-addr ip-adress-of-the-enp0s8-adapter-on-the-manager.swarm-machine` - pe mașina desemnată a fi **manager**
  * output-ul comenzii `docker swarm init` reprezintă comanda ce trebuie executată pe celelalte două mașini pentru a le adăuga în swarm
    * e.g. `docker swarm join --token very-long-token ip-of-enp0s8-adapter-on-the-manager.swarm-machine:2377`

### 2.2 Adăugarea celorlate noduri
În cazul în care se pierde token-ul în terminalul pe care s-a executat `docker swarm init`, acesta poate fi aflat cu comanda `docker swarm join-token -q worker` (va returna **join token**-ul necesar pentru a adăuga un nod de tip **worker** în swarm).

* `docker swarm join --token very-long-token ip-of-enp0s8-adapter-on-the-manager.swarm-machine:2377`

### 2.3 Comenzi utile
*  Vizualizarea infrastructurii nodurilor
   * `docker node ls`
* Vizualizarea infrastructurii serviciilor
   * `docker stack ls`
* Inspectarea stării unui serviciu
  * `docker service ps {name-of-service}`
* Adăugarea unui label pe un nod din swarm
  * `docker node update --label-add {label-name}={label-value} {hostname-of-node}`
* Crearea/actualizarea serviciilor
  * `docker stack deploy -c {name-of-yml-file} {name-of-service-or-app}`

## 3. Compose files
Crearea **compose file**-urile, într-un director la alegere, pe mașina cu rolul de **manager** (e.g. hostname=manager.swarm)

### 3.1 Registry
* setare label pentru mașina cu rolul de manager - `docker node update --label-add registry=true manager.swarm`
* obținerea imaginii - `docker pull registry:latest` - executată pe mașina cu rolul de **manager**
* Crearea directoarelor necesare registry-ului, pe mașina cu rolul de **manager**
  * `mkdir -p /docker/registry-data`
* crearea fișierului **registry.yml** cu următorul conținut (identarea este importantă!) pe mașina cu rolul de **manager**
```
version: '3'

services:
  registry:
    image: registry:latest
    ports:
      - 5000:5000
    volumes:
      - /docker/registry-data:/var/lib/registry
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.labels.registry == true
```
* deploy la serviciu - `docker stack deploy -c registry.yml registry` pe mașina cu rolul de **manager**
* verificare că deploy-ul a fost făcut cu succes - `docker stack services registry` - ar trebui la coloana **Replicas** să fie valoarea **1/1**.

### 3.2 Gitlab
* setare label pentru mașina care va hosta serviciul gitlab (e.g. gitlab.swarm) - `docker node update --label-add gitlab=true gitlab.swarm`
* obținerea imaginii - `docker pull gitlab/gitlab-ce:latest` - executată pe mașina cu hostname-ul **gitlab.swarm**
* setarea **SELINUX** la **permissive** pe mașina cu hostname-ul **gitlab.swarm** - `sed** -i 's/SELINUX=[a-z]*/SELINUX=permissive/g' /etc/selinux/config`
* Crearea directoarelor necesare gitlab pe mașina cu hostname-ul **gitlab.swarm**
  * `mkdir -p /srv/gitlab/config`
  * `mkdir -p /srv/gitlab/logs`
  * `mkdir -p /srv/gitlab/data`
* crearea fișierului **gitlab.yml** cu următorul conținut (identarea este importantă!) pe mașina cu rolul de **manager**
```
version: '3'

services:
  gitlab:
    image: gitlab/gitlab-ce:latest
    hostname: 'gitlab.example.com'
    environment:
      GITLAB_OMNIBUS_CONFIG: |
        external_url 'http://gitlab.example.com:8929'
    ports:
      - 80:80
      - 443:443
      - 22:22
      - 8929:8929
    volumes:
      - /srv/gitlab/config:/etc/gitlab
      - /srv/gitlab/logs:/var/log/gitlab
      - /srv/gitlab/data:/var/opt/gitlab
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.labels.gitlab == true
      restart_policy:
        condition: on-failure
```
* deploy la serviciu - `docker stack deploy -c gitlab.yml gitlab` pe mașina cu rolul de **manager**
* pornirea acestui serviciu durează între 3 și 5 minute. Putem urmări status-ul serviciului folosind comanda `watch -d docker service ps gitlab_gitlab` - a se urmări coloana **Current State**.
* după ce acesta apare cu state-ul de **running** putem accesa serviciul gitlab folosind **ip-ul interfeței enp0s8** de pe mașina *gitlab.swarm* și **portul 8929**.
* după resetarea parolei e necesar crearea unui cont, apoi a unui repository pe acel cont. La secțiunea CI/CD din setările repo-ului se dezactivează **Auto DevOps**. Se deschide apoi rubrica **Runners** pentru a folosi token-ul la înregistrarea serviciului **gitlab-runner**.

### 3.3 Gitlab Runner
* setare label pentru mașina care va hosta serviciul runner (e.g. runner.swarm) `docker node update --label-add runner=true runner.swarm`
* obținerea imaginii - `docker pull gitlab/gitlab-runner:latest` - executată pe mașina cu hostname-ul **runner.swarm**
* Crearea directoarelor necesare gitlab pe mașina cu hostname-ul **runner.swarm**
  * `mkdir -p /srv/gitlab-runner/config`
* crearea fișierului **runner.yml** cu următorul conținut (identarea este importantă!) pe mașina cu rolul de **manager**
```
version: '3'

services:
  runner:
    image: gitlab/gitlab-runner:latest
    volumes:
      - /srv/gitlab-runner/config:/etc/gitlab-runner
      - /var/run/docker.sock:/var/run/docker.sock
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.labels.runner == true
      restart_policy:
        condition: on-failure
```
* deploy la serviciu - `docker stack deploy -c runner.yml runner` pe mașina cu rolul de **manager**

### 4. Gitlab Runner Registration
[Tutorial](https://docs.gitlab.com/runner/register/index.html#docker)
* executarea comenzii `docker run --rm -t -i -v /srv/gitlab-runner/config:/etc/gitlab-runner gitlab/gitlab-runner register` pe mașina pe care rulează serviciul de runner (e.g. runner.swarm) și introducerea datelor pentru register
  * http://ip-of-enp0s8-adapter-from-gitlab.swarm-machine:8929
  * token - se găsește în panoul de control al repository-ul creat pe instanța locală de gitlab, la **secțiunea CI/CD -> Runners**
  * numele runner-ului
  * tag-uri
  * executor -> docker
  * docker image -> alpine:latest
* după terminarea cu succes a procesului de registration se poate verifica că runner-ul este activ în setările repository-ului, la secțiune **CI/CD -> Runners**
