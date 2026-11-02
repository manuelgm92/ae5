# AE5 - Transferencia segura

# Índice

- [dpl-lab](#dpl-lab)
  - [Paso 1 - Usuarios y grupo](#paso-1---usuarios-y-grupo)
  - [Paso 2 - Directorio de despliegue](#paso-2---directorio-de-despliegue)
  - [Paso 3 - Comprobación de usuarios y permisos](#paso-3---comprobación-de-usuarios-y-permisos)
  - [Paso 4 - Configurar vsftpd](#paso-4---configurar-vsftpd)
  - [Paso 5 - Transferencia cifrada con FTPS](#paso-5---transferencia-cifrada-con-ftps)
  - [Paso 6 - Instalar y lanzar OpenSSH](#paso-6---instalar-y-lanzar-openssh)
  - [Paso 7 - Endurecer el servicio](#paso-7---endurecer-el-servicio)
- [dpl-cliente](#dpl-cliente)
  - [Paso 8 - Comprobar endurecimiento](#paso-8---comprobar-endurecimiento)
  - [Paso 9 - Subida del build de Vite con despliegue](#paso-9---subida-del-build-de-vite-con-despliegue)
  - [Paso 10 - Descarga con lector y escritura con lector](#paso-10---descarga-con-lector-y-escritura-con-lector)
  - [Paso 11 - Anónimo debe fallar](#paso-11---anónimo-debe-fallar)
  - [Paso 12 - Modos activo y pasivo](#paso-12---modos-activo-y-pasivo)
  - [Paso 13 - FTPS explícito](#paso-13---ftps-expl%C3%ADcito)
  - [Paso 14 - Claves y transferencia por SSH](#paso-14---claves-y-transferencia-por-ssh)
- [Verificables](#verificables)
  - [1. Servicios activos en dpl-lab](#1-servicios-activos-en-dpl-lab)
  - [2. Configuración y usuarios](#2-configuración-y-usuarios)
  - [3. Transferencias reales desde dpl-cliente (activo y pasivo)](#3-transferencias-reales-desde-dpl-cliente-activo-y-pasivo)
  - [4. Permisos](#4-permisos)
  - [5. Transferencia cifrada](#5-transferencia-cifrada)
  - [6. Comprobación rápida con curl](#6-comprobación-rápida-con-curl)
- [comprobaciones.txt](#comprobacionestxt)

En este proyecto, se trabaja en dos contenedores, `dpl-lab` (servidor) y `dpl-cliente` (cliente) en la red `dpl-net`.


# dpl-lab

Dentro del contenedor `dpl-lab`:

## Paso 1 - Usuarios y grupo

Crear usuarios: `despliegue` (puede subir y bajar) y `lector` (solo descargar), y un grupo `operaciones` al que pertenezcan ambos.

* Grupo `operaciones`:

```bash
groupadd operaciones
```

* Crear los usuarios `despliegue` y `lector`, ambos pertenecientes al grupo `operaciones`:
```bash
useradd -m -s /bin/bash -G operaciones despliegue
useradd -m -s /bin/bash -G operaciones lector
echo 'publicador:CAMBIAR_ESTO' | chpasswd 
echo 'consulta:CAMBIAR_ESTO' | chpasswd
```

## Paso 2 - Directorio de despliegue

Crear directorio de despliegue `/srv/dpl/web` donde solo `despliegue` pueda escribir y `lector` pueda leer (permisos `750`, propietario `despliegue`, grupo `operaciones`).

* Crear `/srv/dpl/web`:
```bash
mkdir -p /srv/dpl/web
```

* Establecer a `despliegue` como propietario:
```bash
chown despliegue:operaciones /srv/dpl/web
```

* Establecer permisos 750 al directorio `/srv/dpl/web` donde solo `despliegue` puede escribir y `lector` puede leer.
```bash
chmod 750 /srv/dpl/web
ls -ld /srv/dpl/web
```

## Paso 3 - Comprobación de usuarios y permisos

```bash
grep -E 'despliegue|lector' /etc/passwd
groups despliegue lector
ls -ld /srv/dpl/web
```

## Paso 4 - Configurar vsftpd 

Dentro de `/etc/vsftpd.conf`, edita las siguientes líneas:

* anonymous_enable=NO 
* write_enable=YES
* local_enable=YES

Y relanzar el servicio, `docker exec -d dpl-lab /usr/sbin/vsftpd /etc/vsftpd.conf`


## Paso 5 - Transferencia cifrada con FTPS

Genera un certificado dentro de `dpl-lab`:
```bash
docker exec dpl-lab bash -c "apt-get install -y openssl && openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/ssl/private/vsftpd.key -out /etc/ssl/certs/vsftpd.crt -subj '/CN=dpl.local'"
```

Añade a `/etc/vsftpd.conf` las directivas SSL y relanza el servicio:

```conf
ssl_enable=YES
allow_anon_ssl=NO
force_local_data_ssl=YES
force_local_logins_ssl=YES
require_ssl_reuse=NO
rsa_cert_file=/etc/ssl/certs/vsftpd.crt
rsa_private_key_file=/etc/ssl/private/vsftpd.key
```
```bash
docker exec dpl-lab pkill vsftpd || true
docker exec -d dpl-lab /usr/sbin/vsftpd /etc/vsftpd.conf
docker exec dpl-lab pgrep -a vsftpd
```

## Paso 6 - Instalar y lanzar OpenSSH

Dentro de `dpl-lab`:

```bash
apt-get install -y openssh-server
mkdir -p /run/sshd && ssh-keygen -A
```

## Paso 7 - Endurecer el servicio

Desactivar contraseña y el login de `root` en `sshd_config`:

```bash
docker exec dpl-lab bash -c "sed -i 's/^#*PasswordAuthentication.*/PasswordAuthentication no/; s/^#*PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config"
docker exec dpl-lab bash -c "grep -E 'PasswordAuthentication|PermitRootLogin' /etc/ssh/sshd_config"
docker exec dpl-lab sshd -t
docker exec dpl-lab pkill -HUP sshd
```

## dpl-cliente

Dentro del contenedor `dpl-cliente`:

Hará falta la ip del contenedor `dpl-lab`, el siguiente comando muestra su ip:

```bash
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' dpl-lab
```


## Paso 8 - Comprobar endurecimiento

```bash
# La autenticación por contraseña debe rechazarse
docker exec dpl-cliente ssh -o PubkeyAuthentication=no -o PreferredAuthentications=password -o StrictHostKeyChecking=accept-new despliegue@IP_DPL_LAB true

# El login de root debe rechazarse aunque exista el servicio
docker exec dpl-cliente ssh -o BatchMode=yes -o StrictHostKeyChecking=accept-new root@IP_DPL_LAB true
```

Salida esperada:
```bash
despliegue@IP_DPL_LAB: Permission denied (publickey).
root@IP_DPL_LAB: Permission denied (publickey).
```

## Paso 9 - Subida del build de Vite con despliegue

Desde el anfitrión, copia el directorio `dist`en `dpl-cliente:/tmp/dist`: `docker cp ~/dpl/ae4/frontend/dist dpl-cliente:/tmp/dist)`

Realizar subida con `despliegue` (subir el `dist/` del build de Vite con mirror -R) y una descarga con `lector`, verificando con `ls` en ambos lados.

```bash
lftp -u despliegue,CAMBIAR_ESTO IP_DPL_LAB \
  -e "set ssl:verify-certificate no; cd /srv/dpl/web; mirror -R /tmp/dist; ls; exit"
```

Desglose de este comando:
* `lftp`: Cliente de transferencia de archivos avanzado que soporta FTP, FTPS, SFTP y HTTP.
* `-u despliegue,CAMBIAR_ESTO`: Especifica las credenciales de autenticación en formato usuario,contraseña.
* `IP_DPL_LAB`: Es la dirección IP del servidor destino (por ejemplo, IP_DPL_LAB).
* `-e "..."`: Indica a lftp que ejecute la cadena de comandos que está entre comillas
* `set ssl:verify-certificate no;`: Desactiva la verificación estricta del certificado SSL/TLS. 
* `cd /srv/dpl/web;`: Cambia el directorio de trabajo en el servidor remoto a la ruta donde se alojan los archivos de la web (/srv/dpl/web).
* `mirror -R /tmp/dist;`: 
    * `mirror`: Sincroniza un directorio local con uno remoto.
    * `-R` (Reverse / Subida): Invierte la dirección habitual para subir el contenido desde el cliente hacia el servidor (en lugar de descargar).
    * `/tmp/dist`: Toma todo el contenido de la carpeta local /tmp/dist (los estáticos de Vite) y lo replica exactamente dentro de la carpeta remota activa (/srv/dpl/web).
* `ls;`: Lista los archivos dentro del directorio remoto (/srv/dpl/web) para verificar visualmente que el contenido subió con éxito.
* `exit`: Cierra la sesión de lftp y finaliza la ejecución del comando.


## Paso 10 - Descarga con lector y escritura con lector

Descarga con lector (permitida) y escritura con lector (denegada, 553/550):

```bash
lftp -u lector,CAMBIAR_ESTO IP_DPL_LAB -e "set ssl:verify-certificate no; cd /srv/dpl/web/dist; get index.html; exit"
lftp -u lector,CAMBIAR_ESTO IP_DPL_LAB -e "set ssl:verify-certificate no; cd /srv/dpl/web; put /tmp/dist/index.html; exit"
```

Si ya existe el archivo, para sobreescribir hay que incluir la directiva `set xfer:clobber yes;` al comando:
```bash
lftp -u lector,CAMBIAR_ESTO IP_DPL_LAB -e "set ssl:verify-certificate no; set xfer:clobber yes; cd /srv/dpl/web/dist; get index.html; exit"
```

## Paso 11 - Anónimo debe fallar

```bash
lftp -u anonymous, IP_DPL_LAB -e "ls; exit"
```

# Paso 12 - Modos activo y pasivo

Comprobar los modos activos y pasivos, debe responder devolviendo exactamente el mismo listado (`assets` e `index.html`):
```bash
lftp -e "set ftp:passive-mode off; set ssl:verify-certificate no; open -u despliegue,CAMBIAR_ESTO IP_DPL_LAB; cd /srv/dpl/web/dist; ls; exit"
lftp -e "set ftp:passive-mode on; set ssl:verify-certificate no; open -u despliegue,CAMBIAR_ESTO IP_DPL_LAB; cd /srv/dpl/web/dist; ls; exit"
```

## Paso 13 - FTPS explícito

```bash
lftp -u despliegue,CAMBIAR_ESTO IP_DPL_LAB -e "set ftp:ssl-force true; set ssl:verify-certificate no; cd /srv/dpl/web/dist; ls; exit"
```

## Paso 14 - Claves y transferencia por SSH

Generación del par de claves SSH, copia de la cable pública al servidor y verificación del acceso por SSH.
```bash
ssh-keygen -t ed25519 -N "" -f /root/.ssh/id_ed25519
ssh-copy-id despliegue@IP_DPL_LAB
ssh despliegue@IP_DPL_LAB "whoami"
```

Copia remota con SCP, creación de un script por lotes para SFTP y eecución no interactiva de SFTP:
```bash
scp /tmp/dist/index.html despliegue@IP_DPL_LAB:/tmp/
printf 'ls -l\n' > /tmp/sftp-batch.txt
sftp -b /tmp/sftp-batch.txt despliegue@IP_DPL_LAB
```

# Verificables

## 1. Servicios activos en dpl-lab 

* (sin systemd: proceso + puerto + versión)

Comandos de comprobación:
```bash
docker exec dpl-lab pgrep -a vsftpd
docker exec dpl-lab ss -lntp | grep ':21'
docker exec dpl-lab pgrep -a sshd
docker exec dpl-lab bash -c "dpkg -l vsftpd | tail -1"
```

Salida:
```bash
86 /usr/sbin/vsftpd /etc/vsftpd.conf
LISTEN 0      32           0.0.0.0:21         0.0.0.0:*    users:(("vsftpd",pid=86,fd=3))
46 sshd: /usr/sbin/sshd -D [listener] 0 of 10-100 startups
ii  vsftpd         3.0.5-0ubuntu3.1 arm64        lightweight, efficient FTP server written for security
```

## 2. Configuración y usuarios

Comandos de comprobación:
```bash
docker exec dpl-lab grep -E 'anonymous_enable|local_enable|write_enable|ssl_enable' /etc/vsftpd.conf
docker exec dpl-lab grep -E '^PasswordAuthentication|^PermitRootLogin' /etc/ssh/sshd_config
docker exec dpl-lab grep -E 'despliegue|lector' /etc/passwd
docker exec dpl-lab groups despliegue lector
docker exec dpl-lab ls -ld /srv/dpl/web
```

Salida:
```bash
anonymous_enable=NO
local_enable=YES
write_enable=YES
#anon_mkdir_write_enable=YES
#ssl_enable=NO
ssl_enable=YES
PermitRootLogin no
PasswordAuthentication no
PermitRootLogin no
PasswordAuthentication no
despliegue:x:1004:1007::/home/despliegue:/bin/bash
lector:x:1005:1005::/home/lector:/bin/bash
despliegue : despliegue operaciones
lector : lector operaciones
drwxr-x--- 4 despliegue operaciones 4096 Nov  1 20:44 /srv/dpl/web
```

## 3. Transferencias reales desde dpl-cliente (activo y pasivo).

Con FTPS forzado, lftp debe aceptar el certificado autofirmado:"set ssl:verify-certificate no". 
Si tu servidor permite FTP en claro, la opción no molesta.

Comando:
```bash
docker exec dpl-cliente bash -c "echo 'prueba de transferencia' > /tmp/prueba.txt"
```
Comando:
```bash
docker exec dpl-cliente lftp -u despliegue,CAMBIAR_ESTO IP_DPL_LAB -e "set ssl:verify-certificate no; cd /srv/dpl/web; put /tmp/prueba.txt; ls; exit"
```
Salida:
```bash
drwxr-xr-x    2 1004     1006         4096 Nov 01 20:31 assets
drwxr-xr-x    3 1004     1007         4096 Nov 01 20:35 dist
-rw-r-----    1 1004     1006          735 Nov 01 20:31 index.html
-rw-r--r--    1 1004     1007           24 Nov 02 12:58 prueba.txt
```
Comando:
```bash
docker exec dpl-cliente lftp -u despliegue,CAMBIAR_ESTO IP_DPL_LAB -e "set ftp:passive-mode off; set ssl:verify-certificate no; cd /srv/dpl/web; ls; exit"
```
Salida:
```bash
drwxr-xr-x    2 1004     1006         4096 Nov 01 20:31 assets
drwxr-xr-x    3 1004     1007         4096 Nov 01 20:35 dist
-rw-r-----    1 1004     1006          735 Nov 01 20:31 index.html
-rw-r--r--    1 1004     1007           24 Nov 02 12:58 prueba.txt
```
Comando:
```bash
docker exec dpl-cliente lftp -u despliegue,CAMBIAR_ESTO IP_DPL_LAB -e "set ftp:passive-mode on; set ssl:verify-certificate no; cd /srv/dpl/web; ls; exit"
```
Salida:
```bash
drwxr-xr-x    2 1004     1006         4096 Nov 01 20:31 assets
drwxr-xr-x    3 1004     1007         4096 Nov 01 20:35 dist
-rw-r-----    1 1004     1006          735 Nov 01 20:31 index.html
-rw-r--r--    1 1004     1007           24 Nov 02 12:58 prueba.txt
```

## 4. Permisos

`lector` descarga sí, escribe no; anónimo no

* `lector` descarga sí:

Comando:
```bash
docker exec dpl-cliente lftp -u lector,CAMBIAR_ESTO IP_DPL_LAB -e "set ssl:verify-certificate no; cd /srv/dpl/web; get prueba.txt; exit"
```
No devuelve ninguna salida, para comprobar la descarga del usuario `lector`:
```bash
docker exec dpl-cliente ls -l /tmp/prueba.txt
```
Salida:
```bash
-rw-r--r-- 1 root root 24 Nov  2 12:58 /tmp/prueba.txt
```

* `lector` escribe no:

Comando:
```bash
docker exec dpl-cliente lftp -u lector,CAMBIAR_ESTO IP_DPL_LAB -e "set ssl:verify-certificate no; cd /srv/dpl/web; put /tmp/prueba.txt; exit"   # debe fallar
```
Salida:
```bash
put: /tmp/prueba.txt: Access failed: 553 Could not create file. (prueba.txt)
```

* anónimo no:

Comando:
```bash
docker exec dpl-cliente lftp -u anonymous, IP_DPL_LAB -e "ls; exit"                                                                             # debe fallar
```
Salida:
```bash
ls: Fatal error: Certificate verification: The certificate is NOT trusted. The certificate issuer is unknown.  (2C:3B:F9:F2:D4:3C:CB:E0:F5:33:C0:3B:20:56:E8:C3:47:0E:1D:F6)
```

## 5. Transferencia cifrada

* FTPS explícito y SFTP con clave

Comando:
```bash
docker exec dpl-cliente lftp -u despliegue,CAMBIAR_ESTO IP_DPL_LAB \
  -e "set ftp:ssl-force true; set ssl:verify-certificate no; cd /srv/dpl/web/dist; ls; exit"
```

Salida:
```bash
drwxr-xr-x    2 1004     1007         4096 Nov 01 20:35 assets
-rw-r--r--    1 1004     1007          735 Nov 01 20:35 index.html
```

Comando:

```bash
docker exec dpl-cliente bash -c "printf 'put /tmp/prueba.txt\nls -l\n' > /tmp/sftp-batch.txt"
docker exec dpl-cliente sftp -b /tmp/sftp-batch.txt despliegue@IP_DPL_LAB
```

Salida:
```bash
sftp> put /tmp/prueba.txt
sftp> ls -l
-rw-r--r--    ? despliegue despliegue       24 Nov  2 13:13 prueba.txt
```

Comando:
```bash
docker exec dpl-cliente ssh -o PubkeyAuthentication=no -o PreferredAuthentications=password despliegue@IP_DPL_LAB true   # debe fallar
```

Salida:
```bash
despliegue@IP_DPL_LAB: Permission denied (publickey).
```

## 6. Comprobación rápida con curl 
* (banner y FTPS explícito; con las directivas ssl_* el FTP en claro se rechaza, así que se usa --ssl-reqd)

```bash
docker exec dpl-cliente curl -sv --ssl-reqd --insecure -u despliegue:CAMBIAR_ESTO ftp://IP_DPL_LAB/ 2>&1 | grep 220
```
Salida:
```bash
< 220 (vsFTPd 3.0.5)
```
Comando:
```bash
docker exec dpl-cliente curl -s --ssl-reqd --insecure -u despliegue:CAMBIAR_ESTO ftp://IP_DPL_LAB/
```
Salida:
```bash
-rw-r--r--    1 1004     1007           24 Nov 02 13:13 prueba.txt
```

# comprobaciones.txt

Se ejecutan los siguientes comandos y se copia la salida en `comprobaciones.txt`:

```bash
# 1. Servicios activos en dpl-lab (sin systemd: proceso + puerto + versión)
docker exec dpl-lab pgrep -a vsftpd
docker exec dpl-lab ss -lntp | grep ':21'
docker exec dpl-lab pgrep -a sshd
docker exec dpl-lab bash -c "dpkg -l vsftpd | tail -1"

# 2. Configuración y usuarios
docker exec dpl-lab grep -E 'anonymous_enable|local_enable|write_enable|ssl_enable' /etc/vsftpd.conf
docker exec dpl-lab grep -E '^PasswordAuthentication|^PermitRootLogin' /etc/ssh/sshd_config
docker exec dpl-lab grep -E 'despliegue|lector' /etc/passwd
docker exec dpl-lab groups despliegue lector
docker exec dpl-lab ls -ld /srv/dpl/web

# 3. Transferencias reales desde dpl-cliente (activo y pasivo)
docker exec dpl-cliente bash -c "echo 'prueba de transferencia' > /tmp/prueba.txt"
docker exec dpl-cliente lftp -u despliegue,CAMBIAR_ESTO IP_DPL_LAB -e "set ssl:verify-certificate no; cd /srv/dpl/web; put /tmp/prueba.txt; ls; exit"
docker exec dpl-cliente lftp -u despliegue,CAMBIAR_ESTO IP_DPL_LAB -e "set ftp:passive-mode off; set ssl:verify-certificate no; cd /srv/dpl/web; ls; exit"
docker exec dpl-cliente lftp -u despliegue,CAMBIAR_ESTO IP_DPL_LAB -e "set ftp:passive-mode on; set ssl:verify-certificate no; cd /srv/dpl/web; ls; exit"

# 4. Permisos: lector descarga sí, escribe no; anónimo no
docker exec dpl-cliente lftp -u lector,CAMBIAR_ESTO IP_DPL_LAB -e "set ssl:verify-certificate no; cd /srv/dpl/web; get prueba.txt; exit"
docker exec dpl-cliente ls -l /tmp/prueba.txt
docker exec dpl-cliente lftp -u lector,CAMBIAR_ESTO IP_DPL_LAB -e "set ssl:verify-certificate no; cd /srv/dpl/web; put /tmp/prueba.txt; exit"   # debe fallar
docker exec dpl-cliente lftp -u anonymous, IP_DPL_LAB -e "ls; exit"                                                                             # debe fallar

# 5. Transferencia cifrada: FTPS explícito y SFTP con clave
docker exec dpl-cliente lftp -u despliegue,CAMBIAR_ESTO IP_DPL_LAB -e "set ftp:ssl-force true; set ssl:verify-certificate no; cd /srv/dpl/web/dist; ls; exit"
docker exec dpl-cliente bash -c "printf 'put /tmp/prueba.txt\nls -l\n' > /tmp/sftp-batch.txt"
docker exec dpl-cliente sftp -b /tmp/sftp-batch.txt despliegue@IP_DPL_LAB
docker exec dpl-cliente ssh -o PubkeyAuthentication=no -o PreferredAuthentications=password despliegue@IP_DPL_LAB true   # debe fallar

# 6. Comprobación rápida con curl (banner y FTPS explícito)
docker exec dpl-cliente curl -sv --ssl-reqd --insecure -u despliegue:CAMBIAR_ESTO ftp://IP_DPL_LAB/ 2>&1 | grep 220
docker exec dpl-cliente curl -s --ssl-reqd --insecure -u despliegue:CAMBIAR_ESTO ftp://IP_DPL_LAB/
```

`cat comprobaciones.txt`:

```bash
86 /usr/sbin/vsftpd /etc/vsftpd.conf
LISTEN 0      32           0.0.0.0:21         0.0.0.0:*    users:(("vsftpd",pid=86,fd=3))
46 sshd: /usr/sbin/sshd -D [listener] 0 of 10-100 startups
ii  vsftpd         3.0.5-0ubuntu3.1 arm64        lightweight, efficient FTP server written for security
anonymous_enable=NO
local_enable=YES
write_enable=YES
#anon_mkdir_write_enable=YES
#ssl_enable=NO
ssl_enable=YES
PermitRootLogin no
PasswordAuthentication no
PermitRootLogin no
PasswordAuthentication no
despliegue:x:1004:1007::/home/despliegue:/bin/bash
lector:x:1005:1005::/home/lector:/bin/bash
despliegue : despliegue operaciones
lector : lector operaciones
drwxr-x--- 4 despliegue operaciones 4096 Nov  1 20:44 /srv/dpl/web
drwxr-xr-x    2 1004     1006         4096 Nov 01 20:31 assets
drwxr-xr-x    3 1004     1007         4096 Nov 01 20:35 dist
-rw-r-----    1 1004     1006          735 Nov 01 20:31 index.html
-rw-r--r--    1 1004     1007           24 Nov 02 13:27 prueba.txt
drwxr-xr-x    2 1004     1006         4096 Nov 01 20:31 assets
drwxr-xr-x    3 1004     1007         4096 Nov 01 20:35 dist
-rw-r-----    1 1004     1006          735 Nov 01 20:31 index.html
-rw-r--r--    1 1004     1007           24 Nov 02 13:27 prueba.txt
drwxr-xr-x    2 1004     1006         4096 Nov 01 20:31 assets
drwxr-xr-x    3 1004     1007         4096 Nov 01 20:35 dist
-rw-r-----    1 1004     1006          735 Nov 01 20:31 index.html
-rw-r--r--    1 1004     1007           24 Nov 02 13:27 prueba.txt
get: /prueba.txt: file already exists and xfer:clobber is unset
-rw-r--r-- 1 root root 24 Nov  2 13:27 /tmp/prueba.txt
put: /tmp/prueba.txt: Access failed: 553 Could not create file. (prueba.txt)
ls: Fatal error: Certificate verification: The certificate is NOT trusted. The certificate issuer is unknown.  (2C:3B:F9:F2:D4:3C:CB:E0:F5:33:C0:3B:20:56:E8:C3:47:0E:1D:F6)
drwxr-xr-x    2 1004     1007         4096 Nov 01 20:35 assets
-rw-r--r--    1 1004     1007          735 Nov 01 20:35 index.html
sftp> put /tmp/prueba.txt
sftp> ls -l
-rw-r--r--    ? despliegue despliegue       24 Nov  2 13:27 prueba.txt
despliegue@IP_DPL_LAB: Permission denied (publickey).
< 220 (vsFTPd 3.0.5)
-rw-r--r--    1 1004     1007           24 Nov 02 13:27 prueba.txt
```