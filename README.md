# Servicios telematicos proyecto final 2025-02
# Lista de proyectos 2 - Proyecto #5: Balanceo de carga de base de datos con MYSQL y MYSQL ROUTER

> **Topología (4 VMs)**  
> `mysql-master`, `mysql-slave1`, `mysql-slave2`, `mysql-router`

---

## 0) Preparación
### Actualizar e instalar MySQL Server (SOLO en master/slaves)
```
sudo apt update
sudo apt install -y mysql-server
sudo systemctl enable mysql
sudo systemctl start mysql
sudo systemctl status mysql
```

### Utilidades de red (En todas las maquinas)
```
sudo apt install -y net-tools
```

💡 **mysql (cliente CLI) — tips útiles**  
- *Entrar a MySQL como root:* `sudo mysql -u root -p`  
- *Salir:* escribe `\q` y presiona Enter  
- *Formato vertical:* agrega `\G` al final de un comando (ej.: `SHOW SLAVE STATUS\G`)  
- *Cambiar de base:* `USE testdb;`  
- *Ver usuarios:* `SELECT user, host FROM mysql.user;`

💡 **vim — tips útiles**  
- *Para salir guardando:* **Esc** → `:wq` → **Enter**  
- *Para salir sin guardar:* **Esc** → `:q!` → **Enter**  
- *Para vaciar todo:* **Esc** → `:%d` → **Enter**  
- *Para buscar la línea `"linea a buscar"`:* **Esc** → `:/linea a buscar` → **Enter**, y usa `n` para siguiente coincidencia

---

## 1) Configuración en **mysql-master**

### 1.1 Habilitar binlogs y escuchar en todas las interfaces
```
# editar la configuración de MySQL del master
sudo vim /etc/mysql/mysql.conf.d/mysqld.cnf
```

**Pega/asegura estas líneas (ajusta solo si ya existen):**
```ini
server-id              = 1
log_bin                = /var/log/mysql/mysql-bin.log
# binlog_expire_logs_seconds = 2592000
max_binlog_size        = 100M
binlog_do_db           = testdb
bind-address           = 0.0.0.0
binlog_ignore_db       = mysql
```

```
# reiniciar MySQL del master
sudo systemctl restart mysql
```

### 1.2 Crear usuario de replicación y obtener posición del binlog
```
sudo mysql -u root -p
```

```sql
-- contraseña: root
CREATE USER 'repl'@'%' IDENTIFIED WITH mysql_native_password BY 'replpass';

-- Opción común (con cliente también):
GRANT REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'repl'@'%';

-- (si tuviste un usuario mal escrito, ignóralo y usa el correcto 'repl')
FLUSH PRIVILEGES;

FLUSH TABLES WITH READ LOCK;
SHOW MASTER STATUS;
-- Anota File y Position (ej.: File='mysql-bin.000001', Position=1897)
```

> **IMPORTANTE:** Mantén la sesión abierta tras `FLUSH TABLES WITH READ LOCK;` hasta finalizar la configuración de los slaves.

---

## 2) Configuración en **mysql-slave1**

### 2.1 Definir `server-id` y `relay_log`, reiniciar
```
sudo vim /etc/mysql/mysql.conf.d/mysqld.cnf
```

**Agrega/asegura:**
```
bind-address = 0.0.0.0
server-id  = 2
relay_log  = /var/log/mysql/mysql-relay-bin.log
```

```
sudo systemctl restart mysql
```

### 2.2 Apuntar al master y arrancar la replicación
```
sudo mysql -u root -p
```

```sql
-- contraseña: root
STOP SLAVE;
RESET SLAVE ALL;
```

## Usa los valores exactos de SHOW MASTER STATUS del master (File y Position)
En este caso nos apareció 

# ⚠ ⚠ ⚠ ⚠ INSERTAR IMAGEN 1 EN WS

Por lo tanto nos quedaría así:
```
CHANGE MASTER TO
  MASTER_HOST='192.168.50.10',
  MASTER_USER='repl',
  MASTER_PASSWORD='replpass',
  MASTER_LOG_FILE='mysql-bin.000001',
  MASTER_LOG_POS=847;

START SLAVE;
SHOW SLAVE STATUS\G
```

---

## 3) Configuración en **mysql-slave2**

### 3.1 Definir `server-id` y `relay_log`, reiniciar
```
sudo vim /etc/mysql/mysql.conf.d/mysqld.cnf
```

**Agrega/asegura:**
```
bind-address = 0.0.0.0
server-id  = 3
relay_log  = /var/log/mysql/mysql-relay-bin.log
```

```
sudo systemctl restart mysql
```

### 3.2 Apuntar al master y arrancar la replicación
```
sudo mysql -u root -p
```

```sql
-- contraseña: root
STOP SLAVE;
RESET SLAVE ALL;

-- Usamos los valores exactos de SHOW MASTER STATUS del master (File y Position) como lo hicimos anteriormente

CHANGE MASTER TO
  MASTER_HOST='192.168.50.10',
  MASTER_USER='repl',
  MASTER_PASSWORD='replpass',
  MASTER_LOG_FILE='mysql-bin.000001',
  MASTER_LOG_POS=847;

START SLAVE;
SHOW SLAVE STATUS\G
```

---

