# Auditoría
### 1. Activa desde SQL*Plus la auditoría de los intentos de acceso exitosos al sistema. Comprueba su funcionamiento.

Con la siguiente consulta podemos comprobar si la auditoría se encuentra activada, como podemos ver a la salida de la consulta nos dice `NONE` , lo que quiere decir que no está activa.

```sql
SQL> select name, VALUE
from v$parameter 
where name like 'audit_trail';  2    3  

NAME
--------------------------------------------------------------------------------
VALUE
--------------------------------------------------------------------------------
audit_trail
NONE
```


Para ello modificaremos el sistema de oracle con el siguiente comando para activarlo.

```sql
SQL> alter system set audit_trail=db scope=spfile;

Sistema modificado.
```


Deberemos de reiniciar la base de datos para que se aplique y empiece a funcionar la auditoría.

```sql
shutdown immediate;
startup;
```

Para activar la auditoria de intentos de acceso exitosos lo realizaremos con el siguiente comando, pues recopilará información sobre los accesos de sesión en oracle.

```sql
SQL> audit create session by access;

Auditoria terminada correctamente.
```


Si hemos accedido a una sesión se recogerá en una vista de auditoria que podemos consultar con la siguiente consulta.
Como podemos ver nos muestra el usuario del sistema, de la base de datos, la fecha y la acción que en este caso es acceder a la sesión.

```sql
SQL> select os_username, username, timestamp, ACTION_NAME from dba_audit_session;

OS_USERNAME
--------------------
USERNAME
--------------------
TIMESTAM
--------
ACTION_NAME
--------------------
debian
ALEX
19/02/24
LOGON
```

### 2. Realiza un procedimiento en PL/SQL que te muestre los accesos fallidos junto con el motivo de los mismos, transformando el código de error almacenado en un mensaje de texto comprensible. Contempla todos los motivos posibles para que un acceso sea fallido.

En primer lugar deberemos activar con el siguiente comando la auditoría para que se registren los intentos de inicio de sesión fallidos.

```sql
SQL> AUDIT CREATE SESSION WHENEVER NOT SUCCESSFUL;

Auditoria terminada correctamente.
```


Creamos una función que guarda en una variable segun el codigo de error que reciba guardará un mensaje u otro de error.

```sql
create or replace function mensaje_error(p_codigo in number) return varchar2 is v_mensaje_error varchar2(100);
begin
  if p_codigo = 911 then
    v_mensaje_error := 'Caracter invalido.';
  elsif p_codigo = 988 then
    v_mensaje_error := 'La contrasena es incorrecta o nula.';
  elsif p_codigo = 1004 then
    v_mensaje_error := 'Usuario predeterminado no compatible.';
  elsif p_codigo = 1005 then
    v_mensaje_error := 'Contrasena nula.';
  elsif p_codigo = 1017 then
    v_mensaje_error := 'Usuario o contrasena invalida.';
  elsif p_codigo = 1045 then
    v_mensaje_error := 'Inicio de sesion con privilegio CREATE SESSION denegado.';
  elsif p_codigo = 1918 then
    v_mensaje_error := 'Usuario no encontrado.';
  elsif p_codigo = 1920 then
    v_mensaje_error := 'El nombre de usuario entra en conflicto con otro nombre de usuario o rol.';
  elsif p_codigo = 9911 then
    v_mensaje_error := 'Contrasena incorrecta.';
  elsif p_codigo = 28000 then
    v_mensaje_error := 'La cuenta esta bloqueada.';
  elsif p_codigo = 28001 then
    v_mensaje_error := 'La contrasena ha expirado.';
  elsif p_codigo = 28002 then
    v_mensaje_error := 'La contrasena expirara pronto.';
  elsif p_codigo = 28003 then
    v_mensaje_error := 'Error en la verificacion de la contrasena.';
  elsif p_codigo = 28007 then
    v_mensaje_error := 'La contrasena no puede ser reutilizada.';
  elsif p_codigo = 28008 then
    v_mensaje_error := 'Contraseña antigua no valida.';
  elsif p_codigo = 28009 then
    v_mensaje_error := 'La conexion como SYS debe ser como SYSDBA o SYSOPER.';
  elsif p_codigo = 28011 then
    v_mensaje_error := 'La contrasena ha expirado.';
  else
    v_mensaje_error := 'Oops, algo salio mal. Codigo de error: ' || p_codigo;
  end if;
  return v_mensaje_error;
end;
/
```


Y un procedimiento que recorrerá un cursor de los accesos fallidos a la base de datos, y mostrará por cada intento el mensaje de error, el usuario y la fecha...

```sql
create or replace procedure accesos_fallidos
is
    cursor c_accesos is select os_username, username, returncode, timestamp from dba_audit_session
        where action_name='LOGON' and returncode != 0 order by timestamp;
    v_mensaje varchar2(100);
begin
    for acceso in c_accesos loop
        v_mensaje := mensaje_error(acceso.returncode);
        dbms_output.put_line('Usuario del sistema: ' || acceso.os_username);
        dbms_output.put_line('Usuario: ' || acceso.username);
        dbms_output.put_line('Fecha: ' || to_char(acceso.timestamp,'YY/MM/DD DY HH24:MI'));
        dbms_output.put_line('Mensaje de error de acceso: ' || v_mensaje);
        dbms_output.put_line('-------------------------------------------------------------');
    end loop;
end;
/
```


Podemos comprobarlo realizando algunos intentos que nos den error:

