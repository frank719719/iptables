
Для реализации данной задачи используем инфраструктуру которую разворачивали для стенда в котором изучали работу сети. 
[Ссылка на проект в Github](https://github.com/frank719719/lan)

# Задачи

Перед нами стоят следующие задачи:
- реализовать knocking port
	- centralRouter может попасть на ssh inetrRouter через knock скрипт.
- добавить inetRouter2, который виден(маршрутизируется (host-only тип сети для виртуалки)) с хоста или форвардится порт через локалхост.)
 - запустить nginx на centralServer.
 - пробросить 80й порт на inetRouter2 8080.
 - дефолт в инет оставить через inetRouter.

# Реализация

## Port knocking на хосте inetRouter

### Установка необходимых пакетов.

Нам пригодятся следующие пакеты:

```shell
apt get update
apt get install iptables iptables-persistent knockd -y
```

### Первичная настройка iptables

Конфигурационный файл и саму реализацию возьмем стенда описанного ранее.

**`/etc/iptables_rules.ipv4`**

```bash
*filter
:INPUT ACCEPT [0:0]
:FORWARD ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
-A INPUT -p icmp -j ACCEPT
-A INPUT -i lo -j ACCEPT
#-A INPUT -p tcp -m state --state NEW -m tcp --dport 22 -j ACCEPT
# Правило разрешающее уже установленные соединения.
-A INPUT -i enp0s9 -m state --state ESTABLISHED,RELATED -j ACCEPT
# Правило которое разрешает подключения к 22 порту через интерфейс enp0s9,
# его мы закоментируем после того как поймем что port knocking работает правильно.
#-A INPUT -i enp0s9 -p tcp --dport 22 -j ACCEPT
# Этим правилом мы блокируем все остальные пакеты которые будут приходить
# с интерфейса enp0s9
-A INPUT -i enp0s9 -j DROP
COMMIT

*nat
:PREROUTING ACCEPT [0:0]
:INPUT ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
:POSTROUTING ACCEPT [0:0]
-A POSTROUTING ! -d 192.168.0.0/16 -o enp0s3 -j MASQUERADE
COMMIT

```

Ключевые моменты:
```ad-info
- Правило разрешающее уже установленные соединения:
`-A INPUT -i enp0s9 -m state --state ESTABLISHED,RELATED -j ACCEPT`

- Этим правилом мы блокируем все остальные пакеты которые будут приходить с интерфейса enp0s9:
  `-A INPUT -i enp0s9 -j DROP`
```


### Установка и настройка knockd

