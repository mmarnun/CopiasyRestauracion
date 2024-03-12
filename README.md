# Copias de Seguridad y Restauración


## 1. Realiza una copia de seguridad lógica de tu base de datos completa, teniendo en cuenta los siguientes requisitos:

- La copia debe estar encriptada y comprimida.
- Debe realizarse en un conjunto de ficheros con un tamaño máximo de 75 MB.
- Programa la operación para que se repita cada día a una hora determinada.


En primer lugar le daremos al usuario alex el privilegio para permitirle realizar copias de seguridad.

```
SQL> GRANT DBA TO alex;

Concesion terminada correctamente.
```

Crearemos un directorio para almacenar las copias de seguridad, además de cambiarle los permisos para que oracle tenga acceso a ella.

```
debian@agb:~$ sudo mkdir /opt/oracle/copias
debian@agb:~$ sudo chown oracle:oinstall /opt/oracle/copias/
```

Desde la linea de comandos de oracle crearemos el directorio con el nombre de **COPIAS** que se refiere al directorio normal indicado. Asimismo le daremos permisos de lectura y escritura al usuario alex en el directorio.

```
SQL> CREATE DIRECTORY COPIAS AS '/opt/oracle/copias/';

Directorio creado.

SQL> GRANT READ,WRITE ON DIRECTORY COPIAS TO alex;

Concesion terminada correctamente.
```

Crearemos un servicio para definir la tarea que hará la copia de seguridad.

```
debian@agb:~$ sudo nano /etc/systemd/system/oracle_copia.service
debian@agb:~$ cat /etc/systemd/system/oracle_copia.service 
[Unit]
Description=Oracle Copias

[Service]
Type=oneshot
ExecStart=/bin/bash -c 'expdp alex/alex DIRECTORY=COPIAS DUMPFILE=copia_$(date +"%Y%m%d%H%M%S").dmp LOGFILE=copia_$(date +"%Y%m%d%H%M%S").log ENCRYPTION_PASSWORD=ALEX COMPRESSION=ALL FULL=Y FILESIZE=75M'

[Install]
WantedBy=multi-user.target
```

Crearemos un archivo temporizador que define que se activará el servicio todos los dias a las 2am.

```
debian@agb:~$ cat /etc/systemd/system/oracle_copia.timer 
[Unit]
Description=Oracle Copias Temporizador

[Timer]
OnCalendar=*-*-* 02:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

Reiniciamos el demonio de systemd, y activamos y habilitamos el temporizador que realizará la copia diariamente.

```
debian@agb:~$ sudo systemctl daemon-reload
debian@agb:~$ sudo systemctl enable oracle_copia.timer
Created symlink /etc/systemd/system/timers.target.wants/oracle_copia.timer → /etc/systemd/system/oracle_copia.timer.
debian@agb:~$ sudo systemctl start oracle_copia.timer
```

Al realizarse una copia se guarda en el directorio indicado con la fecha y hora, aquí podemos ver el log de la copia.

![](imagenes/Pasted%20image%2020240311231741.png)
## 2. Restaura la copia de seguridad lógica creada en el punto anterior.

Primero eliminaremos las tablas por ejemplo del esquema SCOTT, como vemos no tiene las tablas ni datos.

```
SQL> connect SCOTT/TIGER
Conectado.
SQL> select * from emp;
select * from emp
              *
ERROR en linea 1:
ORA-00942: la tabla o vista no existe


SQL> select * from dept;
select * from dept
              *
ERROR en linea 1:
ORA-00942: la tabla o vista no existe
```

Realizaremos la importación de la copia de seguridad con el siguiente comando, indicamos el directorio, el dump de la copia a usar y el log que se va a generar en el mismo directorio de la restauración de la copia.

```
debian@agb:~$ impdp alex/alex directory=COPIAS dumpfile=copia_20240311231141.dmp encryption_password=ALEX full=y logfile=restauracion_$(date +"%Y%m%d%H%M%S").log
```

Si nos dirigimos al log de restauración de la copia, y filtramos por SCOTT podemos ver que las tablas y filas de datos del esquema han sido importadas pues las habíamos eliminado, como las tablas del esquema SCOTTY no las habíamos eliminado se omiten.

```
debian@agb:/opt/oracle/copias$ cat restauracion_20240311232157.log  |grep SCOTT
ORA-39151: La tabla "SCOTTY"."DEPT" existe. Todos los metadados dependientes y los datos se omitirán debido table_exists_action de omitir
ORA-39151: La tabla "SCOTTY"."SALGRADE" existe. Todos los metadados dependientes y los datos se omitirán debido table_exists_action de omitir
ORA-39151: La tabla "SCOTTY"."DUMMY" existe. Todos los metadados dependientes y los datos se omitirán debido table_exists_action de omitir
ORA-39151: La tabla "SCOTTY"."EMP" existe. Todos los metadados dependientes y los datos se omitirán debido table_exists_action de omitir
. . "SCOTT"."BONUS"                                 0 KB       0 filas importadas
. . "SCOTT"."DEPT"                              4.992 KB       4 filas importadas
. . "SCOTT"."EMP"                               5.703 KB      14 filas importadas
. . "SCOTT"."SALGRADE"                          4.921 KB       5 filas importadas
```

En la siguiente imagen podemos ver como nos conectamos al usuario SCOTT, y no tenemos ninguna de las tablas disponibles, pero al restaurar la copia, ya volvemos a tener de nuevo las tablas y los datos tal y como estaban.

![](imagenes/Pasted%20image%2020240311232510.png)


## 3. Pon tu base de datos en modo ArchiveLog y realiza con RMAN una copia de seguridad física en caliente.

La base de datos deberá estar en modo ArchiveLog para realizar una copia de seguridad en caliente, para ello paramos la base de datos instantaneamente, y la iniciaremos de nuevo pero en modo montaje, para permitir la administración de ella, pero sin acceso para los usuarios.

```
SQL> shutdown immediate;
Base de datos cerrada.
Base de datos desmontada.
Instancia ORACLE cerrada.

