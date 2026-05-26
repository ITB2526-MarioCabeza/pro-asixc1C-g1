# pro-asixc1C-g1
Grup format per Laia Coca, Emilia Tikohonova, Brenda Castro i Mario Cabeza.

# 📊 Proyecto SGBD: Plataforma Avanzada InnovateTech

Este repositorio contiene la infraestructura física, scripts de automatización, triggers de seguridad y políticas de hardening del sistema operativo para la plataforma de comunicación, soporte y streaming de **InnovateTech**.

---

## 🗺️ Apartado 1: Diseño del Modelo Relacional (16 Tablas)

El esquema de la base de datos se estructura de forma homogénea en catalán, consolidando un total de **16 tablas** operativas divididas en módulos funcionales.

### 👥 1.1. Gestió de Personal i Recursos Humans
* **departaments:** `codi_dept` (PK), `nom_compleat`, `telefon`.
* **grups_nivells:** `id_grup_nivell` (PK), `nom_grup`, `descripcio`.
* **empleats:** `dni` (PK), `nom`, `cognoms`, `adreca`, `telefon`, `codi_dept` (FK), `id_grup_nivell` (FK).
* **nomines:** `id_nomina` (PK), `dni_empleat` (FK), `mes`, `any`, `salari_base`, `deduccions`, `total_net`.

### 💬 1.2. Sistema de Comunicació, Usuaris i QoS
* **usuaris_sistema:** `id_usuari` (PK), `dni_empleat` (FK, vació para clientes externos), `nom_complet`, `email` (UNIQUE), `extensio_trucades` (UNIQUE), `rol` (ENUM: 'client', 'treballador'), `estat` (ENUM: 'actiu', 'bloquejat'), `enllaç_videotrucada`.
* **qualitats:** `id_calitat` (PK), `nom_perfil` (ENUM: 'alta', 'mitja', 'baixa'), `max_amplada_banda`, `ports_protocols`.
* **registre_trucades:** `id_trucada` (PK), `usuari_origen` (FK), `usuari_desti` (FK), `data_hora_inici`, `data_hora_fi`, `durada_segons`, `id_calitat_usada` (FK), `puntuacio_valoracio` (CHECK), `comentari_valoracio`.

### 🎬 1.3. Streaming i Catàleg de Continguts
* **cataleg_videos:** `id_video` (PK), `titol`, `descripcio`, `categoria`, `durada`, `data_publicacio`, `url_streaming`.

### 📡 1.4. Operacions, Red i Amplada de Banda
* **mesures_ampla_banda:** `id_mesura` (PK), `data_hora`, `equip_mesurat`, `velocitat_baixada` (DECIMAL), `velocitat_pujada` (DECIMAL), `latencia` (DECIMAL), `resultat` (ENUM: 'acceptable', 'no acceptable'), `dni_operari` (FK a empleats), `observacions`.

### 🛒 1.5. Operacions Comercials y Vendes
* **clients:** `id_client` (PK), `nom_empresa`, `cif`, `telefon_contacte`.
* **productes:** `id_producte` (PK), `nom_producte`, `preu` (DECIMAL), `estoc`.
* **comandes:** `id_comanda` (PK), `id_client` (FK), `data_comanda`, `estat_pagament`.
* **cistell:** `id_cistell` (PK), `id_client` (FK), `id_producte` (FK), `quantitat`.

### 🛡️ 1.6. Seguretat, Auditories i Respaldo (Mòdul 0377)
* **taula_avisos:** `id_avis` (PK), `usuari_base_dades`, `taula_afectada`, `operacio_intentada`, `data_hora`.
* **registre_backups:** `id_backup` (PK), `data_hora`, `taules_incloses`, `resultat`.
* **logs_auditorias:** `id_log` (PK), `usuari_sistema`, `accio_realitzada`, `detalls`, `data_hora`.

> [!TIP]
> **Actualización del Diagrama (Draw.io):** Se han eliminado las tablas obsoletas del borrador inicial, garantizando que el diseño gráfico de tablas y relaciones se corresponda estrictamente con este esquema físico de 16 entidades.

---

## 🛠️ Apartado 2: Arquitectura de Datos y Defensa del Diseño

Durante la definición del DDL en el SGBD se han tomado decisiones arquitectónicas clave para blindar la integridad del sistema:

> [!IMPORTANT]
> **Estrategia de Defensa ante el Tribunal (Tablas de Ventas):** Si el docente cuestiona la presencia de un módulo de facturación (`clients`, `productes`, `comandes`, `cistell`) en un proyecto centrado en videollamadas, la justificación técnica es contundente: *El apartado 3.3.1 exige de manera explícita que el rol de ventas gestione clientes, pedidos y cestas, mientras que el apartado 3.3.4 requiere su inclusión en el backup diario. Para salvaguardar la integridad referencial nativa del motor InnoDB, es obligatorio que coexistan dentro del mismo esquema relacional*.

