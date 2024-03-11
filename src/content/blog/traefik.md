---
title: "Traefik como proxy inverso para orquestar contenedores"
description: "Gestiona múltiples contenedores Docker con Traefik y certificados
  SSL."
pubDate: "2024-02-22"
heroImage: "/post_img.webp"
tags: ["VPS", "Docker", "Traefik", "DNS", "Proxy_Inverso"]
---

En esta publicación, exploraremos el uso de Traefik como un proxy inverso para
gestionar de forma eficiente múltiples contenedores Docker en un servidor VPS.
Aprenderemos a instalar y configurar Traefik con certificados SSL mediante
Docker Compose, así como a integrar nuevos contenedores al ecosistema de forma
automática. También abordaremos la configuración del DNS para apuntar al
dominio deseado.

## Introducción

Un proxy inverso actúa como intermediario entre las peticiones externas y los
servicios internos. En el contexto de Docker, un proxy inverso facilita la
gestión de múltiples contenedores al centralizar la configuración de
enrutamiento, seguridad y acceso. Traefik destaca como una opción popular por su
flexibilidad, escalabilidad y facilidad de uso.

## Instalación y configuración de Traefik

1. **Preparar Docker Compose:** Crea un archivo `docker-compose.yml` con la
   configuración de Traefik. Puedes usar la siguiente plantilla como base:

```yml
version: "3.3"

services:
  traefik:
    image: "traefik:v2.10"
    container_name: "traefik"
    command:
      - "--api.dashboard=false"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      - "--certificatesresolvers.myresolver.acme.httpchallenge=true"
      - "--certificatesresolvers.myresolver.acme.httpchallenge.entrypoint=web"
      - "--certificatesresolvers.myresolver.acme.email=example@mail.com"
      - "--certificatesresolvers.myresolver.acme.storage=/letsencrypt/acme.json"
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - "./letsencrypt:/letsencrypt"
      - "/var/run/docker.sock:/var/run/docker.sock:ro"
    networks:
      - traefik
    restart: always

networks:
  traefik:
    name: traefik
    ipam:
      config:
        - subnet: 172.20.0.0/24
```

2. **Iniciar Traefik:** Ejecuta `docker-compose up -d` para iniciar Traefik en
   segundo plano.

## Integración de nuevos contenedores

1. **Definir etiquetas Traefik:** Agrega las etiquetas `traefik.enable=true` y
   `traefik.http.routers.<nombre_servicio>.rule=Host(dominio.com>)` al archivo
   `docker-compose.yml` de cada nuevo servicio.

2. **Especificar puertos:** Define los puertos utiliza cada servicio en su
   archivo `docker-compose.yml`.

   - **Detección automática de puertos**:

   Traefik puede detectar automáticamente los puertos expuestos por un
   contenedor si solo se expone un único puerto. En este caso, no es necesario
   realizar ninguna configuración adicional. Traefik enrutará automáticamente
   las peticiones al puerto detectado.

   - **Configuración manual de puertos**:

   Si el contenedor expone múltiples puertos o ningún puerto, se debe
   especificar el puerto o los puertos a utilizar mediante la etiqueta
   `traefik.http.services.<service_name>.loadbalancer.server.port.`

```yml
version: "3.3"

services:
  whoami:
    image: "traefik/whoami"
    container_name: "simple-service"
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.whoami.rule=Host(`whoami.domain.xyz`)"
      - "traefik.http.routers.whoami.entrypoints=websecure"
      - "traefik.http.routers.whoami.tls.certresolver=myresolver"
      - "traefik.http.services.whoami.loadbalancer.server.port=80"
    networks:
      - traefik

networks:
  traefik:
    name: traefik
    external: true
```

3. **Iniciar el nuevo servicio:** Ejecuta `docker-compose up -d` para iniciar
   el nuevo contenedor. Traefik lo detectará automáticamente y lo enrutará al
   dominio especificado.

## Configuración del DNS

1. **Acceder al panel de control del proveedor de DNS:** Inicia sesión en el
   panel de control de tu proveedor de dominio.

2. **Crear registros A:** Crea registros A para apuntar el dominio y
   subdominios a la dirección IP pública de tu servidor VPS.

3. **Propagar los cambios:** Espera la propagación de los cambios DNS, que puede
   tardar hasta 24 horas.

**Recursos adicionales:**

- Documentación oficial de Traefik:
  [https://doc.traefik.io/traefik/](https://doc.traefik.io/traefik/)

**¡Espero que este artículo te haya sido útil!**
