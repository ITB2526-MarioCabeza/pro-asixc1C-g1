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

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install mariadb-server -y
sudo systemctl status mariadb
