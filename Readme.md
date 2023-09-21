Методическое пособие по выполнению домашнего задания по курсу «Администратор Linux. Professional»
Vagrant-стенд c DNS

Цель домашнего задания
Создать домашнюю сетевую лабораторию. Изучить основы DNS, научиться работать с технологией Split-DNS в Linux-based системах.


Описание домашнего задания
1. взять стенд https://github.com/erlong15/vagrant-bind 
добавить еще один сервер client2
завести в зоне dns.lab имена:
web1 - смотрит на клиент1
web2  смотрит на клиент2
завести еще одну зону newdns.lab
завести в ней запись
www - смотрит на обоих клиентов

2. настроить split-dns
клиент1 - видит обе зоны, но в зоне dns.lab только web1
клиент2 видит только dns.lab

Split DNS (split-horizon или split-brain) — это конфигурация, позволяющая отдавать разные записи зон DNS в зависимости от подсети источника запроса. Данную функцию можно реализовать как с помощью одного DNS-сервера, так и с помощью нескольких DNS-серверов

1. Работа со стендом и настройка DNS

Скачаем себе стенд https://github.com/erlong15/vagrant-bind, перейдём в скаченный каталог и изучим содержимое файлов:

```
neva@Uneva:~/Otus_Kaneva_dz23$ git clone https://github.com/erlong15/vagrant-bind.git
Cloning into 'vagrant-bind'...
remote: Enumerating objects: 27, done.
remote: Total 27 (delta 0), reused 0 (delta 0), pack-reused 27
Receiving objects: 100% (27/27), 4.40 KiB | 4.40 MiB/s, done.
Resolving deltas: 100% (9/9), done.

neva@Uneva:~/Otus_Kaneva_dz23$ cd vagrant-bind

neva@Uneva:~/Otus_Kaneva_dz23/vagrant-bind$ ls -l
total 12
drwxrwxr-x 2 neva neva 4096 сен 14 17:38 provisioning
-rw-rw-r-- 1 neva neva  414 сен 14 17:38 README.md
-rw-rw-r-- 1 neva neva  820 сен 14 17:38 Vagrantfile
```

Мы увидим файл Vagrantfile. Откроем его в любом, удобном для вас текстовом редакторе и добавим необходимую ВМ, приведя его к следующему виду:

```
# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure(2) do |config|
  config.vm.box = "centos/7"

  config.vm.provision "ansible" do |ansible|
    ansible.verbose = "vvv"
    ansible.playbook = "provisioning/playbook.yml"
    ansible.become = "true"
  end


  config.vm.provider "virtualbox" do |v|
          v.memory = 256
  end

  config.vm.define "ns01" do |ns01|
    ns01.vm.network "private_network", ip: "192.168.50.10", virtualbox__intnet: "dns"
    ns01.vm.hostname = "ns01"
  end

  config.vm.define "ns02" do |ns02|
    ns02.vm.network "private_network", ip: "192.168.50.11", virtualbox__intnet: "dns"
    ns02.vm.hostname = "ns02"
  end

  config.vm.define "client" do |client|
    client.vm.network "private_network", ip: "192.168.50.15", virtualbox__intnet: "dns"
    client.vm.hostname = "client"
  end

  config.vm.define "client2" do |client|
    client.vm.network "private_network", ip: "192.168.50.16", virtualbox__intnet: "dns"
    client.vm.hostname = "client"
  end
end
```

Vagranfile описывает создание 4 виртуальных машин на CentOS 7, каждой машине будет выделено по 256 МБ ОЗУ. В начале файла есть модуль, который отвечает за настройку ВМ с помощью Ansible. 

```
neva@Uneva:~/Otus_Kaneva_dz23/vagrant-bind$ vagrant up
neva@Uneva:~/Otus_Kaneva_dz23/vagrant-bind$ vagrant status
Current machine states:

ns01                      running (virtualbox)
ns02                      running (virtualbox)
client                    running (virtualbox)
client2                   running (virtualbox)
```

После того, как у нас получилось добавить виртуальную машину client2, давайте подробнее разберем остальные файлы. Для этого перейдём в каталог provisoning и рассмотрим требуемые нам файлы:
playbook.yml — это Ansible-playbook, в котором содержатся инструкции по настройке нашего стенда
client-motd — файл, содержимое которого будет появляться перед пользователем, который подключился по SSH
named.ddns.lab и named.dns.lab — файлы описания зон ddns.lab и dns.lab соответсвенно
master-named.conf и slave-named.conf — конфигурационные файлы, в которых хранятся настройки DNS-сервера
client-resolv.conf и servers-resolv.conf — файлы, в которых содержатся IP-адреса DNS-серверов

