# Devops-task
A task for Uno-soft

# Структура проекта:

* **docker-compose.yml** - Файл docker compose для поднятия кластера Cassandra.
* **Dockerfile** - Файл для сборки кастомного образа Cassandra для подключения к ноде по ssh.
* **nftables.conf** - Файл конфигурации для nftables.

## Задания:
1. __На машине А (ubuntu 24.04 lts) в локальной сети с ip 192.168.1.197 запускается скрипт docker-compose для поднятия 3 образов с ip адресами 192.168.1.200-202.__
Была создана виртуальная машина в VmWare Workstation Pro с адресом 192.168.1.197 и подняты образы Cassadndra с помощью docker compose.

```
version: "3.8"

networks:
  cassandra-net:
    driver: ipvlan
    driver_opts:
      parent: ens33
      ipvlan_mode: l2
    ipam:
      config:
        - subnet: 192.168.1.0/24
          gateway: 192.168.1.1

services:

  cassandra-1:
    image: "cassandra-ssh-key"
    container_name: "cassandra-1"
    networks:
      cassandra-net:
        ipv4_address: 192.168.1.200
    environment:
      - MAX_HEAP_SIZE=512M
      - HEAP_NEWSIZE=100M
      - CASSANDRA_START_RPC=true
      - CASSANDRA_RPC_ADDRESS=0.0.0.0
      - CASSANDRA_LISTEN_ADDRESS=192.168.1.200
      - CASSANDRA_BROADCAST_ADDRESS=192.168.1.200
      - CASSANDRA_CLUSTER_NAME=my-cluster
      - CASSANDRA_ENDPOINT_SNITCH=GossipingPropertyFileSnitch
      - CASSANDRA_DC=my-datacenter-1
    volumes:
      - cassandra-node-1:/var/lib/cassandra:rw
    restart: on-failure
    healthcheck:
      test: ["CMD-SHELL", "nodetool status"]
      interval: 2m
      start_period: 2m
      timeout: 10s
      retries: 3

  cassandra-2:
    image: "cassandra:latest"
    container_name: "cassandra-2"
    networks:
      cassandra-net:
        ipv4_address: 192.168.1.201
    environment:
      - MAX_HEAP_SIZE=512M
      - HEAP_NEWSIZE=100M
      - CASSANDRA_START_RPC=true
      - CASSANDRA_RPC_ADDRESS=0.0.0.0
      - CASSANDRA_LISTEN_ADDRESS=192.168.1.201
      - CASSANDRA_BROADCAST_ADDRESS=192.168.1.201
      - CASSANDRA_CLUSTER_NAME=my-cluster
      - CASSANDRA_ENDPOINT_SNITCH=GossipingPropertyFileSnitch
      - CASSANDRA_DC=my-datacenter-1
      - CASSANDRA_SEEDS=192.168.1.200
    depends_on:
      cassandra-1:
        condition: service_healthy
    volumes:
      - cassandra-node-2:/var/lib/cassandra:rw
    restart: on-failure
    healthcheck:
      test: ["CMD-SHELL", "nodetool status"]
      interval: 2m
      start_period: 2m
      timeout: 10s
      retries: 3

  cassandra-3:
    image: "cassandra:latest"
    container_name: "cassandra-3"
    networks:
      cassandra-net:
        ipv4_address: 192.168.1.202
    environment:
      - MAX_HEAP_SIZE=512M
      - HEAP_NEWSIZE=100M
      - CASSANDRA_START_RPC=true
      - CASSANDRA_RPC_ADDRESS=0.0.0.0
      - CASSANDRA_LISTEN_ADDRESS=192.168.1.202
      - CASSANDRA_BROADCAST_ADDRESS=192.168.1.202
      - CASSANDRA_CLUSTER_NAME=my-cluster
      - CASSANDRA_ENDPOINT_SNITCH=GossipingPropertyFileSnitch
      - CASSANDRA_DC=my-datacenter-1
      - CASSANDRA_SEEDS=192.168.1.200
    depends_on:
      cassandra-2:
        condition: service_healthy
    volumes:
      - cassandra-node-3:/var/lib/cassandra:rw
    restart: on-failure
    healthcheck:
      test: ["CMD-SHELL", "nodetool status"]
      interval: 2m
      start_period: 2m
      timeout: 10s
      retries: 3

volumes:
  cassandra-node-1:
  cassandra-node-2:
  cassandra-node-3:
```

