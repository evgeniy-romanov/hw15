# PAM

Домашнее задание:

1. Запретить всем пользователям, кроме группы admin логин в выходные (суббота и воскресенье), без учета праздников
* дать конкретному пользователю права работать с докером и возможность рестартить докер сервис


Проверить выполнение дз:
- в выходные должно пускать под admin1, под notadmin1 не пустит.

- Пользователь dockerUser может выполнять команду ```sudo systemctl restart docker```, проверить, что докер перезапущен ``` systemctl status docker```



Описание работы.

*Запретить всем пользователям, кроме группы admin логин в выходные (суббота и воскресенье), без учета праздников*

Реализовано через ```pam_script```.

Установлены epel-release + pam_script
```
yum install epel-release -y
yum install pam_script -y
```
Добавляем в ```/etc/pam.d/sshd``` соответствующую запись ```auth  required  pam_script.so```.
Сам скрипт скопировали из текущей директории на vm и выдали права на выполнение
```
[root@PAM ~]# nano pam_script

#!/usr/bin/env bash

if [[ `grep "admin.*$(echo $PAM_USER)" /etc/group` ]]
then
    exit 0
fi

if [[ `date +%u` > 5 ]]
then
    exit 1
else
    exit 0
fi

[root@PAM ~]# chmod +x pam_script
[root@PAM ~]# cp /root/pam_script /etc/
```
Создаём группу *admin*, создаём пользователя admin и делаем его администратором, задаём пароль.
```
[root@PAM ~]# groupadd admin
[root@PAM ~]# useradd -G admin admin1
[root@PAM ~]# echo "123Pass" | sudo passwd --stdin admin1
Changing password for user admin1.
passwd: all authentication tokens updated successfully.
```
Создаём пользователя notadmin1, задаём пароль.
```
[root@PAM ~]# useradd notadmin1
[root@PAM ~]# echo "123Pass" | sudo passwd --stdin notadmin1
Changing password for user notadmin1.
passwd: all authentication tokens updated successfully.
```
Чтобы быть уверенными, что нам разрешен вход через ssh по паролю выполним:
```
bash -c "sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/g' /etc/ssh/sshd_config"
```
Перезапускаем сервис ```systemctl restart sshd``` - можно проверять

Зайдем по ssh в понедельник (на момент выполнения д/з)
```
[root@PAM ~]# date
Mon Jul 25 15:07:15 UTC 2022

evgeniy@home:~/hw15$ ssh admin1@192.168.56.5
The authenticity of host '192.168.56.5 (192.168.56.5)' can't be established.
ECDSA key fingerprint is SHA256:zg2HnD9hUxaEUT7hh53BOizOcuS4IRenm0OEdeymcwo.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.56.5' (ECDSA) to the list of known hosts.
admin1@192.168.56.5's password: 
[admin1@PAM ~]$ 
[admin1@PAM ~]$ exit
logout
Connection to 192.168.56.5 closed.

evgeniy@home:~/hw15$ ssh notadmin1@192.168.56.5
notadmin1@192.168.56.5's password: 
[notadmin1@PAM ~]$ exit
logout
Connection to 192.168.56.5 closed.
evgeniy@home:~/hw15$ 
```
Изменим дату на выходной день (на 24.07.2022) и проверим:
```
[root@PAM ~]# date +%Y%m%d -s "20220724"

evgeniy@home:~/hw15$ ssh admin1@192.168.56.5
admin1@192.168.56.5's password: 
Last login: Mon Jul 25 15:08:41 2022 from 192.168.56.1
[admin1@PAM ~]$ 
[admin1@PAM ~]$ exit
logout
Connection to 192.168.56.5 closed.
evgeniy@home:~/hw15$ ssh notadmin1@192.168.56.5
notadmin1@192.168.56.5's password: 
Permission denied, please try again.
```
Как видим пользователь notadmin1, не может подключиться в выходной день.
____________________________________
*дать конкретному пользователю права работать с докером и возможность рестартить докер сервис*

Создаём пользователя, задаём пароль. Группа ```docker``` создаётся в процессе установки docker'а, поэтому её отдельно не задаю.
```
[root@PAM ~]# groupadd docker
[root@PAM ~]# useradd dockerUser
[root@PAM ~]# echo "123Pass" | sudo passwd --stdin dockerUser
Changing password for user dockerUser.
passwd: all authentication tokens updated successfully.
```
Манипуляции для установки docker'а:
```
[root@PAM ~]# yum check-update
[root@PAM ~]# curl -fsSL https://get.docker.com/ | sh
[root@PAM ~]# systemctl start docker
```
Чтобы разрешить пользователю dockerUser работать с docker, его необходимо добавить в группу:
```
[root@PAM ~]# usermod -aG docker dockerUser
[root@PAM ~]# su dockerUser
[dockerUser@PAM root]$ docker run hello-world

Hello from Docker!
This message shows that your installation appears to be working correctly.
```
Делаем рестарт Docker-a:
```
[dockerUser@PAM root]$ systemctl restart docker
==== AUTHENTICATING FOR org.freedesktop.systemd1.manage-units ===
Для управления системными службами и юнитами, необходимо пройти аутентификацию.
Authenticating as: dockerUser
Password: 
==== AUTHENTICATION COMPLETE ===
[dockerUser@PAM root]$ 
```
Так же есть ещё стандартный способ, это добавить пользователя в sudoers, т.е. дать пользователю право выполнять команду от имени root. Здесь необходимо добавить пользователя в группу wheel командой ```usermod -aG wheel dockerUser```, после чего можно выполнять команды с приставкой sudo.