SQL> startup mount;
Instancia ORACLE iniciada.

Total System Global Area 1644164416 bytes
Fixed Size		    9135424 bytes
Variable Size		 1258291200 bytes
Database Buffers	  369098752 bytes
Redo Buffers		    7639040 bytes
Base de datos mon![](imagenes/Evaluación%20Inicial%20-%20Manuel%20Alejandro%20(1)%20(1).pdf)tada.
```

Ahora cambiaremos la configuración de la base de datos habilitando el modo de ArchiveLog que nos permitirá la creación y almacenamineto de archivos de logs.
Podemos comprobar el modo con la consulta:

```
SQL> alter database archivelog;

Base de datos modificada.

SQL> SELECT log_mode FROM V$DATABASE;

LOG_MODE
------------
ARCHIVELOG
```

Al usuario alex le daremos los permisos necesarios para la administración del catalogo de recuperación.

```
SQL> GRANT RECOVERY_CATALOG_OWNER TO alex;

Concesion terminada correctamente.
```

Crearemos un nuevo espacio de tablas donde se guardará la información de este catalogo, modificaremos las propiedades de alex para que su tablespace por defecto sea el que hemos creado y su cuota sea ilimitada.

```
SQL> CREATE TABLESPACE TS_RMAN DATAFILE '/opt/oracle/oradata/ORCLCDB/rman.dbf' SIZE 700M AUTOEXTEND ON NEXT 20M MAXSIZE UNLIMITED;

Tablespace creado.

SQL> ALTER USER alex DEFAULT TABLESPACE TS_RMAN QUOTA UNLIMITED ON TS_RMAN;

Usuario modificado.
```

Desde una terminal bash ejecutamos la herramienta de RMAN, para entrar en su linea de comandos, nos conectaremos al catalogo de recuperación con las credenciales del usuario alex que es al que le hemos dado los permisos para la administración de estas copias en el catalago.
Con él crearemos un catálogo para la recuperacion almacenado en el espacio de tablas TS_RMAN.

```
debian@agb:~$ rman

Recovery Manager: Release 19.0.0.0.0 - Production on Mon Mar 11 23:46:19 2024
Version 19.3.0.0.0

Copyright (c) 1982, 2019, Oracle and/or its affiliates.  All rights reserved.

RMAN> connect CATALOG alex/alex;

connected to recovery catalog database

RMAN> CREATE CATALOG TABLESPACE TS_RMAN;

recovery catalog created
```

Una vez creado, entraremos de nuevo con el usuario alex pero nos conectaremos asimismo a la base de datos, donde registraremos las base de datos para que se guarde y RMAN más adelante pueda ejecutar restauraciones de copias.

```
debian@agb:~$ rman target =/ catalog alex/alex

Recovery Manager: Release 19.0.0.0.0 - Production on Mon Mar 11 23:46:58 2024
Version 19.3.0.0.0

Copyright (c) 1982, 2019, Oracle and/or its affiliates.  All rights reserved.

connected to target database: ORCLCDB (DBID=2922678409)
connected to recovery catalog database

RMAN> REGISTER DATABASE;

