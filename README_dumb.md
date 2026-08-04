# Explicación sencilla de la imagen Python

Este documento explica qué hace `python/Dockerfile` y cómo recibe Python 3.12
cuando publicamos la imagen en GHCR.

## Resumen

La construcción tiene dos pasos:

1. El workflow construye la imagen base de Pulumi sobre
   `python:3.12-slim-bookworm`.
2. `python/Dockerfile` añade Git, crea un entorno virtual e instala
   `python/requirements.txt`.

Por tanto, no copiamos manualmente Python entre imágenes. La propia imagen base
ya contiene Python 3.12.

## De dónde sale Python 3.12

En `.github/workflows/publish-ghcr-python.yml`, la construcción de la base pasa
este argumento:

```yaml
build-args: |
  BASE_IMAGE=python:3.12-slim-bookworm
  PULUMI_VERSION=${{ env.PULUMI_VERSION }}
```

`base-debian/Dockerfile` acepta `BASE_IMAGE` como su primera imagen. Para esta
publicación concreta, la base contiene Python 3.12 y encima se instala el CLI
de Pulumi.

Después, `python/Dockerfile` utiliza esa imagen ya preparada.

## Argumentos del Dockerfile

```dockerfile
ARG REPOSITORY_BASE_PATH
ARG RELEASE_VERSION
```

- `REPOSITORY_BASE_PATH` es la ubicación del registro. En la publicación será
  `ghcr.io/crixaxpo`.
- `RELEASE_VERSION` es la etiqueta de la imagen base, por ejemplo `3.253.0`.

## Imagen base

```dockerfile
FROM ${REPOSITORY_BASE_PATH}/runner-pulumi-base-debian:${RELEASE_VERSION}
```

Utiliza la imagen construida en el paso anterior. Esa imagen ya contiene:

- Python 3.12.
- El CLI de Pulumi.
- El usuario `spacelift` con UID `1983`.

## Instalación como administrador

```dockerfile
USER root
```

Docker cambia temporalmente a `root` porque instalar paquetes del sistema y
crear `/opt/pulumi-python` requiere permisos de administrador.

## Instalación de Git y herramientas Python

```dockerfile
RUN apt-get update -o Acquire::Retries=5 \
  -o Acquire::http::No-Cache=true \
  -o Acquire::http::Pipeline-Depth=0 \
  && apt-get install -y --no-install-recommends \
    git python3 python3-pip python3-venv \
  && ln -sf python3 /usr/bin/python \
  && rm -rf /var/lib/apt/lists/*
```

Este bloque:

1. Actualiza el catálogo de paquetes de Debian.
2. Reintenta la descarga si hay un fallo temporal.
3. Instala Git y las herramientas necesarias para crear entornos virtuales.
4. Permite usar `python` además de `python3`.
5. Elimina el catálogo de APT para reducir el tamaño de la imagen.

La imagen base ya contiene Python 3.12 en `/usr/local/bin`. Ese directorio
aparece antes que `/usr/bin` en `PATH`, por lo que `python3` utiliza Python 3.12
durante la construcción.

## Copia de las dependencias

```dockerfile
COPY requirements.txt /tmp/requirements.txt
```

Copia `python/requirements.txt` a un fichero temporal dentro de la imagen.

El fichero contiene la unión de versiones que necesitan `pulumi-payen` y
`github-pulumi-automation`:

```text
azure-appconfiguration==1.9.0
azure-identity==1.25.3
databricks-sdk==0.120.0
msal==1.37.0
pulumi==3.253.0
pulumi-azure-native==2.92.2
pulumi-azuread==6.10.0
pulumi-command==1.2.1
pulumi-databricks==1.101.0
pulumi-github==6.14.0
PyGithub==2.9.1
python-dotenv==1.2.2
PyYAML==6.0.3
truststore==0.10.4
```

## Entorno virtual e instalación

```dockerfile
RUN python3 -m venv /opt/pulumi-python \
  && /opt/pulumi-python/bin/python -m pip install --upgrade pip setuptools wheel \
  && /opt/pulumi-python/bin/python -m pip install \
    --no-cache-dir --no-compile -r /tmp/requirements.txt
```

Este bloque:

1. Crea un entorno virtual en `/opt/pulumi-python`.
2. Actualiza `pip`, `setuptools` y `wheel` dentro del entorno.
3. Instala las dependencias de `requirements.txt`.

Las dependencias quedan instaladas durante el build. Spacelift no necesita
descargarlas desde PyPI al ejecutar un run.

## Limpieza

```dockerfile
find /opt/pulumi-python -type d -name "__pycache__" \
  -prune -exec rm -rf {} + \
  && rm /tmp/requirements.txt
```

Elimina cachés de Python y el fichero temporal de requisitos para reducir el
tamaño final.

## Variables de entorno

```dockerfile
ENV VIRTUAL_ENV="/opt/pulumi-python"
ENV PYTHONDONTWRITEBYTECODE="1"
ENV PATH="/opt/pulumi-python/bin:$PATH:./.pulumi/bin"
```

- `VIRTUAL_ENV` identifica el entorno virtual activo.
- `PYTHONDONTWRITEBYTECODE` evita crear nuevos `__pycache__` durante los runs.
- `PATH` hace que `python`, `pip` y los paquetes instalados se obtengan primero
  desde `/opt/pulumi-python`.

## Usuario final

```dockerfile
USER spacelift
```

La imagen termina ejecutándose como `spacelift`, no como `root`. Este es el
usuario que utilizará el worker.

## Plugins de Pulumi

El Dockerfile no ejecuta manualmente:

```text
pulumi plugin install ...
```

Los paquetes Python sí están preinstalados. Si falta un plugin binario de un
provider, Pulumi lo resolverá siguiendo su comportamiento normal durante el
preview o el update, igual que hacía la imagen anterior.

## Flujo final

```text
python:3.12-slim-bookworm
           |
           | base-debian añade Pulumi CLI y usuario spacelift
           v
runner-pulumi-base-debian
           |
           | python/Dockerfile instala requirements.txt
           v
ghcr.io/crixaxpo/runner-pulumi-python:<tag>
```
