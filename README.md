<div align="center">
  <h1>📊 Proyecto SGBD: Plataforma Avanzada InnovateTech</h1>
  <p><i>Documentació tècnica completa del projecte transversal d'infraestructura, seguretat i administració de bases de dades.</i></p>

  ![MariaDB](https://img.shields.io/badge/MariaDB-003545?style=for-the-badge&logo=mariadb&logoColor=white)
  ![Linux](https://img.shields.io/badge/Linux-FCC624?style=for-the-badge&logo=linux&logoColor=black)
  ![Bash](https://img.shields.io/badge/Bash-4EAA25?style=for-the-badge&logo=gnu-bash&logoColor=white)
  ![AWS](https://img.shields.io/badge/AWS_EC2-FF9900?style=for-the-badge&logo=amazon-aws&logoColor=white)
</div>

---

## 👥 Grup de Treball (pro-asixc1C-g1)
* **Laia Coca**
* **Emilia Tikohonova**
* **Brenda Castro**
* **Mario Cabeza**

---

## 🗺️ Apartat 1: Disseny i Arquitectura del Model Relacional

L'esquema físic de la base de dades `innovatetech_db` s'ha normalitzat i traduït homogèniament al **català**, consolidant un total de **16 taules** operatives distribuïdes en mòduls funcionals.

### 🔄 Actualització del Diagrama (Draw.io)
Per garantir la correspondència exacta amb el model físic final de 16 entitats, s'han dut a terme les següents accions:
* Eliminació exhaustiva de taules obsoletes de l'esborrany inicial.
* Canvi complet de noms de l'esquema a l'idioma **català**.
* Creació de les taules estructurals: `grups_nivells`, `qualitats` i `registre_trucades`.
* Disseny i enllaç de claus estrangeres (FK) crítiques (com la columna `grup_nivell`).

---

## 🗂️ Mòduls de la Base de Dades (16 Taules)

### 1.1. Gestió de Personal i Recursos Humans
Centralitza l'organització de l'empresa, dades dels empleats i finances salarials.

| Taula | Clau Primària (PK) | Claus Foranes (FK) | Atributs i Restriccions |
| :--- | :--- | :--- | :--- |
| **`departaments`** | `codi_dept` | - | `nom_compleat`, `telefon` |
| **`grups_nivells`**| `id_grup_nivell` | - | `nom_grup`, `descripcio` |
| **`empleats`** | `dni` | `codi_dept`, `id_grup_nivell` | `nom`, `cognoms`, `adreca`, `telefon` |
| **`nomines`** | `id_nomina` | `dni_empleat` | `mes`, `any`, `salari_base`, `deduccions`, `total_net` |

> [!IMPORTANT]
> **Regla de Seguretat `ON DELETE RESTRICT`:** S'ha blindat la relació entre departaments i empleats. Si s'intenta eliminar un departament que encara conté treballadors actius, el SGBD denegarà l'acció nativament per evitar deixar personal "orfe".

### 1.2. Sistema de Comunicació, Usuaris i QoS
Controla l'accés a la plataforma de videotrucades i paràmetres de Qualitat de Servei (QoS).

| Taula | Clau Primària (PK) | Claus Foranes (FK) | Atributs i Restriccions |
| :--- | :--- | :--- | :--- |
| **`usuaris_sistema`**| `id_usuari` | `dni_empleat` *(Podeu NULL)* | `nom_complet`, `email` (**UNIQUE**), `extensio_trucades` (**UNIQUE**), `rol` (ENUM), `estat` (ENUM), `enllaç_videotrucada` |
| **`qualitats`** | `id_calitat` | - | `nom_perfil` (ENUM: 'alta', 'mitja', 'baixa'), `max_amplada_banda`, `ports_protocols` |
| **`registre_trucades`**| `id_trucada` | `usuari_origen`, `usuari_desti`, `id_calitat_usada` | `data_hora_inici`, `data_hora_fi`, `durada_segons`, `puntuacio_valoracio` (**CHECK**), `comentari_valoracio` |

> [!TIP]
> **El "misteri" dels clients externs (dni_empleat NULL):** El sistema és utilitzat tant per treballadors com per clients externs. Si l'usuari és un client, no té DNI a la taula de RRHH. Per això es permet un valor `NULL`. Si s'esborra un empleat, el seu usuari de trucades no desapareix, passa a `DELETE SET NULL`.
> 
> **Restriccions de seguretat (UNIQUE):** Hem blindat el correu (`uq_email_usuario`) i l'extensió telefònica (`uq_extension`) per evitar col·lisions a la xarxa VoIP.

### 1.3. Streaming i Catàleg de Continguts
Administra els continguts audiovisuals sota demanda.

| Taula | Clau Primària (PK) | Claus Foranes (FK) | Atributs i Restriccions |
| :--- | :--- | :--- | :--- |
| **`cataleg_videos`** | `id_video` | - | `titol`, `descripcio`, `categoria`, `durada`, `data_publicacio`, `url_streaming` |

### 1.4. Operacions, Xarxa i Amplada de Banda
Auditoria contínua del rendiment de xarxa per als operaris tècnics.

| Taula | Clau Primària (PK) | Claus Foranes (FK) | Atributs i Restriccions |
| :--- | :--- | :--- | :--- |
| **`mesures_ampla_banda`**| `id_mesura` | `dni_operari` | `data_hora`, `equip_mesurat`, `velocitat_baixada` (**DECIMAL**), `velocitat_pujada` (**DECIMAL**), `latencia` (**DECIMAL**), `resultat` (ENUM), `observacions` |

> [!NOTE]
> **Control d'errors amb CONSTRAINT CHECK:** S'assegura que les puntuacions de trucada només estiguin en el rang de 1 a 5. Qualsevol intent de la capa web d'inserir un valor fora de rang serà bloquejat de forma nativa.
>
> **Tipus de dades exactes (DECIMAL):** Per a les mètriques de xarxa s'utilitza `DECIMAL(6,2)` en lloc de FLOAT, evitant pèrdues de precisió matemàtica o arrodoniments estranys en els tests de velocitat. Víncul de l'operari directe amb la taula d'empleats per auditories internes.

### 1.5. Operacions Comercials i Vendes
Mòdul integrat per respondre als requeriments comercials de l'enunciat.

| Taula | Clau Primària (PK) | Claus Foranes (FK) | Atributs i Restriccions |
| :--- | :--- | :--- | :--- |
| **`clients`** | `id_client` | - | `nom_empresa`, `cif`, `telefon_contacte` |
| **`productes`** | `id_producte` | - | `nom_producte`, `preu` (DECIMAL), `estoc` |
| **`comandes`** | `id_comanda` | `id_client` | `data_comanda`, `estat_pagament` |
| **`cistell`** | `id_cistell` | `id_client`, `id_producte`| `quantitat` |

> [!WARNING]
> **🔥 Estratègia de defensa:** Si el professor pregunta: *«Per què has creat taules de compres si el projecte parla de videotrucades?»*, la teva resposta impecable serà: *"Perquè a l'apartat 3.3.1 s'especifica que el rol de vendes ha de gestionar clients, comandes, productes i cistells, i en el 3.3.4 es demana incloure-les al backup diari. Per garantir la integritat referencial del SGBD, havien de coexistir en el mateix esquema."*
>
> **DELETE CASCADE vs RESTRICT:** Si un client buida el seu compte, el seu `cistell` s'esborra automàticament (CASCADE). En canvi, a les `comandes` (factures) s'usa `RESTRICT` per evitar esborrar dades històriques de comptabilitat per error.

### 1.6. Seguretat, Auditories i Respatller (Mòdul 0377)
Mòdul d'administració per a seguretat lògica i persistència.

| Taula | Clau Primària (PK) | Claus Foranes (FK) | Atributs i Restriccions |
| :--- | :--- | :--- | :--- |
| **`taula_avisos`** | `id_avis` | - | `usuari_base_dades`, `taula_afectada`, `operacio_intentada`, `data_hora` |
| **`registre_backups`** | `id_backup` | - | `data_hora`, `taules_incloses`, `resultat` |
| **`logs_auditorias`** | `id_log` | - | `usuari_sistema`, `accio_realitzada`, `detalls`, `data_hora` |

---

## ⚙️ Desplegament i Preparació de l'Entorn

### 1. Instal·lació del Servidor SGBD
```bash
# Actualització del sistema i instal·lació de MariaDB
sudo apt update && sudo apt upgrade -y
sudo apt install mariadb-server -y

# Comprovació del correcte funcionament del servei
sudo systemctl status mariadb
