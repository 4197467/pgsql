* Installer mais comment ?

  - À partir des sources (notre tp)

  - À partir d'un paquet RPM (Redhat/CentOS/Rocky/Fedora)
      * Dans le dépot de la distribution : vérifier la version dispo
      * Les dépots RPM préparé par posgresql.org : version plus récente
      * On peut aussi contruire un paquet RPM à partir des sources (ça dépasse
        le contexte de notre cours)
   - Même principe pour les distribution Debian/Ubuntu/etc.

Une fois les sources téléchargées (dans ~/Téléchargement par ex) :

~~~~Bash
$ cd ~
$ tar xvf ~/Téléchargement/post<tab>
$ cd post<tab>
$ ./configure
$ sudo yum install readline-devel # si le configure échoue sur erreur concernant readline
$ make # et on prend un café
$ make check
$ sudo make install
~~~~


Commandes complémentaires pour le TP (si travaille pas en root et si
le compte postgres est "système" :

~~~~
sudo adduser --system postgres
sudo mkdir /home/postgres
sudo chown -R postgres:postgres /home/postgres
sudo su - postgres
nano .bash_profile

PATH=$PATH:$HOME/bin:/usr/local/pgsql/bin
export PATH
~~~~

Vérifications :

~~~~
$ du -sh /usr/local/pgsql
$ ls -l /usr/local/pgsql/bin
$ sudo su - postgres
$ echo $PATH
/usr/local/bin:/usr/bin:/usr/local/sbin:/usr/sbin:/home/postgres/bin:/usr/local/pgsql/bin
~~~~

* Quelles versions de PostgreSQL sont disponibles en RPM pour nous ?

- De base (sans activer les dépôts RPM gérés par postgresql.org ?

~~~~
$ yum search postgresql
$ yum info postgresql-server
Version      : 13.7 (sur une Rocky 9)
~~~~

- En ayant activé les dépôt de postgresql.org : Download|Linux|Redhat/Rocky/etc.
- après sélection de Rocky 9/Pg 14 on trouve la suggestion :

~~~~
# Install the repository RPM:
sudo dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm

# Disable the built-in PostgreSQL module:
sudo dnf -qy module disable postgresql

# Install PostgreSQL:
sudo dnf install -y postgresql14-server

# Optionally initialize the database and enable automatic start:
sudo /usr/pgsql-14/bin/postgresql-14-setup initdb
sudo systemctl enable postgresql-14
sudo systemctl start postgresql-14
~~~~

Installation à partir des sources : création d'une première instance 

~~~~
$ sudo su - postgres
$ type initdb 
$ type pg_ctl
$ initdb -D /home/postgres/data
$ pg_ctl -D /home/postgres/data -l logfile start
$ ps fxo pid,cmd | grep [p]ostgres 
~~~~

Pour une installation à partir d'un RPM, tout est décrit dans
le fichier de doc : `/usr/share/doc/postgresqlXX/README.rpm-dist`

## Modifications initiales de `/home/postgres/data/postgresql.conf`

~~~~
listen_address = '*'
shared_buffers = 1GB                    # min 128kB
work_mem = 8MB
~~~~

## Configuration avec systemd

Source d'info : https://www.postgresql.org/docs/current/server-start.html

Recompiler avec l'option `--with-systemd`, on retourne dans le répertoire des 
sources. On arrête le serveur si il tourne
~~~~
yum install systemd-devel
cd postgres-...
make clean
./configure --with-systemd
make
sudo make install
~~~~

Le fichier à copier dans `/lib/systemd/system` sous le nom
`postgresql.service` est :

~~~~
[Unit]
Description=PostgreSQL database server
Documentation=man:postgres(1)

[Service]
Type=notify
User=postgres
ExecStart=/usr/local/pgsql/bin/postgres -D /home/postgres/data
ExecReload=/bin/kill -HUP $MAINPID
KillMode=mixed
KillSignal=SIGINT
TimeoutSec=infinity

[Install]
WantedBy=multi-user.target
~~~~

Ensuite pour que postgresql (notre version compilée par nous) 
soit lancée à chaque démarrage

~~~~
sudo systemctl start postgresql
sudo systemctl status postgresql 
sudo systemctl enable postgresql # pour les prochains démarrages
~~~~

On peut rebooter pour vérifier que notre instance est lancée
automatiquement au démarrage.

On peut contrôler l'ordre de lancement, ex : tomcat après postgresl, dans le fichier `tomcat.service` on peut spécifier : 

~~~~
[Unit]
Description=Tomcat bla bla
After=postgres.service
~~~~

ou encore (dépendance + ordre) :

~~~~
Description=Tomcat bla bla...
Wants=postgres.service
~~~~

**Note : dans la suite des TPs nous ferons `sudo systemctl stop/start/restart postgresql` au lieu d'utiliser `pg_ctl`**

**Important pour la suite : vérifier que postgresql est bien défini comme un service actif pour systemd. Redémarrer votre système `sudo reboot` et ensuite :**

~~~~
sudo systemctl status postgresql
postgresql.service - PostgreSQL database server
     Loaded: loaded (/usr/lib/systemd/system/postgresql.service; enabled; vendo>
     Active: active (running) since Mon 2022-10-03 16:45:05 CEST; 16h ago
       Docs: man:postgres(1)
   Main PID: 814 (postgres)
      Tasks: 7 (limit: 23436)
     Memory: 49.2M
        CPU: 1.698s
     CGroup: /system.slice/postgresql.service
             ├─814 /usr/local/pgsql/bin/postgres -D /home/postgres/data
             ├─861 "postgres: checkpointer "
             ├─862 "postgres: background writer "
             ├─863 "postgres: walwriter "
             ├─864 "postgres: autovacuum launcher "
             ├─865 "postgres: stats collector "
             └─866 "postgres: logical replication launcher "
~~~~

Si ça ne fonctionne pas, signalez le.

## Installation sous Debian (via les dépots deb de postgresql.org)

1. Télécharger l'image netinstall de Debian GNU/Linux 11
2. L'installer dans une machine virtuelle Virtual Box (4 coeurs, 4096Mio de RAM, 100Gio de disque principal)
3. Faire de l'utilisateur créé à l'installation un admin (sudo) :

~~~~
su -
usermod -a -G sudo login_de_l'utilisateur
~~~~

Fermez la session et reconnectez-vous.

Doc à jour : https://wiki.postgresql.org/wiki/Apt

~~~~
sudo apt install curl ca-certificates gnupg
curl https://www.postgresql.org/media/keys/ACCC4CF8.asc | gpg --dearmor | sudo tee /etc/apt/trusted.gpg.d/apt.postgresql.org.gpg >/dev/null

sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'

sudo apt update
sudo apt install postgresql-14

~~~~

L'installation crée automatiquement un cluster (instance)
de la version installée nommé `main` :

~~~~
$ sudo pg_lsclusters
14 main ...
~~~~

Le service systemd associé est nommé `postgresql@14-main` :

~~~~
$ systemctl status postgresql@14-main
~~~~

Pour plus d'information voir : https://www.percona.com/blog/2019/06/24/managing-multiple-postgresql-instances-on-ubuntu-debian/

## TP 2 (adapté à notre configuration)

1. Initialiser un cluster dans /home/postgres/data : DÉJÀ FAIT
2. Inutile car on peut lancer/arrêter/relancer notre cluster avec
   `sudo systemctl restart postgresql` (pas besoin de `pg_ctl`)
3. Dans `/home/postgresql/data/postgresql.conf` modifier (valeurs
   calculées pour notre système qui a 4Gio de RAM) :

~~~~
   shared_buffer = 1GB
   work_mem = 8M
   max_connections = 100
   listen_addresses = '*'
~~~~

4. Vérifier que ça fonctionne et visualiser le journal (systemd) de notre
   cluster :

~~~~
$ sudo systemctl restart postgresql
$ sudo systemctl status postgresql
$ sudo journalctl -f -u postgresql
~~~~

5. Utiliser ALTER SYSTEM :

~~~~
$ sudo su - postgres
$ /usr/local/pgsql/bin/psql
postgres=# SHOW work_mem;
 work_mem 
----------
 8MB
(1 row)
postgres=# ALTER SYSTEM SET work_mem='15MB';
ALTER SYSTEM
# SHOW work_mem;
 work_mem 
----------
 8MB
(1 row)
~~~~

Après avoir rechargé/relancé le cluster : `systemctl reload/restart postgresql` :

~~~~
$ /usr/local/pgsql/bin/psql
psql (14.5)
Type "help" for help.

postgres=# SHOW work_mem;
 work_mem 
----------
 15MB
(1 row)
~~~~

Retrouver les modification apportées par `ALTER SYSTEM` :

~~~~
$ cat data/postgresql.auto.conf 
# Do not edit this file manually!
# It will be overwritten by the ALTER SYSTEM command.
checkpoint_timeout = '5min'
work_mem = '15MB'
~~~~

## Création de bases 

Demo :

~~~~
$ /usr/local/pgsql/bin/createdb test
~~~~

## Tablespaces : ajout d'un nouveau disque sous Linux/Virtual Box

_Optionel_ : ajouter un disque physique avec Virtual Box :

1. Éteindre la vm : `sudo poweroff`
2. Virtual Box|Configuration|Stockage
3. Cliquez sur le contrôleur concerné (SATA)
4. Bouton : Ajouter un nouvel accessoire de stockage
5. Créer un nouveau disque de 50Gio
6. Validez

La VM démarrée, on vérifie que le nouveau disque est détectée
et on repère son nom (important : éviter de détruire le disque
système d'origine).

~~~~
$ lsblk
NAME        MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda           8:0    0   50G  0 disk 
sdb           8:16   0  100G  0 disk 
├─sdb1        8:17   0    1G  0 part /boot
└─sdb2        8:18   0   99G  0 part 
  ├─rl-root 253:0    0 63,9G  0 lvm  /
  ├─rl-swap 253:1    0  3,9G  0 lvm  [SWAP]
  └─rl-home 253:2    0 31,2G  0 lvm  /home
~~~~

ICI : sous sdb on voit des volumes logique (ou des partitions),
ici c'est sda qui ne contient rien. C'est sda que je vais
partitionner et formation (ou y mettre des volumes logique,
etc. cf. cours d'admin Linux) :

ATTENTION : ne vous pas trompez pas de disque.

AU MOINDRE DOUTE : STOP !!!

1. Création d'une partition unique sur le disque : `sudo cfdisk /dev/sd?`
2. Comme le disque est complète vierge, il nous demande le type
   d'étiquette (format de table de partitions) : GPT (GUID Partition
   Table, l'état de l'art actuel sur PC 64bits)
3. Nouvelle|Taille : laisse le defaut (tout le disques)
4. Ecrire
5. Valide (si on 100% sûr de nous) : oui et Quitter

On peut vérifier que la partition est bien reconnue :

~~~~
 lsblk 
NAME        MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda           8:0    0   50G  0 disk 
└─sda1        8:1    0   50G  0 part 
sdb           8:16   0  100G  0 disk 
├─sdb1        8:17   0    1G  0 part /boot
└─sdb2        8:18   0   99G  0 part 
  ├─rl-root 253:0    0 63,9G  0 lvm  /
  ├─rl-swap 253:1    0  3,9G  0 lvm  [SWAP]
  └─rl-home 253:2    0 31,2G  0 lvm  /home
~~~~

Chez moi c'est sda1.

On formatte ensuite la partition, quel format ?
format courant actuel : ext4, xfs, experimental : btrfs.

1. Formater la partition : `mkfs.xfs /dev/sd?1`
2. Repérer son UUID : avec `sudo blkid` ou `lsblk --fs`

~~~~
$ lsblk --fs
...
$ lsblk --fs
NAME FSTYPE FSVER LABEL UUID                                   FSAVAIL FSUSE% MOUNTPOINTS
sd?                                                                           
└─sd?1
     xfs                e3ff9cc4-d2d4-4502-ba2a-e650ecc65f8c                  
...
~~~~

L'uuid est (ici) `e3ff9cc4-d2d4-4502-ba2a-e650ecc65f8c`

3. On ajoute une entrée à `/etc/fstab` :

~~~~
UUID=e3ff9cc4-d2d4-4502-ba2a-e650ecc65f8c /srv/dbdata xfs defaults 0 2
~~~~

En adaptant l'UUID

~~~~
$ sudo systemctl daemon-reload
$ sudo mkdir /srv/dbdata
$ sudo mount -a 
$ lsblk
$ lsblk 
NAME        MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
....
sd?           8:0    0   50G  0 disk 
└─sd?1        8:1    0   50G  0 part /srv/dbdata
....
~~~~

On voit la colonne MOUNTPOINTS qui contient /srv/dbdata.
Il est bien monté à partir de la partition concernée.

Pour que Posgres puisse y écire on peut changer le propritaire
du répertoire de base : `sudo chown postgres:postgres /srv/dbdata`.

Au final la sitution : nous avons un nouveau disque, partitioné
en une seule partition (nous aurions pu aussi créer un volume
logique, cf. cours d'Admin Linux) formatée (en XFS), on peut
vérifier quel est l'espace disponible dans lequel on pourra
créer un _tablespace_ :

~~~~
$ df -h 
Sys. de fichiers    Taille Utilisé Dispo Uti% Monté sur
...
/dev/sd?1              50G    390M   50G   1% /srv/dbdata
~~~~

50Gio de dispo ici...

## Tablespaces

Premier essai : dans `/home/postgresql`

~~~~
$ sudo su - postgres
$ mkdir ~/tpsp1
$ /usr/local/pgsql/bin/psql
# CREATE TABLESPACE tbsp1 LOCATION '/home/postgres/tbsp1';
CREATE TABLESPACE
Time: 4,286 ms
~~~~

## TP : Créer un table space dans /srv/dbdata (disque dédié)

1. Créer un répertoire vide dans /srv/dbdata nommé tstest1
   
   Si ça échoue, il est probable que /srv/dbdata n'appartient pas à
   postgres : `sudo chown postgres:postgres /srv/dbdata`

2. Créer le tablespace correspondant dans Postgres (même nom)
3. Créer une base de données `test_sp1` associé à ce tablespace
4. (si vous voulez : créer une table ou deux)
5. Supprimez la base de donnée `test_sp1`
6. Supprimez le tablespace et le répertoire

**Solution (exemple)**

~~~~
[postgres@localhost ~]$ whoami 
postgres
[postgres@localhost ~]$ mkdir /srv/dbdata/tstest1
[postgres@localhost ~]$ /usr/local/pgsql/bin/psql 
[local]:5432 postgres@postgres=# CREATE TABLESPACE tstest1 LOCATION '/srv/dbdata/tstest1';
CREATE TABLESPACE
Time: 7,706 ms
[local]:5432 postgres@postgres=# CREATE DATABASE test_sp1 TABLESPACE tstest1;
CREATE DATABASE
Time: 152,191 ms
[local]:5432 postgres@postgres=# \c test_sp1
You are now connected to database "test_sp1" as user "postgres".
[local]:5432 postgres@test_sp1=# CREATE TABLE people(id SERIAL, name TEXT, age INT);
CREATE TABLE
Time: 8,832 ms
[local]:5432 postgres@test_sp1=# CREATE TABLE town(id SERIAL, name TEXT, pop INT);
CREATE TABLE
Time: 6,415 ms
[local]:5432 postgres@test_sp1=# DROP DATABASE test_sp1;
ERROR:  55006: cannot drop the currently open database
LOCATION:  dropdb, dbcommands.c:877
Time: 0,438 ms
[local]:5432 postgres@test_sp1=# \c postgres
You are now connected to database "postgres" as user "postgres".
[local]:5432 postgres@postgres=# DROP DATABASE test_sp1;
DROP DATABASE
Time: 19,780 ms
[local]:5432 postgres@postgres=# DROP TABLESPACE tstest1 ;
DROP TABLESPACE
Time: 3,623 ms
[local]:5432 postgres@postgres=# \q
[postgres@localhost ~]$ rmdir /srv/dbdata/tstest1
~~~~

## Authentification et réseau

On va modifié la configation de notre VM Virtual Box pour qu'elle soit
dans le réseau local M2I :

- Configuration de la VM
- Réseau 
- Mode d'accès réseau : Accès par pont
- Notez l'adresse IP de votre serveur Rocky : `ip addr`
  Pour moi : 10.145.39.105/24
  Notez bien la vôtre !!!

**Pensez à arrêter et désactiver le firewall intégré à Rocky/CentOS/RHEL**

~~~~
$ sudo systemctl stop firewalld
$ sudo systemctl disable firewalld 
~~~~

Que souhaite-t-on comme permissions d'accès :

- Nous sommes administrateur du système Linux, donc en local
  on continue à se connecter en rôle "postgres" sans authentification
- On souhaite :
  - Permettre au rôle "postgres" de se connecter avec un mot de passe
    à partir de deux IP : celle du sytème Linux lui-même (pratique pour
    tester) et celle de notre poste Windows (`ipconfig` dans cmd pour la
    connaître et la noter)
- Plus tard on configura l'accès local ou distant d'autre rôle 
  (identifiants)

On associe un mot de passe au rôle "postgres" (ça n'empêche pas les
connexions locales, par socket, puisqu'elles sont déclarés en "trust")
(mettez un mot de passe à vous, pas trop simple)

~~~~
$ sudo su - postgres
$ /usr/local/pgsql/bin/psql
[local]:5432 postgres@postgres=# ALTER ROLE postgres PASSWD 'bidule123';
[local]:5432 postgres@postgres=# \q
~~~~ 

À la fin du fichier `data/pg_hba.conf` on ajoute :

~~~~
# IP de mon poste Windows (voir ipconfig dans cmd)
host    all             postgres        10.145.39.61/32         scram-sha-256
# IP du système Linux (pour test) 
# /usr/local/pgsql/bin/psql -h 10.145.39.105 -U postgres 
host    all             postgres        10.145.39.105/32        scram-sha-256 
~~~~

On relance le serveur et on teste en local en passant par le réseau :
(XX correspondant à l'IP de notre système Linux) :

~~~~
$ /usr/local/pgsql/bin/psql -h 10.145.39.XX -U postgres
~~~~

On peut installer pgAdmin sous Windows et l'utiliser pour 
se connecter à notre PosgreSQL.

