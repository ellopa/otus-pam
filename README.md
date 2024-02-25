## Vagrant-стенд c PAM

**Цель домашнего задания**

Научиться создавать пользователей и добавлять им ограничения

**Описание домашнего задания**
1) Запретить всем пользователям, кроме группы admin, логин в выходные (суббота и воскресенье), без учета праздников

2) *дать конкретному пользователю права работать с докером и возможность рестартить докер сервис

### Введение

Почти все операционные системы Linux — многопользовательские. Администратор Linux должен уметь создать и настраивать пользователей.

В Linux есть 3 группы пользователей: 
**Администраторы** — привелегированные пользователи с полным доступом к системе. По умолчанию в ОС есть такой пользователь — root
**Локальные пользователи** — их учётные записи создаёт администратор, их права ограничены. Администраторы могут изменять права локальных пользователей
**Системные пользователи** — учетный записи, которые создаются системой для внутрениих процессов и служб. Например пользователь — nginx

У каждого пользователя есть свой уникальный идентификатор — UID. Чтобы упростить процесс настройки прав для новых пользователей, их объединяют в группы. Каждая группа имеет свой набор прав и ограничений. Любой пользователь, создаваемый или добавляемый в такую группу, автоматически их наследует. Если при добавлении пользователя для него не указать группу, то у него будет своя, индивидуальная группа — с именем пользователя. 
Один пользователь может одновременно входить в несколько групп.
**Информацию о каждом пользователе сервера можно посмотреть в файле /etc/passwd**

Для более точных настроек пользователей можно использовать подключаемые модули аутентификации (PAM)
**PAM (Pluggable Authentication Modules - подключаемые модули аутентификации) — набор библиотек, которые позволяют интегрировать различные методы аутентификации в виде единого API.**

PAM решает следующие задачи: 
**Аутентификация** — процесс подтверждения пользователем своей подлиности. Например: ввод логина и пароля, ssh-ключ и т д. 
**Авторизация** — процесс наделения пользователя правами
**Отчетность** — запись информации о произошедших событиях

PAM может быть реализован несколькоми способами: 
**Модуль pam_time** — настройка доступа для пользователя с учётом времени
**Модуль pam_exec** — настройка доступа для пользователей с помощью скириптов
И т. д. 

### Функциоанльные и нефункциональные требования
ПК на Unix c 8ГБ ОЗУ или виртуальная машина с включенной Nested Virtualization.
Созданный аккаунт на GitHub - https://github.com/ 
Если Вы находитесь в России, для корректной работы Вам может потребоваться VPN.

Предварительно установленное и настроенное следующее ПО:

[Hashicorp Vagrant](https://www.vagrantup.com/downloads) 
[Oracle VirtualBox](https://www.virtualbox.org/wiki/Linux_Downloads). 
Любой редактор кода, например Visual Studio Code, Atom и т.д.
192.168.56.11 generic/centos8s

### Настройка запрета для всех пользователей (кроме группы Admin) логина в выходные дни (Праздники не учитываются)

- Подключаемся к нашей созданной ВМ: vagrant ssh
- Переходим в root-пользователя: sudo -i
```
elena_leb@ubuntunbleb:~/PAM_DZ$ vagrant ssh
[vagrant@pam ~]$ sudo -i
```
- Создаём пользователя otusadm и otus: sudo useradd otusadm && sudo useradd otus
```
[root@pam ~]# useradd otusadm && useradd otus
```
- Создаём пользователям пароли: echo "Otus2022!" | sudo passwd --stdin otusadm && echo "Otus2022!" | sudo passwd --stdin otus
Для примера мы указываем одинаковые пароли для пользователя otus и otusadm
```
[root@pam ~]# echo "Otus2022!" | passwd --stdin otusadm && echo "Otus2022!" | passwd --stdin otus
Changing password for user otusadm.
passwd: all authentication tokens updated successfully.
Changing password for user otus.
passwd: all authentication tokens updated successfully.
```
- Создаём группу admin: sudo groupadd -f admin
```
groupadd -f admin
```
- Добавляем пользователей vagrant,root и otusadm в группу admin:
  usermod otusadm -a -G admin && usermod root -a -G admin && usermod vagrant -a -G admin
```
[root@pam ~]# usermod otusadm -a -G admin && usermod root -a -G admin && usermod vagrant -a -G admin
```
>- Обратите внимание, что мы просто добавили пользователя otusadm в группу admin. Это не делает пользователя otusadm администратором.

После создания пользователей, нужно проверить, что они могут подключаться по SSH к нашей ВМ. Для этого пытаемся подключиться с хостовой машины: 
ssh otus@192.168.56.11
Далее вводим наш созданный пароль. 
```
elena_leb@ubuntunbleb:~/PAM_DZ$ ssh otus@192.168.56.11
otus@192.168.56.11's password: 
[otus@pam ~]$ whoami
otus
[otus@pam ~]$ exit
logout
Connection to 192.168.56.11 closed.
elena_leb@ubuntunbleb:~/PAM_DZ$ ssh otusadm@192.168.56.11
otusadm@192.168.56.11's password: 
[otusadm@pam ~]$ whoami
otusadm
[otusadm@pam ~]$ exit
logout
Connection to 192.168.56.11 closed.
```
### Далее настроим правило, по которому все пользователи кроме тех, что указаны в группе admin не смогут подключаться в выходные дни:

- Проверим, что пользователи root, vagrant и otusadm есть в группе admin:
```
[root@pam ~]# cat /etc/group | grep admin
admin:x:1003:otusadm,root,vagrant
```
>- **Информация о группах и пользователях в них хранится в файле /etc/group, пользователи указываются через запятую.** 

- Выберем метод PAM-аутентификации, так как у нас используется только ограничение по времени, то было бы логично использовать метод pam_time, однако, данный метод не работает с локальными группами пользователей, и, получается, что использование данного метода добавит нам большое количество однообразных строк с разными пользователями. В текущей ситуации лучше написать небольшой скрипт контроля и использовать модуль pam_exec

- Создадим файл-скрипт /usr/local/bin/login.sh

vi /usr/local/bin/login.sh

```
#!/bin/bash
#Первое условие: если день недели суббота или воскресенье
if [ $(date +%a) = "Sat" ] || [ $(date +%a) = "Sun" ]; then
 #Второе условие: входит ли пользователь в группу admin
 if getent group admin | grep -qw "$PAM_USER"; then
        #Если пользователь входит в группу admin, то он может подключиться
        exit 0
      else
        #Иначе ошибка (не сможет подключиться)
        exit 1
    fi
  #Если день не выходной, то подключиться может любой пользователь
  else
    exit 0
fi
```
В скрипте подписаны все условия. Скрипт работает по принципу: 
Если сегодня суббота или воскресенье, то нужно проверить, входит ли пользователь в группу admin, если не входит — то подключение запрещено. При любых других вариантах подключение разрешено. 

- Добавим права на исполнение файла: chmod +x /usr/local/bin/login.sh
```
chmod +x /usr/local/bin/login.sh
```
- Меняем /etc/pam.d/sshd для прохождения дополнительной аутентификации через модуль pam_exec:
```
sudo sed  -i -E "s/account.+required.+pam_nologin.so/account    required     pam_nologin.so\naccount    required    pam_exec.so    \/usr\/local\/bin\/login.sh/" /etc/pam.d/sshd
```
```
#%PAM-1.0
auth	   substack     password-auth
auth	   include	postlogin
account    required     pam_sepermit.so
account    required     pam_nologin.so
account    required    pam_exec.so    /usr/local/bin/login.sh
account    include	password-auth
password   include	password-auth
# pam_selinux.so close should be the first session rule
session    required     pam_selinux.so close
session    required     pam_loginuid.so
# pam_selinux.so open should only be followed by sessions to be executed in the user context
session    required     pam_selinux.so open env_params
session    required     pam_namespace.so
session    optional     pam_keyinit.so force revoke
session    optional     pam_motd.so
session    include	password-auth
session    include	postlogin
```
### Проверка
```
elena_leb@ubuntunbleb:~/PAM_DZ$ ssh otus@192.168.56.11
otus@192.168.56.11's password: 
/usr/local/bin/login.sh failed: exit code 1
Connection closed by 192.168.56.11 port 22
elena_leb@ubuntunbleb:~/PAM_DZ$ ssh otusadm@192.168.56.11
otusadm@192.168.56.11's password: 
Last failed login: Sun Feb 25 14:36:01 UTC 2024 from 192.168.56.1 on ssh:notty
There was 1 failed login attempt since the last successful login.
Last login: Sun Feb 25 14:10:54 2024 from 192.168.56.1
```