По заданию нам нужно изменить playbook.yml таким образом, чтобы у нас была вторая клиентская машина:.

```
- hosts: client,client2
  become: yes
  tasks:
  - name: copy resolv.conf to the client
    copy: src=client-resolv.conf dest=/etc/resolv.conf owner=root group=root mode=0644

#Копирование конфигруационного файла rndc
  - name: copy rndc conf file
    copy: src=rndc.conf dest=/home/vagrant/rndc.conf owner=vagrant group=vagrant mode=0644
#Настройка сообщения при входе на сервер
  - name: copy motd to the client
    copy: src=client-motd dest=/etc/motd owner=root group=root mode=0644
```

Для нормальной работы DNS-серверов, на них должно быть настроено одинаковое время. Для того, чтобы на всех серверах было одинаковое время, нам потребуется настроить NTP.
Пример данной настройки в Ansible (YAML-формат):

```
 - name: stop and disable chronyd
    service: 
      name: chronyd
      state: stopped
      enabled: false
      
  - name: start and enable ntpd 
    service: 
      name: ntpd
      state: started
```

Перед выполнением следующих заданий, нужно обратить внимание, на каком адресе и порту работают наши DNS-сервера.

```
[vagrant@ns01 ~]$ ss -tulpn
Netid State      Recv-Q Send-Q                                                                       Local Address:Port                                                                                      Peer Address:Port
udp   UNCONN     0      0                                                                            192.168.50.10:53                                                                                                   *:*
udp   UNCONN     0      0                                                                                127.0.0.1:323                                                                                                  *:*
udp   UNCONN     0      0                                                                                        *:68                                                                                                   *:*
udp   UNCONN     0      0                                                                                        *:111                                                                                                  *:*
udp   UNCONN     0      0                                                                                        *:929                                                                                                  *:*
udp   UNCONN     0      0                                                                                    [::1]:53                                                                                                [::]:*
udp   UNCONN     0      0                                                                                    [::1]:323                                                                                               [::]:*
udp   UNCONN     0      0                                                                                     [::]:111                                                                                               [::]:*
udp   UNCONN     0      0                                                                                     [::]:929                                                                                               [::]:*
tcp   LISTEN     0      128                                                                                      *:111                                                                                                  *:*
tcp   LISTEN     0      10                                                                           192.168.50.10:53                                                                                                   *:*
tcp   LISTEN     0      128                                                                                      *:22                                                                                                   *:*
tcp   LISTEN     0      128                                                                          192.168.50.10:953                                                                                                  *:*
tcp   LISTEN     0      100                                                                              127.0.0.1:25                                                                                                   *:*
tcp   LISTEN     0      128                                                                                   [::]:111                                                                                               [::]:*
tcp   LISTEN     0      10                                                                                   [::1]:53                                                                                                [::]:*
tcp   LISTEN     0      128                                                                                   [::]:22                                                                                                [::]:*
tcp   LISTEN     0      100                                                                                  [::1]:25                                                                                                [::]:*
```

Исходя из данной информации, нам нужно подкорректировать файл /etc/resolv.conf для DNS-серверов: на хосте ns01 указать nameserver 192.168.50.10, а на хосте ns02 — 192.168.50.11 
В Ansible для этого можно воспользоваться шаблоном с Jinja. Изменим имя файла servers-resolv.conf на servers-resolv.conf.j2 и укажем там следующие условия:

```
domain dns.lab
search dns.lab
#Если имя сервера ns02, то указываем nameserver 192.168.50.11
{% if ansible_hostname == 'ns02' %}
nameserver 192.168.50.11
{% endif %}
#Если имя сервера ns01, то указываем nameserver 192.168.50.10
{% if ansible_hostname == 'ns01' %}
nameserver 192.168.50.10
{% endif %}
```

После внесение изменений в файл, внесём измения в ansible-playbook:
Используем вместо модуля copy модуль template:

```
- name: copy resolv.conf to the servers
  template: src=servers-resolv.conf.j2 dest=/etc/resolv.conf owner=root group=root mode=0644
```


Добавление имён в зону dns.lab

Давайте проверим, что зона dns.lab уже существует на DNS-серверах:

Фрагмент файла /etc/named.conf на сервере ns01:

```
// Имя зоны
zone "dns.lab" {
    type master;
    // Тем, у кого есть ключ zonetransfer.key можно получать копию файла зоны 
    allow-transfer { key "zonetransfer.key"; };
    // Файл с настройками зоны
    file "/etc/named/named.dns.lab";
};
```

Похожий фрагмент файла /etc/named.conf  находится на slave-сервере ns02:

```
// Имя зоны
zone "dns.lab" {
    type slave;
    // Адрес мастера, куда будет обращаться slave-сервер
    masters { 192.168.50.10; };
};
```

Также на хосте ns01 мы видим файл /etc/named/named.dns.lab с настройкой зоны:

```
[root@ns01 ~]# cat /etc/named/named.dns.lab
$TTL 3600
$ORIGIN dns.lab.
@               IN      SOA     ns01.dns.lab. root.dns.lab. (
                            2711201407 ; serial
                            3600       ; refresh (1 hour)
                            600        ; retry (10 minutes)
                            86400      ; expire (1 day)
                            600        ; minimum (10 minutes)
                        )

                IN      NS      ns01.dns.lab.
                IN      NS      ns02.dns.lab.

; DNS Servers
ns01            IN      A       192.168.50.10
ns02            IN      A       192.168.50.11
```

Именно в этот файл нам потребуется добавить имена. Допишем в конец файла следующие строки: 

```
;Web
web1            IN      A       192.168.50.15
web2            IN      A       192.168.50.16
```

Если изменения внесены вручную, то для применения настроек нужно:
Перезапустить службу named: systemctl restart named 
Изменить значение Serial (добавить +1 к числу 2711201407), изменение значения serial укажет slave-серверам на то, что были внесены изменения и что им надо обновить свои файлы с зонами.

После внесения изменений, выполним проверку с клиента:

```
[vagrant@client ~]$ dig @192.168.50.10 web1.dns.lab

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.14 <<>> @192.168.50.10 web1.dns.lab
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 25967
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 2, ADDITIONAL: 3

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;web1.dns.lab.                  IN      A

;; ANSWER SECTION:
web1.dns.lab.           3600    IN      A       192.168.50.15

;; AUTHORITY SECTION:
dns.lab.                3600    IN      NS      ns01.dns.lab.
dns.lab.                3600    IN      NS      ns02.dns.lab.

;; ADDITIONAL SECTION:
ns01.dns.lab.           3600    IN      A       192.168.50.10
ns02.dns.lab.           3600    IN      A       192.168.50.11

;; Query time: 1 msec
;; SERVER: 192.168.50.10#53(192.168.50.10)
;; WHEN: Mon Sep 18 16:13:41 UTC 2023
;; MSG SIZE  rcvd: 127



[vagrant@client ~]$ dig @192.168.50.11 web2.dns.lab

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.9 <<>> @192.168.50.11 web2.dns.lab
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 36834
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 2, ADDITIONAL: 3

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;web2.dns.lab.			IN	A

;; ANSWER SECTION:
web2.dns.lab.		3600	IN	A	192.168.50.16

;; AUTHORITY SECTION:
dns.lab.		3600	IN	NS	ns01.dns.lab.
dns.lab.		3600	IN	NS	ns02.dns.lab.

;; ADDITIONAL SECTION:
ns01.dns.lab.		3600	IN	A	192.168.50.10
ns02.dns.lab.		3600	IN	A	192.168.50.11

;; Query time: 2 msec
;; SERVER: 192.168.50.11#53(192.168.50.11)
;; WHEN: Sun Mar 27 00:37:28 UTC 2022
;; MSG SIZE  rcvd: 127
```

Добавление имён в зону dns.lab с помощью Ansible

В существующем Ansible-playbook менять ничего не потребуется. Нам потребуется добавить стоки с новыми именами в файл named.dns.lab и снова запустить Ansible.


Создание новой зоны и добавление в неё записей

Для того, чтобы прописать на DNS-серверах новую зону нам потребуется: 
На хосте ns01 добавить зону в файл /etc/named.conf:

```
// lab's newdns zone
zone "newdns.lab" {
    type master;
    allow-transfer { key "zonetransfer.key"; };
    allow-update { key "zonetransfer.key"; };
    file "/etc/named/named.newdns.lab";
};
```

На хосте ns02 также добавить зону и указать с какого сервера запрашивать информацию об этой зоне (фрагмент файла /etc/named.conf):