```sql
SQL> exec accesos_fallidos;

Procedimiento PL/SQL terminado correctamente.

SQL> set serveroutput on;
SQL> 
SQL> exec accesos_fallidos;
Usuario del sistema: debian
Usuario: ALEX
Fecha: 20/02/24 SAB 12:31
Mensaje de error de acceso: Usuario o contrasena invalida.
-------------------------------------------------------------
Usuario del sistema: debian
Usuario: ???W
Fecha: 20/02/24 SAB 12:35
Mensaje de error de acceso: Usuario o contrasena invalida.
-------------------------------------------------------------
Usuario del sistema: debian
Usuario: AMANDA
Fecha: 20/02/24 SAB 13:10
Mensaje de error de acceso: Inicio de sesion con privilegio CREATE SESSION
denegado.
-------------------------------------------------------------
Usuario del sistema: debian
Usuario: ALEX
Fecha: 20/02/24 SAB 13:11
Mensaje de error de acceso: La cuenta esta bloqueada.
-------------------------------------------------------------

Procedimiento PL/SQL terminado correctamente.
```


### 3. Activa la auditoría de las operaciones DML realizadas por SCOTT. Comprueba su funcionamiento.

Deberemos establecer la auditoría que registre las operaciones de INSERT, UPDATE y DELETE realizadas por el usuario SCOTT a nivel de acceso.

```sql
SQL> audit insert table, update table, delete table by SCOTT by access;

Auditoria terminada correctamente.
```


Realizamos varias operaciones para comprobar que se registra.

```sql
SQL> insert into emp values (1597, 'Alejandra', 'CLERK', 7782, to_date('2024-02-20', 'YYYY-MM-DD'), 1200, null, 10);

1 fila creada.

SQL> update emp set sal = 3200 where empno = 7902;

1 fila actualizada.

SQL> delete from emp where empno = 1597;

1 fila suprimida.
```


Como podemos ver las operaciones ejecutadas se han registrado con éxito.

```sql
SQL> select obj_name, action_name, timestamp 
from  dba_audit_object 
where username='SCOTT';

OBJ_NAME
--------------------------------------------------------------------------------
ACTION_NAME		     TIMESTAM
---------------------------- --------
EMP
INSERT			     20/02/24

EMP
UPDATE			     20/02/24

EMP
DELETE			     20/02/24
```

### 4. Realiza una auditoría de grano fino para almacenar información sobre la inserción de empleados con sueldo superior a 2000 en la tabla emp de scott.

Con este bloque de código anónimo utilizamos el paquete `dbms_fga` para agregar una política de auditoría de grano fino a la tabla `EMP` en el esquema `SCOTT`.
La política, le podemos dar el nombre como `sal2k_alex`, se configura para auditar inserciones (`INSERT`) en la tabla `EMP` donde el valor de la columna `sal` sea mayor a 2000. Esto hará que cada vez que se realice una inserción en la tabla `EMP` y el valor de la columna `sal` sea superior a 2000, se registrará una entrada de auditoría.

```sql
SQL> begin
  dbms_fga.add_policy (
  object_schema => 'SCOTT',
  object_name => 'EMP',
  policy_name => 'sal2k_alex',
  audit_condition => 'sal > 2000',
  statement_types => 'INSERT'
  );
end;
/

Procedimiento PL/SQL terminado correctamente.
```

Entonces realizamos un insert con un salario de mas de 2000.

```sql
insert into emp values (1597, 'Alejandra', 'CLERK', 7782, to_date('2024-02-20', 'YYYY-MM-DD'), 2200, null, 10);

```


Y finalmente comprobamos filtrando por el nombre de la política los registros.

```sql
SQL> select db_user, object_name, object_schema, sql_text, timestamp 
from dba_fga_audit_trail 
where policy_name='SAL2K_ALEX'
order by timestamp;

DB_USER     | OBJECT_NAME | OBJECT_SCHEMA | SQL_TEXT                                               | TIMESTAMP
-------------------------------------------------------------------------------------------
SCOTT       | EMP         | SCOTT         | insert into emp values (1597, 'Alejandra', 'CLERK', 7782, | 20/02/24
            |             |               | to_date('2024-02-20', 'YYYY-MM-DD'), 2200, null, 10)     |
```


### 5. Explica la diferencia entre auditar una operación by access o by session ilustrándolo con ejemplos.

#### by access
Auditar una operación "by access", es el registro que se crea por cada acceso a un objeto e la base de datos da igual que la sesión del usuario haya terminado o no, y las operaciones en objetos como SELECT, INSERT, UPDATE y DELETE.

Con el siguiente comando estableceremos la auditoria al usuario SCOTT por acceso.

```sql
SQL> audit insert table, update table, delete table by SCOTT by access;

Auditoria terminada correctamente.
```


Ejecutamos varias operaciones.

```sql
SQL> delete from emp where empno = 1597;

1 fila suprimida.

SQL> insert into emp values (1597, 'Alejandra', 'CLERK', 7782, to_date('2024-02-20', 'YYYY-MM-DD'), 1200, null, 10);

1 fila creada.

SQL> update emp set sal = 3200 where empno = 7902;

1 fila actualizada.
```

Comprobamos en los registros que se han recopilado correctamente.

```sql
SQL> select username, owner, obj_name, action, action_name, timestamp
from dba_audit_trail
where username = 'SCOTT'
order by extended_timestamp desc
fetch first 3 rows only;

USERNAME | OWNER | OBJ_NAME | ACTION | ACTION_NAME | TIMESTAMP
----------------------------------------------------------------
SCOTT    | SCOTT | EMP      | 6      | UPDATE      | 20/02/24
SCOTT    | SCOTT | EMP      | 2      | INSERT      | 20/02/24
SCOTT    | SCOTT | EMP      | 7      | DELETE      | 20/02/24
```


#### by session
Auditar "by session" creará un registro para cada sesión de usuario que accede a la base de datos, generará registros de todas las operaciones que se ejecuten durante esa sesión.

La diferencia fundamental está en el nivel de detalle de los registros de auditoría: "by access" genera registros para cada operación, mientras que "by session" genera registros para cada sesión de usuario.

```sql
SQL> audit insert table, update table, delete table by SCOTT by session;

Auditoria terminada correctamente.
```

