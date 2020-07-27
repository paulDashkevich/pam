# pam Домашнее задание:
 1. Запретить всем пользователям, кроме группы admin логин в выходные (суббота и воскресенье), без учета праздников.
Запрет на вход в консоль можно осуществить редактированием файлов: **/etc/pam.d/sshd и /etc/pam.d/login**
Добавляем необходимые модули PAM: **pam_access.so** (настройкa доступа групп, не имеет возможности ограничений по времени) и **pam_time.so** (настройкa ограничений по времени)
```
[root@pam centos]# cat /etc/pam.d/login
#%PAM-1.0
auth [user_unknown=ignore success=ok ignore=ignore default=bad] pam_securetty.so
auth       substack     system-auth
auth       include      postlogin
account    required     pam_access.so
account    required     pam_time.so
account    required     pam_nologin.so
account    include      system-auth
password   include      system-auth
# pam_selinux.so close should be the first session rule
session    required     pam_selinux.so close
session    required     pam_loginuid.so
session    optional     pam_console.so
# pam_selinux.so open should only be followed by sessions to be executed in the user context
session    required     pam_selinux.so open
session    required     pam_namespace.so
session    optional     pam_keyinit.so force revoke
session    include      system-auth
session    include      postlogin
-session   optional     pam_ck_connector.so
```
```
[root@pam centos]# cat /etc/pam.d/login
#%PAM-1.0
auth [user_unknown=ignore success=ok ignore=ignore default=bad] pam_securetty.so
auth       substack     system-auth
auth       include      postlogin
account    required     pam_access.so
account    required     pam_time.so
account    required     pam_nologin.so
account    include      system-auth
password   include      system-auth
# pam_selinux.so close should be the first session rule
session    required     pam_selinux.so close
session    required     pam_loginuid.so
session    optional     pam_console.so
# pam_selinux.so open should only be followed by sessions to be executed in the user context
session    required     pam_selinux.so open
session    required     pam_namespace.so
session    optional     pam_keyinit.so force revoke
session    include      system-auth
session    include      postlogin
-session   optional     pam_ck_connector.so
```
Вносим изменения в файл /etc/security/time.conf, чтобы указать время запрета доступа пользователя **day**, добавляем в конец файла следущую строку, где **Wk** - обозначает выходные 
```
*;*;day;!Wk
```
При входе в консоль в выходной день получаем нечто похожее на это:
```
[root@pam centos]# ssh day@localhos
day@localhost's password:
Authentication failed.
```
 1.2 Назначаем ограничения по группе пользователей
 * Создаём группу admin
 ```
 groupadd admin
 ```
 * добавляем в неё нашего пользователя **day**
```
usermod -aG admin day
```
 * устанавливаем в систему и прописываем РАМ-модуль в ранее указанных файлах с указанием скрипта, который будет смотреть чтобы входящий пользователь был в составе группы **admin** и выполнялись условия ограничения на вход (запрет входа по выходным дням)
```
yum install pam_script -y
```
```
#%PAM-1.0

auth       required     pam_script.so /usr/local/bin/test_login.sh #указанный выше модуль cо скрпитом-сепаратором :)
auth       required     pam_sepermit.so
auth       substack     password-auth
auth       include      postlogin
# Used with polkit to reauthorize users in remote sessions
-auth      optional     pam_reauthorize.so prepare
account required pam_access.so
account required pam_time.so
account    required     pam_nologin.so
account    include      password-auth
password   include      password-auth
# pam_selinux.so close should be the first session rule
session    required     pam_selinux.so close
session    required     pam_loginuid.so
# pam_selinux.so open should only be followed by sessions to be executed in the user context
session    required     pam_selinux.so open env_params
session    required     pam_namespace.so
session    optional     pam_keyinit.so force revoke
session    include      password-auth
session    include      postlogin
# Used with polkit to reauthorize users in remote sessions
-session   optional     pam_reauthorize.so prepare
```
 * прописываем скрипт в файлe **/usr/local/bin/test_login.sh** сохраняем **:wq** и делаем исполняемым **sudo cmod +x**
```
#!/bin/bash

if [[ `grep $PAM_USER /etc/group | grep 'day'` ]]
then
exit 0
fi
if [[ `date +%u` > 5 ]]
then
exit 1
fi
```
> Скрипт запускается РАМ-модулем и проверяет логинящегося пользователя по ограничениям. Если пользователь входит в группу admin и сегодня день недели меньше, чем 6 - тогда впускаем, если день недели больше чем 5 и пользователь в группе admin - возвращаем 1 и логин запрещается.