database registered in recovery catalog
starting full resync of recovery catalog
full resync complete
```


En la siguiente imagen podemos ver la ejecución del comando para realizar la copia de seguridad totalmente completa de la base de datos, donde se incluirán los archivos de datos, de control,  registros, etc.

![](imagenes/Pasted%20image%2020240311234741.png)

Con el siguiente comando podemos comprobar que la copia se ha realizado correctamente.

![](imagenes/Pasted%20image%2020240311234846.png)

## 4. Borra un fichero de datos de un tablespace e intenta recuperar la instancia de la base de datos a partir de la copia de seguridad creada en el punto anterior.

Primero eliminaremos un archivo de datos del tablespace de USERS por ejemplo.
```
root@agb:/opt/oracle/oradata/ORCLCDB# rm users01.dbf 
```

Al intentar acceder a la base de datos, vemos que nos salta un error pues nos falta un fichero de datos del espacio de tablas.

```
SQL> connect SCOTT/TIGER
Conectado.
SQL> select * from emp;
select * from emp
              *
ERROR en linea 1:
ORA-01116: error al abrir el archivo de base de datos 7
ORA-01110: archivo de datos 7: '/opt/oracle/oradata/ORCLCDB/users01.dbf'
ORA-27041: no se ha podido abrir el archivo
Linux-x86_64 Error: 2: No such file or directory
Additional information: 3
```

De nuevo inicaremos RMAN conectado con el usuario alex a la base de datos y pondremos el el tablespace de USERS en modo offline.

```
debian@agb:~$ rman target =/ catalog alex/alex

Recovery Manager: Release 19.0.0.0.0 - Production on Tue Mar 12 00:03:42 2024
Version 19.3.0.0.0

Copyright (c) 1982, 2019, Oracle and/or its affiliates.  All rights reserved.

connected to target database: ORCLCDB (DBID=2922678409)
connected to recovery catalog database

RMAN> SQL "ALTER TABLESPACE USERS OFFLINE IMMEDIATE";

sql statement: ALTER TABLESPACE USERS OFFLINE IMMEDIATE
```

Entonces realizamos una restauración del tablespace que da problemas debido al archivo eliminado, podemos ver en la salida que indica que está restaurando el datafile eliminado.

```
channel ORA_DISK_1: restoring datafile 00007 to /opt/oracle/oradata/ORCLCDB/users01.dbf
```

```
RMAN> RESTORE TABLESPACE USERS;

Starting restore at 12-MAR-24
allocated channel: ORA_DISK_1
channel ORA_DISK_1: SID=276 device type=DISK

channel ORA_DISK_1: starting datafile backup set restore
channel ORA_DISK_1: specifying datafile(s) to restore from backup set
channel ORA_DISK_1: restoring datafile 00007 to /opt/oracle/oradata/ORCLCDB/users01.dbf
channel ORA_DISK_1: reading from backup piece /opt/oracle/product/19c/dbhome_1/dbs/022lfck1_1_1
channel ORA_DISK_1: piece handle=/opt/oracle/product/19c/dbhome_1/dbs/022lfck1_1_1 tag=TAG20240311T234712
channel ORA_DISK_1: restored backup piece 1
channel ORA_DISK_1: restore complete, elapsed time: 00:00:01
Finished restore at 12-MAR-24
```

Recuperaremos el tablespace de USERS y lo pondremos en modo online para su puesta en marcha despues de la recuperación.

```
RMAN> RECOVER TABLESPACE USERS;

Starting recover at 12-MAR-24
using channel ORA_DISK_1

starting media recovery
media recovery complete, elapsed time: 00:00:00

Finished recover at 12-MAR-24

RMAN> SQL "ALTER TABLESPACE USERS ONLINE";

sql statement: ALTER TABLESPACE USERS ONLINE
```


En la imagen podemos ver que se intentaba realizar consultas sobre las tablas pero daban un error, al recuperar el archivo eliminado y activar de nuevo el tablespace podemos ver que ya funcionaba de manera correcta.

![](imagenes/Pasted%20image%2020240312000905.png)

Aquí de igual manera podemos ver como el archivo antes no estaba, y al realizar la restauración se ha generado de nuevo.

![](imagenes/Pasted%20image%2020240312000929.png)
## 5. Borra un fichero de control e intenta recuperar la base de datos a partir de la copia de seguridad creada en el punto anterior.

Comprobamos desde RMAN que las copias de seguridad existen y en que ruta y directorio se encuentran.

```
RMAN> list backup of controlfile;


List of Backup Sets
===================


BS Key  Type LV Size       Device Type Elapsed Time Completion Time
------- ---- -- ---------- ----------- ------------ ---------------
6       Full    17.95M     DISK        00:00:01     11-MAR-24      
        BP Key: 6   Status: AVAILABLE  Compressed: NO  Tag: TAG20240311T234745
        Piece Name: /opt/oracle/product/19c/dbhome_1/dbs/c-2922678409-20240311-00
  Control File Included: Ckp SCN: 6255970      Ckp time: 11-MAR-24

BS Key  Type LV Size       Device Type Elapsed Time Completion Time
------- ---- -- ---------- ----------- ------------ ---------------
7       Full    17.95M     DISK        00:00:00     12-MAR-24      
        BP Key: 7   Status: AVAILABLE  Compressed: NO  Tag: TAG20240312T001402
        Piece Name: /opt/oracle/product/19c/dbhome_1/dbs/c-2922678409-20240312-00
  Control File Included: Ckp SCN: 6259578      Ckp time: 12-MAR-24