```
SQL> delete from emp where empno = 1597;

1 fila suprimida.

SQL> update emp set sal = 3200 where empno = 7902;

1 fila actualizada.

SQL> insert into emp values (1597, 'Alejandra', 'CLERK', 7782, to_date('2024-02-20', 'YYYY-MM-DD'), 1200, null, 10);

1 fila creada.
```

### 6. Documenta las diferencias entre los valores db y db, extended del parámetro audit_trail de ORACLE. Demuéstralas poniendo un ejemplo de la información sobre una operación concreta recopilada con cada uno de ellos.

En Oracle, el parámetro `audit_trail` determina la ubicación de almacenamiento de los registros de auditoría generados por el sistema de auditoría del mismo. Este parámetro tiene varios valores posibles, como son `db` y `db,extended`.

#### db

Cuando `audit_trail` está configurado como `db`, los registros de auditoría se almacenan en la tabla `aud$` dentro de la base de datos. Es el valor predeterminado para el parámetro y proporciona registros de auditoría básicos que tienen información como la hora de la operación, el tipo de operación (SELECT, INSERT, UPDATE, DELETE), el nombre del objeto usado y el nombre del usuario que ejecutó la operación. Es un nivel de auditoría que ofrece una visión muy general de las operaciones en la base de datos, no incluye detalles adicionales como el SQL completo ejecutado o los valores de los datos antes y después de una operación.

Con el siguiente comando estableceremos donde se guardarán los registros.

```sql
SQL> alter system set audit_trail=db scope=spfile;

Sistema modificado.
```

Realizamos algunas operaciones...

```sql
SQL> insert into emp values (1597, 'Alejandra', 'CLERK', 7782, to_date('2024-02-20', 'YYYY-MM-DD'), 1200, null, 10);

1 fila creada.

SQL> update emp set sal = 3200 where empno = 7902;

1 fila actualizada.

SQL> delete from emp where empno = 1597;

1 fila suprimida.
```


Con la siguiente consulta veremos las 3 operaciones que ha realizado el usuario SCOTT.

```sql
SQL> select username, action_name, sql_text, timestamp
from dba_audit_trail
where username = 'SCOTT'
order by extended_timestamp desc
fetch first 3 rows only;

USERNAME | ACTION_NAME | SQL_TEXT | TIMESTAMP
---------------------------------------------
SCOTT    | SESSION REC |          | 20/02/24
SCOTT    | SESSION REC |          | 20/02/24
SCOTT    | SESSION REC |          | 20/02/24
```


#### db,extended

Cuando `audit_trail` se configura como `db, extended`, los registros de auditoría también se almacenan en la tabla `aud$` pero con registros extendidos de información. Además de los registros básicos, este nivel captura detalles como el SQL completo ejecutado, los valores de los datos antes y después de una operación. Aunque esta opción aumenta la complejidad y el tamaño de los registros, nos va a proporcionar mucha más información para análisis detallados.

Con el siguiente comando estableceremos donde se guardarán los registros.

```sql
SQL> alter system set audit_trail=db,extended scope=spfile;

Sistema modificado.
```


Realizamos algunas operaciones...

```sql
SQL> insert into emp values (1597, 'Alejandra', 'CLERK', 7782, to_date('2024-02-20', 'YYYY-MM-DD'), 1200, null, 10);

1 fila creada.

SQL> delete from emp where empno = 1597;

1 fila suprimida.

SQL> update emp set sal = 3200 where empno = 7902;

1 fila actualizada.
```


Con la siguiente consulta veremos las 3 operaciones que ha realizado el usuario SCOTT podemos ver que también nos muestra el comando de la operación.

```sql
SQL> select username, action_name, sql_text, timestamp
from dba_audit_trail
where username = 'SCOTT'
order by extended_timestamp desc
fetch first 3 rows only;

USERNAME | ACTION_NAME | SQL_TEXT                                                         | TIMESTAMP
--------------------------------------------------------------------------------------------------------------
SCOTT    | SESSION REC | update emp set sal = 3200 where empno = 7902                     | 20/02/24
SCOTT    | SESSION REC | delete from emp where empno = 1597                                | 20/02/24
SCOTT    | SESSION REC | insert into emp values (1597, 'Alejandra', 'CLERK', 7782, to_date | 20/02/24
         |             | ('2024-02-20', 'YYYY-MM-DD'), 1200, null, 10)                     |
```


Finalmente la diferencia principal entre `db` y `db, extended` está en los detalles de la información en los registros de auditoría, pues `db` ofrece información básica sobre las operaciones, y `db, extended` proporciona unos registros más detallados.

### 7. Averigua si en Postgres se pueden realizar los cuatro primeros apartados. Si es así, documenta el proceso adecuadamente.

#### Ejercicio 1

En postgres activamos la auditoria con el siguiente comando.

```sql
postgres=# alter system set log_connections = 'ON';
ALTER SYSTEM
```

Reiniciamos postgres.

```bash
systemctl restart postgresql
```

Y si miramos el archivo de logs podemos ver los accesos con éxito.

```bash
tail /var/log/postgresql/postgresql-15-main.log

2024-02-20 21: 20:44.495 CET [3305] postgres@postgres LOG:  conexión autenticada: identidad=«postgres» método=peer (/etc/postgresql/15/main/pg_hba.conf:90)
2024-02-20 21: 20:44.495 CET [3305] postgres@postgres LOG:  conexión autorizada: usuario=postgres base_de_datos=postgres nombre_de_aplicación=psql
```

#### Ejercicio 2

Igualmente que en el anterior ejercicio en el mismo archivo de log, se pueden ver asimismo los accesos fallidos.

```bash
tail /var/log/postgresql/postgresql-15-main.log

2024-02-20 21:30:39.375 CET [3350] [desconocido]@[desconocido] LOG:  conexión recibida: host=127.0.0.1 port=63260
2024-02-20 21:30:39.385 CET [3350] alex@camiones FATAL:  la autentificación password falló para el usuario «alex»
2024-02-20 21:30:39.385 CET [3350] alex@camiones DETALLE:  La conexión coincidió con la línea 97 de pg_hba.conf: «host    all             all             127.0.0.1/32            scram-sha-256»

```

