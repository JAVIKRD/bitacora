# Enumeración de Microsoft SQL Server (MSSQL)

Consultas y comandos SQL esenciales para auditorías y pentesting en entornos con **Microsoft SQL Server**.

---

## 1. Versión y Datos del Servidor

Identificar la versión exacta y nivel de parches del motor SQL para buscar vectores conocidos o vulnerabilidades.

```sql
-- Versión detallada de SQL Server
SELECT @@version;

-- Propiedades del producto y edición
SELECT SERVERPROPERTY('ProductVersion');
SELECT SERVERPROPERTY('ProductLevel');
SELECT SERVERPROPERTY('Edition');

-- Nombre del equipo y de la instancia
SELECT @@SERVERNAME;
SELECT SERVERPROPERTY('MachineName');
```

---

## 2. Enumeración de Bases de Datos

```sql
-- Listar todas las bases de datos disponibles
SELECT name FROM sys.databases;
SELECT name FROM master.dbo.sysdatabases;

-- Base de datos actualmente en uso
SELECT DB_NAME();

-- Información y fechas de creación
SELECT name, database_id, create_date FROM sys.databases;

-- Espacio y tamaño de las bases de datos
EXEC sp_helpdb;
```

---

## 3. Enumeración de Usuarios y Permisos

```sql
-- Listar principales del servidor y usuarios
SELECT name FROM master.sys.server_principals;
SELECT name FROM sys.sysusers;

-- Usuario actual con el que se ejecuta la sesión
SELECT USER_NAME();
SELECT SYSTEM_USER;
SELECT CURRENT_USER;

-- Permisos efectivos del usuario actual en el servidor
SELECT * FROM fn_my_permissions(NULL, 'SERVER');

-- Listar usuarios con el rol sysadmin (máximo privilegio)
SELECT name FROM master.sys.server_principals 
WHERE IS_SRVROLEMEMBER('sysadmin', name) = 1;

-- Comprobar si el usuario actual es sysadmin (1 = Sí, 0 = No)
SELECT IS_SRVROLEMEMBER('sysadmin');
```

---

## 4. Enumeración de Tablas y Columnas

```sql
-- Listar tablas en la base de datos actual
SELECT table_name FROM information_schema.tables;

-- Listar columnas de una tabla específica
SELECT column_name, data_type 
FROM information_schema.columns 
WHERE table_name = 'users';

-- Buscar columnas con palabras clave sensibles (password, token, hash, secret)
SELECT table_name, column_name 
FROM information_schema.columns 
WHERE column_name LIKE '%password%' OR column_name LIKE '%pass%' OR column_name LIKE '%hash%';

-- Contar número de filas por tabla
SELECT t.name, p.rows 
FROM sys.tables t
INNER JOIN sys.partitions p ON t.object_id = p.object_id
WHERE p.index_id < 2;

-- Información detallada de columnas con procedimiento almacenado
EXEC sp_columns 'nombre_tabla';
```

---

## 5. Roles y Permisos en Bases de Datos

```sql
-- Listar todos los roles del servidor
SELECT name FROM master.sys.server_principals WHERE type = 'R';

-- Ver permisos asignados a objetos
EXEC sp_helprotect;

-- Miembros de roles en la base de datos
EXEC sp_helprolemember;
```

---

## 6. Servidores Vinculados (Linked Servers)

La enumeración de servidores vinculados permite realizar movimientos laterales e interactuar con otras bases de datos en la red corporativa.

```sql
-- Listar servidores vinculados configurados
EXEC sp_linkedservers;
SELECT * FROM sys.servers;

-- Probar conexión y extraer versión del servidor remoto
SELECT * FROM OPENQUERY([LinkedServerName], 'SELECT @@version');

-- Ejecutar consulta directa en el servidor vinculado
EXEC ('SELECT @@version') AT [LinkedServerName];
```