```

Una vez ubicadas podemos eliminar esta vez un fichero de control de la base de datos.

```
root@agb:/opt/oracle/oradata/ORCLCDB# rm control01.ctl
```

Para recueprarlo, apagaremos la base de datos y la iniciaremos en modo nomount para recuperar el archivo de control.

```
SQL> shutdown abort;
Instancia ORACLE cerrada.

SQL> startup nomount;
Instancia ORACLE iniciada.

Total System Global Area 1644164416 bytes
Fixed Size		    9135424 bytes
Variable Size		 1258291200 bytes
Database Buffers	  369098752 bytes
Redo Buffers		    7639040 bytes
```

Entraremos a la linea de comandos de RMAN, y restauraremos el fichero de control indicando la ruta de la copia de seguridad desde la que vamos a recuperar.

Podemos ver que se ha recueprado ela rchivo de control eliminado.

```
debian@agb:~$ rman

Recovery Manager: Release 19.0.0.0.0 - Production on Tue Mar 12 13:08:41 2024
Version 19.3.0.0.0

Copyright (c) 1982, 2019, Oracle and/or its affiliates.  All rights reserved.

RMAN> connect target

connected to target database: ORCLCDB (not mounted)

RMAN> restore controlfile from '/opt/oracle/product/19c/dbhome_1/dbs/c-2922678409-20240312-00';

Starting restore at 12-MAR-24
using target database control file instead of recovery catalog
allocated channel: ORA_DISK_1
channel ORA_DISK_1: SID=256 device type=DISK

channel ORA_DISK_1: restoring control file
channel ORA_DISK_1: restore complete, elapsed time: 00:00:01
output file name=/opt/oracle/oradata/ORCLCDB/control01.ctl
output file name=/opt/oracle/oradata/ORCLCDB/control02.ctl
Finished restore at 12-MAR-24
```

Comprobamos que se ha recuperado el archivo de control.

```
root@agb:/opt/oracle/oradata/ORCLCDB# ls -hil
total 3,2G
291237 -rw-r----- 1 oracle oinstall  18M mar 12 13:09 control01.ctl
262962 -rw-r----- 1 oracle oinstall  18M mar 12 13:09 control02.ctl
262697 drwxr-x--- 2 oracle oinstall 4,0K oct 30 21:54 ORCLPDB1
262698 drwxr-x--- 2 oracle oinstall 4,0K oct 30 21:42 pdbseed
263010 -rw-r----- 1 oracle oinstall 201M mar 12 13:07 redo01.log
263013 -rw-r----- 1 oracle oinstall 201M mar 12 13:07 redo02.log
263014 -rw-r----- 1 oracle oinstall 201M mar 12 13:07 redo03.log
291075 -rw-r----- 1 oracle oinstall 701M mar 12 13:07 rman.dbf
262931 -rw-r----- 1 oracle oinstall 631M mar 12 13:07 sysaux01.dbf
262928 -rw-r----- 1 oracle oinstall 941M mar 12 13:07 system01.dbf
263028 -rw-r----- 1 oracle oinstall  33M mar 11 23:23 temp01.dbf
262932 -rw-r----- 1 oracle oinstall 331M mar 12 13:07 undotbs01.dbf
291124 -rw-r----- 1 oracle oinstall  27M mar 12 13:07 users01.dbf
```

Entonces ya podremos recuperar la base de datos para que esté puesta para su funcionamiento correcto sin fallos, ademas de que se iniciará con nuevos archivos del punto en el que está la base de datos inicada para que los datos cumplan con la integridad.

```
RMAN> recover database;

Starting recover at 12-MAR-24
allocated channel: ORA_DISK_1
channel ORA_DISK_1: SID=23 device type=DISK

starting media recovery

archived log for thread 1 with sequence 44 is already on disk as file /opt/oracle/oradata/ORCLCDB/redo02.log
archived log for thread 1 with sequence 45 is already on disk as file /opt/oracle/oradata/ORCLCDB/redo03.log
archived log file name=/opt/oracle/oradata/ORCLCDB/redo02.log thread=1 sequence=44
archived log file name=/opt/oracle/oradata/ORCLCDB/redo03.log thread=1 sequence=45
media recovery complete, elapsed time: 00:00:00
Finished recover at 12-MAR-24

RMAN> alter database open resetlogs;


