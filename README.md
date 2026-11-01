# AE5 - Transferencia segura

# Índice


En este microproyecto, se trabaja en dos contenedores, `dpl-lab` (servidor) y `dpl-cliente` (cliente) en la red `dpl-net`.

# Paso 1

Crear usuarios: despliegue (puede subir y bajar) y lector (solo descargar), y un grupo operaciones al que pertenezcan ambos.

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

# Pso 2

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

