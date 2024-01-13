## Задание

#### 1. Запустить nginx на нестандартном порту 3-мя разными способами:
#### - переключатели setsebool;
#### - добавление нестандартного порта в имеющийся тип;
#### - формирование и установка модуля SELinux.


## Решение
#### Система: Linux selinux 3.10.0-1127.el7.x86_64 #1 SMP Tue Mar 31 23:36:51 UTC 2020 x86_64 x86_64 x86_64 GNU/Linux
Результатом выполнения команды vagrant up станет созданная виртуальная машина с установленным nginx, который работает на порту TCP 4881. Порт TCP 4881 уже проброшен до хоста. SELinux включен.
Во время развёртывания стенда попытка запустить nginx завершилась с ошибкой:

    selinux: Jan 13 09:46:03 selinux nginx[3579]: nginx: configuration file /etc/nginx/nginx.conf test failed
    selinux: Jan 13 09:46:03 selinux systemd[1]: nginx.service: control process exited, code=exited status=1
    selinux: Jan 13 09:46:03 selinux systemd[1]: Failed to start The nginx HTTP and reverse proxy server.
    selinux: Jan 13 09:46:03 selinux systemd[1]: Unit nginx.service entered failed state.
    selinux: Jan 13 09:46:03 selinux systemd[1]: nginx.service failed.

Данная ошибка появляется из-за того, что SELinux блокирует работу nginx на нестандартном порту.

#### Запустим nginx на нестандартном порту 3-мя разными способами
Для начала проверим, что в ОС отключен файервол: systemctl status firewalld:
root@selinux ~]# systemctl status firewalld
  firewalld.service - firewalld - dynamic firewall daemon
   Loaded: loaded (/usr/lib/systemd/system/firewalld.service; disabled; vendor preset: enabled)
   Active: inactive (dead)
     Docs: man:firewalld(1)

Также можно проверить, что конфигурация nginx настроена без ошибок: nginx -t:
[root@selinux ~]# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful

Далее проверим режим работы SELinux: getenforce:
[root@selinux ~]# getenforce
Enforcing

Отображается режим Enforcing. Данный режим означает, что SELinux будет блокировать запрещенную активность.
##### Разрешим в SELinux работу nginx на порту TCP 4881 c помощью переключателей setsebool
Находим в логах (/var/log/audit/audit.log) информацию о блокировании порта:
[root@selinux ~]# nano /var/log/audit/audit.log
type=AVC msg=audit(1705139163.512:1035): avc:  denied  { name_bind } for  pid=3579 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:

Копируем время, в которое был записан этот лог, и, с помощью утилиты audit2why смотрим:
root@selinux ~]# grep 1705139163.512:1035 /var/log/audit/audit.log | audit2why
type=AVC msg=audit(1705139163.512:1035): avc:  denied  { name_bind } for  pid=3579 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0

	Was caused by:
	The boolean nis_enabled was set incorrectly.
	Description:
	Allow nis to enabled

	Allow access by executing:
	# setsebool -P nis_enabled 1

Утилита audit2why покаpзала почему трафик блокируется. Исходя из вывода утилиты, мы видим, что нам нужно поменять параметр nis_enabled.
Включим параметр nis_enabled и перезапустим nginx: setsebool -P nis_enabled on:
root@selinux ~]# setsebool -P nis_enabled on
[root@selinux ~]# systemctl restart nginx
[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Sat 2024-01-13 10:05:16 UTC; 8s ago
  Process: 3887 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 3885 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 3884 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 3889 (nginx)
   CGroup: /system.slice/nginx.service
           ├─3889 nginx: master process /usr/sbin/nginx
           └─3891 nginx: worker process

Jan 13 10:05:16 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Jan 13 10:05:16 selinux nginx[3885]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Jan 13 10:05:16 selinux nginx[3885]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Jan 13 10:05:16 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
[root@selinux ~]#

Также можно проверить работу nginx из браузера. Заходим в любой браузер на хостовой машине и переходим по адресу http://192.168.56.101:4881
#### См. Pic 1

Проверить статус параметра можно с помощью команды: getsebool -a | grep nis_enabled
root@selinux ~]# getsebool -a | grep nis_enabled
nis_enabled --> on