Statement processed
```


## 6. Documenta el empleo de las herramientas de copia de seguridad y restauración de Postgres.

Para realizar copias de seguridad y restaurarlas en Postgres usaremos herramientos como **pg_dumpall** con ella podremos volcar datos tosas las bases de datos.

Crearemos un archivo `copia.sql` que guardará todo lo necesario para realizar una resaturación de todas las bases de datos.

```
postgres@agb:~$ pg_dumpall -U postgres -f copia.sql
postgres@agb:~$ ls -hil
total 208K
266518 drwxr-xr-x 3 postgres postgres 4,0K ene 22 22:46 15
289941 -rw-r--r-- 1 postgres postgres  12K feb 25 21:36 audit.sql
291366 -rw-r--r-- 1 postgres postgres 143K mar 12 17:09 copia.sql
415115 drwx------ 3 postgres postgres 4,0K feb 14 19:52 datos
415172 drwxr-xr-x 2 postgres postgres 4,0K mar  4 23:22 exp_psql
268372 -rw-r--r-- 1 postgres postgres  40K mar  4 23:10 exp_scott.sql
```

Para probarla entraremos a una base de datos, veremos que tenemos las tablas y datos correctamente almacenados.

```
debian@agb:~$ psql -U alexcond -d alexcond -h localhost
Contraseña para usuario alexcond: 
psql (15.5 (Debian 15.5-0+deb12u1))
Conexión SSL (protocolo: TLSv1.3, cifrado: TLS_AES_256_GCM_SHA384, compresión: desactivado)
Digite «help» para obtener ayuda.

alexcond=> \d
         Listado de relaciones
 Esquema |  Nombre   | Tipo  |  Dueño   
---------+-----------+-------+----------
 public  | conductor | tabla | alexcond
(1 fila)

alexcond=> select * from conductor;

 codigo |  nombre  | apellido1 | apellido2 |    dni    |     calle     | num_calle |   provincia    |      poblacion       | telefono  |             correo             
--------+----------+-----------+-----------+-----------+---------------+-----------+----------------+----------------------+-----------+--------------------------------
 15974  | Maria    | Gutierrez | Nunez     | 82206485A | LOS ROSALES   | 00008     | Sevilla        | Dos Hermanas         | 635792048 | mariaguati08@hotmail.com
 95320  | Almudena | Garcia    | Camacho   | 25301477D | FELIPE II     | 00015     | Sevilla        | Dos Hermanas         | 785220159 | almugar@hotmail.com
 52014  | Juan     | Perez     | Rodriguez | 63201598D | LA JARA       | 00025     | Cadiz          | Jerez De La Frontera | 789201564 | juanprodri025@gmail.es
 65204  | Alex     | Martin    | Nunez     | 41578964D | QUEVEDO       | 00011     | Sevilla        | Los Palacios         | 658912044 | alexmarnun435@outlook.com
 00158  | Sofia    | Valassi   | Moreno    | 85003495A | COCO          | 00115     | Barcelona      | Sitges               | 621685420 | sofiavalasi-moreno@hotmail.cat
 32058  | Cesar    | Paredes   | Espinar   | 48620159W | SA TAULERA    | 00018     | Islas Baleares | Palma                | 971520384 | cesarpeesp-432@gmail.com
 45068  | Maite    | Antunez   | Gonzalez  | 52300168X | CONSOLACION   | 00148     | Sevilla        | Alcala De Guadaira   | 752156308 | maiteangon8@hotmail.com
 08951  | Jose     | Fernandez | Murillo   | 85210684A | MAYOR         | 00025     | Madrid         | Madrid               | 695201597 | josefermur@hotmail.es
 54321  | Natalia  | Rodriguez | Garcia    | 87654321B | SAN FRANCISCO | 00021     | Sevilla        | Sevilla              | 971562018 | natalyrodricia-234@outlook.org
(9 filas)

```

Pues vamos a eliminar la tabla.

```
alexcond=> drop table conductor;
DROP TABLE
alexcond=> select * from conductor;
ERROR:  no existe la relación «conductor»
LÍNEA 1: select * from conductor;
                       ^
```

O incluso la base de datos.

```
postgres=# drop database alexcond;
DROP DATABASE
```

Para a continuación restaurarla desde la copia que habíamos realizado al inicio, tan simple como indicar el comando `psql` con el usuario postgres y con el parametro `-f` la ruta o el nombre del archivo de la copia.

```
postgres@agb:~$ psql -U postgres -f copia.sql
```

Podemos ver en la imagen que algunos usuarios que ya existen pues no los crea y saltan errores.

![](imagenes/Pasted%20image%2020240312172457.png)



Pero no hay problema porque si que restaura todo lo que se haya perdido o eliminado pues en esta parte de la ejecución del restaurado de la base de datos vemos que ha creado la base de datos de `alexcond` que es la que habíamos eliminado y empieza a crear las tablas e insertar los datos en ella.

![](imagenes/Pasted%20image%2020240312172511.png)

Para que se realicen las copias diariamente crearemos también un servicio que ejecute el comando que realiza la copia y lo guardará en una ruta concreta y se guardará asimismo junto a la fecha y hora.

```
debian@agb:~$ sudo nano /etc/systemd/system/postgres_copia.service
debian@agb:~$ cat /etc/systemd/system/postgres_copia.service 
[Unit]
Description=Postgres Copias