#### Ejercicio 3

Para auditar operaciones DML, deberemos descargar el archivo audit.sql que es un script SQL que podemos utilizar para crear un disparador de auditoría en una base de datos de postgres.

```bash
postgres@agb:~$ wget https://raw.githubusercontent.com/2ndQuadrant/audit-trigger/master/audit.sql
--2024-02-20 21:36:33--  https://raw.githubusercontent.com/2ndQuadrant/audit-trigger/master/audit.sql
Resolviendo raw.githubusercontent.com (raw.githubusercontent.com)... 185.199.108.133, 185.199.109.133, 185.199.111.133, ...
Conectando con raw.githubusercontent.com (raw.githubusercontent.com)[185.199.108.133]:443... conectado.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 11832 (12K) [text/plain]
Grabando a: «audit.sql»

audit.sql                             100%[=======================================================================>]  11,55K  --.-KB/s    en 0s      

2024-02-20 21:36:33 (73,1 MB/s) - «audit.sql» guardado [11832/11832]

```

Después entramos a la base de datos y con el usuario a auditar, y ejecutamos el sql.

```sql
camiones=# \i audit.sql 
CREATE EXTENSION
CREATE SCHEMA
REVOKE
COMMENT
CREATE TABLE
REVOKE
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
CREATE INDEX
CREATE INDEX
CREATE INDEX
CREATE FUNCTION
COMMENT
CREATE FUNCTION
COMMENT
CREATE FUNCTION
CREATE FUNCTION
COMMENT
CREATE VIEW
COMMENT
```

Con la siguiente consulta indicaremos que tabla vamos a auditar

```sql
camiones=# SELECT audit.audit_table('public.ciudad');
NOTICE:  disparador «audit_trigger_row» para la relación «ciudad» no existe, omitiendo
NOTICE:  disparador «audit_trigger_stm» para la relación «ciudad» no existe, omitiendo
NOTICE:  CREATE TRIGGER audit_trigger_row AFTER INSERT OR UPDATE OR DELETE ON ciudad FOR EACH ROW EXECUTE PROCEDURE audit.if_modified_func('true');
NOTICE:  CREATE TRIGGER audit_trigger_stm AFTER TRUNCATE ON ciudad FOR EACH STATEMENT EXECUTE PROCEDURE audit.if_modified_func('true');
 audit_table 
-------------
 
(1 fila)
```

Realizamos varias operaciones DML.

```sql
camiones=# insert into ciudad values ('12345', 'Sevilla', 'Andalucia', '12345');
INSERT 0 1
camiones=# update ciudad
set nombre = 'Dos Hermanas'
where codigo = '12345';
UPDATE 1
camiones=# delete from ciudad
where codigo = '12345';
DELETE 1
```

Y con la siguiente consulta veremos las operaciones realizas por quien y cuando.

```sql
camiones=# select session_user_name, action, table_name, action_tstamp_clk, client_query 
from audit.logged_actions;
 session_user_name | action | table_name |       action_tstamp_clk       |                             client_query                              
-------------------+--------+------------+-------------------------------+-----------------------------------------------------------------------
 alex              | I      | ciudad     | 2024-02-20 21:48:58.057991+01 | insert into ciudad values ('12345', 'Sevilla', 'Andalucia', '12345');
 alex              | U      | ciudad     | 2024-02-20 21:48:58.061036+01 | update ciudad                                                        +
                   |        |            |                               | set nombre = 'Dos Hermanas'                                          +
                   |        |            |                               | where codigo = '12345';
 alex              | D      | ciudad     | 2024-02- 20 21:48:58.063397+01 | delete from ciudad                                                   +
                   |        |            |                               | where codigo = '12345';
(3 filas)
```


#### Ejercicio 4

Primero crearemos una tabla donde se introducirán los registros que se van a auditar.

```sql
postgres=# create table public.audit_emp (
    id serial primary key,
    db_user varchar(50),
    object_name varchar(50),
    object_schema varchar(50),
    sql_text text,
    timestamp timestamptz default current_timestamp
);
CREATE TABLE
```

Crearemos una función que si se inserta un salario de mas de 2000, se realizará una insercción de registro en la tabla de auditoria de la tabla de empleo.

```sql
postgres=# create or replace function public.audit_sal2k()
returns trigger as $$
begin
    if new.sal > 2000 then
        insert into public.audit_emp values (current_user, tg_table_name, tg_table_schema, tg_op);
    end if;
    return new;
end;
$$ language plpgsql;
CREATE FUNCTION
```

Creamos el trigger que saltará cuando se inserten datos en la tabla emp.

```sql
postgres=# create or replace trigger sal2k_audit_trigger
after insert on public.emp
for each row execute function public.audit_sal2k();
CREATE TRIGGER
```

Entonces realizamos la prueba insertando una fila con un salario mayor de 2000, y con la siguiente consulta podemos ver el registro creado.

```sql
postgres=# insert into public.emp values (1597, 'Alejandra', 'CLERK', 7782, to_date('2024-02-20', 'YYYY-MM-DD'), 2200, null, 10);
INSERT 0 1


postgres=# select *
from public.audit_emp;

 id | db_user  | object_name | object_schema | sql_text |           timestamp           
----+----------+-------------+---------------+----------+-------------------------------
  1 | postgres | emp         | public        | INSERT   | 2024-02-20 22:33:14.488852+01
(1 fila)
```


### 8. Averigua si en MySQL se pueden realizar los apartados 1, 3 y 4. Si es así, documenta el proceso adecuadamente.

#### Ejercicio 1

Para activar de alguna manera la auditoria, desde el fichero de configuración activaremos, descomentando las siguientes lineas que crearán archivos de registro para ver los logs del uso en la base de datos.
Deberemos reiniciar después el servicio de la base de datos.