Информация для этой части взята из публикации на сайте  https://interface31.ru/
[Ссылка на статью](https://interface31.ru/post/nastraivaem-port-knocking-v-linux-debian-ubuntu/)

Команда для установки:
```bash
apt install knockd
```

Затем откроем файл **/etc/default/knockd** и установим следующую опцию:
```
START_KNOCKD=1
```

Затем создадим файл юнита **systemd**:
```
nano /etc/systemd/system/knockd.service
```

И внесем в него следующие строки:
```
[Unit]
Description=Port-Knock Daemon
After=network.target
Requires=network.target
Documentation=man:knockd(1)

[Service]
EnvironmentFile=-/etc/default/knockd
ExecStartPre=/usr/bin/sleep 1
ExecStart=/usr/sbin/knockd $KNOCKD_OPTS
ExecReload=/bin/kill -HUP $MAINPID
KillMode=mixed
Restart=always
SuccessExitStatus=0 2 15
ProtectSystem=full
CapabilityBoundingSet=CAP_NET_RAW CAP_NET_ADMIN

[Install]
WantedBy=multi-user.target
```

Затем перечитаем изменения и добавим сервис **knockd** в автозагрузку:
```
systemctl daemon-reload
systemctl enable knockd
```

Но не спешим запускать саму службу, с большой долей вероятности мы получим ошибку, связанную с неправильными настройками.

**Правим конфигурационный файл knockd**

Откроем файл `/etc/knockd.conf` и удалим оттуда все, кроме первой секции **options**, которую приведем к следующему виду:
```
[options]
     # Эта опция указывает записывать события в системный лог
     UseSyslog
     # Сюда записываем имя интерфеса к которому собираемся подключаться
     Interface = enp0s9
# Секция для подключения по SSH
[SSH]
     # Последовательность портов для стука
     sequence = 7000,9000,8000
     seq_timeout = 5
     tcpflags = syn
     start_command = /sbin/iptables -I INPUT -s %IP% -p tcp --dport 22 -j ACCEPT
     stop_command = /sbin/iptables -D INPUT -s %IP% -p tcp --dport 22 -j ACCEPT
     cmd_timeout = 60

```

Пояснение:
Опция **sequence** задает последовательность "стуков", по умолчанию используется протокол **tcp**, если вы хотите использовать **udp**, то следует указать его через двоеточие, например:

```
sequence = 7000,9000:udp,8000
```

**seq_timeout** - время в секундах за которое клиент должен совершить всю последовательность "стуков", следует сделать его минимальным, но при этом следует учитывать возможные задержки каналов связи как сервера, так и клиента, **tcpflags** - определяет TCP-флаги пакетов, которые будут участвовать в последовательности, **syn** рекомендуется использовать совместно с SSH, так как SSH-трафик может мешать knockd, делая последовательность недействительной.

Опции **start_command** и **stop_command** определяют команды которые должен выполнить knoсkd с интервалом указанном в опции **cmd_timeout**. Принцип действия здесь прост - получив указанную последовательность "стуков" сервис выполняет команду из опции **start_command**, а по истечении заданного таймаута команду **stop_command**.

Наша задача - открыть доступ к порту SSН на время достаточное для установления подключения. Поэтому первой командой мы добавляем самым первым правилом в цепочку INPUT разрешение подключиться к SSH с адреса источника "стука". Обратите внимание, что для этого мы используем ключ **-I**, который без указания номера добавляет правило первой строкой, а также подстановочную конструкцию **-s %IP%**, которая подставит в правило текущий адрес "стучавшего". Вторая команда удаляет уже добавленное правило.

Таким образом порт будет открыт на подключение только для адреса клиента и только на время, указанное в таймауте. Этот момент обычно вызывает у начинающих некоторое недоумение: а каким образом будет работать соединение, если мы закрыли порт? Все очень просто, доступ к порту 22 нам нужен только для установления соединения, после чего оно перейдет в состояние ESTABLISHED и будет разрешено первым заданным нами правилом в цепочке INPUT, которое разрешает установленные и связанные подключения.

Сохраним конфигурацию и запустим сервис:

```
systemctl start knockd
```

```ad-warning
Важное примечание. Перед попыткой подключиться в нашем случае надо поправить одну строчку на машине к которой мы подключаемся.
Открываем файл `nano /etc/ssh/sshd_config.d/60-cloudimg-settings.conf` и редактируем единственную строчку на `PasswordAuthentication yes`, тем самым мы разрешаем подключение по SSH с использованием пароля.
```

### Подключение с помошью port knock

Команда для подключения:
```
knock ip-adress 7000 9000 8000
```

В нашем случае:
```
root@centralRouter:~# knock 192.168.255.1 7000 9000 8000
root@centralRouter:~# ssh vagrant@192.168.255.1
vagrant@192.168.255.1's password: 
Welcome to Ubuntu 22.04.5 LTS (GNU/Linux 5.15.0-198-generic x86_64)

```

Смотрим на хосте что временное правило появилось в списке iptables:
```
root@inetRouter:/etc/iptables# iptables -vnL
Chain INPUT (policy ACCEPT 922 packets, 52068 bytes)
 pkts bytes target     prot opt in     out     source               destination         
   18  3402 ACCEPT     tcp  --  *      *       192.168.255.2        0.0.0.0/0            tcp dpt:22
    0     0 ACCEPT     icmp --  *      *       0.0.0.0/0            0.0.0.0/0           
    0     0 ACCEPT     all  --  lo     *       0.0.0.0/0            0.0.0.0/0           
   39  2680 ACCEPT     all  --  enp0s9 *       0.0.0.0/0            0.0.0.0/0            state RELATED,ESTABLISHED
   34  2040 DROP       all  --  enp0s9 *       0.0.0.0/0            0.0.0.0/0           

Chain FORWARD (policy ACCEPT 157 packets, 27368 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain OUTPUT (policy ACCEPT 784 packets, 99904 bytes)
 pkts bytes target     prot opt in     out     source               destination
```

Видим что подключение отобразилось на машине:
```
root@inetRouter:/etc/iptables# ss -tulpan
Netid       State         Recv-Q        Send-Q                  Local Address:Port                Peer Address:Port        Process                                                      
udp         UNCONN        0             0                       127.0.0.53%lo:53                       0.0.0.0:*            users:(("systemd-resolve",pid=580,fd=13))                   
udp         UNCONN        0             0                    10.0.2.15%enp0s3:68                       0.0.0.0:*            users:(("systemd-network",pid=578,fd=20))                   
tcp         LISTEN        0             128                           0.0.0.0:22                       0.0.0.0:*            users:(("sshd",pid=2838,fd=3))                              
tcp         LISTEN        0             4096                    127.0.0.53%lo:53                       0.0.0.0:*            users:(("systemd-resolve",pid=580,fd=14))                   
tcp         ESTAB         0             0                       192.168.255.1:22                 192.168.255.2:40832        users:(("sshd",pid=3784,fd=4),("sshd",pid=3706,fd=4))       
tcp         ESTAB         0             0                           10.0.2.15:22                      10.0.2.2:34618        users:(("sshd",pid=1247,fd=4),("sshd",pid=1182,fd=4))       
tcp         LISTEN        0             128                              [::]:22                          [::]:*            users:(("sshd",pid=2838,fd=4))
```

### Готово

## Создание inetRouter2

```ad-info
Данный роутер добавим по той же схеме что и `inetRouter`. Одним интерфейсом он будет смотреть в `centralRouter` а другим в интернет.
```

Создаем машину inetRouter2 путем добавления следующего блока в vagrantfile:
```ruby
:inetRouter2 => {
    :box_name => "ubuntu/jammy64",
    :vm_name => "inetRouter2",
    :net => [
      # Адрес для NAT-сети (для Ansible)
      ["192.168.50.14", 2, "255.255.255.0"],
      # Сеть router-net: inetRouter2 <-> centralRouter
      ["192.168.255.13", 3, "255.255.255.252", "router2-net"],
    ]
  },
```

Подправим блок с centralRouter добавив туда интерфейс который будет связан с inetRouter2:

```ruby
:centralRouter => {
    :box_name => "ubuntu/jammy64",
    :vm_name => "centralRouter",
    :net => [
      ["192.168.50.11", 2, "255.255.255.0"],
      ["192.168.255.2", 3, "255.255.255.252", "router-net"],
      # central: directors 192.168.0.0/28
      ["192.168.0.1", 4, "255.255.255.240", "dir-net"],
      # central: office hardware 192.168.0.32/28
      ["192.168.0.33", 5, "255.255.255.240", "hw-net"],
      # central: wifi 192.168.0.64/26
      ["192.168.0.65", 6, "255.255.255.192", "wifi-net"],
      # centralRouter <-> office1Router
      ["192.168.255.9", 7, "255.255.255.252", "central-office1"],
      # centralRouter <-> inetRouter2  (192.168.255.12/30)
      ["192.168.255.14", 8, "255.255.255.252", "router2-net"],
    ]
  },
```

Создадим отдельный файл netplan для хоста inetRouter2:

```
nano ./ansible/templates/50-vagrant_inetRouter2.yaml
```

```yaml
network:
  version: 2
  ethernets:
    enp0s8:
      addresses:
        - 192.168.50.14/24
    enp0s9:
      addresses:
        - 192.168.255.13/30
      routes:
        - to: 192.168.0.0/16
          via: 192.168.255.14
        - to: 192.168.1.0/24
          via: 192.168.255.14
        - to: 192.168.2.0/24
          via: 192.168.255.14
```

Создадим отдельный файл iptables_rules_inetRouter2.ipv4 для inetRouter2:

```bash
*filter
:INPUT ACCEPT [0:0]
:FORWARD ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
-A INPUT -p icmp -j ACCEPT
-A INPUT -i lo -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 22 -j ACCEPT
#
-A FORWARD -p tcp -d 192.168.0.2 --dport 80 -m state --state NEW,ESTABLISHED,RELATED -j ACCEPT
COMMIT

*nat
:PREROUTING ACCEPT [0:0]
:INPUT ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
:POSTROUTING ACCEPT [0:0]
# Правило для того что бы работал доступ с любого другого устройства в сети при обращении к хостовой машине
# Проброс порта 8080 на порт 9090 хостовой машины настроен в vagrantfile
-A PREROUTING -p tcp -i enp0s3 --dport 8080 -j DNAT --to-destination 192.168.0.2:80
# Правило для того что бы работал доступ с хоста в виртуальную машину по интерфейсу virtualbox (адрес 192.168.50.14)
-A PREROUTING -p tcp -i enp0s8 --dport 8080 -j DNAT --to-destination 192.168.0.2:80
# Подмена адреса источника т.к. внутренний сервер должен отвечать через тот же шлюз обратно во внешнюю сеть
-A POSTROUTING -d 192.168.0.2 -p tcp --dport 80 -j MASQUERADE
#-A POSTROUTING ! -d 192.168.0.0/16 -o enp0s3 -j MASQUERADE
COMMIT

```

Копировать его будем с помошью ansible вставив следующий блок в файл provision.yml
```ruby
- name: Copy iptables rules file (only inetRouter2)
      template:
        src: iptables_rules_inetRouter2.ipv4
        dest: /etc/iptables_rules.ipv4
        owner: root
        group: root
        mode: "0644"
      when: ansible_hostname in ["inetRouter2"]
```

В vagrantfile добавим блок в котором пропишем проброс портов с хостовой машины на виртуальную `inetRouter2`

```ruby
if boxconfig[:vm_name] == "inetRouter2"
        box.vm.network "forwarded_port",
          guest: 8080, host: 9090, id: "nginxweb",
          auto_correct: true
      end
```

**Итоговые файлы:**
(Прошу не обращать внимание на лишние строки т.к. это простая переделка другого стенда)

vagrantfile:
```ruby
# -*- mode: ruby -*-
# vi: set ft=ruby :

MACHINES = {
  :inetRouter => {
    :box_name => "ubuntu/jammy64",
    :vm_name => "inetRouter",
    :net => [
      # Адрес для NAT-сети (для Ansible)
      ["192.168.50.10", 2, "255.255.255.0"],
      # Сеть router-net: inetRouter <-> centralRouter
      ["192.168.255.1", 3, "255.255.255.252", "router-net"],
    ]
  },

  :inetRouter2 => {
    :box_name => "ubuntu/jammy64",
    :vm_name => "inetRouter2",
    :net => [
      # Адрес для NAT-сети (для Ansible)
      ["192.168.50.14", 2, "255.255.255.0"],
      # Сеть router2-net: inetRouter <-> centralRouter
      ["192.168.255.13", 3, "255.255.255.252", "router2-net"],
    ]
  },

  :centralRouter => {
    :box_name => "ubuntu/jammy64",
    :vm_name => "centralRouter",
    :net => [
      ["192.168.50.11", 2, "255.255.255.0"],
      ["192.168.255.2", 3, "255.255.255.252", "router-net"],
      # central: directors 192.168.0.0/28
      ["192.168.0.1", 4, "255.255.255.240", "dir-net"],
      # central: office hardware 192.168.0.32/28
      ["192.168.0.33", 5, "255.255.255.240", "hw-net"],
      # central: wifi 192.168.0.64/26
      ["192.168.0.65", 6, "255.255.255.192", "wifi-net"],
      # centralRouter <-> office1Router
      ["192.168.255.9", 7, "255.255.255.252", "central-office1"],
      # centralRouter <-> inetRouter2  (192.168.255.12/30)
      ["192.168.255.14", 8, "255.255.255.252", "router2-net"],
    ]
  },

#  :office1Router => {
#    :box_name => "ubuntu/jammy64",
#    :vm_name => "office1Router",
#    :net => [
#      ["192.168.50.20", 2, "255.255.255.0"],
#      ["192.168.255.10", 3, "255.255.255.252", "central-office1"],
#      # dev 192.168.2.0/26
#      ["192.168.2.1", 4, "255.255.255.192", "office1-dev"],
#      # test servers 192.168.2.64/26
#      ["192.168.2.65", 5, "255.255.255.192", "office1-test"],
#      # managers 192.168.2.128/26
#      ["192.168.2.129", 6, "255.255.255.192", "office1-managers"],
#      # office hardware 192.168.2.192/26
#      ["192.168.2.193", 7, "255.255.255.192", "office1-hw"],
#    ]
#  },

#  :office2Router => {
#    :box_name => "ubuntu/jammy64",
#    :vm_name => "office2Router",
#    :net => [
#      ["192.168.50.30", 2, "255.255.255.0"],
#      ["192.168.255.6", 3, "255.255.255.252", "central-office2"],
#      # dev 192.168.1.0/25
#      ["192.168.1.1", 4, "255.255.255.128", "office2-dev"],
#      # test servers 192.168.1.128/26
#      ["192.168.1.129", 5, "255.255.255.192", "office2-test"],
#      # office hardware 192.168.1.192/26
#      ["192.168.1.193", 6, "255.255.255.192", "office2-hw"],
#    ]
#  },

  :centralServer => {
    :box_name => "ubuntu/jammy64",
    :vm_name => "centralServer",
    :net => [
      ["192.168.50.12", 2, "255.255.255.0"],
      # directors 192.168.0.0/28
      ["192.168.0.2", 3, "255.255.255.240", "dir-net"],
    ]
  },

#  :office1Server => {
#    :box_name => "ubuntu/jammy64",
#    :vm_name => "office1Server",
#    :net => [
#      ["192.168.50.21", 2, "255.255.255.0"],
#      # managers 192.168.2.128/26
#      ["192.168.2.130", 3, "255.255.255.192", "office1-managers"],
#    ]
#  },

#  :office2Server => {
#    :box_name => "ubuntu/jammy64",
#    :vm_name => "office2Server",
#    :net => [
#      ["192.168.50.22", 2, "255.255.255.0"],
#      # dev 192.168.1.0/25
#      ["192.168.1.2", 3, "255.255.255.128", "office2-dev"],
#    ]
#  },
}

Vagrant.configure("2") do |config|
  MACHINES.each do |boxname, boxconfig|
    config.vm.define boxname do |box|
      box.vm.box = boxconfig[:box_name]
      box.vm.host_name = boxconfig[:vm_name]

      box.vm.provider "virtualbox" do |v|
        v.memory = 1500
        v.cpus = 2
      end

      boxconfig[:net].each do |ipconf|
        box.vm.network(
          "private_network",
          ip: ipconf[0],
          adapter: ipconf[1],
          netmask: ipconf[2],
          virtualbox__intnet: ipconf[3]
        )
      end

      if boxconfig[:vm_name] == "inetRouter2"
        box.vm.network "forwarded_port",
          guest: 8080, host: 9090, id: "nginxweb",
          auto_correct: true
      end

      if boxconfig.key?(:public)
        box.vm.network "public_network", boxconfig[:public]
      end

      # Копируем SSH-ключи для root
      box.vm.provision "shell", inline: <<-SHELL
        mkdir -p /root/.ssh
        cp ~vagrant/.ssh/authorized_keys /root/.ssh/
        chmod 600 /root/.ssh/authorized_keys
      SHELL
      # Удаляем ненужный cloud-init файл в `/etc/netplan/`
      box.vm.provision "shell", inline: <<-SHELL
        rm /etc/netplan/50-cloud-init.yaml
      SHELL
      # Создаем отдельный файл с настройкой интерфеса для выхода в интернет
       if ["inetRouter", "inetRouter2"].include?(boxconfig[:vm_name])
       box.vm.provision "shell", inline: <<-SHELL
      # Убедиться, что enp0s3 управляется Netplan и включён DHCP
         cat > /etc/netplan/00-inet-nat.yaml <<'EOF'
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: true
      dhcp4-overrides:
        use-routes: true
EOF
    chmod 600 /etc/netplan/00-inet-nat.yaml
    netplan apply
      SHELL
      end
      # Ansible запускается только после создания последней ВМ
      if boxconfig[:vm_name] == "centralServer"
        box.vm.provision "ansible" do |ansible|
          ansible.playbook = "ansible/provision.yml"
          ansible.inventory_path = "ansible/hosts"
          ansible.host_key_checking = "false"
          ansible.limit = "all"
        end
      end
    end
  end
end

```

**`./ansible/provision.yml`:**

```ruby
---
- name: Base setup for all hosts
  hosts: all
  become: true
  gather_facts: true
  tasks:
    - name: Install useful packages
      apt:
        name:
          - traceroute
          - iptables
          - iptables-persistent
        state: present
        update_cache: true

- name: Install nginx on centralServer
  hosts: all
  become: true
  gather_facts: true
  tasks:
    - name: nstall nginx on centralServer
      apt:
        name:
          - nginx
        state: present
        update_cache: true
      when: ansible_hostname in ["centralServer"]

- name: Setup routers (NAT + forwarding)
  hosts: routers
  become: true
  gather_facts: true
  tasks:
    - name: Enable IPv4 forwarding
      sysctl:
        name: net.ipv4.conf.all.forwarding
        value: '1'
        state: present
        sysctl_set: true
        reload: true

    - name: Disable ufw (not needed, using iptables)
      systemd:
        name: ufw
        state: stopped
        enabled: false
      ignore_errors: true

    - name: Copy iptables rules file (only inetRouter)
      template:
        src: iptables_rules.ipv4
        dest: /etc/iptables_rules.ipv4
        owner: root
        group: root
        mode: "0644"
      when: ansible_hostname in ["inetRouter"]

    - name: Copy iptables rules file (only inetRouter2)
      template:
        src: iptables_rules_inetRouter2.ipv4
        dest: /etc/iptables_rules.ipv4
        owner: root
        group: root
        mode: "0644"
      when: ansible_hostname in ["inetRouter2"]

    - name: Install iptables-restore systemd unit (only inetRouter)
      copy:
        dest: /etc/systemd/system/iptables-restore.service
        mode: "0644"
        content: |
          [Unit]
          Description=Restore iptables rules
          After=network-online.target
          Wants=network-online.target

          [Service]
          Type=oneshot
          ExecStart=/sbin/iptables-restore /etc/iptables_rules.ipv4
          RemainAfterExit=yes

          [Install]
          WantedBy=multi-user.target
      when: ansible_hostname in ["inetRouter", "inetRouter2"]

    - name: Enable and start iptables-restore.service (only inetRouter)
      systemd:
        name: iptables-restore.service
        enabled: true
        state: started
        daemon_reload: true
      when: ansible_hostname in ["inetRouter", "inetRouter2"]


#    - name: Copy iptables restore script (only inetRouter)
#      template:
#        src: iptables_restore
#        dest: /etc/network/if-pre-up.d/iptables
#        owner: root
#        group: root
#        mode: "0755"
#      when: ansible_hostname == "inetRouter"

    - name: Apply iptables rules now (only inetRouter)
      shell: iptables-restore < /etc/iptables_rules.ipv4
      when: ansible_hostname in ["inetRouter", "inetRouter2"]

- name: Disable default route on eth0 for all except inetRouter
  hosts: all
  become: true
  gather_facts: true
  tasks:
    - name: Disable dhcp routes on eth0
      template:
        src: 00-installer-config.yaml
        dest: /etc/netplan/00-installer-config.yaml
        owner: root
        group: root
        mode: "0600"
      when: ansible_hostname not in ["inetRouter", "inetRouter2"]

- name: Configure static routes (netplan)
  hosts: all
  become: true
  gather_facts: true
  tasks:
    - name: Copy netplan config with routes
      template:
        src: "50-vagrant_{{ ansible_hostname }}.yaml"
        dest: /etc/netplan/50-vagrant.yaml
        owner: root
        group: root
        mode: "0600"

    - name: Apply netplan
      command: netplan apply

    - name: Reboot to verify persistence
      reboot:
        reboot_timeout: 600

```

## Проверка

**После того как стенд запущен можем залогиниться на `centralServer` и посмотреть установлен ли там nginx:**

```bash
vagrant ssh centralServer
```

```bash
vagrant@centralServer:~$ systemctl status nginx
● nginx.service - A high performance web server and a reverse proxy server
     Loaded: loaded (/lib/systemd/system/nginx.service; enabled; vendor preset: enabled)
     Active: active (running) since Thu 2026-11-05 21:58:01 UTC; 19min ago
       Docs: man:nginx(8)
    Process: 641 ExecStartPre=/usr/sbin/nginx -t -q -g daemon on; master_process on; (code=exited, status=0/SUCCESS)
    Process: 692 ExecStart=/usr/sbin/nginx -g daemon on; master_process on; (code=exited, status=0/SUCCESS)
   Main PID: 693 (nginx)
      Tasks: 3 (limit: 1659)
     Memory: 12.7M
        CPU: 24ms

```

```bash
vagrant@centralServer:~$ ss -tulpan
Netid             State              Recv-Q             Send-Q                            Local Address:Port                         Peer Address:Port              Process             
udp               UNCONN             0                  0                                 127.0.0.53%lo:53                                0.0.0.0:*                                     
udp               UNCONN             0                  0                              10.0.2.15%enp0s3:68                                0.0.0.0:*                                     
tcp               LISTEN             0                  511                                     0.0.0.0:80                                0.0.0.0:*                                     
tcp               LISTEN             0                  128                                     0.0.0.0:22                                0.0.0.0:*                                     
tcp               LISTEN             0                  4096                              127.0.0.53%lo:53                                0.0.0.0:*                                     
tcp               ESTAB              0                  0                                     10.0.2.15:22                               10.0.2.2:40954                                 
tcp               LISTEN             0                  511                                        [::]:80                                   [::]:*                                     
tcp               LISTEN             0                  128                                        [::]:22                                   [::]:*                                     
```

**Проверяем доступ в интернет, он должен остаться через `inetRouter2`:**

```bash
vagrant@centralServer:~$ traceroute 8.8.8.8
traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets
 1  _gateway (192.168.0.1)  1.046 ms  0.992 ms  0.973 ms
 2  192.168.255.1 (192.168.255.1)  1.759 ms  1.741 ms  1.587 ms
 3  10.0.2.2 (10.0.2.2)  1.863 ms  1.779 ms  1.756 ms
 4  router.lan (192.168.78.1)  5.175 ms  5.156 ms  5.136 ms
 5  * * *
 6  lag-4-435.bbr01.nn.ertelecom.ru (109.194.184.18)  5.859 ms  6.129 ms  6.062 ms
 7  bbr03.spb.ertelecom.ru (188.234.152.203)  33.824 ms  36.201 ms  35.965 ms
 8  ae16-0.RT.SL.SPB.retn.ru (139.45.238.94)  35.882 ms  35.851 ms  35.824 ms
 9  ae6-106.SR.SL.SPB.retn.ru (46.46.189.6)  34.036 ms  34.001 ms  33.973 ms
10  ae10-92.rt.tnr.hki.fi.retn.net (87.245.248.42)  39.269 ms  39.871 ms  39.090 ms
11  * * *
12  * * *
13  dns.google (8.8.8.8)  41.702 ms  41.577 ms  41.551 ms
vagrant@centralServer:~$ 
```
Видим что маршрут правильный.

**Переходим на роутер inetRouter2 и проверяем правила проброса портов:**

```bash
vagrant@inetRouter2:~$ sudo iptables -t nat -L -n -v
Chain PREROUTING (policy ACCEPT 2 packets, 104 bytes)
 pkts bytes target     prot opt in     out     source               destination         
    3   132 DNAT       tcp  --  enp0s3 *       0.0.0.0/0            0.0.0.0/0            tcp dpt:8080 to:192.168.0.2:80
    0     0 DNAT       tcp  --  enp0s8 *       0.0.0.0/0            0.0.0.0/0            tcp dpt:8080 to:192.168.0.2:80

Chain INPUT (policy ACCEPT 2 packets, 104 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain OUTPUT (policy ACCEPT 13 packets, 894 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain POSTROUTING (policy ACCEPT 13 packets, 894 bytes)
 pkts bytes target     prot opt in     out     source               destination         
    3   132 MASQUERADE  tcp  --  *      *       0.0.0.0/0            192.168.0.2          tcp dpt:80
vagrant@inetRouter2:~$ sudo iptables -t filter -L -n -v
Chain INPUT (policy ACCEPT 141 packets, 25922 bytes)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 ACCEPT     icmp --  *      *       0.0.0.0/0            0.0.0.0/0           
    8  1372 ACCEPT     all  --  lo     *       0.0.0.0/0            0.0.0.0/0           
    2   104 ACCEPT     tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            state NEW tcp dpt:22

Chain FORWARD (policy ACCEPT 28 packets, 2079 bytes)
 pkts bytes target     prot opt in     out     source               destination         
   20   937 ACCEPT     tcp  --  *      *       0.0.0.0/0            192.168.0.2          tcp dpt:80 state NEW,RELATED,ESTABLISHED

Chain OUTPUT (policy ACCEPT 122 packets, 20234 bytes)
 pkts bytes target     prot opt in     out     source               destination         
vagrant@inetRouter2:~$ 

```

Видим что все нужные нам правила на месте.

**Проверим доступность `nginx` с роутера `inetRouter2`**

```bash
vagrant@inetRouter2:~$ curl 192.168.0.2
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>

```

**Проверим доступность с хостовой машины:**

```bash
usertest@desktop:~/otus/iptables$ curl 192.168.50.14:8080
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

**Проверим доступность сервиса через проброс который делали на порт 9090 хостовой машины:**
```bash
usertest@desktop:~/otus/iptables$ curl 127.0.0.1:9090
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

Все проверки успешно пройдены.
Задачи выполнены.

# Готово