* **Precisión Métricas de Red (`DECIMAL`):** Las velocidades de bajada, subida y latencia en `mesures_ampla_banda` utilizan el tipo de dato `DECIMAL(6,2)` en lugar de un tipo flotante impreciso (`FLOAT`/`DOUBLE`). Esto evita errores de redondeo matemático acumulado en las auditorías de red.
* **El Enigma de los Clientes Externos (`dni_empleat NULL`):** La plataforma da soporte tanto a trabajadores internos como a clientes externos. Al no poseer estos últimos un registro en Recursos Humanos, el campo `dni_empleat` de la tabla `usuaris_sistema` se define como nullable. Además, se añade la regla `ON DELETE SET NULL` para asegurar que, si un operario causa baja en la empresa, el histórico de sus videollamadas no sea destruido.
* **Prevención de Colisiones de Red (`UNIQUE`):** Los campos de correo electrónico (`email`) y extensión de VoIP (`extensio_trucades`) están blindados con restricciones de unicidad para evitar suplantaciones de identidad o colisiones en la centralita de telefonía.
* **Restricciones de Integridad Referencial Avanzada (`ON DELETE`):**
  * `RESTRICT`: Implementado en `departaments` y `comandes` para bloquear la eliminación accidental de registros históricos de contabilidad o dejar trabajadores activos sin un departamento asignado.
  * `CASCADE`: Configurado en la tabla `cistell`. Si un producto se retira de la venta o se purga un cliente, los elementos de sus cestas activas se eliminan automáticamente del sistema de forma limpia.
* **Validación de Datos en Motor (`CONSTRAINT CHECK`):** La tabla de registros de llamadas contiene una regla de comprobación física (`chk_puntuacio`) que restringe las notas del servicio del 1 al 5, bloqueando de manera nativa inserciones anómalas de la aplicación web.

---

## ⚙️ Apartado 3: Despliegue del Servidor MariaDB e Instalación

### 3.1. Actualización e Instalación en AWS EC2
Para preparar el entorno de producción y verificar el correcto levantamiento del demonio del SGBD, se ejecutan los siguientes comandos en la terminal de Linux:

-- Primer paso: Creación e ingreso al esquema principal
CREATE DATABASE innovatetech_db;
USE innovatetech_db;
3.3. Inserción de Datos de Prueba
Se ha poblado el SGBD de manera controlada para dar cumplimiento a los requisitos funcionales:

Alta de Departamentos (Requisito 3.1) e inserción de trabajadores en la tabla Empleados.

Definición de perfiles de Calidad de Servicio (QoS) en la tabla qualitats.

Inserción de Usuarios de Sistema vinculando tanto DNI de personal interno como accesos de clientes externos.

Carga del catálogo de contenidos en la tabla cataleg_videos.

Datos operacionales cruzados: asignación de nóminas, generación de productos, apertura de cestas de compra e historial base en registre_trucades.

🔐 Apartado 4: Control de Accesos, Roles y Matriz de Permisos
El control de accesos se rige bajo la premisa de la restricción implícita: todo lo que no está explícitamente concedido por un administrador está prohibido por defecto por el motor MariaDB.

SQL
-- Declaración de los roles exigidos en el apartado 3.3.1
CREATE ROLE 'admin_rol', 'vendes_rol', 'administracio_rol', 'treballador_rol';
4.1. Matriz de Privilegios del Sistema
Admin (admin_rol): Ostenta el control absoluto de la base de datos (ALL PRIVILEGES). Cuenta en exclusiva con la asignación del privilegio global FILE. Este permiso crítico del sistema operativo faculta a MariaDB para realizar operaciones de lectura y escritura directa sobre el sistema de almacenamiento local de la instancia EC2, acción necesaria para la ejecución automatizada de las copias de seguridad.

Vendes (vendes_rol): Privilegios de escritura y lectura (SELECT, INSERT, UPDATE) restringidos exclusivamente al flujo de negocio: tablas clients, comandes, productes, cistell y acceso consultivo a llamadas.

Administració (administracio_rol): Permisos de gestión completa sobre el área de recursos humanos (empleats, nomines, departaments, grups_nivells). Por defecto, MariaDB les deniega el acceso a la tabla registre_trucades al no existir una concesión explícita.

Treballador (treballador_rol): Aplicación estricta del Principio de Mínimo Privilegio. El operario tiene permisos de lectura sobre el catálogo de vídeos y productos, y facultades de INSERT en registre_trucades para dejar constancia de su actividad. No dispone de privilegios de UPDATE ni DELETE sobre las llamadas, evitando que un empleado altere de manera fraudulenta las duraciones o métricas de soporte.

🛡️ Apartado 5: Triggers de Seguridad y Reglas de Negocio
Para asegurar que la lógica de negocio y las políticas de la empresa se cumplan de manera automatizada en la capa de datos, se han implementado los siguientes Triggers (Disparadores) activos:

Filtro de Duplicados en Alta: Detiene la ejecución lanzando un error controlado mediante SIGNAL SQLSTATE si la capa de aplicación intenta duplicar identificadores o usuarios existentes.