```bash
nano /etc/mysql/mariadb.conf.d/50-server.cnf

general_log_file       = /var/log/mysql/mysql.log
general_log            = 1
log_error = /var/log/mysql/error.log


systemctl restart mariadb
```

Y mirando en el archivo podemos ver los registros de acceso exitosos y fallidos.

```bash
root@agb:/var/log/mysql# tail -f mysql.log 
		    29 Query	select count(*) into @discard from `mysql`.`table_stats`
		    29 Quit	
		    30 Connect	root@localhost on  using Socket
		    30 Query	select count(*) into @discard from `mysql`.`help_relation`
		    30 Quit	
		    31 Connect	root@localhost on  using Socket
		    31 Query	select count(*) into @discard from `mysql`.`help_category`
		    31 Quit	
2402 20 22:44:52	    32 Connect	alex@localhost on  using Socket
		    32 Query	select @@version_comment limit 1
240220 22:45:07	    32 Quit	
240220 22:45:12	    33 Connect	prac@localhost on  using Socket
		    33 Query	select @@version_comment limit 1


root@agb:/var/log/mysql# tail -f error.log 
2024-02-20 22:46:35 34 [Warning] Access denied for user 'alex'@'localhost' (using password: YES)

```


#### Ejercicio 3

Para auditar operaciones DML deberemos instalar el siguiente plugin.

```sql
MariaDB [(none)]> INSTALL SONAME 'server_audit';
Query OK, 0 rows affected (0,006 sec)
```

En el archivo de configuración escribiremos las siguientes lineas:
- `server_audit_events=CONNECT,QUERY,TABLE`: especifica qué eventos serám auditados. En este caso, se estarán auditando eventos de conexión, consultas y eventos relacionados con las tablas.

- `server_audit_logging=ON`: Esto habilita el registro de auditoría.

- `server_audit_incl_users=scott`: qué usuarios serán auditados.

- `server_audit_file_path=/var/log/mysql/audit.log`: ubicación del archivo de registro de auditoría. 

```bash
nano /etc/mysql/mariadb.conf.d/50-server.cnf

[server]
server_audit_events=CONNECT,QUERY,TABLE
server_audit_logging=ON
server_audit_incl_users=scott
server_audit_file_path=/var/log/mysql/audit.log

systemctl restart mariadb
```

Realizamos algunas consultas DML.

```sql
MariaDB [scott]> insert into emp values (1597, 'Alejandra', 'CLERK', 7782, STR_TO_DATE('20/02/2024', '%d/%m/%Y'), 1200, null, 10);
Query OK, 1 row affected (0,002 sec)

MariaDB [scott]> delete from emp where empno = 1597;
Query OK, 1 row affected (0,002 sec)

MariaDB [scott]> update emp set sal = 3200 where empno = 7902;
Query OK, 1 row affected (0,002 sec)
Rows matched: 1  Changed: 1  Warnings: 0
```

Y mirando en el archivo indicado antes veremos los registros.

```bash
root@agb:/var/log/mysql# tail -f  audit.log 
20240220 23:02:38,agb,scott,localhost,42,82,QUERY,,'select @@version_comment limit 1',0
20240220 23:02:45,agb,scott,localhost,42,83,QUERY,,'SELECT DATABASE()',0
20240220 23:02:45,agb,scott,localhost,42,85,QUERY,scott,'show databases',0
20240220 23:02:45,agb,scott,localhost,42,86,QUERY,scott,'show tables',0
20240220 23:02:52,agb,scott,localhost,42,89,WRITE,scott,emp,
20240220 23:02:52,agb,scott,localhost,42,89,QUERY,scott,'insert into emp values (1597, \'Alejandra\', \'CLERK\', 7782, STR_TO_DATE(\'20/02/2024\', \'%d/%m/%Y\'), 1200, null, 10)',0
20240220 23:02:56,agb,scott,localhost,42,90,WRITE,scott,emp,
20240220 23:02:56,agb,scott,localhost,42,90,QUERY,scott,'delete from emp where empno = 1597',0
20240220 23:02:59,agb,scott,localhost,42,91,WRITE,scott,emp,
20240220 23:02:59,agb,scott,localhost,42,91,QUERY,scott,'update emp set sal = 3200 where empno = 7902',0

```


#### Ejercicio 4

Al igual que con postgres crearemos una tabla para registrar las auditorias.

```sql
MariaDB [scott]> create table audit_emp (
    ->     id int auto_increment primary key,
    ->     db_user varchar(50),
    ->     object_name varchar(50),
    ->     object_schema varchar(50),
    ->     sql_text varchar(50),
    ->     timestamp timestamp default current_timestamp
    -> );
Query OK, 0 rows affected (0,021 sec)
```

Crearemos un trigger que saltará cuando se inserte una fila en la tabla emp con el salario mayor de 2000, e introducirá en la tabla de auditoría el usuarios, la fecha, etc.

```sql
MariaDB [scott]> delimiter //
MariaDB [scott]> create or replace trigger sal2k_audit_insert
    -> after insert on emp
    -> for each row
    -> begin
    ->     if new.sal > 2000 then
    ->         insert into public.audit_emp (db_user, object_name, object_schema, sql_text)
    ->         values (current_user(), 'public.emp', 'public', 'INSERT');
    ->     end if;
    -> end;
    -> //
Query OK, 0 rows affected (0,026 sec)

MariaDB [scott]> delimiter;
```

Realizaremos una insercción con una fila de mas de 2000 de salario, y si consultamos la tabla podemos ver que se ha almacenado el registro.