[Service]
Type=oneshot

ExecStart=/bin/bash -c 'pg_dumpall -U postgres -f /home/postgres/copias/copia_$(date +"%Y%m%d%H%M%S").sql'

[Install]
WantedBy=multi-user.target
```

Debemos crear conjuntamente un temporizador para que se ejecute diariamente a las 2 am.

```
debian@agb:~$ sudo nano /etc/systemd/system/postgres_copia.timer
debian@agb:~$ cat /etc/systemd/system/postgres_copia.timer 
[Unit]
Description=Postgres Copias Temporizador

[Timer]
OnCalendar=*-*-* 02:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

Reiniciamos el demonio de systemd, y activamos y habilitamos el temporizador que realizará la copia diariamente.

```
debian@agb:~$ sudo systemctl daemon-reload
debian@agb:~$ sudo systemctl enable postgres_copia.timer
Created symlink /etc/systemd/system/timers.target.wants/postgres_copia.timer → /etc/systemd/system/postgres_copia.timer.
debian@agb:~$ sudo systemctl start postgres_copia.timer
```

## 7. Documenta el empleo de las herramientas de copia de seguridad y restauración de MySQL.

Para las copias y restauraciones en Mysql usaremos la herramienta `mysqldump` indicando con el parametro `--all-databases` para que se realice la copia de toda y todas las bases de datos.

```
debian@agb:~$ sudo mysqldump -u root -p --all-databases > copia.sql
Enter password: 

debian@agb:~$ ls -hil
total 2,5G
  4360 -rw-r--r--  1 debian debian   2,5M mar 12 17:33 copia.sql
```

Entramos a mysql y vemos las bases de datos, procederemos a eliminar algunas para su futura recuperación.

```
debian@agb:~$ sudo mysql
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 37
Server version: 10.11.4-MariaDB-1~deb12u1-log Debian 12

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| camiones           |
| camionesimp        |
| examen             |
| gn                 |
| information_schema |
| mysql              |
| performance_schema |
| phpmyadmin         |
| prac               |
| scott              |
| sys                |
| tablascsv          |
+--------------------+
12 rows in set (0,000 sec)

MariaDB [(none)]> drop database examen;
Query OK, 3 rows affected (3,526 sec)

MariaDB [(none)]> drop database camiones;
Query OK, 16 rows affected (11,924 sec)

MariaDB [(none)]> drop database scott;
Query OK, 3 rows affected (3,285 sec)

MariaDB [(none)]> use scott;
ERROR 1049 (42000): Unknown database 'scott'
MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| camionesimp        |
| gn                 |
| information_schema |
| mysql              |
| performance_schema |
| phpmyadmin         |
| prac               |
| sys                |
| tablascsv          |
+--------------------+
9 rows in set (0,000 sec)
```

Para recueprar, con el comando `mysql` e indicando el fichero que hemos creado de la copia de antes.
Al restaurar comprobaremos entrando de nuevo y veremos que las bases de datos y los datos se han recuperado.

```
debian@agb:~$ sudo mysql -u root -p < copia.sql
Enter password: 
debian@agb:~$ sudo mysql
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 39
Server version: 10.11.4-MariaDB-1~deb12u1-log Debian 12

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| camiones           |
| camionesimp        |
| examen             |
| gn                 |
| information_schema |
| mysql              |
| performance_schema |
| phpmyadmin         |
| prac               |
| scott              |
| sys                |
| tablascsv          |
+--------------------+
12 rows in set (0,000 sec)

MariaDB [(none)]> use scott;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
MariaDB [scott]> show tables;
+-----------------+
| Tables_in_scott |
+-----------------+
| audit_emp       |
| dept            |
| emp             |
+-----------------+
3 rows in set (0,000 sec)

MariaDB [scott]> select * from dept;
+--------+------------+----------+
| DEPTNO | DNAME      | LOC      |
+--------+------------+----------+
|     10 | ACCOUNTING | NEW YORK |
|     20 | RESEARCH   | DALLAS   |
|     30 | SALES      | CHICAGO  |
|     40 | OPERATIONS | BOSTON   |
+--------+------------+----------+
4 rows in set (0,000 sec)
```

Para que se realicen las copias diariamente crearemos también un servicio que ejecute el comando que realiza la copia y lo guardará en una ruta concreta y se guardará asimismo junto a la fecha y hora.