Был использован ipvlan для выделения адресов в сети хоста. Также, с помощью переменных окружения **MAX_HEAP_SIZE** и **HEAP_NEWSIZE** была ограничена потребляемая память нод. Был использован кастомный образ Cassandra, который будет представлен далее.
Запускается с помощью:
```
docker compose -f docker-compose.yml up -d
```
Результат работы:
![alt text](https://github.com/strikess735/Devops-task/blob/main/images/dc-up.png)

2. __Затем с машины Б (ubuntu 24.04 lts) из той же локальной сети с ip 192.168.1.198 необходимо подключиться через cqlsh к каждой из машин-образов.__
Была создана виртуальная машина с адресом 192.168.1.198 в той же подсети, установлен python3 и pip, установлен cqlsh в окружение и успешно произведено подключение к каждой ноде:

![alt text](https://github.com/strikess735/Devops-task/blob/main/images/cqlsh.png)

3. __Настроить ssh для возможности подключения к 1.200 с 1.197.__

Для подключения по ssh к ноде с адресом 192.168.1.200 с машины 192.168.1.197 был написан кастомный Dockerfile с установкой openssh и добавлением публичного ключа в authorized_keys.
```
FROM cassandra:latest

RUN apt-get update && apt-get install -y openssh-server && rm -rf /var/lib/apt/lists/*

RUN useradd -m -s /bin/bash admin

COPY id_cassandra.pub /home/admin/.ssh/authorized_keys

RUN chown -R admin:admin /home/admin/.ssh && chmod 700 /home/admin/.ssh && chmod 600 /home/admin/.ssh/authorized_keys

RUN sed -i 's/#PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config && \
    sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin no/' /etc/ssh/sshd_config

RUN mkdir /var/run/sshd

EXPOSE 22
CMD ["/bin/bash", "-c", "service ssh start && exec docker-entrypoint.sh cassandra -f"]

```
Также, добавлен интерфейс cassandra-shim для возможности подключения к ноде с хоста(192.168.1.197) по ssh и создан маршрут:

Создание интерфейса:
```
sudo ip link add cassandra-shim link ens33 type ipvlan mode l2
sudo ip addr add 192.168.1.250/24 dev cassandra-shim
sudo ip link set cassandra-shim up
```

Создание маршрута:
```
sudo ip route add 192.168.1.200/29 dev cassandra-shim
```

Это необходимо для ipvlan l2, так как из-за особенностей драйвера, не может быть доступа с хоста в контейнер. Как альтернатива, можно использовать ipvlan l3, но тогда нужно будет прописывать маршруты на каждом устройстве, которому нужен доступ к нодам. Также, можно использовать Macvlan, но я решил его не использовать, т.к. в случае развертывания кластера в облаке, провайдер может запретить новые mac адреса контейнеров.

Результат работы:

![alt text](https://github.com/strikess735/Devops-task/blob/main/images/ssh.png)

Также были созданы правила **nftables** разрашающие подключения только в соответствии с заданиями.

```
#!/usr/sbin/nft -f

flush ruleset

table inet filter {
	chain input {
		type filter hook input priority 0; policy drop;
                iifname "lo" accept
                ct state established,related accept
                iifname "cassandra-shim" ip saddr 192.168.1.197 tcp dport 22 accept
                iifname "cassandra-shim" ip saddr 192.168.1.198 tcp dport 9042 accept
                ip protocol icmp icmp type echo-request accept
                #for debug
                ip saddr 192.168.1.1 tcp dport 22 accept
	}
	chain forward {
		type filter hook forward priority filter;
	}
	chain output {
		type filter hook output priority filter;
	}
}

```
![alt text](https://github.com/strikess735/Devops-task/blob/main/images/nftables.png)
