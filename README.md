# Projecte Transversal ASIXc1 — InnovateTech
 
> Infraestructura CPD híbrida (física + AWS) amb serveis de directori, logs centralitzats, streaming multimèdia i base de dades per a la plataforma **InnovateTech**.
 
**Curs:** 2025/2026 · **Grup:** ASIXc1C · **Equip:** pro-asixc1C-g1
 
---
 
## Equip de Desenvolupament
 
| Membre | Rol principal |
| :--- | :--- |
| Laia Coca | Serveis d'àudio i vídeo |
| Emilia Tikohonova | Servidor de logs (Rsyslog + ELK) |
| Brenda Castro | Màquina web, SFTP i LDAP |
| Mario Cabeza | Base de dades i automatització |
 
---
 
## Índex
 
1. [Descripció del Projecte](#1-descripció-del-projecte)
2. [Arquitectura General AWS](#2-arquitectura-general-aws)
   - 2.1 [Prerequisit Global: Accés i Administració de Màquines](#21-prerequisit-global-accés-i-administració-de-màquines)
3. [Servidor LDAP (OpenLDAP)](#3-servidor-ldap-openldap)
4. [Màquina Web: Nginx + SFTP](#4-màquina-web-nginx--sftp)
5. [Servidor de Logs: Rsyslog + ELK Stack](#5-servidor-de-logs-rsyslog--elk-stack)
6. [Base de Dades (MariaDB)](#6-base-de-dades-mariadb)
---
 
## 1. Descripció del Projecte
 
**InnovateTech** és una empresa de provisió de serveis tecnològics que requereix modernitzar la seva capacitat operativa. Aquest projecte dissenya i implanta la infraestructura tecnològica completa: des de la proposta de CPD físic fins al desplegament al núvol AWS, incloent-hi serveis multimèdia, directori d'usuaris, centralització de logs i una base de dades integral.
 
L'arquitectura resultant cobreix:
 
- Disseny físic del CPD (ubicació, racks, SAI, seguretat física i lògica).
- Desplegament al núvol AWS amb serveis de directori actiu (LDAP), web, SFTP i logs centralitzats.
- Infraestructura de streaming d'àudio i vídeo, i servei de videoconferència.
- Base de dades relacional amb control d'accés per rols, triggers d'auditoria i còpies de seguretat automatitzades.
- Documentació en Markdown publicada a GitHub i automatització amb Ansible.
---
 
## 2. Arquitectura General AWS
 
La infraestructura de producció s'ha desplegat íntegrament a **Amazon EC2** (sense RDS ni AMIs del marketplace), seguint el principi de màxim control sobre el sistema operatiu i la configuració.
 
| Instància | Hostname | IP Privada | IP Pública | Serveis |
| :--- | :--- | :--- | :--- | :--- |
| EC2 Ubuntu | `srv-ldap` | `10.0.1.224` | — | OpenLDAP (slapd) |
| EC2 Ubuntu | `srv-web-ftps` | `10.0.1.112` | `98.83.3.235` | Nginx, SFTP, SSSD |
| EC2 Ubuntu | `srv-logs` | `10.0.6.239` | `52.4.133.45` | Rsyslog, ELK Stack |
| EC2 Ubuntu | `srv-bbdd` | — | — | MariaDB |
 
> Cada servei resideix en una instància independent, a excepció del servei web i SFTP que cohabiten a `srv-web-ftps`.
 
---
 
### 2.1 Prerequisit Global: Accés i Administració de Màquines
 
Totes les instàncies EC2 del projecte segueixen el mateix patró d'accés: **un usuari d'administració dedicat per màquina**, autenticat exclusivament per clau pública RSA. No s'utilitza mai l'usuari per defecte (`ubuntu`) ni l'autenticació per contrasenya.
 
#### Creació de l'usuari d'administració
 
El procediment és idèntic per a totes les màquines. A continuació s'il·lustra amb l'usuari `emilia` al servidor de logs (per a la resta de màquines, substituir el nom d'usuari corresponent).
 
```bash
# Crear el nou usuari d'administració
sudo adduser emilia
 
# Afegir-lo al grup sudo
sudo usermod -aG sudo emilia
 
# Canviar a l'usuari creat
su - emilia
```
 
#### Configuració de l'accés per clau pública
 
**Pas 1 — Generar el parell de claus a la màquina local** (si no existeix ja):
 
```bash
ssh-keygen -t rsa -b 4096
# Genera:
#   ~/.ssh/id_rsa       → clau privada (no compartir mai)
#   ~/.ssh/id_rsa.pub   → clau pública
```
 
**Pas 2 — Instal·lar la clau pública a la instància** (des de la màquina local):
 
```bash
ssh-copy-id -i ~/.ssh/id_rsa.pub usuari@IP_INSTANCIA
```
 
O manualment des de la instància:
 
```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
echo "ssh-rsa AAAA..." >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```
 
**Pas 3 — Verificar l'accés sense contrasenya:**
 
```bash
ssh emilia@10.0.6.239
# Welcome to Ubuntu...  → connexió establerta sense password
```
 
#### Permisos sudo sense contrasenya (requerit per Ansible)
 
Per permetre que Ansible executi tasques privilegiades de forma no interactiva, l'usuari d'administració necessita `NOPASSWD` al sudoers:
 
```bash
sudo visudo
```
 
Afegir al final del fitxer:
 
```
# Permisos d'administració sense contrasenya (necessari per a Ansible)
emilia  ALL=(ALL) NOPASSWD:ALL
```
 
> **Nota de seguretat:** Aquesta configuració s'aplica únicament als usuaris d'administració del projecte, en un entorn de laboratori controlat. En producció real, s'hauria de restringir a les comandes estrictament necessàries.
 
---
 
## 3. Servidor LDAP (OpenLDAP)
 
![OpenLDAP](https://img.shields.io/badge/OpenLDAP-003366?style=for-the-badge&logo=openldap&logoColor=white)
![Ubuntu](https://img.shields.io/badge/Ubuntu-E95420?style=for-the-badge&logo=ubuntu&logoColor=white)
![AWS EC2](https://img.shields.io/badge/AWS_EC2-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white)
 
El servidor `srv-ldap` actua com a directori centralitzat d'usuaris per a tota la infraestructura d'InnovateTech. Tots els serveis (SFTP, autenticació de sistema) resolen els usuaris contra aquest directori.
 
### Paràmetres del servidor
 
| Paràmetre | Valor |
| :--- | :--- |
| **Hostname** | `srv-ldap` |
| **Usuari d'administració** | `emilia` |
| **IP privada** | `10.0.1.224` |
| **Base DN** | `dc=innovatetech,dc=cat` |
| **Admin DN** | `cn=admin,dc=innovatetech,dc=cat` |
| **Domini DNS** | `innovatetech.cat` |
| **Organització** | `Innovate Tech` |
 
---
 
### 3.1. Instal·lació d'OpenLDAP
 
Si existeix una instal·lació prèvia defectuosa, cal eliminar-la completament abans de reinstal·lar:
 
```bash
# Eliminar completament la instal·lació anterior
sudo apt-get purge -y slapd ldap-utils
sudo rm -rf /var/lib/ldap /etc/ldap
sudo apt-get autoremove -y && sudo apt-get autoclean
 
# Instal·lar de nou
sudo apt-get install -y slapd ldap-utils
```
 
> L'ús de `purge` en lloc de `remove` garanteix que s'eliminen els fitxers de configuració residuals, evitant conflictes en la reinstal·lació.
 
#### Configuració interactiva
 
Durant la instal·lació (o amb `sudo dpkg-reconfigure slapd`), introduir els paràmetres següents:
 
| Pregunta | Valor |
| :--- | :--- |
| Omit OpenLDAP server configuration? | No |
| DNS domain name | `innovatetech.cat` |
| Organization name | `Innovate Tech` |
| Administrator password | *(vegeu credencials del projecte)* |
 
---
 
### 3.2. Creació de l'Estructura del Directori
 
L'estructura del directori s'organitza en dues unitats organitzatives (OU): `users` i `groups`.
 
#### `base.ldif` — Unitats Organitzatives
 
```ldif
dn: ou=users,dc=innovatetech,dc=cat
objectClass: organizationalUnit
ou: users
 
dn: ou=groups,dc=innovatetech,dc=cat
objectClass: organizationalUnit
ou: groups
```
 
```bash
ldapadd -x -D "cn=admin,dc=innovatetech,dc=cat" -W -f base.ldif
# adding new entry "ou=users,dc=innovatetech,dc=cat"
# adding new entry "ou=groups,dc=innovatetech,dc=cat"
```
 
#### `groups.ldif` — Grups POSIX
 
```ldif
dn: cn=brenda,ou=groups,dc=innovatetech,dc=cat
objectClass: posixGroup
cn: brenda
gidNumber: 10001
 
dn: cn=emilia,ou=groups,dc=innovatetech,dc=cat
objectClass: posixGroup
cn: emilia
gidNumber: 10002
 
dn: cn=laia,ou=groups,dc=innovatetech,dc=cat
objectClass: posixGroup
cn: laia
gidNumber: 10003
 
dn: cn=mario,ou=groups,dc=innovatetech,dc=cat
objectClass: posixGroup
cn: mario
gidNumber: 10004
```
 
```bash
ldapadd -x -D "cn=admin,dc=innovatetech,dc=cat" -W -f groups.ldif
# adding new entry "cn=brenda,ou=groups,dc=innovatetech,dc=cat"
# adding new entry "cn=emilia,ou=groups,dc=innovatetech,dc=cat"
# adding new entry "cn=laia,ou=groups,dc=innovatetech,dc=cat"
# adding new entry "cn=mario,ou=groups,dc=innovatetech,dc=cat"
```
 
#### Encriptació de contrasenyes
 
Abans de crear els usuaris, es genera el hash SSHA de la contrasenya:
 
```bash
slappasswd
# New password:
# Re-enter new password:
# {SSHA}NBA6g6DGt+Hsh9rRTFODuxQiaR+8YIeU
```
 
#### `users.ldif` — Usuaris POSIX
 
```ldif
dn: uid=brenda,ou=users,dc=innovatetech,dc=cat
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
uid: brenda
sn: Brenda
cn: Brenda
uidNumber: 10001
gidNumber: 10001
homeDirectory: /home/brenda
loginShell: /bin/bash
userPassword: {SSHA}NBA6g6DGt+Hsh9rRTFODuxQiaR+8YIeU
 
dn: uid=emilia,ou=users,dc=innovatetech,dc=cat
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
uid: emilia
sn: Emilia
cn: Emilia
uidNumber: 10002
gidNumber: 10002
homeDirectory: /home/emilia
loginShell: /bin/bash
userPassword: {SSHA}NBA6g6DGt+Hsh9rRTFODuxQiaR+8YIeU
 
dn: uid=laia,ou=users,dc=innovatetech,dc=cat
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
uid: laia
sn: Laia
cn: Laia
uidNumber: 10003
gidNumber: 10003
homeDirectory: /home/laia
loginShell: /bin/bash
userPassword: {SSHA}NBA6g6DGt+Hsh9rRTFODuxQiaR+8YIeU
 
dn: uid=mario,ou=users,dc=innovatetech,dc=cat
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
uid: mario
sn: Mario
cn: Mario
uidNumber: 10004
gidNumber: 10004
homeDirectory: /home/mario
loginShell: /bin/bash
userPassword: {SSHA}NBA6g6DGt+Hsh9rRTFODuxQiaR+8YIeU
```
 
> **Ordre d'importació important:** Primer `base.ldif` (OU pare), després `groups.ldif` i finalment `users.ldif`. Invertir l'ordre provoca errors de referència perquè les entrades pare no existiran al directori.
 
```bash
ldapadd -x -D "cn=admin,dc=innovatetech,dc=cat" -W -f users.ldif
# adding new entry "uid=brenda,ou=users,dc=innovatetech,dc=cat"
# adding new entry "uid=emilia,ou=users,dc=innovatetech,dc=cat"
# adding new entry "uid=laia,ou=users,dc=innovatetech,dc=cat"
# adding new entry "uid=mario,ou=users,dc=innovatetech,dc=cat"
```
 
---
 
### 3.3. Verificació de Logs i Connectivitat
 
```bash
# Verificar que rsyslog escolta al port 514 (reenviarà logs a srv-logs)
sudo ss -tulnp | grep 514
# udp  UNCONN  0.0.0.0:514   0.0.0.0:*  users:(("rsyslogd",pid=755,fd=5))
# tcp  LISTEN  0.0.0.0:514   0.0.0.0:*  users:(("rsyslogd",pid=755,fd=7))
 
# Prova d'enviament de log al servidor centralitzat
logger "TEST AUDITORIA LDAP"
 
# Verificació de la recepció (des de srv-logs)
sudo tail -n 5 /var/log/remote/srv-ldap.log
# 2026-05-26T07:11:23+00:00 srv-ldap emilia: TEST AUDITORIA LDAP
```
 
---
 
## 4. Màquina Web: Nginx + SFTP
 
![Nginx](https://img.shields.io/badge/Nginx-009639?style=for-the-badge&logo=nginx&logoColor=white)
![SFTP](https://img.shields.io/badge/SFTP-4D4D4D?style=for-the-badge&logo=openssh&logoColor=white)
![LDAP](https://img.shields.io/badge/LDAP-0052CC?style=for-the-badge&logo=ldap&logoColor=white)
![AWS EC2](https://img.shields.io/badge/AWS_EC2-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white)
 
La màquina `srv-web-ftps` allotja el servidor web Nginx i el servei SFTP integrat amb autenticació LDAP centralitzada via SSSD.
 
### Paràmetres de la màquina
 
| Paràmetre | Valor |
| :--- | :--- |
| **Hostname** | `srv-web-ftps` |
| **Usuari d'administració** | `brenda` |
| **IP pública** | `98.83.3.235` |
| **IP privada** | `10.0.1.112` |
| **Sistema operatiu** | Ubuntu 24.04 LTS |
 
---
 
### 4.1. Configuració del Servei Web (Nginx)
 
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install nginx -y
sudo systemctl enable --now nginx
```
 
#### Verificació de l'estat del servei
 
```bash
sudo systemctl status nginx
# ● nginx.service - A high performance web server and a reverse proxy server
#      Active: active (running) since Thu 2026-05-21 07:17:50 UTC
```
 
Accedint a `http://98.83.3.235` es confirma que el servidor web respon correctament.
 
---
 
### 4.2. Integració LDAP amb SSSD
 
Per autenticar els usuaris LDAP al sistema operatiu, s'instal·la i configura el dimoni SSSD.
 
#### Paquets necessaris
 
| Paquet | Funció |
| :--- | :--- |
| `sssd` + `sssd-ldap` | Resolució d'identitats contra LDAP |
| `libpam-sss` + `libnss-sss` | Integració amb PAM i NSS |
| `ldap-utils` | Eines de diagnòstic LDAP |
 
```bash
sudo apt install -y sssd sssd-ldap libpam-sss libnss-sss ldap-utils openssh-server
```
 
#### Fitxer `/etc/sssd/sssd.conf`
 
```ini
[sssd]
services = nss, pam
config_file_version = 2
domains = innovatetech.cat
 
[domain/innovatetech.cat]
id_provider = ldap
auth_provider = ldap
ldap_uri = ldap://10.0.1.224
ldap_search_base = dc=innovatetech,dc=cat
ldap_user_search_base  = ou=users,dc=innovatetech,dc=cat
ldap_group_search_base = ou=groups,dc=innovatetech,dc=cat
ldap_default_bind_dn  = cn=admin,dc=innovatetech,dc=cat
ldap_default_authtok  = <CONTRASENYA_ADMIN>
ldap_schema = rfc2307
ldap_tls_reqcert = never
override_homedir = /home/%u
fallback_homedir = /home/%u
default_shell = /bin/bash
cache_credentials = true
ldap_id_use_start_tls = false
ldap_user_object_class   = posixAccount
ldap_user_name           = uid
ldap_user_uid_number     = uidNumber
ldap_user_gid_number     = gidNumber
ldap_user_home_directory = homeDirectory
ldap_user_shell          = loginShell
ldap_user_password       = userPassword
ldap_auth_disable_tls_never_use_in_production = true
ldap_pwd_policy = none
```
 
```bash
# Permisos obligatoris perquè SSSD arrenqui
sudo chmod 600 /etc/sssd/sssd.conf
sudo chown root:root /etc/sssd/sssd.conf
sudo systemctl restart sssd
```
 
#### Verificació de la resolució d'usuaris
 
```bash
getent passwd emilia
# emilia:*:10002:10002:Emilia:/home/emilia:/bin/bash
 
getent passwd brenda
# brenda:*:10001:10001:Brenda:/home/brenda:/bin/bash
```
 
---
 
### 4.3. Configuració del Servei SFTP amb Chroot
 
Es restringeix el grup `sftp-users` a accés exclusiu SFTP dins del seu directori personal.
 
#### Modificació de `/etc/ssh/sshd_config`
 
```
PasswordAuthentication yes
KbdInteractiveAuthentication yes
UsePAM yes
Subsystem sftp internal-sftp
 
Match Group sftp-users
    ChrootDirectory /home/%u
    ForceCommand internal-sftp
    AllowTcpForwarding no
    X11Forwarding no
```
 
#### Creació de directoris i assignació de permisos
 
El directori arrel del chroot **ha de ser propietat de `root:root`** (requisit de seguretat del dimoni SSH). L'usuari té accés d'escriptura únicament al subdirectori `files`.
 
```bash
for user in brenda emilia laia mario; do
  sudo mkdir -p /home/${user}/files
  sudo chown root:root /home/${user}
  sudo chmod 755 /home/${user}
  sudo chown ${user}:${user} /home/${user}/files
  sudo chmod 700 /home/${user}/files
  sudo usermod -aG sftp-users ${user}
done
```
 
#### Verificació
 
```bash
getent group sftp-users
# sftp-users:x:1002:brenda,emilia,laia,mario
 
sftp brenda@localhost
# Connected to localhost.
# sftp> ls
# files
```
 
---
 
## 5. Servidor de Logs: Rsyslog + ELK Stack
 
![Ansible](https://img.shields.io/badge/Ansible-EE0000?style=for-the-badge&logo=ansible&logoColor=white)
![Elastic](https://img.shields.io/badge/Elastic-005571?style=for-the-badge&logo=elastic&logoColor=white)
![Kibana](https://img.shields.io/badge/Kibana-005571?style=for-the-badge&logo=kibana&logoColor=white)
![Ubuntu](https://img.shields.io/badge/Ubuntu-E95420?style=for-the-badge&logo=ubuntu&logoColor=white)
![AWS EC2](https://img.shields.io/badge/AWS_EC2-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white)
 
El servidor `srv-logs` centralitza els logs de tota la infraestructura mitjançant Rsyslog (recepció) i l'ELK Stack (indexació i visualització). Tota la configuració s'automatitza amb **Ansible**.
 
> La creació de l'usuari d'administració i la configuració d'accés per clau pública d'aquesta màquina es descriu a la [secció §2.1](#21-prerequisit-global-accés-i-administració-de-màquines), que és comuna a totes les instàncies.
 
### Paràmetres del servidor
 
| Paràmetre | Valor |
| :--- | :--- |
| **Hostname** | `srv-logs` |
| **Usuari d'administració** | `emilia` |
| **IP privada** | `10.0.6.239` |
| **IP pública** | `52.4.133.45` |
| **Port Rsyslog** | `514` (UDP + TCP) |
| **Port Kibana** | `5601` |
| **Port Elasticsearch** | `9200` |
| **Versió Ansible** | `2.20.1` |
 
---
 
### 5.1. Instal·lació i Estructura d'Ansible
 
```bash
sudo apt update && sudo apt install ansible -y
ansible --version
# ansible [core 2.20.1]
 
# Crear l'estructura de treball
mkdir ~/ansible && cd ~/ansible
```
 
Contingut de `inventory.ini`:
 
```ini
[logs]
localhost ansible_connection=local
```
 
> Ansible s'administra a si mateix: el servidor de logs és alhora el controlador Ansible i el node gestionat.
 
---
 
### 5.2. Playbook Rsyslog: Recepció Centralitzada de Logs
 
El playbook `rsyslog.yml` configura Rsyslog per acceptar logs de totes les màquines de la xarxa via UDP i TCP al port 514.
 
```yaml
---
- hosts: logs
  become: yes
 
  tasks:
    - name: Instal·lar rsyslog
      apt:
        name: rsyslog
        state: present
        update_cache: yes
 
    - name: Habilitar recepció UDP
      lineinfile:
        path: /etc/rsyslog.conf
        regexp: '^#module\(load="imudp"\)'
        line: 'module(load="imudp")'
 
    - name: Obrir port UDP 514
      lineinfile:
        path: /etc/rsyslog.conf
        regexp: '^#input\(type="imudp" port="514"\)'
        line: 'input(type="imudp" port="514")'
 
    - name: Habilitar recepció TCP
      lineinfile:
        path: /etc/rsyslog.conf
        regexp: '^#module\(load="imtcp"\)'
        line: 'module(load="imtcp")'
 
    - name: Obrir port TCP 514
      lineinfile:
        path: /etc/rsyslog.conf
        regexp: '^#input\(type="imtcp" port="514"\)'
        line: 'input(type="imtcp" port="514")'
      notify: Reiniciar rsyslog
 
  handlers:
    - name: Reiniciar rsyslog
      service:
        name: rsyslog
        state: restarted
        enabled: yes
```
 
#### Execució i idempotència
 
```bash
# Primera execució: aplica tots els canvis
ansible-playbook -i inventory.ini rsyslog.yml
# localhost : ok=7  changed=5  unreachable=0  failed=0
 
# Execucions posteriors: idempotent, sense canvis
ansible-playbook -i inventory.ini rsyslog.yml
# localhost : ok=7  changed=0  unreachable=0  failed=0
```
 
#### Verificació del port 514
 
```bash
sudo ss -tulnp | grep 514
# udp  UNCONN  0.0.0.0:514  ...  users:(("rsyslogd",pid=3104,fd=5))
# tcp  LISTEN  0.0.0.0:514  ...  users:(("rsyslogd",pid=3104,fd=7))
```
 
---
 
### 5.3. Configuració de les Màquines Client
 
A cada màquina de la infraestructura s'afegeix una línia al fitxer de Rsyslog per reenviar tots els logs cap a `srv-logs`:
 
```bash
echo "*.* @@10.0.6.239" | sudo tee -a /etc/rsyslog.d/50-default.conf
sudo systemctl restart rsyslog
```
 
> `@@` indica TCP (connexió fiable). En entorns de producció on la pèrdua de logs no és acceptable, sempre s'ha d'usar TCP en lloc d'UDP (`@`).
 
#### Regles del Security Group AWS
 
| Regla | Protocol | Port | Origen |
| :--- | :--- | :--- | :--- |
| Rsyslog UDP | UDP | 514 | `10.0.0.0/16` |
| Rsyslog TCP | TCP | 514 | `10.0.0.0/16` |
 
---
 
### 5.4. Separació de Logs per Hostname
 
Per facilitar l'auditoria, cada màquina escriu els seus logs en un fitxer independent:
 
```bash
sudo mkdir /var/log/remote
sudo nano /etc/rsyslog.d/remote.conf
```
 
Contingut de `remote.conf`:
 
```
$template RemoteLogs,"/var/log/remote/%HOSTNAME%.log"
*.* ?RemoteLogs
& stop
```
 
```bash
sudo chmod 755 /var/log/remote
sudo chown syslog:adm /var/log/remote
sudo systemctl restart rsyslog
```
 
#### Verificació de la recepció per màquina
 
```bash
ls /var/log/remote
# srv-ldap.log  srv-logs.log
 
sudo tail -f /var/log/remote/srv-ldap.log
# 2026-05-26T07:11:23+00:00 srv-ldap emilia: TEST AUDITORIA LDAP
```
 
---
 
### 5.5. Playbook ELK Stack: Elasticsearch + Kibana + Filebeat
 
El playbook `elk.yml` desplega automàticament l'stack complet de visualització de logs.
 
```yaml
---
- name: Configurar Servidor Central de Logs (Rsyslog + ELK Stack)
  hosts: localhost
  become: yes
  tasks:
 
  # ── 1. PREREQUISITS I REPOSITORIS ──────────────────────────────────────────
    - name: Instal·lar dependències bàsiques
      apt:
        name: [apt-transport-https, gnupg2, curl]
        state: present
        update_cache: yes
 
    - name: Eliminar clau GPG anterior si existeix
      file:
        path: /usr/share/keyrings/elasticsearch-keyring.gpg
        state: absent
 
    - name: Descarregar i desencriptar la clau GPG d'Elastic
      shell: |
        curl -fsSL https://artifacts.elastic.co/GPG-KEY-elasticsearch \
          | gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg
      args:
        creates: /usr/share/keyrings/elasticsearch-keyring.gpg
 
    - name: Assegurar permisos correctes a la clau GPG
      file:
        path: /usr/share/keyrings/elasticsearch-keyring.gpg
        mode: '0644'
        owner: root
        group: root
 
    - name: Afegir el repositori oficial d'Elastic 7.x
      apt_repository:
        repo: deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] https://artifacts.elastic.co/packages/7.x/apt stable main
        state: present
        filename: elastic-7.x
 
  # ── 2. CONFIGURACIÓ DE RSYSLOG ──────────────────────────────────────────────
    - name: Assegurar el directori de logs remots amb permisos correctes
      file:
        path: /var/log/remote
        state: directory
        owner: syslog
        group: adm
        mode: '0755'
 
    - name: Habilitar recepció UDP a Rsyslog
      lineinfile:
        path: /etc/rsyslog.conf
        regexp: '^#module\(load="imudp"\)'
        line: 'module(load="imudp")'
      notify: Reiniciar rsyslog
 
    - name: Habilitar recepció TCP 514 a Rsyslog
      lineinfile:
        path: /etc/rsyslog.conf
        regexp: '^#input\(type="imtcp" port="514"\)'
        line: 'input(type="imtcp" port="514")'
      notify: Reiniciar rsyslog
 
    - name: Crear la plantilla de separació de logs per hostname
      copy:
        dest: /etc/rsyslog.d/remote.conf
        content: |
          $template RemoteLogs,"/var/log/remote/%HOSTNAME%.log"
          *.* ?RemoteLogs
          & stop
      notify: Reiniciar rsyslog
 
  # ── 3. ELASTICSEARCH ────────────────────────────────────────────────────────
    - name: Instal·lar Elasticsearch
      apt:
        name: elasticsearch
        state: present
 
    - name: Configurar Elasticsearch per escoltar localment
      lineinfile:
        path: /etc/elasticsearch/elasticsearch.yml
        regexp: '^#?network.host:'
        line: 'network.host: 127.0.0.1'
      notify: Reiniciar elasticsearch
 
    - name: Assegurar que Elasticsearch està actiu i s'inicia amb el sistema
      service:
        name: elasticsearch
        state: started
        enabled: yes
 
  # ── 4. KIBANA ───────────────────────────────────────────────────────────────
    - name: Instal·lar Kibana
      apt:
        name: kibana
        state: present
 
    - name: Configurar Kibana per ser accessible externament
      lineinfile:
        path: /etc/kibana/kibana.yml
        regexp: '^#?server.host:'
        line: 'server.host: "0.0.0.0"'
      notify: Reiniciar kibana
 
    - name: Connectar Kibana amb Elasticsearch local
      lineinfile:
        path: /etc/kibana/kibana.yml
        regexp: '^#?elasticsearch.hosts:'
        line: 'elasticsearch.hosts: ["http://localhost:9200"]'
      notify: Reiniciar kibana
 
    - name: Assegurar que Kibana està actiu i s'inicia amb el sistema
      service:
        name: kibana
        state: started
        enabled: yes
 
  # ── 5. FILEBEAT ─────────────────────────────────────────────────────────────
    - name: Instal·lar Filebeat
      apt:
        name: filebeat
        state: present
 
    - name: Configurar Filebeat per llegir /var/log/remote/
      copy:
        dest: /etc/filebeat/filebeat.yml
        content: |
          filebeat.inputs:
          - type: log
            enabled: true
            paths:
              - /var/log/remote/*.log
 
          output.elasticsearch:
            hosts: ["localhost:9200"]
      notify: Reiniciar filebeat
 
    - name: Assegurar que Filebeat està actiu i s'inicia amb el sistema
      service:
        name: filebeat
        state: started
        enabled: yes
 
  # ── HANDLERS ────────────────────────────────────────────────────────────────
  handlers:
    - name: Reiniciar rsyslog
      service: { name: rsyslog, state: restarted }
    - name: Reiniciar elasticsearch
      service: { name: elasticsearch, state: restarted }
    - name: Reiniciar kibana
      service: { name: kibana, state: restarted }
    - name: Reiniciar filebeat
      service: { name: filebeat, state: restarted }
```
 
#### Execució del playbook
 
```bash
ansible-playbook -i inventory.ini elk.yml
# PLAY RECAP *******************************************************************
# localhost : ok=24  changed=11  unreachable=0  failed=0  skipped=0  rescued=0
```
 
---
 
### 5.6. Verificació a Kibana
 
Un cop desplegat l'ELK Stack, s'accedeix a Kibana i es crea un index pattern per visualitzar els logs:
 
| Paràmetre | Valor |
| :--- | :--- |
| **URL Kibana** | `http://52.4.133.45:5601` |
| **Index pattern** | `filebeat*` |
| **Timestamp field** | `@timestamp` |
| **Índexos detectats** | `filebeat-7.17.29`, `filebeat-7.17.29-2026.05.28-000001` |
| **Registres indexats** | **26.328 hits** |
 
> Kibana detecta correctament els dos índexos de Filebeat i mostra els 26.328 registres de logs de tota la infraestructura en temps real, identificats per `agent.hostname: srv-logs`.
 
---

## 🗺️ Apartat 6: Disseny del Model Relacional (16 Taules)

L'esquema de la base de dades s'estructura de forma homogènia en **català**, consolidant un total de **16 taules** operatives. Aquestes taules estan dividides lògicament en diversos mòduls funcionals per facilitar-ne el manteniment i l'escalabilitat.

### 🏢 6.1. Gestió de Personal i Recursos Humans
Aquest mòdul centralitza l'estructura organitzativa de l'empresa, les dades dels treballadors i la gestió financera de les nòmines.

| Taula | Clau Primària (PK) | Claus Foranes (FK) | Columnes i Restriccions Addicionals |
| :--- | :--- | :--- | :--- |
| **`departaments`** | `codi_dept` | - | `nom_compleat`, `telefon` |
| **`grups_nivells`** | `id_grup_nivell` | - | `nom_grup`, `descripcio` |
| **`empleats`** | `dni` | `codi_dept`, `id_grup_nivell` | `nom`, `cognoms`, `adreca`, `telefon` |
| **`nomines`** | `id_nomina` | `dni_empleat` | `mes`, `any`, `salari_base`, `deduccions`, `total_net` |

### 💬 6.2. Sistema de Comunicació, Usuaris i QoS
Gestió dels usuaris de la plataforma (tant interns com externs) i registre de la qualitat de les trucades.

| Taula | Clau Primària (PK) | Claus Foranes (FK) | Columnes i Restriccions Addicionals |
| :--- | :--- | :--- | :--- |
| **`usuaris_sistema`** | `id_usuari` | `dni_empleat` *(Buit per a clients)* | `nom_complet`, `email` (UNIQUE), `extensio_trucades` (UNIQUE), `rol` (ENUM: 'client', 'treballador'), `estat` (ENUM: 'actiu', 'bloquejat'), `enllaç_videotrucada` |
| **`qualitats`** | `id_calitat` | - | `nom_perfil` (ENUM: 'alta', 'mitja', 'baixa'), `max_amplada_banda`, `ports_protocols` |
| **`registre_trucades`**| `id_trucada` | `usuari_origen`, `usuari_desti`, `id_calitat_usada` | `data_hora_inici`, `data_hora_fi`, `durada_segons`, `puntuacio_valoracio` (CHECK), `comentari_valoracio` |

### 🎬 6.3. Streaming i Catàleg de Continguts
Administració del repositori de vídeos i recursos multimèdia de la plataforma.

| Taula | Clau Primària (PK) | Claus Foranes (FK) | Columnes i Restriccions Addicionals |
| :--- | :--- | :--- | :--- |
| **`cataleg_videos`** | `id_video` | - | `titol`, `descripcio`, `categoria`, `durada`, `data_publicacio`, `url_streaming` |

### 📡 6.4. Operacions, Xarxa i Amplada de Banda
Monitoratge tècnic del rendiment de la infraestructura per assegurar l'estabilitat del streaming.

| Taula | Clau Primària (PK) | Claus Foranes (FK) | Columnes i Restriccions Addicionals |
| :--- | :--- | :--- | :--- |
| **`mesures_ampla_banda`**| `id_mesura` | `dni_operari` | `data_hora`, `equip_mesurat`, `velocitat_baixada` (DEC), `velocitat_pujada` (DEC), `latencia` (DEC), `resultat` (ENUM: 'acceptable', 'no acceptable'), `observacions` |

### 🛒 6.5. Operacions Comercials i Vendes
Mòdul encarregat de la gestió de clients empresarials, estoc de productes i facturació.

| Taula | Clau Primària (PK) | Claus Foranes (FK) | Columnes i Restriccions Addicionals |
| :--- | :--- | :--- | :--- |
| **`clients`** | `id_client` | - | `nom_empresa`, `cif`, `telefon_contacte` |
| **`productes`** | `id_producte` | - | `nom_producte`, `preu` (DECIMAL), `estoc` |
| **`comandes`** | `id_comanda` | `id_client` | `data_comanda`, `estat_pagament` |
| **`cistell`** | `id_cistell` | `id_client`, `id_producte`| `quantitat` |

### 🛡️ 6.6. Seguretat, Auditories i Respatller (Mòdul 0377)
Garanteix la traçabilitat de les operacions i l'estat de les còpies de seguretat del SGBD.

| Taula | Clau Primària (PK) | Claus Foranes (FK) | Columnes i Restriccions Addicionals |
| :--- | :--- | :--- | :--- |
| **`taula_avisos`** | `id_avis` | - | `usuari_base_dades`, `taula_afectada`, `operacio_intentada`, `data_hora` |
| **`registre_backups`** | `id_backup` | - | `data_hora`, `taules_incloses`, `resultat` |
| **`logs_auditorias`** | `id_log` | - | `usuari_sistema`, `accio_realitzada`, `detalls`, `data_hora` |

---

> [!TIP]
> **Actualització del Diagrama (Draw.io):** S'han eliminat les taules obsoletes de l'esborrany inicial, garantint que el disseny gràfic de relacions es correspongui de manera estricta i fidel amb aquest esquema físic definitiu de 16 entitats.

---

### 🔄 Actualització del Diagrama (Draw.io / Lucidchart)
Per garantir la correspondència exacta amb el model físic final de 16 entitats, s'han dut a terme les següents accions:
* Eliminació exhaustiva de taules obsoletes de l'esborrany inicial.
* Canvi complet de noms de l'esquema a l'idioma **català**.
* Creació de les taules estructurals: `grups_nivells`, `qualitats` i `registre_trucades`.
* Disseny i enllaç de claus estrangeres (FK) crítiques (com la columna `grup_nivell`).

**Diagrama Entitat-Relació (Model Físic):**
<div align="center">
  <img src="bbdd/PROYECTO_TRANSVERSAL.png" alt="Diagrama ER InnovateTech" width="1000"/>
</div>

---

### ☁️ Justificació de la Infraestructura (EC2 vs RDS)
S'ha optat per desplegar el SGBD MariaDB directament sobre una instància **Amazon EC2** en comptes d'utilitzar Amazon RDS pels següents motius:
1. **Control Absolut del Sistema:** L'ús d'una instància EC2 ens permet gestionar directament el sistema operatiu (Ubuntu), gestionar de forma nativa fitxers de configuració del motor i utilitzar eines de tasques programades del sistema com `cron`.
2. **Optimització de Costos:** Amazon RDS presenta uns costos de manteniment, emmagatzematge i instància molt elevats que no s'ajusten a la política econòmica de contenció de despeses d'InnovateTech per a aquest entorn.

---

### 📅 Justificació de la Freqüència de Còpies de Seguretat
S'ha programat una tasca síncrona mitjançant el dimoni `cron` de Linux que s'executa de forma automatitzada **cada dia a les 23:00 hores** (`0 23 * * *`). 
* **Motiu:** Es tria aquesta franja horària nocturna perquè coincideix amb el moment de menor activitat laboral i càrrega transaccional de l'empresa. Això garanteix que la sentència de backup no bloquegi les taules crítiques (`empleats`, `clients`, `comandes` i `registre_trucades`) durant l'operativa diària de l'empresa, assegurant un punt de recuperació òptim sense pèrdua de dades rellevants.

---

## ⚙️ Desplegament i Preparació de l'Entorn

Per posar en marxa l'entorn de la base de dades, cal preparar el servidor instal·lant el sistema gestor. Executa el següent bloc de comandes en un entorn basat en Debian/Ubuntu:

```bash
# 1. Actualitzar els repositoris i paquets del sistema
sudo apt update && sudo apt upgrade -y

# 2. Instal·lar el servidor MariaDB
sudo apt install mariadb-server -y

# 3. Comprovar l'estat del servei per verificar la correcta instal·lació
sudo systemctl status mariadb

chmod 600 ~/.ssh/authorized_keys