```
// lab's newdns zone
zone "newdns.lab" {
    type slave;
    masters { 192.168.50.10; };
    file "/etc/named/named.newdns.lab";
};
```

На хосте ns01 создадим файл /etc/named/named.newdns.lab

```
$TTL 3600
$ORIGIN newdns.lab.
@               IN      SOA     ns01.dns.lab. root.dns.lab. (
                            2711201007 ; serial
                            3600       ; refresh (1 hour)
                            600        ; retry (10 minutes)
                            86400      ; expire (1 day)
                            600        ; minimum (10 minutes)
                        )

                IN      NS      ns01.dns.lab.
                IN      NS      ns02.dns.lab.

; DNS Servers
ns01            IN      A       192.168.50.10
ns02            IN      A       192.168.50.11

;WWW
www             IN      A       192.168.50.15
www             IN      A       192.168.50.16
```

У файла должны быть права 660, владелец — root, группа — named. 

```
[root@ns01 ~]# ls -l /etc/named/
total 16
-rw-rw----. 1 root named 600 Sep 18 17:21 named.ddns.lab
-rw-rw----. 1 root named 697 Sep 18 17:21 named.dns.lab
-rw-rw----. 1 root named 625 Sep 18 17:20 named.dns.lab.rev
-rw-rw----. 1 root named 690 Sep 20 07:58 named.newdns.lab
```

После внесения данных изменений, изменяем значение serial (добавлем +1 к значению 2711201007) и перезапускаем named:

```
[root@ns01 ~]# systemctl restart named
[root@ns01 ~]# systemctl status named.service
● named.service - Berkeley Internet Name Domain (DNS)
   Loaded: loaded (/usr/lib/systemd/system/named.service; enabled; vendor preset: disabled)
   Active: active (running) since Wed 2023-09-20 08:49:03 UTC; 5s ago
  Process: 24876 ExecStop=/bin/sh -c /usr/sbin/rndc stop > /dev/null 2>&1 || /bin/kill -TERM $MAINPID (code=exited, status=0/SUCCESS)
  Process: 24908 ExecStart=/usr/sbin/named -u named -c ${NAMEDCONF} $OPTIONS (code=exited, status=0/SUCCESS)
  Process: 24906 ExecStartPre=/bin/bash -c if [ ! "$DISABLE_ZONE_CHECKING" == "yes" ]; then /usr/sbin/named-checkconf -z "$NAMEDCONF"; else echo "Checking of zone files is disabled"; fi (code=exited, status=0/SUCCESS)
 Main PID: 24910 (named)
   CGroup: /system.slice/named.service
           └─24910 /usr/sbin/named -u named -c /etc/named.conf

Sep 20 08:49:03 ns01 named[24910]: network unreachable resolving './DNSKEY/IN': 2001:7fe::53#53
Sep 20 08:49:03 ns01 named[24910]: network unreachable resolving './NS/IN': 2001:7fe::53#53
Sep 20 08:49:03 ns01 named[24910]: network unreachable resolving './DNSKEY/IN': 2001:500:a8::e#53
Sep 20 08:49:03 ns01 named[24910]: network unreachable resolving './NS/IN': 2001:500:a8::e#53
Sep 20 08:49:03 ns01 named[24910]: network unreachable resolving './DNSKEY/IN': 2001:500:1::53#53
Sep 20 08:49:03 ns01 named[24910]: network unreachable resolving './NS/IN': 2001:500:1::53#53
Sep 20 08:49:03 ns01 named[24910]: network unreachable resolving './DNSKEY/IN': 2001:500:12::d0d#53
Sep 20 08:49:03 ns01 named[24910]: network unreachable resolving './NS/IN': 2001:500:12::d0d#53
Sep 20 08:49:03 ns01 named[24910]: managed-keys-zone: Key 20326 for zone . acceptance timer complete: key now trusted
Sep 20 08:49:03 ns01 named[24910]: resolver priming query complete
```

Создание новой зоны и добавление в неё записей с помощью Ansible

Для создания зоны и добавления в неё записей, добавляем зону в файл /etc/named.conf на хостах ns01 и ns02, а также создаем файл named.newdns.lab, который далее отправим на сервер ns01.
Добавим в модуль copy наш файл named.newdns.lab:

```
- name: copy zones
    copy: src={{ item }} dest=/etc/named/ owner=root group=named mode=0660
    with_fileglob:
      - named.d*
      - named.newdns.lab
```