```
debian@agb:~$ sudo nano /etc/systemd/system/mysql_copia.service
debian@agb:~$ cat /etc/systemd/system/mysql_copia.service 
[Unit]
Description=Mysql Copias

[Service]
Type=oneshot

ExecStart=/bin/bash -c 'mysqldump -u root -p --all-databases > /home/debian/copias_mysql/copia_$(date +"%Y%m%d%H%M%S").sql'

[Install]
WantedBy=multi-user.target
```

Debemos crear conjuntamente un temporizador para que se ejecute diariamente a las 2 am.

```
debian@agb:~$ sudo nano /etc/systemd/system/mysql_copia.timer
debian@agb:~$ cat /etc/systemd/system/mysql_copia.timer 
[Unit]
Description=Mysql Copias Temporizador

[Timer]
OnCalendar=*-*-* 02:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

Reiniciamos el demonio de systemd, y activamos y habilitamos el temporizador que realizará la copia diariamente.

```
debian@agb:~$ sudo systemctl daemon-reload
debian@agb:~$ sudo systemctl enable mysql_copia.timer
Created symlink /etc/systemd/system/timers.target.wants/mysql_copia.timer → /etc/systemd/system/mysql_copia.timer.
debian@agb:~$ sudo systemctl start mysql_copia.timer
```

## 8. Documenta el empleo de las herramientas de copia de seguridad y restauración de MongoDB.

```
vagrant@mongo-alex:~$ mongodump -u alex -p alex --db camiones --authenticationDatabase camiones --out copia
2024-03-12T17:39:30.850+0000	writing camiones.Remolque to copia/camiones/Remolque.bson
2024-03-12T17:39:30.851+0000	done dumping camiones.Remolque (12 documents)
2024-03-12T17:39:30.851+0000	writing camiones.Ciudad to copia/camiones/Ciudad.bson
2024-03-12T17:39:30.854+0000	writing camiones.Parque to copia/camiones/Parque.bson
2024-03-12T17:39:30.854+0000	writing camiones.tablas to copia/camiones/tablas.bson
2024-03-12T17:39:30.854+0000	writing camiones.camiones to copia/camiones/camiones.bson
2024-03-12T17:39:30.856+0000	done dumping camiones.Ciudad (11 documents)
2024-03-12T17:39:30.857+0000	done dumping camiones.Parque (8 documents)
2024-03-12T17:39:30.858+0000	done dumping camiones.tablas (4 documents)
2024-03-12T17:39:30.859+0000	done dumping camiones.camiones (0 documents)
2024-03-12T17:39:30.860+0000	writing camiones.Remolque_Normal to copia/camiones/Remolque_Normal.bson
2024-03-12T17:39:30.862+0000	writing camiones.Remolque_Frigorifico to copia/camiones/Remolque_Frigorifico.bson
2024-03-12T17:39:30.863+0000	done dumping camiones.Remolque_Normal (0 documents)
2024-03-12T17:39:30.863+0000	writing camiones.Remolque_Cisterna to copia/camiones/Remolque_Cisterna.bson
2024-03-12T17:39:30.864+0000	done dumping camiones.Remolque_Frigorifico (0 documents)
2024-03-12T17:39:30.865+0000	done dumping camiones.Remolque_Cisterna (0 documents)
```

```
Enterprise test> show dbs;
admin         240.00 KiB
camiones      296.00 KiB
camionesford   56.00 KiB
config         60.00 KiB
local          80.00 KiB
recetas        44.00 KiB
Enterprise test> use camiones;
switched to db camiones
Enterprise camiones> show collections;
camiones
Ciudad
Parque
Remolque
Remolque_Cisterna
Remolque_Frigorifico
Remolque_Normal
tablas
```

```
Enterprise test> use camiones;
switched to db camiones
Enterprise camiones> db.dropDatabase();
{ ok: 1, dropped: 'camiones' }
```

```
vagrant@mongo-alex:~$ mongorestore -u alex --db camiones --authenticationDatabase camiones copia/camiones
Enter password for mongo user:

