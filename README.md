# pro-asixc1C-g1
Grup format per Laia Coca, Emilia Tikohonova, Brenda Castro i Mario Cabeza.

# 📊 Proyecto: Sistema de Gestión de Base de Datos (InnovateTech)

Este repositorio contiene la documentación, el diseño físico, los scripts de automatización y las políticas de seguridad implementadas en MariaDB para la plataforma de comunicación y streaming **InnovateTech**.

---

## 🗺️ 1. Diseño de la Base de Datos y Modelo Relacional

A continuación se detallan las tablas principales del sistema separadas por módulos funcionales. Haz clic en cada sección para desplegar la estructura física:

<details>
<summary>👥 1. Gestió de Personal</summary>

* **departaments:** `codi_dept` (PK), `nom_compleat`, `telefon`.
* **empleats:** `dni` (PK), `nom`, `cognoms`, `adreca`, `telefon`, `codi_dept` (FK).
</details>

<details>
<summary>💬 2. Sistema de Comunicació i Usuaris</summary>

* **usuaris_sistema:** `id_usuari` (PK), `dni_empleat` (FK, per a treballadors interns), `nom_complet`, `email` (UNIQUE), `extensio_trucades` (UNIQUE), `rol` (ENUM: 'client', 'treballador'), `estat` (ENUM: 'actiu', 'bloquejat'), `enllaç_videotrucada`.
* **configuracion_qualitat:** `id_config` (PK), `nom_grup` (alta, mitja, baixa), `max_amplada_banda`, `parametres_servidor` (ports, protocols).
* **registre_trucades:** `id_trucada` (PK), `usuari_origen` (FK), `usuari_desti` (FK), `data_hora_inici`, `data_hora_fi`, `durada_segons`, `id_config_usada` (FK), `puntuacio_valoracio` (CHECK 1-5), `comentari_valoracio`.
</details>

<details>
<summary>🎬 3. Streaming i Catàleg</summary>

* **cataleg_videos:** `id_video` (PK), `titol`, `descripcio`, `categoria`, `durada`, `data_publicacio`, `url_streaming`.
</details>

<details>
<summary>📡 4. Operacions i Amplada de Banda (Molt important per a ASIX)</summary>

* **mesures_amplada_banda:** `id_mesura` (PK), `data_hora`, `equip_mesurat`, `velocitat_baixada` (DECIMAL), `velocitat_pujada` (DECIMAL), `latencia` (DECIMAL), `resultat` (ENUM: 'acceptable', 'no acceptable'), `dni_operari` (FK a empleats), `observacions`.
</details>

<details>
<summary>🛡️ 5. Seguretat i Auditoria (Mòdul 0377)</summary>

* **taula_avisos (Log d'auditoria):** `id_avis` (PK), `usuari_base_dades`, `taula_afectada`, `operacio_intentada` (INSERT, UPDATE, etc.), `data_hora`.
* **control_backups:** `id_backup` (PK), `data_hora`, `taules_incloses`, `resultat`.
</details>

> [!TIP]
> **Diagrama Draw.io:** [Insertar aquí el enlace a tu imagen o archivo de Draw.io con las relaciones correspondientes].

---

## 🛠️ 2. Justificación de Decisiones Técnicas (Estrategias de Diseño)

Para garantizar la robustez e integridad del SGBD, se han aplicado las siguientes restricciones físicas y de lógica de negocio:

> [!IMPORTANT]
> **Estrategia de defensa (Tablas de Ventas):** Aunque el núcleo del proyecto son las videollamadas, se han creado las tablas `productes`, `comandes` y `cistell`. La justificación es que el **apartado 3.3.1** especifica que el rol de ventas debe gestionar clientes, pedidos, productos y cestas, y el **apartado 3.3.4** pide incluirlas en el backup diario. Para garantizar la integridad referencial del SGBD, debían coexistir en el mismo esquema.

* **Tipos de datos exactos (`DECIMAL`):** Para medir las velocidades y latencia en `mesures_amplada_banda` usamos `DECIMAL(6,2)` en lugar de `FLOAT`. Esto evita la pérdida de precisión decimal, asegurando que si un test da 300.50 Mbps, se guarde exactamente ese valor sin redondeos imprecisos.
* **El "misterio" de los clientes externos (`dni_empleat NULL`):** El sistema lo usan tanto trabajadores como clientes externos. Si el usuario es cliente, no tiene DNI en RRHH. Por ello `dni_empleat` permite nulos. Si un empleado es borrado, su usuario queda en `NULL` (`DELETE SET NULL`), evitando borrar su historial de llamadas.
* **Prevención de colisiones (`UNIQUE`):** Se ha blindado el correo (`uq_email_usuario`) y la extensión telefónica (`uq_extension`) en `usuaris_sistema` para evitar conflictos en la red VoIP.
* **Restricciones de dominio (`ENUM` y `CHECK`):** En la configuración de calidad (`QoS`) usamos `ENUM('alta', 'mitja', 'baixa')`. Además, en las llamadas, la restricción `chk_puntuacio` asegura nativamente que las valoraciones del servicio solo puedan ser del 1 al 5.
* **Integridad Relacional (`ON DELETE`):**
  * **RESTRICT:** En departamentos (evita borrar un departamento con trabajadores activos) y en facturas/pedidos (evita borrar datos históricos de contabilidad).
  * **CASCADE:** En la cesta (`cistell`), de modo que si se borra un producto del catálogo, se retira automáticamente de las cestas.

---

## ⚙️ 3. Despliegue de MariaDB y Creación del Entorno

### Actualización del sistema e Instalación
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install mariadb-server -y
sudo systemctl status mariadb