Control de Usuarios Bloqueados: Un disparador de tipo BEFORE INSERT en registre_trucades analiza el estado del emisor en usuaris_sistema. Si el flag está marcado como 'bloquejat', la inserción de la llamada es abortada de forma nativa.

Cortafuegos Antiactividad Diaria (DoS): Trigger que computa el número de llamadas realizadas por un usuario específico en el día actual. Al alcanzar el límite estricto de 50 llamadas, el disparador modifica automáticamente el estado de ese usuario a 'bloquejat' en la tabla correspondiente para mitigar usos indebidos de la red VoIP.

Umbral de Consumo Mensual: Restringe el inicio de nuevas videollamadas si el sumatorio de minutos acumulados por el identificador del usuario supera los 1000 minutos mensuales.

Vigilancia de Estructuras Financieras: Trigger centrado en la tabla nomines que detecta el usuario del sistema operativo que intenta hacer modificaciones. Bloquea de inmediato cualquier operación de UPDATE o DELETE sobre salarios si proviene de los roles de treballador o vendes.

Aislamiento Organizacional de Comunicaciones: Disparador que deniega e interrumpe cualquier interacción de inserción o alteración sobre el sistema de registros de llamadas si el operador está asignado al rol de administracio.

🤖 Apartado 6: Scripting de Backup y Automatización (Bash/Cron)
Para el resguardo diario de la información corporativa, se ha programado un script en Bash que realiza un volcado lógico de la base de datos (mysqldump), registrando el resultado de la operación en la tabla registre_backups y enviando alertas en caso de fallo a la entidad taula_avisos (Log de auditoría exigido en el apartado 3.3.3 del módulo 0377).

6.1. Automatización con Crontab
Para evitar la degradación del rendimiento de la red corporativa durante las horas de producción, el script se ha automatizado mediante el demonio cron del sistema operativo para ejecutarse de manera persistente a las 23:00h:

Bash
# Permisos de ejecución aplicados al script
chmod +x /home/mario.cabeza.7e9/scripts/backup.sh

# Entrada programada en el crontab del usuario del sistema
0 23 * * * /home/mario.cabeza.7e9/scripts/backup.sh
🖥️ Apartado 7: Hardening, Gestión de Usuarios y Admin de Linux (AWS EC2)
Se han implementado políticas de robustecimiento e infraestructura a nivel del sistema operativo Linux para securizar el nodo de AWS:

Creación de Operador Alternativo: Se ha dado de alta al usuario del sistema mario.cabeza.7e9, otorgándole privilegios administrativos elevados mediante su inclusión controlada en el archivo de configuración /etc/sudoers.

Migración de Entorno y Persistencia: Para consolidar el entorno de trabajo bajo el nuevo operador, se han transferido los scripts y ficheros generados originalmente en el terminal por el usuario por defecto (ubuntu), aplicando comandos chown y chmod para otorgar la propiedad absoluta al nuevo usuario de manera persistente.

Hardening SSH (Autenticación por Contraseña Forzada): Se modificaron las directivas del archivo /etc/ssh/sshd_config, forzando al demonio SSH a exigir obligatoriamente una contraseña robusta para el usuario alternativo, anulando accesos directos sin validación de credenciales como contramedida de seguridad.

Persistencia de Identidad de Red (Hostname): Se configuró permanentemente el nombre de red de la máquina. La persistencia del cambio se ha validado de manera exitosa tras un ciclo completo de reinicio de la instancia (sudo reboot).

🔍 Apartado 8: Validaciones Finales y Comprobación del SGBD
Para constatar que el estado final de la base de datos es óptimo y se corresponde con la reestructuración física de las 16 tablas, se realizaron consultas de auditoría desde el entorno del operador mario.cabeza.7e9:

8.1. Verificación del Catálogo de Tablas
SQL
-- Comprobación del número total de entidades creadas
SHOW TABLES;

-- Validación de la estructura de campos en tablas core
DESCRIBE departaments;
DESCRIBE grups_nivells;
DESCRIBE empleats;
DESCRIBE nomines;
DESCRIBE usuaris_sistema;
DESCRIBE qualitats;
DESCRIBE registre_trucades;
8.2. Inspección DDL y Reglas de Integridad
SQL
-- Extracción y auditoría de restricciones físicas nativas
SHOW CREATE TABLE empleats;        -- Verifica relaciones FK y reglas RESTRICT
SHOW CREATE TABLE registre_trucades; -- Verifica restricciones CONSTRAINT CHECK (1-5)
SHOW CREATE TABLE cistell;         -- Verifica la cascada activa (ON DELETE CASCADE)
8.3. Auditoría de Seguridad de Usuarios y Disparadores
SQL
-- Consulta de mapeo de usuarios del motor y privilegios asignados
SELECT User, Host FROM mysql.user;
SHOW GRANTS FOR 'mario.cabeza.7e9'@'localhost';

-- Verificación de la presencia de los Triggers de control perimetral
SHOW TRIGGERS;

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install mariadb-server -y
sudo systemctl status mariadb