Вернём запрет работы nginx на порту 4881 обратно. Для этого отключим nis_enabled: setsebool -P nis_enabled off
После отключения nis_enabled служба nginx снова не запустится:
root@selinux ~]# setsebool -P nis_enabled off
[root@selinux ~]# systemctl restart nginx
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
[root@selinux ~]# systemctl status nginx
 nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: failed (Result: exit-code) since Sat 2024-01-13 10:09:20 UTC; 2s ago
  Process: 3887 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 3911 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=1/FAILURE)
  Process: 3910 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 3889 (code=exited, status=0/SUCCESS)

Jan 13 10:09:20 selinux systemd[1]: Stopped The nginx HTTP and reverse proxy server.
Jan 13 10:09:20 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Jan 13 10:09:20 selinux nginx[3911]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Jan 13 10:09:20 selinux nginx[3911]: nginx: [emerg] bind() to 0.0.0.0:4881 failed (13: Permission denied)
Jan 13 10:09:20 selinux nginx[3911]: nginx: configuration file /etc/nginx/nginx.conf test failed
Jan 13 10:09:20 selinux systemd[1]: nginx.service: control process exited, code=exited status=1
Jan 13 10:09:20 selinux systemd[1]: Failed to start The nginx HTTP and reverse proxy server.
Jan 13 10:09:20 selinux systemd[1]: Unit nginx.service entered failed state.
Jan 13 10:09:20 selinux systemd[1]: nginx.service failed.


##### Теперь разрешим в SELinux работу nginx на порту TCP 4881 c помощью добавления нестандартного порта в имеющийся тип:

Поиск имеющегося типа, для http трафика: semanage port -l | grep http:
root@selinux ~]# semanage port -l | grep http
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989

Добавим порт в тип http_port_t: emanage port -a -t http_port_t -p tcp 4881:
root@selinux ~]# semanage port -a -t http_port_t -p tcp 4881
[root@selinux ~]# semanage port -l | grep  http_port_t
http_port_t                    tcp      4881, 80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988

Теперь перезапустим службу nginx и проверим её работу: systemctl restart nginx:
root@selinux ~]# systemctl restart nginx
[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Sat 2024-01-13 10:14:40 UTC; 2s ago
  Process: 3936 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 3934 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 3933 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 3938 (nginx)
   CGroup: /system.slice/nginx.service
           ├─3938 nginx: master process /usr/sbin/nginx
           └─3940 nginx: worker process

Jan 13 10:14:40 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Jan 13 10:14:40 selinux nginx[3934]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Jan 13 10:14:40 selinux nginx[3934]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Jan 13 10:14:40 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.

Также можно проверить работу nginx из браузера. Заходим в любой браузер на хосте и переходим по адресу http://192.168.56.101:4881

#### См. Pic 1

Удалить нестандартный порт из имеющегося типа можно с помощью команды: semanage port -d -t http_port_t -p tcp 4881:
root@selinux ~]# semanage port -d -t http_port_t -p tcp 4881
[root@selinux ~]# semanage port -l | grep  http_port_t
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
[root@selinux ~]#
[root@selinux ~]# systemctl restart nginx
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
[root@selinux ~]# systemctl status nginx
 nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: failed (Result: exit-code) since Sat 2024-01-13 10:17:43 UTC; 1s ago
  Process: 3936 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 3959 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=1/FAILURE)
  Process: 3958 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 3938 (code=exited, status=0/SUCCESS)

Jan 13 10:17:43 selinux systemd[1]: Stopped The nginx HTTP and reverse proxy server.
Jan 13 10:17:43 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Jan 13 10:17:43 selinux nginx[3959]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Jan 13 10:17:43 selinux nginx[3959]: nginx: [emerg] bind() to 0.0.0.0:4881 failed (13: Permission denied)
Jan 13 10:17:43 selinux nginx[3959]: nginx: configuration file /etc/nginx/nginx.conf test failed
Jan 13 10:17:43 selinux systemd[1]: nginx.service: control process exited, code=exited status=1
Jan 13 10:17:43 selinux systemd[1]: Failed to start The nginx HTTP and reverse proxy server.
Jan 13 10:17:43 selinux systemd[1]: Unit nginx.service entered failed state.
Jan 13 10:17:43 selinux systemd[1]: nginx.service failed.

##### Разрешим в SELinux работу nginx на порту TCP 4881 c помощью формирования и установки модуля SELinux:
Воспользуемся утилитой audit2allow для того, чтобы на основе логов SELinux сделать модуль, разрешающий работу nginx на нестандартном порту:
root@selinux ~]# grep nginx /var/log/audit/audit.log | audit2allow -M nginx
******************** IMPORTANT ***********************
To make this policy package active, execute:

semodule -i nginx.pp

Audit2allow сформировал модуль, и сообщил нам команду, с помощью которой можно применить данный модуль: semodule -i nginx.pp:
root@selinux ~]# semodule -i nginx.pp
[root@selinux ~]# systemctl restart nginx
[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Sat 2024-01-13 10:21:29 UTC; 1s ago
  Process: 3987 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 3985 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 3984 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 3989 (nginx)
   CGroup: /system.slice/nginx.service
           ├─3989 nginx: master process /usr/sbin/nginx
           └─3991 nginx: worker process

Jan 13 10:21:29 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Jan 13 10:21:29 selinux nginx[3985]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Jan 13 10:21:29 selinux nginx[3985]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Jan 13 10:21:29 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.

После добавления модуля nginx запустился без ошибок. При использовании модуля изменения сохранятся после перезагрузки.

Просмотр всех установленных модулей: semodule -l
Результатом будет длинный список моделей.
Для удаления модуля воспользуемся командой: semodule -r nginx:
root@selinux ~]# semodule -r nginx
libsemanage.semanage_direct_remove_key: Removing last nginx module (no other nginx module exists at another priority).


#### 2. Обеспечение работоспособности приложения при включенном SELinux
Выполним клонирование репозитория: git clone https://github.com/mbfx/otus-linux-adm.git
Перейдём в каталог со стендом: cd otus-linux-adm/selinux_dns_problems
Развернём 2 ВМ с помощью vagrant: vagrant up
После того, как стенд развернется, проверим ВМ с помощью команды: vagrant status
sabo@sabo-HP-Pavilion-Notebook:~/LESSONS/otus-linux-adm/selinux_dns_problems$ vagrant status
Current machine states:

ns01                      running (virtualbox)
client                    running (virtualbox)

This environment represents multiple VMs. The VMs are all listed
above with their current state. For more information about a specific
VM, run `vagrant status NAME`.

Подключимся к клиенту: vagrant ssh client:

Попробуем внести изменения в зону: nsupdate -k /etc/named.zonetransfer.key
[root@client ~]# nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
update failed: SERVFAIL
> quit

Изменения внести не получилось. Давайте посмотрим логи SELinux, чтобы понять в чём может быть проблема.
Для этого воспользуемся утилитой audit2why: 	
[root@client ~]# cat /var/log/audit/audit.log | audit2why
[root@client ~]#

Тут мы видим, что на клиенте отсутствуют ошибки.
Не закрывая сессию на клиенте, подключимся к серверу ns01 и проверим логи SELinux:
[root@ns01 ~]# cat /var/log/audit/audit.log | audit2why
type=AVC msg=audit(1705142271.672:1933): avc:  denied  { create } for  pid=5194 comm="isc-worker0000" name="named.ddns.lab.view1.jnl" scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file permissive=0

	Was caused by:
		Missing type enforcement (TE) allow rule.

		You can use audit2allow to generate a loadable module to allow this access.

В логах мы видим, что ошибка в контексте безопасности. Вместо типа named_t используется тип etc_t.
Проверим данную проблему в каталоге /etc/named:
[root@ns01 ~]# ls -laZ /etc/named
drw-rwx---. root named system_u:object_r:etc_t:s0       .
drwxr-xr-x. root root  system_u:object_r:etc_t:s0       ..
drw-rwx---. root named unconfined_u:object_r:etc_t:s0   dynamic
-rw-rw----. root named system_u:object_r:etc_t:s0       named.50.168.192.rev
-rw-rw----. root named system_u:object_r:etc_t:s0       named.dns.lab
-rw-rw----. root named system_u:object_r:etc_t:s0       named.dns.lab.view1
-rw-rw----. root named system_u:object_r:etc_t:s0       named.newdns.lab

Тут мы также видим, что контекст безопасности неправильный. Проблема заключается в том, что конфигурационные файлы лежат в другом каталоге. Посмотреть в каком каталоги должны лежать, файлы, чтобы на них распространялись правильные политики SELinux можно с помощью команды: sudo semanage fcontext -l | grep named

[root@ns01 ~]# sudo semanage fcontext -l | grep named
/etc/rndc.*                                        regular file       system_u:object_r:named_conf_t:s0
/var/named(/.*)?                                   all files          system_u:object_r:named_zone_t:s0