Соответственно файл  named.newdns.lab будет скопирован на хост ns01 по адресу /etc/named/named.newdns.lab

Остальную часть playbook можно оставить без изменений.



2. Настройка Split-DNS 

У нас уже есть прописанные зоны dns.lab и newdns.lab. Однако по заданию client1  должен видеть запись web1.dns.lab и не видеть запись web2.dns.lab. Client2 может видеть обе записи из домена dns.lab, но не должен видеть записи домена newdns.lab Осуществить данные настройки нам поможет технология Split-DNS.
Для настройки Split-DNS нужно: 
1) Создать дополнительный файл зоны dns.lab, в котором будет прописана только одна запись: 

```
[root@ns01 ~]# vi /etc/named/named.dns.lab.client
$TTL 3600
$ORIGIN dns.lab.
@               IN      SOA     ns01.dns.lab. root.dns.lab. (
                            2711201407 ; serial
                            3600       ; refresh (1 hour)
                            600        ; retry (10 minutes)
                            86400      ; expire (1 day)
                            600        ; minimum (10 minutes)
                        )

                IN      NS      ns01.dns.lab.
                IN      NS      ns02.dns.lab.

; DNS Servers
ns01            IN      A       192.168.50.10
ns02            IN      A       192.168.50.11

;Web
web1            IN      A       192.168.50.15
```

Имя файла может отличаться от указанной зоны. У файла должны быть права 660, владелец — root, группа — named.
2) Внести изменения в файл /etc/named.conf на хостах ns01 и ns02

Прежде всего нужно сделать access листы для хостов client и client2. Сначала сгенерируем ключи для хостов client и client2, для этого на хосте ns01 запустим утилиту tsig-keygen (ключ может генериться 5 минут и более): 

```
[root@ns01 ~]# tsig-keygen
key "tsig-key" {
        algorithm hmac-sha256;
        secret "XUeIVPZMYYGJk771qTR4HW5TDGILijvjjMfpXjKICoE=";
};
[root@ns01 ~]# tsig-keygen
key "tsig-key" {
        algorithm hmac-sha256;
        secret "B34Bp49mUHVTIfDoxzbzzuj4xECKzmpcHgyb3yjOeHk=";
};
```

После генерации, мы увидим ключ (secret) и алгоритм с помощью которого он был сгенерирован. Оба этих параметра нам потребуются в access листе. 
Всего нам потребуется 2 таких ключа. После их генерации добавим блок с access листами в конец файла /etc/named.conf сервера ns01.

```
#Описание ключа для хоста client
key "client-key" {
        algorithm hmac-sha256;
        secret "XUeIVPZMYYGJk771qTR4HW5TDGILijvjjMfpXjKICoE=";
};
#Описание ключа для хоста client2
key "client2-key" {
        algorithm hmac-sha256;
        secret "B34Bp49mUHVTIfDoxzbzzuj4xECKzmpcHgyb3yjOeHk=";
};
#Описание access-листов
acl client { !key client2-key; key client-key; 192.168.50.15; };
acl client2 { !key client-key; key client2-key; 192.168.50.16; };
```

Описание ключей и access листов будет одинаковое для master и slave сервера, поэтому дописываем аналогичные настройки в /etc/named.conf сервера nso2.

Технология Split-DNS реализуется с помощью описания представлений (view), для каждого отдельного acl. В каждое представление (view) добавляются только те зоны, которые разрешено видеть хостам, адреса которых указаны в access листе.

Все ранее описанные зоны должны быть перенесены в модули view. Вне view зон быть недолжно, зона any должна всегда находиться в самом низу. 

После применения всех вышеуказанных правил на хосте ns01 мы получим следующее содержимое файла /etc/named.conf