```sql
MariaDB [scott]> insert into emp values (1597, 'Alejandra', 'CLERK', 7782, STR_TO_DATE('20/02/2024', '%d/%m/%Y'), 2200, null, 10);
Query OK, 1 row affected (0,002 sec)

MariaDB [scott]> select *
    -> from audit_emp;
    
+----+---------+-------------+---------------+----------+---------------------+
| id | db_user | object_name | object_schema | sql_text | timestamp           |
+----+---------+-------------+---------------+----------+---------------------+
|  1 | scott@% | emp         | SCOTT         | INSERT   | 2024-02-20 23:28:02 |
+----+---------+-------------+---------------+----------+---------------------+
1 row in set (0,000 sec)
```



### 9. Averigua las posibilidades que ofrece MongoDB para auditar los cambios que va sufriendo un documento. Demuestra su funcionamiento.

Para auditar el seguimiento de cambios en los documentos de MongoDB, podemos usar diferentes opciones de auditoría que ofrece la base de datos, sobretodo si tenemos la edición Enterprise. En la documentación oficial de MongoDB nos muestra cuatro métodos mediante los cuales podemos obtener eventos de auditoría.

En primer lugar, tenemos la opción de redirigir los registros de auditoría hacia la consola, con el siguiente comando:

```bash
mongod --dbpath data/db --auditDestination console
```

Otra alternativa es dirigir los registros hacia el sistema de syslog:

```bash
mongod --dbpath data/db --auditDestination syslog
```

Además, es posible configurar la auditoría para que los registros se almacenen en archivos, pudiendo elegir entre formatos JSON o BSON:

```bash
mongod --dbpath data/db --auditDestination file --auditFormat JSON --auditPath data/db/auditLog.json
```

```bash
mongod --dbpath data/db --auditDestination file --auditFormat BSON --auditPath data/db/auditLog.bson
```

Asimismo podemos también ajustar la auditoría a través del archivo de configuración principal de MongoDB (generalmente ubicado en `/etc/mongod.conf`). Por ejemplo:

```yaml
auditLog:
   destination: console
```

```yaml
auditLog:
   destination: syslog
```

```yaml
auditLog:
   destination: file
   format: JSON
   path: data/db/auditLog.json
```

```yaml
auditLog:
   destination: file
   format: BSON
   path: data/db/auditLog.bson
```

 Yo voy a realizar las pruebas configurando la auditoría en formato JSON, habilitándola mediante la modificación del archivo de configuración, he agregado que de más detalles a la hora de recoger los datos auditados:

```bash
nano /etc/mongod.conf

auditLog:
   destination: file
   format: JSON
   path: /var/log/mongodb/auditLog.json
   component:
    write:
    verbosity: 2


systemctl restart mongod
```

Antes de empezar, debemos instalar `jq`, una herramienta para procesar JSON desde la línea de comandos:

```bash
apt install -y jq
```

Vamos a crear algunos registros...

```yaml
Enterprise camiones> db.createCollection("Ciudad", {
...     validator: {
...       $jsonSchema: {
...         bsonType: "object",
...         required: ["codigo_postal", "codigo"],
...         properties: {
...           codigo: {
...             bsonType: "string",
...             maxLength: 5
...           },
...           nombre: {
...             bsonType: "string",
...             maxLength: 20
...           },
...           comunidadautonoma: {
...             bsonType: "string",
...             maxLength: 30
...           },
...           codigo_postal: {
...             bsonType: "string",
...             maxLength: 5
...           }
...         }
...       }
...     }
...   })
{ ok: 1 }

Enterprise camiones> db.Ciudad.insertMany([
...     { codigo: "12345", nombre: "Madrid", comunidadautonoma: "Madrid", codigo_postal: "28001" },
...     { codigo: "23456", nombre: "Barcelona", comunidadautonoma: "Cataluna", codigo_postal: "08001" },
...     { codigo: "34567", nombre: "Valencia", comunidadautonoma: "Comunidad Valenciana", codigo_postal: "46001" },
...     { codigo: "45678", nombre: "Sevilla", comunidadautonoma: "Andalucia", codigo_postal: "41004" },
...     { codigo: "56789", nombre: "Zaragoza", comunidadautonoma: "Aragon", codigo_postal: "50003" },
...     { codigo: "67890", nombre: "Malaga", comunidadautonoma: "Andalucia", codigo_postal: "29001" },
...     { codigo: "78901", nombre: "Bilbao", comunidadautonoma: "Pais Vasco", codigo_postal: "48001" },
...     { codigo: "89012", nombre: "Murcia", comunidadautonoma: "Region De Murcia", codigo_postal: "30002" }
...   ])
{
  acknowledged: true,
  insertedIds: {
    '0': ObjectId('65dcb36383db3e2fc378cd9c'),
    '1': ObjectId('65dcb36383db3e2fc378cd9d'),
    '2': ObjectId('65dcb36383db3e2fc378cd9e'),
    '3': ObjectId('65dcb36383db3e2fc378cd9f'),
    '4': ObjectId('65dcb36383db3e2fc378cda0'),
    '5': ObjectId('65dcb36383db3e2fc378cda1'),
    '6': ObjectId('65dcb36383db3e2fc378cda2'),
    '7': ObjectId('65dcb36383db3e2fc378cda3')
  }
}

Enterprise camiones> db.Ciudad.updateOne(
...    { nombre: "Barcelona" },
...    { $set: { nombre: "Barna" } }
... )
{
  acknowledged: true,
  insertedId: null,
  matchedCount: 1,
  modifiedCount: 1,
  upsertedCount: 0
}

Enterprise camiones> db.Ciudad.deleteOne({ codigo_postal: "28001" })
{ acknowledged: true, deletedCount: 1 }

Enterprise camiones> db.createRole({
...    role: "consulta_ciudades",
...    privileges: [
...      {
...        resource: { db: "camiones", collection: "Ciudad" },
...        actions: [ "find" ]
...      }
...    ],
...    roles: []
... })
{ ok: 1 }

Enterprise camiones> db.createUser(
...   {
...     user: "pepe",
...     pwd: "pepe",
...     roles: ["consulta_ciudades"]
...   }
... )
{ ok: 1 }
```

Registros en auditoria:

```yaml
tail -f /var/log/mongodb/auditLog.json | jq


#Creacion de la colección
{
  "atype": "createCollection",
  "ts": {
    "$date": "2024-02-26T15:59:17.595+00:00"
  },
  "uuid": {
    "$binary": "Mx8e7ywAR7iNEYjSF7ZzKA==",
    "$type": "04"
  },
  "local": {
    "ip": "127.0.0.1",
    "port": 27017
  },
  "remote": {
    "ip": "127.0.0.1",
    "port": 35274
  },
  "users": [
    {
      "user": "alex",
      "db": "camiones"
    }
  ],
  "roles": [
    {
      "role": "dbAdmin",
      "db": "camiones"
    },
    {
      "role": "readWrite",
      "db": "camiones"
    }
  ],
  "param": {
    "ns": "camiones.Ciudad"
  },
  "result": 0
}



#Creacion del rol
{
  "atype": "createRole",
  "ts": {
    "$date": "2024-02-26T15:53:34.344+00:00"
  },
  "uuid": {
    "$binary": "Mx8e7ywAR7iNEYjSF7ZzKA==",
    "$type": "04"
  },
  "local": {
    "ip": "127.0.0.1",
    "port": 27017
  },
  "remote": {
    "ip": "127.0.0.1",
    "port": 35274
  },
  "users": [
    {
      "user": "alex",
      "db": "camiones"
    }
  ],
  "roles": [
    {
      "role": "dbAdmin",
      "db": "camiones"
    },
    {
      "role": "readWrite",
      "db": "camiones"
    }
  ],
  "param": {
    "role": "consulta_ciudades",
    "db": "camiones",
    "roles": [],
    "privileges": [
      {
        "resource": {
          "db": "camiones",
          "collection": "Ciudad"
        },
        "actions": [
          "find"
        ]
      }
    ]
  },
  "result": 0
}


# Creacion de usuario con rol
{
  "atype": "createUser",
  "ts": {
    "$date": "2024-02-26T15:54:00.743+00:00"
  },
  "uuid": {
    "$binary": "Mx8e7ywAR7iNEYjSF7ZzKA==",
    "$type": "04"
  },
  "local": {
    "ip": "127.0.0.1",
    "port": 27017
  },
  "remote": {
    "ip": "127.0.0.1",
    "port": 35274
  },
  "users": [
    {
      "user": "alex",
      "db": "camiones"
    }
  ],
  "roles": [
    {
      "role": "dbAdmin",
      "db": "camiones"
    },
    {
      "role": "readWrite",
      "db": "camiones"
    }
  ],
  "param": {
    "user": "pepe",
    "db": "camiones",
    "roles": [
      {
        "role": "consulta_ciudades",
        "db": "camiones"
      }
    ]
  },
  "result": 0
}
```


### 10. Averigua si en MongoDB se pueden auditar los accesos a una colección concreta. Demuestra su funcionamiento.

Para aduitar los accesos a una coleccion concreta deberemos de ampliar la configuracion de auditoria en el archivo de configuración que usamos antes:

```bash
nano /etc/mongod.conf

auditLog:
   destination: file
   format: JSON
   path: /var/log/mongodb/auditLog_ciudad.json
   filter: '{ atype: "authCheck", "param.ns": "camiones.Ciudad", "param.command": { $in: ["insert", "update", "delete", "find"] } }'

setParameter: { auditAuthorizationSuccess: true }

```

Luego, reiniciamos el servicio.

```bash
systemctl restart mongod
```

Y vamos a realizar algunas operaciones para que se registren:

```yaml
Enterprise camiones> db.Ciudad.insertOne({ codigo: "56789", nombre: "Valencia", comunidadautonoma: "Comunitat Valenciana", codigo_postal: "46002" })
{
  acknowledged: true,
  insertedId: ObjectId('65dcbfe40b239cd8da37198d')
}


Enterprise camiones> db.Ciudad.updateOne(
...    { codigo: "45678" },
...    { $set: { comunidadautonoma: "Andalucía" } }
... )
{
  acknowledged: true,
  insertedId: null,
  matchedCount: 1,
  modifiedCount: 1,
  upsertedCount: 0
}



Enterprise camiones> db.Ciudad.deleteOne({ nombre: "Zaragoza" })
{ acknowledged: true, deletedCount: 1 }


Enterprise camiones> db.Ciudad.find()
[
  {
    _id: ObjectId('65dcb58283db3e2fc378cda4'),
    codigo: '12345',
    nombre: 'Madrid Capital',
    comunidadautonoma: 'Madrid',
    codigo_postal: '28001'
  },
  {
    _id: ObjectId('65dcb58283db3e2fc378cda5'),
    codigo: '23456',
    nombre: 'Barcelona',
    comunidadautonoma: 'Cataluna',
    codigo_postal: '08001'
  },
  {
    _id: ObjectId('65dcb58283db3e2fc378cda7'),
    codigo: '45678',
    nombre: 'Sevilla',
    comunidadautonoma: 'Andalucía',
    codigo_postal: '41004'
  },
  {
    _id: ObjectId('65dcb58283db3e2fc378cda9'),
    codigo: '67890',
    nombre: 'Malaga',
    comunidadautonoma: 'Andalucia',
    codigo_postal: '29001'
  },
  {
    _id: ObjectId('65dcb58283db3e2fc378cdaa'),
    codigo: '78901',
    nombre: 'Bilbao',
    comunidadautonoma: 'Pais Vasco',
    codigo_postal: '48001'
  },
  {
    _id: ObjectId('65dcb58283db3e2fc378cdab'),
    codigo: '89012',
    nombre: 'Murcia',
    comunidadautonoma: 'Region De Murcia',
    codigo_postal: '30002'
  },
  {
    _id: ObjectId('65dcb8811516110d66b22694'),
    codigo: '123',
    nombre: 'Barcelona',
    comunidadautonoma: 'Catalunya',
    codigo_postal: '08002'
  },
  {
    _id: ObjectId('65dcbc8e0e18f7200486e4fc'),
    codigo: '123',
    nombre: 'Barcelona',
    comunidadautonoma: 'Catalunya',
    codigo_postal: '08002'
  },
  {
    _id: ObjectId('65dcbced0e18f7200486e4fd'),
    codigo: '56789',
    nombre: 'Valencia',
    comunidadautonoma: 'Comunitat Valenciana',
    codigo_postal: '46002'
  },
  {
    _id: ObjectId('65dcbf4e0e18f7200486e4fe'),
    codigo: '56789',
    nombre: 'Valencia',
    comunidadautonoma: 'Comunitat Valenciana',
    codigo_postal: '46002'
  },
  {
    _id: ObjectId('65dcbfe40b239cd8da37198d'),
    codigo: '56789',
    nombre: 'Valencia',
    comunidadautonoma: 'Comunitat Valenciana',
    codigo_postal: '46002'
  }
]
```