Изменим тип контекста безопасности для каталога /etc/named: sudo chcon -R -t named_zone_t /etc/named
[root@ns01 ~]# sudo chcon -R -t named_zone_t /etc/named
[root@ns01 ~]# ls -laZ /etc/named
drw-rwx---. root named system_u:object_r:named_zone_t:s0 .
drwxr-xr-x. root root  system_u:object_r:etc_t:s0       ..
drw-rwx---. root named unconfined_u:object_r:named_zone_t:s0 dynamic
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.50.168.192.rev
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.dns.lab
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.dns.lab.view1
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.newdns.lab

Попробуем снова внести изменения с клиента:
[root@client ~]# nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
> quit

[root@client ~]# dig www.ddns.lab

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.15 <<>> www.ddns.lab
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 15982
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 2

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;www.ddns.lab.			IN	A

;; ANSWER SECTION:
www.ddns.lab.		60	IN	A	192.168.50.15

;; AUTHORITY SECTION:
ddns.lab.		3600	IN	NS	ns01.dns.lab.

;; ADDITIONAL SECTION:
ns01.dns.lab.		3600	IN	A	192.168.50.10

;; Query time: 0 msec
;; SERVER: 192.168.50.10#53(192.168.50.10)
;; WHEN: Sat Jan 13 10:47:30 UTC 2024
;; MSG SIZE  rcvd: 96

Видим, что изменения применились. Попробуем перезагрузить хосты и ещё раз сделать запрос с помощью dig:
[root@client ~]# dig www.ddns.lab

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.15 <<>> www.ddns.lab
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NXDOMAIN, id: 50375
;; flags: qr rd ra; QUERY: 1, ANSWER: 0, AUTHORITY: 1, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 65494
;; QUESTION SECTION:
;www.ddns.lab.			IN	A

;; AUTHORITY SECTION:
.			86400	IN	SOA	a.root-servers.net. nstld.verisign-grs.com. 2024011300 1800 900 604800 86400

;; Query time: 272 msec
;; SERVER: 10.0.2.3#53(10.0.2.3)
;; WHEN: Sat Jan 13 10:49:58 UTC 2024
;; MSG SIZE  rcvd: 116

Всё правильно. После перезагрузки настройки сохранились.
Для того, чтобы вернуть правила обратно, можно ввести команду: restorecon -v -R /etc/named
[root@ns01 ~]# restorecon -v -R /etc/named
restorecon reset /etc/named context system_u:object_r:named_zone_t:s0->system_u:object_r:etc_t:s0
restorecon reset /etc/named/named.dns.lab.view1 context system_u:object_r:named_zone_t:s0->system_u:object_r:etc_t:s0
restorecon reset /etc/named/named.dns.lab context system_u:object_r:named_zone_t:s0->system_u:object_r:etc_t:s0
restorecon reset /etc/named/dynamic context unconfined_u:object_r:named_zone_t:s0->unconfined_u:object_r:etc_t:s0
restorecon reset /etc/named/dynamic/named.ddns.lab context system_u:object_r:named_zone_t:s0->system_u:object_r:etc_t:s0
restorecon reset /etc/named/dynamic/named.ddns.lab.view1 context system_u:object_r:named_zone_t:s0->system_u:object_r:etc_t:s0
restorecon reset /etc/named/dynamic/named.ddns.lab.view1.jnl context system_u:object_r:named_zone_t:s0->system_u:object_r:etc_t:s0
restorecon reset /etc/named/named.newdns.lab context system_u:object_r:named_zone_t:s0->system_u:object_r:etc_t:s0
restorecon reset /etc/named/named.50.168.192.rev context system_u:object_r:named_zone_t:s0->system_u:object_r:etc_t:s0



