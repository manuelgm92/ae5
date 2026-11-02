# AE5 - Transferencia segura

# Índice


En este microproyecto, se trabaja en dos contenedores, `dpl-lab` (servidor) y `dpl-cliente` (cliente) en la red `dpl-net`.


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