```
[root@ns01 ~]# cat /etc/named.conf
options {

    // network
        listen-on port 53 { 192.168.50.10; };
        listen-on-v6 port 53 { ::1; };

    // data
        directory       "/var/named";
        dump-file       "/var/named/data/cache_dump.db";
        statistics-file "/var/named/data/named_stats.txt";
        memstatistics-file "/var/named/data/named_mem_stats.txt";

    // server
        recursion yes;
        allow-query     { any; };
    allow-transfer { any; };

    // dnssec
        dnssec-enable yes;
        dnssec-validation yes;

    // others
        bindkeys-file "/etc/named.iscdlv.key";
        managed-keys-directory "/var/named/dynamic";
        pid-file "/run/named/named.pid";
        session-keyfile "/run/named/session.key";
};

logging {
        channel default_debug {
                file "data/named.run";
                severity dynamic;
        };
};

// RNDC Control for client
key "rndc-key" {
    algorithm hmac-md5;
    secret "GrtiE9kz16GK+OKKU/qJvQ==";
};
controls {
        inet 192.168.50.10 allow { 192.168.50.15; } keys { "rndc-key"; };
};
key "client-key" {
        algorithm hmac-sha256;
        secret "XUeIVPZMYYGJk771qTR4HW5TDGILijvjjMfpXjKICoE=";
};
key "client2-key" {
        algorithm hmac-sha256;
        secret "B34Bp49mUHVTIfDoxzbzzuj4xECKzmpcHgyb3yjOeHk=";
};

// ZONE TRANSFER WITH TSIG
include "/etc/named.zonetransfer.key";
server 192.168.50.11 {
    keys { "zonetransfer.key"; };
};
// Указание Access листов
acl client { !key client2-key; key client-key; 192.168.50.15; };
acl client2 { !key client-key; key client2-key; 192.168.50.16; };
// Настройка первого view
view "client" {
    // Кому из клиентов разрешено подключаться, нужно указать имя access-листа
    match-clients { client; };

    // Описание зоны dns.lab для client
    zone "dns.lab" {
        // Тип сервера — мастер
        type master;
        // Добавляем ссылку на файл зоны, который создали в прошлом пункте
        file "/etc/named/named.dns.lab.client";
        // Адрес хостов, которым будет отправлена информация об изменении зоны
        also-notify { 192.168.50.11 key client-key; };
    };

    // newdns.lab zone
    zone "newdns.lab" {
        type master;
        file "/etc/named/named.newdns.lab";
        also-notify { 192.168.50.11 key client-key; };
    };
};

// Описание view для client2
view "client2" {
    match-clients { client2; };

    // dns.lab zone
    zone "dns.lab" {
        type master;
        file "/etc/named/named.dns.lab";
        also-notify { 192.168.50.11 key client2-key; };
    };

    // dns.lab zone reverse
    zone "50.168.192.in-addr.arpa" {
        type master;
        file "/etc/named/named.dns.lab.rev";
        also-notify { 192.168.50.11 key client2-key; };
    };
};
// Зона any, указана в файле самой последней
view "default" {
    match-clients { any; };
// root zone
zone "." IN {
        type hint;
        file "named.ca";
};

// zones like localhost
include "/etc/named.rfc1912.zones";
// root's DNSKEY
include "/etc/named.root.key";

// lab's zone
zone "dns.lab" {
    type master;
    allow-transfer { key "zonetransfer.key"; };
    file "/etc/named/named.dns.lab";
};

// lab's zone reverse
zone "50.168.192.in-addr.arpa" {
    type master;
    allow-transfer { key "zonetransfer.key"; };
    file "/etc/named/named.dns.lab.rev";
};

// lab's ddns zone
zone "ddns.lab" {
    type master;
    allow-transfer { key "zonetransfer.key"; };
    allow-update { key "zonetransfer.key"; };
    file "/etc/named/named.ddns.lab";
};

// lab's newdns zone
zone "newdns.lab" {
    type master;
    allow-transfer { key "zonetransfer.key"; };
    allow-update { key "zonetransfer.key"; };
    file "/etc/named/named.newdns.lab";
};
};
```

Далее внесем изменения в файл /etc/named.conf на сервере ns02. Файл будет похож на файл, лежащий на ns01, только в настройках будет указание забирать информацию с сервера ns01:

```
[root@ns02 ~]# cat /etc/named.conf
options {

    // network
        listen-on port 53 { 192.168.50.11; };
        listen-on-v6 port 53 { ::1; };

    // data
        directory       "/var/named";
        dump-file       "/var/named/data/cache_dump.db";
        statistics-file "/var/named/data/named_stats.txt";
        memstatistics-file "/var/named/data/named_mem_stats.txt";

    // server
        recursion yes;
        allow-query     { any; };
    allow-transfer { any; };

    // dnssec
        dnssec-enable yes;
        dnssec-validation yes;

    // others
        bindkeys-file "/etc/named.iscdlv.key";
        managed-keys-directory "/var/named/dynamic";
        pid-file "/run/named/named.pid";
        session-keyfile "/run/named/session.key";
};

logging {
        channel default_debug {
                file "data/named.run";
                severity dynamic;
        };
};

// RNDC Control for client
key "rndc-key" {
    algorithm hmac-md5;
    secret "GrtiE9kz16GK+OKKU/qJvQ==";
};
controls {
        inet 192.168.50.11 allow { 192.168.50.15; } keys { "rndc-key"; };
};

key "client-key" {
        algorithm hmac-sha256;
        secret "XUeIVPZMYYGJk771qTR4HW5TDGILijvjjMfpXjKICoE=";
};
key "client2-key" {
        algorithm hmac-sha256;
        secret "B34Bp49mUHVTIfDoxzbzzuj4xECKzmpcHgyb3yjOeHk=";
};

// ZONE TRANSFER WITH TSIG
include "/etc/named.zonetransfer.key";
server 192.168.50.10 {
    keys { "zonetransfer.key"; };
};
// Указание Access листов
acl client { !key client2-key; key client-key; 192.168.50.15; };
acl client2 { !key client-key; key client2-key; 192.168.50.16; };
// Настройка первого view
view "client" {
    // Кому из клиентов разрешено подключаться, нужно указать имя access-листа
    match-clients { client; };

    // Описание зоны dns.lab для client
    zone "dns.lab" {
        type slave;
        masters { 192.168.50.10; };
        file "/etc/named/named.dns.lab.client";
};

    // newdns.lab zone
    zone "newdns.lab" {
        type slave;
        masters { 192.168.50.10; };
        file "/etc/named/named.newdns.lab";
};
};
// Описание view для client2
view "client2" {
    match-clients { client2; };

    // dns.lab zone
    zone "dns.lab" {
        type slave;
        masters { 192.168.50.10; };
        file "/etc/named/named.dns.lab";
};
 // dns.lab zone reverse
    zone "50.168.192.in-addr.arpa" {
        type slave;
        masters { 192.168.50.10; };
        file "/etc/named/named.dns.lab.rev";
};
};
// Зона any, указана в файле самой последней
view "default" {
    match-clients { any; };
// root zone
zone "." IN {
        type hint;
        file "named.ca";
};

// zones like localhost
include "/etc/named.rfc1912.zones";
// root's DNSKEY
include "/etc/named.root.key";

// lab's zone
zone "dns.lab" {
    type slave;
    masters { 192.168.50.10; };
    file "/etc/named/named.dns.lab";
};

// lab's zone reverse
zone "50.168.192.in-addr.arpa" {
    type slave;
    masters { 192.168.50.10; };
    file "/etc/named/named.dns.lab.rev";
};

// lab's ddns zone
zone "ddns.lab" {
    type slave;
    masters { 192.168.50.10; };
    file "/etc/named/named.ddns.lab";
};

// lab's newdns zone
zone "newdns.lab" {
    type slave;
    masters { 192.168.50.10; };
    file "/etc/named/named.newdns.lab";
};
};
```


Так как файлы с конфигурациями получаются достаточно большими — возрастает вероятность сделать ошибку. При их правке можно воспользоваться утилитой niamed-checkconf. Она укажет в каких строчках есть ошибки. Использование данной утилиты рекомендуется после изменения настроек на DNS-сервере. 

После внесения данных изменений можно перезапустить (по очереди) службу named на серверах ns01 и ns02.

```
[root@ns02 ~]# named-checkconf
/etc/named.conf:98: syntax error near '}'
[root@ns02 ~]# vi /etc/named.conf
[root@ns02 ~]# named-checkconf
[root@ns02 ~]# systemctl restart named
```

Далее, нужно будет проверить работу Split-DNS с хостов client и client2. Для проверки можно использовать утилиту ping:

```
[vagrant@client ~]$ ping www.newdns.lab
PING www.newdns.lab (192.168.50.15) 56(84) bytes of data.
64 bytes from client (192.168.50.15): icmp_seq=1 ttl=64 time=0.033 ms
64 bytes from client (192.168.50.15): icmp_seq=2 ttl=64 time=0.028 ms
64 bytes from client (192.168.50.15): icmp_seq=3 ttl=64 time=0.025 ms
64 bytes from client (192.168.50.15): icmp_seq=4 ttl=64 time=0.024 ms
64 bytes from client (192.168.50.15): icmp_seq=5 ttl=64 time=0.024 ms
64 bytes from client (192.168.50.15): icmp_seq=6 ttl=64 time=0.029 ms
64 bytes from client (192.168.50.15): icmp_seq=7 ttl=64 time=0.026 ms
64 bytes from client (192.168.50.15): icmp_seq=8 ttl=64 time=0.028 ms
64 bytes from client (192.168.50.15): icmp_seq=9 ttl=64 time=0.029 ms
64 bytes from client (192.168.50.15): icmp_seq=10 ttl=64 time=0.024 ms
64 bytes from client (192.168.50.15): icmp_seq=11 ttl=64 time=0.022 ms
64 bytes from client (192.168.50.15): icmp_seq=12 ttl=64 time=0.022 ms
64 bytes from client (192.168.50.15): icmp_seq=13 ttl=64 time=0.025 ms
^C
--- www.newdns.lab ping statistics ---
13 packets transmitted, 13 received, 0% packet loss, time 12000ms
rtt min/avg/max/mdev = 0.022/0.026/0.033/0.003 ms
[vagrant@client ~]$ ping web1.dns.lab
PING web1.dns.lab (192.168.50.15) 56(84) bytes of data.
64 bytes from client (192.168.50.15): icmp_seq=1 ttl=64 time=0.011 ms
64 bytes from client (192.168.50.15): icmp_seq=2 ttl=64 time=0.022 ms
64 bytes from client (192.168.50.15): icmp_seq=3 ttl=64 time=0.023 ms
64 bytes from client (192.168.50.15): icmp_seq=4 ttl=64 time=0.027 ms
64 bytes from client (192.168.50.15): icmp_seq=5 ttl=64 time=0.022 ms
^C
--- web1.dns.lab ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4001ms
rtt min/avg/max/mdev = 0.011/0.021/0.027/0.005 ms
[vagrant@client ~]$ ping web2.dns.lab
ping: web2.dns.lab: Name or service not known
```

На хосте мы видим, что client видит обе зоны (dns.lab и newdns.lab), однако информацию о хосте web2.dns.lab он получить не может. 

Проверка на client2: 

```
[vagrant@client ~]$ ping www.newdns.lab
ping: www.newdns.lab: Name or service not known
[vagrant@client ~]$ ping web1.dns.lab
PING web1.dns.lab (192.168.50.15) 56(84) bytes of data.
64 bytes from 192.168.50.15 (192.168.50.15): icmp_seq=1 ttl=64 time=1.16 ms
64 bytes from 192.168.50.15 (192.168.50.15): icmp_seq=2 ttl=64 time=0.633 ms
64 bytes from 192.168.50.15 (192.168.50.15): icmp_seq=3 ttl=64 time=0.573 ms
^C
--- web1.dns.lab ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2001ms
rtt min/avg/max/mdev = 0.573/0.790/1.166/0.268 ms
[vagrant@client ~]$ ping web2.dns.lab
PING web2.dns.lab (192.168.50.16) 56(84) bytes of data.
64 bytes from client (192.168.50.16): icmp_seq=1 ttl=64 time=0.031 ms
64 bytes from client (192.168.50.16): icmp_seq=2 ttl=64 time=0.027 ms
64 bytes from client (192.168.50.16): icmp_seq=3 ttl=64 time=0.027 ms
64 bytes from client (192.168.50.16): icmp_seq=4 ttl=64 time=0.025 ms
64 bytes from client (192.168.50.16): icmp_seq=5 ttl=64 time=0.025 ms
^C
--- web2.dns.lab ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4001ms
rtt min/avg/max/mdev = 0.025/0.027/0.031/0.002 ms
```

Тут мы понимаем, что client2 видит всю зону dns.lab и не видит зону newdns.lab
Для того, чтобы проверить что master и slave сервера отдают одинаковую информацию, в файле /etc/resolv.conf можно удалить на время nameserver 192.168.50.10 и попробовать выполнить все те же проверки. Результат должен быть идентичный. 

```
[vagrant@client ~]$ cat /etc/resolv.conf
domain dns.lab
search dns.lab
#nameserver 192.168.50.10
nameserver 192.168.50.11
[vagrant@client ~]$ ping www.newdns.lab
ping: www.newdns.lab: Name or service not known
```

Настройка Split-DNS c помощью Ansible

В существующем Ansible-playbook менять ничего не потребуется. Нам потребуется изменить содержимое файлов master-named.conf и slave-named.conf, а также добавить файл named.dns.lab.client.



