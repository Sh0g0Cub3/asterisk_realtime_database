Centos7にAsterisk Realtime Databaseを構築する手順
===

## selinuxが無効か確認する
```
 ~]# getenforce
Disabled
```

## yumでパッケージの更新とパッケージインストールを行う

```
 ~]# yum update -y

 ~]# reboot

 ~]# yum -y groupinstall "Additional Development"

 ~]#  yum install -y epel-release dmidecode gcc-c++ ncurses-devel libxml2-devel make wget openssl-devel newt-devel kernel-devel sqlite-devel libuuid-devel gtk2-devel jansson-devel binutils-devel libedit libedit-devel mysql-connector-odbc  libtool-ltdl-devel
```

## Asteriskユーザを作成する

```
 ~]#  adduser asterisk -c "Asterisk User"

 ~]# passwd asterisk
ユーザー asterisk のパスワードを変更。
新しいパスワード:
新しいパスワードを再入力してください:
passwd: すべての認証トークンが正しく更新できました。

 ~]#  usermod -aG wheel asterisk

```

## Asteriskをインストールする

- make menuselectでは、日本語音声を有効化を行う。

```
 ~]# cd /usr/src/
 src]# wget http://downloads.asterisk.org/pub/telephony/asterisk/asterisk-16-current.tar.gz
--2021-08-29 14:05:53--  http://downloads.asterisk.org/pub/telephony/asterisk/asterisk-16-current.tar.gz
downloads.asterisk.org (downloads.asterisk.org) をDNSに問いあわせています... 170.249.154.172
downloads.asterisk.org (downloads.asterisk.org)|170.249.154.172|:80 に接続しています... 接続しました。
HTTP による接続要求を送信しました、応答を待っています... 200 OK
長さ: 27888074 (27M) [application/x-gzip]
`asterisk-16-current.tar.gz' に保存中

100%[====================================================================================================================================================================================================================================>] 27,888,074  4.46MB/s 時間 10s

2021-08-29 14:06:04 (2.57 MB/s) - `asterisk-16-current.tar.gz' へ保存完了 [27888074/27888074]

 src]#

 asterisk-16.20.0]# ./configure --with-jansson-bundled

 asterisk-16.20.0]# make menuselect

 asterisk-16.20.0]# make && make install && make samples && make config

 asterisk-16.20.0]# chown asterisk. /var/run/asterisk
 asterisk-16.20.0]# chown asterisk. -R /etc/asterisk
 asterisk-16.20.0]# chown asterisk. -R /var/{lib,log,spool}/asterisk
 asterisk-16.20.0]# chkconfig asterisk on
 asterisk-16.20.0]# systemctl start asterisk
```

## MariaDBをインストールする

```
 asterisk-16.20.0]# yum install -y mysql-connector-odbc mariadb mariadb-server 

 asterisk-16.20.0]# systemctl enable mariadb
Created symlink from /etc/systemd/system/multi-user.target.wants/mariadb.service to /usr/lib/systemd/system/mariadb.service.
 asterisk-16.20.0]# systemctl start mariadb

 asterisk-16.20.0]# mysql_secure_installation

NOTE: RUNNING ALL PARTS OF THIS SCRIPT IS RECOMMENDED FOR ALL MariaDB
      SERVERS IN PRODUCTION USE!  PLEASE READ EACH STEP CAREFULLY!

In order to log into MariaDB to secure it, we'll need the current
password for the root user.  If you've just installed MariaDB, and
you haven't set the root password yet, the password will be blank,
so you should just press enter here.

Enter current password for root (enter for none):
OK, successfully used password, moving on...

Setting the root password ensures that nobody can log into the MariaDB
root user without the proper authorisation.

Set root password? [Y/n] y
New password:
Re-enter new password:
Password updated successfully!
Reloading privilege tables..
 ... Success!


By default, a MariaDB installation has an anonymous user, allowing anyone
to log into MariaDB without having to have a user account created for
them.  This is intended only for testing, and to make the installation
go a bit smoother.  You should remove them before moving into a
production environment.

Remove anonymous users? [Y/n]
 ... Success!

Normally, root should only be allowed to connect from 'localhost'.  This
ensures that someone cannot guess at the root password from the network.

Disallow root login remotely? [Y/n] n
 ... skipping.

By default, MariaDB comes with a database named 'test' that anyone can
access.  This is also intended only for testing, and should be removed
before moving into a production environment.

Remove test database and access to it? [Y/n] y
 - Dropping test database...
 ... Success!
 - Removing privileges on test database...
 ... Success!

Reloading the privilege tables will ensure that all changes made so far
will take effect immediately.

Reload privilege tables now? [Y/n] y
 ... Success!

Cleaning up...

All done!  If you've completed all of the above steps, your MariaDB
installation should now be secure.

Thanks for using MariaDB!
```

## alembicを利用するためのpython関連パッケージをインストールする。

```
 asterisk-16.20.0]# yum install MySQL-python python3-pip python3-devel -y

 asterisk-16.20.0]# pip3 install alembic mysqlclient
```

## Realtime Database用のデータベースを作成する。

- config.iniは sqlalchemy.url にMariaDBのアカウントを設定する。

```
 asterisk-16.20.0]# cd /usr/src/asterisk-16.20.0/contrib/ast-db-manage/
 ast-db-manage]#

 ast-db-manage]# mv config.ini.sample config.ini

 ast-db-manage]# vim config.ini
 ast-db-manage]# cat config.ini
(略)
sqlalchemy.url = mysql://root:パスワード@localhost/asterisk
(略)

 ast-db-manage]# mysql -u root -p
Enter password:
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 10
Server version: 5.5.68-MariaDB MariaDB Server

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> CREATE DATABASE asterisk;
Query OK, 1 row affected (0.00 sec)

MariaDB [(none)]> exit
Bye
 ast-db-manage]#

 ast-db-manage]# alembic -c config.ini upgrade head
```

