# Auditoría

**Índice:**
- [**Ejercicio 1**](#1-activa-desde-sqlplus-la-auditoría-de-los-intentos-de-acceso-exitosos-al-sistema-comprueba-su-funcionamiento)

- [**Ejercicio 2**](#2-realiza-un-procedimiento-en-plsql-que-te-muestre-los-accesos-fallidos-junto-con-el-motivo-de-los-mismos-transformando-el-código-de-error-almacenado-en-un-mensaje-de-texto-comprensible-contempla-todos-los-motivos-posibles-para-que-un-acceso-sea-fallido)

- [**Ejercicio 3**](#3-activa-la-auditoría-de-las-operaciones-dml-realizadas-por-scott-comprueba-su-funcionamiento)

- [**Ejercicio 4**](#4-realiza-una-auditoría-de-grano-fino-para-almacenar-información-sobre-la-inserción-de-empleados-con-sueldo-superior-a-2000-en-la-tabla-emp-de-scott)

- [**Ejercicio 5**](#5-explica-la-diferencia-entre-auditar-una-operación-by-access-o-by-session-ilustrándolo-con-ejemplos)

- [**Ejercicio 6**](#6-documenta-las-diferencias-entre-los-valores-db-y-db-extended-del-parámetro-audit_trail-de-oracle-demuéstralas-poniendo-un-ejemplo-de-la-información-sobre-una-operación-concreta-recopilada-con-cada-uno-de-ellos)

- [**Ejercicio 7**](#7-averigua-si-en-postgres-se-pueden-realizar-los-cuatro-primeros-apartados-si-es-así-documenta-el-proceso-adecuadamente)

- [**Ejercicio 8**](#8-averigua-si-en-mysql-se-pueden-realizar-los-apartados-1-3-y-4-si-es-así-documenta-el-proceso-adecuadamente)

- [**Ejercicio 9**](#9-averigua-las-posibilidades-que-ofrece-mongodb-para-auditar-los-cambios-que-va-sufriendo-un-documento-demuestra-su-funcionamiento)

- [**Ejercicio 10**](#10-averigua-si-en-mongodb-se-pueden-auditar-los-accesos-a-una-colección-concreta-demuestra-su-funcio)

---

### 1. Activa desde SQL*Plus la auditoría de los intentos de acceso exitosos al sistema. Comprueba su funcionamiento.

```sql
SQL> select name, VALUE
  2  from v$parameter 
  3  where name like 'audit_trail';

NAME
--------------------------------------------------------------------------------
VALUE
--------------------------------------------------------------------------------
audit_trail
NONE
```

```sql
SQL> alter system set audit_trail=db scope=spfile;

System altered.
```

Reiniciar base de datos
```sql
shutdown immediate;
startup;
```

### 2. Realiza un procedimiento en PL/SQL que te muestre los accesos fallidos junto con el motivo de los mismos, transformando el código de error almacenado en un mensaje de texto comprensible. Contempla todos los motivos posibles para que un acceso sea fallido.



### 3. Activa la auditoría de las operaciones DML realizadas por SCOTT. Comprueba su funcionamiento.



### 4. Realiza una auditoría de grano fino para almacenar información sobre la inserción de empleados con sueldo superior a 2000 en la tabla emp de scott.



### 5. Explica la diferencia entre auditar una operación by access o by session ilustrándolo con ejemplos.



### 6. Documenta las diferencias entre los valores db y db, extended del parámetro audit_trail de ORACLE. Demuéstralas poniendo un ejemplo de la información sobre una operación concreta recopilada con cada uno de ellos.



### 7. Averigua si en Postgres se pueden realizar los cuatro primeros apartados. Si es así, documenta el proceso adecuadamente.


### 8. Averigua si en MySQL se pueden realizar los apartados 1, 3 y 4. Si es así, documenta el proceso adecuadamente.



### 9. Averigua las posibilidades que ofrece MongoDB para auditar los cambios que va sufriendo un documento. Demuestra su funcionamiento.



### 10. Averigua si en MongoDB se pueden auditar los accesos a una colección concreta. Demuestra su funcionamiento.