Vamos a mirar que en el archivo que se habrá creado para los registros de auditoria de la colección Ciudad.

```yaml

#Insert
{
  "atype": "authCheck",
  "ts": {
    "$date": "2024-02-26T16:44:20.791+00:00"
  },
  "uuid": {
    "$binary": "faKSne8RQxGeKwaW6GCPWg==",
    "$type": "04"
  },
  "local": {
    "ip": "127.0.0.1",
    "port": 27017
  },
  "remote": {
    "ip": "127.0.0.1",
    "port": 50708
  },
  "users": [
    {
      "user": "alex",
      "db": "camiones"
    }
  ],
  "roles": [
    {
      "role": "dbAdmin",
      "db": "camiones"
    },
    {
      "role": "readWrite",
      "db": "camiones"
    }
  ],
  "param": {
    "command": "insert",
    "ns": "camiones.Ciudad",
    "args": {
      "insert": "Ciudad",
      "documents": [
        {
          "codigo": "56789",
          "nombre": "Valencia",
          "comunidadautonoma": "Comunitat Valenciana",
          "codigo_postal": "46002",
          "_id": {
            "$oid": "65dcbfe40b239cd8da37198d"
          }
        }
      ],
      "ordered": true,
      "lsid": {
        "id": {
          "$binary": "KAs202TYTb+seGy5bDCX9Q==",
          "$type": "04"
        }
      },
      "$db": "camiones"
    }
  },
  "result": 0
}


#Update
{
  "atype": "authCheck",
  "ts": {
    "$date": "2024-02-26T16:44:43.712+00:00"
  },
  "uuid": {
    "$binary": "faKSne8RQxGeKwaW6GCPWg==",
    "$type": "04"
  },
  "local": {
    "ip": "127.0.0.1",
    "port": 27017
  },
  "remote": {
    "ip": "127.0.0.1",
    "port": 50708
  },
  "users": [
    {
      "user": "alex",
      "db": "camiones"
    }
  ],
  "roles": [
    {
      "role": "dbAdmin",
      "db": "camiones"
    },
    {
      "role": "readWrite",
      "db": "camiones"
    }
  ],
  "param": {
    "command": "update",
    "ns": "camiones.Ciudad",
    "args": {
      "update": "Ciudad",
      "updates": [
        {
          "q": {
            "codigo": "45678"
          },
          "u": {
            "$set": {
              "comunidadautonoma": "Andalucía"
            }
          }
        }
      ],
      "ordered": true,
      "lsid": {
        "id": {
          "$binary": "KAs202TYTb+seGy5bDCX9Q==",
          "$type": "04"
        }
      },
      "$db": "camiones"
    }
  },
  "result": 0
}


# Delete
{
  "atype": "authCheck",
  "ts": {
    "$date": "2024-02-26T16:44:46.196+00:00"
  },
  "uuid": {
    "$binary": "faKSne8RQxGeKwaW6GCPWg==",
    "$type": "04"
  },
  "local": {
    "ip": "127.0.0.1",
    "port": 27017
  },
  "remote": {
    "ip": "127.0.0.1",
    "port": 50708
  },
  "users": [
    {
      "user": "alex",
      "db": "camiones"
    }
  ],
  "roles": [
    {
      "role": "dbAdmin",
      "db": "camiones"
    },
    {
      "role": "readWrite",
      "db": "camiones"
    }
  ],
  "param": {
    "command": "delete",
    "ns": "camiones.Ciudad",
    "args": {
      "delete": "Ciudad",
      "deletes": [
        {
          "q": {
            "nombre": "Zaragoza"
          },
          "limit": 1
        }
      ],
      "ordered": true,
      "lsid": {
        "id": {
          "$binary": "KAs202TYTb+seGy5bDCX9Q==",
          "$type": "04"
        }
      },
      "$db": "camiones"
    }
  },
  "result": 0
}

# Find
{
  "atype": "authCheck",
  "ts": {
    "$date": "2024-02-26T16:45:56.896+00:00"
  },
  "uuid": {
    "$binary": "faKSne8RQxGeKwaW6GCPWg==",
    "$type": "04"
  },
  "local": {
    "ip": "127.0.0.1",
    "port": 27017
  },
  "remote": {
    "ip": "127.0.0.1",
    "port": 50708
  },
  "users": [
    {
      "user": "alex",
      "db": "camiones"
    }
  ],
  "roles": [
    {
      "role": "dbAdmin",
      "db": "camiones"
    },
    {
      "role": "readWrite",
      "db": "camiones"
    }
  ],
  "param": {
    "command": "find",
    "ns": "camiones.Ciudad",
    "args": {
      "find": "Ciudad",
      "filter": {},
      "lsid": {
        "id": {
          "$binary": "KAs202TYTb+seGy5bDCX9Q==",
          "$type": "04"
        }
      },
      "$db": "camiones"
    }
  },
  "result": 0
}
```

Esto demuestra que la auditoría se realiza solo sobre la colección "Ciudad".