## 4) Pruebas de replicación (DDL/DML)

### 4.1 En **mysql-master**
```
sudo mysql -u root -p
```
> En caso de tener el lock aberto usar: ``` UNLOCK TABLES; ``` 

```sql
-- contraseña: root
CREATE DATABASE IF NOT EXISTS testdb;
USE testdb;

CREATE TABLE IF NOT EXISTS rep_check_1 (
  id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  msg VARCHAR(120) NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP NULL DEFAULT NULL ON UPDATE CURRENT_TIMESTAMP
) ENGINE=InnoDB;

INSERT INTO rep_check_1 (msg) VALUES ('hola r1'), ('hola r2'), ('hola r3');

SELECT * FROM rep_check_1;
```

### 4.2 En **mysql-slave1** y **mysql-slave2**
```
sudo mysql -u root -p
```

```sql
-- contraseña: root
USE testdb;
SELECT * FROM rep_check_1;
```

### 4.3 DDL extra para forzar replicación a ambos slaves — en **mysql-master**
```
sudo mysql -u root -p
```

```sql
-- contraseña: root
USE testdb;

CREATE TABLE rep_check_2 (
  id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  msg VARCHAR(120) NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP NULL DEFAULT NULL ON UPDATE CURRENT_TIMESTAMP
) ENGINE=InnoDB;

INSERT INTO rep_check_2 (msg) VALUES ('hola desde master'), ('probando slave2');
```

### 4.4 Verificar en **mysql-slave1** y **mysql-slave2**
```
sudo mysql -u root -p
```

```sql
-- contraseña: root
USE testdb;
SHOW TABLES LIKE 'rep_check_2';
SELECT * FROM rep_check_2;
```

---

## 5) Usuario de aplicación

### 5.1 En **mysql-master** (permisos DML/DDL)
```
sudo mysql -u root -p
```

```sql
-- contraseña: root
CREATE USER IF NOT EXISTS 'appuser'@'192.168.50.13' IDENTIFIED BY '1234';
GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, DROP, INDEX, ALTER ON testdb.* TO 'appuser'@'192.168.50.13';
FLUSH PRIVILEGES;
```

### 5.2 En **mysql-slave1** y **mysql-slave2** (solo lectura)
```
sudo mysql -u root -p
```

```sql
-- contraseña: root
CREATE USER IF NOT EXISTS 'appuser'@'192.168.50.13' IDENTIFIED BY '1234';
GRANT SELECT ON testdb.* TO 'appuser'@'192.168.50.13';
FLUSH PRIVILEGES;
```

---

## 6) MySQL Router en **mysql-router**

### 6.1 Instalar MySQL Router y cliente
```
sudo apt update
wget https://dev.mysql.com/get/mysql-apt-config_0.8.36-1_all.deb
sudo dpkg -i mysql-apt-config_0.8.36-1_all.deb
```
 > En este espacio nos saldrá una ventana, debemos dar a la flecha abajo en nuestro teclado para seleccionar "ok" y salir.
```
sudo apt-get update
sudo apt-get install -y mysql-router
sudo apt --fix-broken install -y
sudo apt install -y mysql-client
```

### 6.2 Configurar `mysqlrouter.conf`
```
cd /etc/mysqlrouter/
sudo vim mysqlrouter.conf
```
**Contenido de mysqlrouter.conf**
```
[DEFAULT]
name = manual_replication_router
user = mysqlrouter
keyring_path = /var/lib/mysqlrouter/keyring
logging_folder = /var/log/mysqlrouter
runtime_folder = /var/run/mysqlrouter
config_folder = /etc/mysqlrouter

[logger]
level = INFO

[routing:master]
bind_address = 0.0.0.0
bind_port = 6446
destinations = 192.168.50.10:3306
mode = read-write
routing_strategy = first-available

[routing:slaves]
bind_address = 0.0.0.0
bind_port = 6447
destinations = 192.168.50.11:3306,192.168.50.12:3306
mode = read-only
routing_strategy = round-robin

[keepalive]
interval = 60

```
> Reiniciamos para guardar los cambios
```
sudo systemctl restart mysqlrouter
```

---

## 7) Pruebas contra el Router desde **mysql-router**
### Prueba de SOLO LECTURA (6447). Ejecuta varias veces para ver el balanceo entre slave1/slave2.

```
mysql -u appuser -p'1234' -h 127.0.0.1 -P 6447 -e "USE testdb; SELECT @@hostname, @@port, COUNT(*) FROM rep_check_2;"
mysql -u appuser -p'1234' -h 127.0.0.1 -P 6447 -e "USE testdb; SELECT @@hostname, @@port, COUNT(*) FROM rep_check_2;"
```

### Prueba de LECTURA/ESCRITURA (6446). Debe escribir en el master.
```
mysql -u appuser -p'1234' -h 127.0.0.1 -P 6446 -e "USE testdb; INSERT INTO rep_check_2 (msg) VALUES ('prueba via router'); SELECT @@hostname AS backend_host, @@port AS backend_port; SELECT * FROM rep_check_2 ORDER BY id DESC LIMIT 5;"
```

---

## 8) Desbloquear tablas en el master cuando termines el bootstrap
```
sudo mysql -u root -p -e "UNLOCK TABLES;"
```