## データベースが作成できたことを確認し、mysqlアカウントにasteriskを追加する。

```
 ast-db-manage]# mysql -uroot -p
Enter password:
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 12
Server version: 5.5.68-MariaDB MariaDB Server

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> USE asterisk;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
MariaDB [asterisk]> show tables;
+-----------------------------+
| Tables_in_asterisk          |
+-----------------------------+
| alembic_version_config      |
| extensions                  |
| iaxfriends                  |
| meetme                      |
| musiconhold                 |
| musiconhold_entry           |
| ps_aors                     |
| ps_asterisk_publications    |
| ps_auths                    |
| ps_contacts                 |
| ps_domain_aliases           |
| ps_endpoint_id_ips          |
| ps_endpoints                |
| ps_globals                  |
| ps_inbound_publications     |
| ps_outbound_publishes       |
| ps_registrations            |
| ps_resource_list            |
| ps_subscription_persistence |
| ps_systems                  |
| ps_transports               |
| queue_members               |
| queue_rules                 |
| queues                      |
| sippeers                    |
| voicemail                   |
+-----------------------------+
26 rows in set (0.00 sec)
˜
MariaDB [asterisk]>  grant all privileges on asterisk.* to 'asterisk'@'localhost' identified by 'asterisk';
Query OK, 0 rows affected (0.00 sec)

MariaDB [asterisk]> FLUSH PRIVILEGES;
Query OK, 0 rows affected (0.01 sec)

MariaDB [asterisk]> exit
Bye
 ast-db-manage]#
```

## Asteriskがodbc経由でMariaDBに接続する設定を行う。

- /etc/odbc.iniを新規作成する。
- Asteriskの設定ファイルを修正する前に/home/rootにバックアップを取得する。

```
 ast-db-manage]# vim /etc/odbc.ini

 ast-db-manage]# cat /etc/odbc.ini
[asterisk]
Description = MySQL Asterisk
Driver = MySQL
Database = asterisk
Server = localhost
User = asterisk
Password = asterisk
Port = 3306
Option = 3

 ast-db-manage]# cd ~

 ~]# tar czvfph etc-asterisk-`date "+%Y%m%d"`.tar.gz /etc/asterisk

 ~]# vim /etc/asterisk/res_odbc.conf

 ~]# cat /etc/asterisk/res_odbc.conf
[asterisk]
enabled => yes
dsn => asterisk
username => asterisk
password => asterisk
pre-connect => yes
sanitysql => select 1
max_connections => 20
connect_timeout => 5
negative_connection_cache => 600

 ~]# vim /etc/asterisk/extconfig.conf

 ~]# cat /etc/asterisk/extconfig.conf
[settings]
ps_endpoints => odbc,asterisk
ps_auths => odbc,asterisk
ps_aors => odbc,asterisk
ps_domain_aliases => odbc,asterisk
ps_endpoint_id_ips => odbc,asterisk
ps_contacts => odbc,asterisk
voicemail => odbc,asterisk
;queues => odbc,asterisk
;queue_members => odbc,asterisk
extensions => odbc,asterisk

 ~]# vim /etc/asterisk/pjsip.conf

 ~]# cat /etc/asterisk/pjsip.conf
[global]
type=global
endpoint_identifier_order=ip,username,anonymous,header,auth_username

[0.0.0.0-udp]
type=transport
protocol=udp
bind=0.0.0.0:5060
allow_reload=yes

 ~]# vim /etc/asterisk/sorcery.conf

 ~]# cat /etc/asterisk/sorcery.conf
[res_pjsip]
endpoint=realtime,ps_endpoints
auth=realtime,ps_auths
aor=realtime,ps_aors
domain_alias=realtime,ps_domain_aliases

[res_pjsip_endpoint_identifier_ip]
identify=realtime,ps_endpoint_id_ips

 ~]# vim /etc/asterisk/modules.conf

 ~]# cat /etc/asterisk/modules.conf
[modules]
autoload=yes
preload => res_odbc.so
preload => res_config_odbc.so
load => func_realtime.so
load => pbx_realtime.so

noload => chan_sip.so
noload = chan_alsa.so
noload = chan_console.so
noload = res_hep.so
noload = res_hep_pjsip.so
noload = res_hep_rtcp.so

 ~]# systemctl restart asterisk
```

## データベースに内線の値をINSERTする。

- 例として1000の内線を作成する。
- ps_authsにINSERTするpasswordは変更する。
- from-internalのDialplanを内線が利用すると想定。

```
 ~]# mysql -u root -p
Enter password:
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 47
Server version: 5.5.68-MariaDB MariaDB Server

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> USE asterisk;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
MariaDB [asterisk]>

MariaDB [asterisk]> INSERT INTO ps_aors (id, max_contacts, qualify_frequency) VALUES (1000, 3, 60);

MariaDB [asterisk]> INSERT INTO ps_auths (id, auth_type, password, username) VALUES (1000, 'userpass', パスワード, 1000);

MariaDB [asterisk]> INSERT INTO ps_endpoints (id, transport, aors, auth, context, disallow, allow, direct_media, deny, permit, mailboxes) 
VALUES 
(100, '0.0.0.0-udp', '1000', '1000', 'from-internal', 'all', 'all', 'no', '0.0.0.0/0', '0.0.0.0/0');
```

## 内線が作成されることを確認する。

```
 ~]# asterisk -rx 'pjsip show endpoints'