# Дополнительно
### Поиск информации по отказу. Запустить команду: grep "SELinux is preventing" /var/log/messages
[root@ns01 ~]# grep "SELinux is preventing" /var/log/messages
Jan 13 10:37:55 localhost setroubleshoot: SELinux is preventing isc-worker0000 from create access on the file named.ddns.lab.view1.jnl. For complete SELinux messages run: sealert -l 7adef65e-8fee-4e76-895f-9090c9caf91d
Jan 13 10:37:55 localhost python: SELinux is preventing isc-worker0000 from create access on the file named.ddns.lab.view1.jnl.#012#012*****  Plugin catchall_labels (83.8 confidence) suggests   *******************#012#012If you want to allow isc-worker0000 to have create access on the named.ddns.lab.view1.jnl file#012Then you need to change the label on named.ddns.lab.view1.jnl#012Do#012# semanage fcontext -a -t FILE_TYPE 'named.ddns.lab.view1.jnl'#012where FILE_TYPE is one of the following: dnssec_trigger_var_run_t, ipa_var_lib_t, krb5_host_rcache_t, krb5_keytab_t, named_cache_t, named_log_t, named_tmp_t, named_var_run_t, named_zone_t.#012Then execute:#012restorecon -v 'named.ddns.lab.view1.jnl'#012#012#012*****  Plugin catchall (17.1 confidence) suggests   **************************#012#012If you believe that isc-worker0000 should be allowed create access on the named.ddns.lab.view1.jnl file by default.#012Then you should report this as a bug.#012You can generate a local policy module to allow this access.#012Do#012allow this access for now by executing:#012# ausearch -c 'isc-worker0000' --raw | audit2allow -M my-iscworker0000#012# semodule -i my-iscworker0000.pp#012
..........................
### Далее запускаем предложенную команду:
sealert -l 7adef65e-8fee-4e76-895f-9090c9caf91d
[root@ns01 ~]# sealert -l 7adef65e-8fee-4e76-895f-9090c9caf91d
SELinux is preventing isc-worker0000 from create access on the file tmp-gVzXFDP0wW.

*****  Plugin catchall_labels (83.8 confidence) suggests   *******************

If you want to allow isc-worker0000 to have create access on the tmp-gVzXFDP0wW file
Then you need to change the label on tmp-gVzXFDP0wW
Do
# semanage fcontext -a -t FILE_TYPE 'tmp-gVzXFDP0wW'
where FILE_TYPE is one of the following: dnssec_trigger_var_run_t, ipa_var_lib_t, krb5_host_rcache_t, krb5_keytab_t, named_cache_t, named_log_t, named_tmp_t, named_var_run_t, named_zone_t.
Then execute:
restorecon -v 'tmp-gVzXFDP0wW'


*****  Plugin catchall (17.1 confidence) suggests   **************************

If you believe that isc-worker0000 should be allowed create access on the tmp-gVzXFDP0wW file by default.
Then you should report this as a bug.
You can generate a local policy module to allow this access.
Do
allow this access for now by executing:
# ausearch -c 'isc-worker0000' --raw | audit2allow -M my-iscworker0000
# semodule -i my-iscworker0000.pp


Additional Information:
Source Context                system_u:system_r:named_t:s0
Target Context                system_u:object_r:etc_t:s0
Target Objects                tmp-gVzXFDP0wW [ file ]
Source                        isc-worker0000
Source Path                   isc-worker0000
Port                          <Unknown>
Host                          ns01
Source RPM Packages           
Target RPM Packages           
Policy RPM                    selinux-policy-3.13.1-266.el7.noarch
Selinux Enabled               True
Policy Type                   targeted
Enforcing Mode                Enforcing
Host Name                     ns01
Platform                      Linux ns01 3.10.0-1127.el7.x86_64 #1 SMP Tue Mar
                              31 23:36:51 UTC 2020 x86_64 x86_64
Alert Count                   3
First Seen                    2024-01-13 10:37:51 UTC
Last Seen                     2024-01-13 18:11:13 UTC
Local ID                      7adef65e-8fee-4e76-895f-9090c9caf91d

Raw Audit Messages
type=AVC msg=audit(1705169473.800:746): avc:  denied  { create } for  pid=704 comm="isc-worker0000" name="tmp-gVzXFDP0wW" scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file permissive=0

# Также при поиске отказа первоначально можно отключить SElinux, чтобы убедиться что причина отказа - это политики SELinux.
# ИТОГО:
Список устранения неполадок выглядит следующим образом:

1. Проверьте исключения брандмауэра для портов вашего приложения.

2. Проверьте разрешения файловой системы, чтобы убедиться, что ваша учетная запись службы имеет правильные разрешения на чтение, запись и выполнение там, где это необходимо.

3. Проверьте предварительные требования и зависимости вашего приложения.

4. Проверьте файлы /var/log/messages и /var/log/audit/audit.log на наличие отказов SELinux.

Разрешающий режим SELinux можно ненадолго использовать, чтобы проверить, не является ли SELinux виновником того, что ваше приложение не работает. После того, как вы определили, что это проблема, верните ее в принудительный режим и начните изменять соответствующие контексты.
