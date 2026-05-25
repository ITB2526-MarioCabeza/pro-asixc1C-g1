# pro-asixc1C-g1
Grup format per Laia Coca, Emilia Tikohonova, Brenda Castro i Mario Cabeza.

BASES DE DATOS

#### 1. Gestió de Personal

* departaments: codi_dept (PK), nom_compleat, telefon.
* empleats: dni (PK), nom, cognoms, adreca, telefon, codi_dept (FK).

#### 2. Sistema de Comunicació i Usuaris

* usuaris_sistema: id_usuari (PK), dni_empleat (FK, per a treballadors interns), nom_complet, email (UNIQUE), extensio_trucades, rol (ENUM: 'client', 'treballador'), estat (ENUM: 'actiu', 'bloquejat'), enllaç_videotrucada.
* configuracion_qualitat: id_config (PK), nom_grup (alta, mitja, baixa), max_amplada_banda, parametres_servidor (ports, protocols).
* registre_trucades: id_trucada (PK), usuari_origen (FK), usuari_desti (FK), data_hora_inici, data_hora_fi, durada_segons, id_config_usada (FK), puntuacio_valoracio, comentari_valoracio.

#### 3. Streaming i Catàleg

* cataleg_videos: id_video (PK), titol, descripcio, categoria, durada, data_publicacio, url_streaming.

#### 4. Operacions i Amplada de Banda (Molt important per a ASIX)

* mesures_amplada_banda: id_mesura (PK), data_hora, equip_mesurat, velocitat_baixada, velocitat_pujada, latencia, resultat (ENUM: 'acceptable', 'no acceptable'), dni_operari (FK a empleats), observacions.

#### 5. Seguretat i Auditoria (Mòdul 0377)

