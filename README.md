# Servicios telematicos proyecto final 2025-02
### Lista de proyectos 2 - Proyecto #5: Balanceo de carga de base de datos con MYSQL y MYSQL ROUTER

## Configuraciones en master y esclavo 1 y 2

> ⚠️ **Nota general**: Asegurese de adaptar valores (IP, `server-id`, archivo/posición del binlog, usuarios) según su entorno real.

```
sudo apt install -y mysql-server
sudo systemctl enable mysql
sudo systemctl start mysql
sudo systemctl status mysql
sudo apt install -y net-tools
```

> 💡 **Sugerencia**: Verifica versión de MySQL (`mysql --version`). En MySQL ≥ 8.0.22

## EN MASTER

```
sudo mysql -u root -p
root
CREATE USER 'repl'@'%' IDENTIFIED WITH mysql_native_password BY 'replpass';
GRANT REPLICATION SLAVE ON . TO 'rel'@'%'; sino
GRANT REPLICATION SLAVE, REPLICATION CLIENT ON . TO 'repl'@'%';
FLUSH PRIVILEGES;
FLUSH TABLES WITH READ LOCK;
SHOW MASTER STATUS;
```

> 🧭 **Archivo/posición del binlog**: Anota **`File`** y **`Position`** que devuelve `SHOW MASTER STATUS;`. Los necesitarás en `CHANGE MASTER TO` en los Slaves.

## EN SLAVE 1 y 2
EN /etc/mysql/mysql.conf.d/mysqld.cnf EDITAR
> ⚠️ **ID único por instancia**: Aquí se muestra `server-id = 2` como ejemplo. Sin embargo, **Cada servidor debe tener un `server-id` único**. Por ejemplo: **master=1**, **slave1=2**, **slave2=3**,  **router=❗❗**
> 
> 💡 **Ojo**: La línea `sudo systemctl restart mysql` es un **comando de sistema**, no una directiva de configuración. Por lo tanto, ejecútelo en terminal luego de editar el archivo.

```
server-id              = 2
relay_log = /var/log/mysql/mysql-relay-bin.log
sudo systemctl restart mysql
```

## EN MASTER
EN EDITAR  cd /etc/mysql/mysql.conf.d/ 

> ⚠️ **Binlog y filtros**: `binlog_do_db = testdb` registra solo ese esquema. Si deseas replicar más bases, ajuste (o comente) según su necesidad.
>
> 🌐 **Red**: `bind-address = 0.0.0.0` expone MySQL en todas las interfaces. Asegura firewall y reglas de red adecuadas.

```
sudo vim mysqld.cnf
editar
```
> Colocar
```
 server-id              = 1
 log_bin                        = /var/log/mysql/mysql-bin.log
# binlog_expire_logs_seconds    = 2592000
max_binlog_size   = 100M
binlog_do_db           = testdb
bind-address = 0.0.0.0
binlog_ignore_db       = mysql
```

## EN SLAVE1 y 2

> 🧩 **Resincronización**: `RESET SLAVE ALL;` borra la configuración previa del Slave.
>
> 🔁 **Terminología**: En versiones nuevas, `STOP SLAVE/START SLAVE` puede ser `STOP REPLICA/START REPLICA`.

```
sudo mysql -u root -p
STOP SLAVE;
RESET SLAVE ALL;
```
> 🧠 **Rellena con tus valores**: Con lo que salga en `SHOW MASTER STATUS;`. Cambiar `MASTER_HOST`, `MASTER_LOG_FILE` y `MASTER_LOG_POS` con lo obtenido en el master. Solo realiza el reemplazo al ejecutarlos.
```
CHANGE MASTER TO  MASTER_HOST='192.168.50.10', MASTER_USER='repl', MASTER_PASSWORD='replpass', MASTER_LOG_FILE='mysql-bin.000001', MASTER_LOG_POS=1897;

START SLAVE;
SHOW SLAVE STATUS\G
```
(no debe salir error)

## probar que funciona

### EN MASTER 

> ✅ **Prueba DML**: Inserciones en `rep_check_1` deben aparecer en ambos Slaves si la replicación está bien.
> 🧱 **Esquemas**: Asegúrate de estar usando `testdb`, tal como lo especifica `binlog_do_db` en el master.

```
CREATE DATABASE IF NOT EXISTS testdb;
USE testdb;

CREATE TABLE IF NOT EXISTS rep_check_1 (
  id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  msg VARCHAR(120) NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP NULL DEFAULT NULL ON UPDATE CURRENT_TIMESTAMP
) ENGINE=InnoDB;

INSERT INTO rep_check_1 (msg) VALUES
  ('hola r1'), ('hola r2'), ('hola r3');

SELECT * FROM rep_check_1;
```

### EN SLAVE 

> 🔎 **Verificación**: Debes ver los mismos registros que en el master.

```
USE testdb;
SELECT * FROM rep_check_1;
```

## Otra prueba desde master

> 🧪 **Prueba DDL**: La creación de `rep_check_2` verifica que también se repliquen cambios de esquema (DDL).

```
USE testdb;

-- crea una tabla nueva para forzar que el DDL viaje a ambos slaves
CREATE TABLE rep_check_2 (
  id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  msg VARCHAR(120) NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP NULL DEFAULT NULL ON UPDATE CURRENT_TIMESTAMP
) ENGINE=InnoDB;

INSERT INTO rep_check_2 (msg) VALUES ('hola desde master'), ('probando slave2');
```

## EN SLAVE

> 🔍 **Comprobación final**: La tabla debe existir y contener los registros insertados desde el master.

```
USE testdb;
SHOW TABLES LIKE 'rep_check_2';
SELECT * FROM rep_check_2;
```