2024-03-12T17:43:29.296+0000	The --db and --collection flags are deprecated for this use-case; please use --nsInclude instead, i.e. with --nsInclude=${DATABASE}.${COLLECTION}
2024-03-12T17:43:29.297+0000	building a list of collections to restore from copia/camiones dir
2024-03-12T17:43:29.298+0000	reading metadata for camiones.Remolque_Normal from copia/camiones/Remolque_Normal.metadata.json
2024-03-12T17:43:29.298+0000	reading metadata for camiones.camiones from copia/camiones/camiones.metadata.json
2024-03-12T17:43:29.299+0000	reading metadata for camiones.tablas from copia/camiones/tablas.metadata.json
2024-03-12T17:43:29.299+0000	reading metadata for camiones.Ciudad from copia/camiones/Ciudad.metadata.json
2024-03-12T17:43:29.300+0000	reading metadata for camiones.Parque from copia/camiones/Parque.metadata.json
2024-03-12T17:43:29.300+0000	reading metadata for camiones.Remolque from copia/camiones/Remolque.metadata.json
2024-03-12T17:43:29.301+0000	reading metadata for camiones.Remolque_Cisterna from copia/camiones/Remolque_Cisterna.metadata.json
2024-03-12T17:43:29.301+0000	reading metadata for camiones.Remolque_Frigorifico from copia/camiones/Remolque_Frigorifico.metadata.json
2024-03-12T17:43:29.330+0000	restoring camiones.Parque from copia/camiones/Parque.bson
2024-03-12T17:43:29.339+0000	restoring camiones.Ciudad from copia/camiones/Ciudad.bson
2024-03-12T17:43:29.341+0000	finished restoring camiones.Parque (8 documents, 0 failures)
2024-03-12T17:43:29.351+0000	finished restoring camiones.Ciudad (11 documents, 0 failures)
2024-03-12T17:43:29.366+0000	restoring camiones.Remolque from copia/camiones/Remolque.bson
2024-03-12T17:43:29.373+0000	restoring camiones.tablas from copia/camiones/tablas.bson
2024-03-12T17:43:29.380+0000	finished restoring camiones.Remolque (12 documents, 0 failures)
2024-03-12T17:43:29.386+0000	finished restoring camiones.tablas (4 documents, 0 failures)
2024-03-12T17:43:29.389+0000	restoring camiones.Remolque_Frigorifico from copia/camiones/Remolque_Frigorifico.bson
2024-03-12T17:43:29.408+0000	restoring camiones.Remolque_Normal from copia/camiones/Remolque_Normal.bson
2024-03-12T17:43:29.419+0000	finished restoring camiones.Remolque_Frigorifico (0 documents, 0 failures)
2024-03-12T17:43:29.436+0000	restoring camiones.camiones from copia/camiones/camiones.bson
2024-03-12T17:43:29.440+0000	restoring camiones.Remolque_Cisterna from copia/camiones/Remolque_Cisterna.bson
2024-03-12T17:43:29.454+0000	finished restoring camiones.camiones (0 documents, 0 failures)
2024-03-12T17:43:29.454+0000	finished restoring camiones.Remolque_Normal (0 documents, 0 failures)
2024-03-12T17:43:29.465+0000	finished restoring camiones.Remolque_Cisterna (0 documents, 0 failures)
2024-03-12T17:43:29.465+0000	no indexes to restore for collection camiones.Remolque_Frigorifico
2024-03-12T17:43:29.465+0000	no indexes to restore for collection camiones.Remolque_Normal
2024-03-12T17:43:29.466+0000	no indexes to restore for collection camiones.camiones
2024-03-12T17:43:29.466+0000	no indexes to restore for collection camiones.tablas
2024-03-12T17:43:29.466+0000	no indexes to restore for collection camiones.Ciudad
2024-03-12T17:43:29.466+0000	no indexes to restore for collection camiones.Parque
2024-03-12T17:43:29.466+0000	no indexes to restore for collection camiones.Remolque
2024-03-12T17:43:29.467+0000	no indexes to restore for collection camiones.Remolque_Cisterna
2024-03-12T17:43:29.467+0000	35 document(s) restored successfully. 0 document(s) failed to restore.
```


Para que se realicen las copias diariamente crearemos también un servicio que ejecute el comando que realiza la copia y lo guardará en una ruta concreta y se guardará asimismo junto a la fecha y hora.

```
vagrant@mongo-alex:~$ sudo nano /etc/systemd/system/mongo_copia.service
vagrant@mongo-alex:~$ cat /etc/systemd/system/mongo_copia.service 
[Unit]
Description=Mongo Copias

[Service]
Type=oneshot

ExecStart=/bin/bash -c 'mongodump -u alex -p alex --db camiones --authenticationDatabase camiones --out /home/vagrant/copia_mongo/copia$(date +%%Y%%m%%d%%H%%M%%S)'

[Install]
WantedBy=multi-user.target
```

Debemos crear conjuntamente un temporizador para que se ejecute diariamente a las 2 am.

```
vagrant@mongo-alex:~$ sudo nano /etc/systemd/system/mongo_copia.timer
vagrant@mongo-alex:~$ cat /etc/systemd/system/mongo_copia.timer 
[Unit]
Description=Mongo Copias Temporizador

[Timer]
OnCalendar=*-*-* 02:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

Reiniciamos el demonio de systemd, y activamos y habilitamos el temporizador que realizará la copia diariamente.

```
debian@agb:~$ sudo systemctl daemon-reload
vagrant@mongo-alex:~$ sudo systemctl enable mongo_copia.timer
Created symlink /etc/systemd/system/timers.target.wants/mongo_copia.timer → /etc/systemd/system/mongo_copia.timer.
vagrant@mongo-alex:~$ sudo systemctl start mongo_copia.timer
``