* taula_avisos (Log d'auditoria): id_avis (PK), usuari_base_dades, taula_afectada, operacio_intentada (INSERT, UPDATE, etc.), data_hora.
* control_backups: id_backup (PK), data_hora, taules_incloses, resultat.

DIAGRAMA DRAW.IO

TABLAS CON SUS RELACIONES CORRESPONDIENTES


GITHUB

BASE DE DATOS

Actualización sistema + instalación base de datos mariadb.

Para ver si el servicio mariadb se ha iniciado correctamente.

Primer paso para crear la base de datos innovatetech_db.

Creación de la tabla Departamentos

Creación de la tabla Empleados.

ON DELETE RESTRICT: Es una regla de seguridad. Si intentas borrar un departamento que todavía tiene empleados dentro trabajando, la base de datos te dará un error y lo impedirá para no dejar trabajadores "huérfanos".

Creación tabla de configuración de calidad de servicio (QoS)

Creación tabla de usuarios del sistema de comunicación.

configuracion_calidad: Aquí se guardan los parámetros técnicos que te pide el enunciado (puertos, protocolos y perfiles). Al usar un ENUM('alta', 'mitja', 'baixa'), obligamos a que nadie pueda inventarse un perfil raro.

El "misterio" de los clientes externos (dni_empleat NULL): El enunciado dice que el sistema lo usan tanto trabajadores internos como clientes externos. Si el usuario es un cliente, no tiene DNI en nuestra tabla de recursos humanos. Por eso, permitimos que dni_empleat sea NULL (vacío). Si se borra un empleado, su usuario de videollamadas no se borra, simplemente se queda en NULL (DELETE SET NULL).

Restricciones de seguridad (UNIQUE): Hemos blindado el correo (uq_email_usuario) y la extensión telefónica (uq_extension). Así el sistema impedirá que dos usuarios tengan la misma extensión o el mismo correo, evitando colisiones en la red de telefonía VoIP.

Tabla de catálogo de vídeos en streaming.

Tabla de registros de llamadas (Historial Completo)

Tabla de medidas de ancho de banda (Auditoría de red).

Control de errores con CONSTRAINT CHECK: En la tabla de llamadas hemos blindado los datos con reglas físicas. Por ejemplo, chk_puntuacio asegura que las notas del servicio solo puedan ser del 1 al 5. Si la app web intentara meter un "6" o un "0", la base de datos la frenará en seco de forma nativa.

Tipos de datos exactos (DECIMAL): Para medir las velocidades de bajada, subida y latencia de red, usamos DECIMAL(6,2). Los tipos FLOAT o DOUBLE pueden perder precisión decimal con operaciones matemáticas. Con DECIMAL, nos aseguramos de que si un test da 300.50 Mbps, se guarde exactamente ese valor sin redondeos raros.

Vínculo del Operario: La tabla de medidas requiere un dni_operari que apunta directamente a la tabla de empleados. Esto cumple el requisito de saber qué técnico de soporte ha realizado el test de velocidad a ese usuario.

Tabla de auditoría: Tabla de avisos (Exigida en el apartado 3.3.3)

Tabla de ventas (Mencionadas en el rol de ventas y backups).

Creación tablas productes, comandes y cistell.

Creación tablas de administración (Mencionadas en el rol de administració)

Estrategia de defensa: Si el profesor te pregunta "¿Por qué has creado tablas de compras si el proyecto habla de videollamadas?", tu respuesta impecable será: "Porque en el apartado 3.3.1 se especifica que el rol de ventas debe gestionar clientes, pedidos, productos y cestas, y en el 3.3.4 se pide incluirlas en el backup diario. Para garantizar la integridad referencial del SGBD, debían coexistir en el mismo esquema".

La regla DELETE CASCADE en la cesta (cistell): Si un cliente vacía su cuenta o se elimina un producto del catálogo, la cesta de la compra asociada se borra automáticamente. En cambio, en las facturas/pedidos (comandes), usamos RESTRICT para evitar borrar datos históricos de contabilidad por error.

Insertar en la tabla Departamentos (Requisito 3.1)

Insertar en la tabla Empleados (RRHH)

Insertar Perfiles de calidad de Servicio (QoS)

Insertar usuarios del sistema (Empleados y clientes externos)

Insertar catálogo de videos en streaming.

Insertar clientes, productos y nóminas (para soportar roles).

Demostración de la tabla Departamentos y Usuarios_sistema.

Creación roles del sistema (Apartado 3.3.1)

Asignar permisos al rol Admin.

Asignar permisos al rol Vendes.

Permisos SELECT, INSERT, UPDATE en clientes, comandos, productos, cesta y llamadas.

Asignar permisos al rol Administració

Permisos SELECT, INSERT, UPDATE en personal, nóminas, departamentos y grupos.

Asignar permisos al rol Treballador

Permisos SELECT, INSERT, UPDATE en productos, vídeos, configuraciones.

Permisos especiales para backups (Apartado 3.3.2)

GRANT FILE para poder hacer backups con ficheros

Se asigna al admin al ser un permiso global del servidor.

El principio de mínimo privilegio: Fíjate en el rol treballador. El enunciado dice que "puede registrar su actividad de llamadas", por lo que le hemos dado SELECT e INSERT en registre_trucades. Sin embargo, no tiene permiso de UPDATE ni DELETE, lo que significa que un trabajador puede registrar que ha hecho una llamada, pero nunca podrá borrarla ni modificar la duración para engañar a la empresa.

Restricción implícita: No hace falta escribir código para "prohibir" cosas. En las bases de datos, lo que no se permite explícitamente está prohibido por defecto. Al no darle al rol administracio ningún permiso sobre registre_trucades, el propio motor de MariaDB le denegará el acceso de forma automática.

El privilegio FILE: Es un permiso crítico a nivel de sistema operativo que permite a MariaDB leer y escribir archivos directamente en el disco duro de tu máquina AWS EC2 (necesario para las copias de seguridad automáticas que haremos más adelante). Por seguridad, solo se lo otorgamos al rol admin.

SCRIPT DE AUTOMATIZACIÓN EN BASH

Permisos y comprobación del archivo creado.

Intento de crear un usuarios que no existe

Al intentar crear un usuario que ya existe el sistema directamente hace que salte un error para no poder seguir adelante.

Trigger para controlar el bloqueo de usuarios

Si el usuario está bloqueado impide que inserte llamadas.

Trigger para controlar el número máximo de llamadas diarias

Límite máximo de 50 llamadas por usuario al día. Si este se pasa se bloquea.

Trigger para limitar el máximo de tiempo por usuario a 1000 minutos mensuales.

CREACIÓN USUARIO ALTERNATIVO (mario.cabeza.7e9)

Añadimos permisos sudo al usuario mario.cabeza.7e9

Trigger de vigilancia que bloquea si “treballador” o “vendes” intentan modificar salarios.

Trigger de vigilancia sobre el sistema de llamadas que bloquea si el personal “administració” intenta interactuar con llamadas.

Permisos + programar tarea cron para que todos los días se ejecute este script exactamente a las 23:00h.

Taula para registrar los eventos de backups realizados.

Comprobación del fichero de backup funcionando.

Para ver que todo se ha creado correctamente.

Creación de un nuevo hostname

Vemos como después de hacer un sudo reboot la máquina me cambia automáticamente el hostname al que yo he creado anteriormente.

PERMITIR EL ACCESO PONIENDO LA CLAVE (AUTENTIFICACIÓN)

Para que se tenga que entrar por contraseña obligatoriamente.

Ahora vemos cómo en vez de entrar directamente con este usuario te pide la contraseña como método de defensa.

Comprobación para ver que todo lo que he creado anteriormente de tablas, usuarios y permisos se ha guardado dentro de la base de datos MariaDB y que también se puede ver desde el nuevo usuario creado mario.cabeza.7e9.

Lo que he hecho aquí es copiar los dos archivos que creé desde el usuario ubuntu con el comando nano a mi nuevo usuario mario.cabeza.7e9 para no tener que volver a crearlos. Ya que como esto se crea directamente desde el usuario y no desde la base de datos, estos no se guardan al entrar en el nuevo hostname.

Cambio de permisos para que el nuevo usuario pueda tener completo uso de estos archivos creados. 
