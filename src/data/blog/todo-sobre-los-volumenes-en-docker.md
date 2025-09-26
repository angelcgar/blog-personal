---
author: Angel Contreras Garcia
pubDatetime: 2025-09-26T01:44:12Z
modDatetime: 2025-09-26T01:44:12Z
title: Todo sobre los Volúmenes en Docker
slug: todo-sobre-los-volumenes-en-docker
featured: true
draft: false
tags:
  - configuration
  - docs
description: Todo lo que necesitas saber sobre los volumenes de Docker
---

## Introducción

Docker ha cambiado las reglas del juego en el desarrollo y despliegue de aplicaciones. Nos permite empaquetar nuestro código en contenedores ligeros y portátiles que funcionan igual en cualquier lugar. Pero, si eres nuevo en este mundo, seguro te has topado con una pregunta clave: **¿Qué pasa con los datos cuando un contenedor se detiene o se elimina?**

Por defecto, cualquier dato generado dentro de un contenedor es **efímero**. Esto significa que si el contenedor se apaga o se borra, toda esa información se pierde para siempre. 😱 Aquí es donde los volúmenes de Docker se convierten en nuestros mejores aliados, ofreciendo una solución elegante para que nuestros datos persistan más allá del ciclo de vida de un contenedor.

## ¿Qué son exactamente los Volúmenes de Docker?

Un volumen en Docker es un mecanismo diseñado para almacenar y gestionar datos **fuera del sistema de archivos del contenedor**. De esta forma, los datos se desacoplan del contenedor.

* **Sin un volumen:** los datos viven y mueren con el contenedor.
* **Con un volumen:** los datos se guardan en una ubicación especial gestionada por Docker en la máquina anfitriona (el *host*), totalmente independiente del contenedor.

👉 La mejor analogía es pensar en un volumen como **un disco duro externo que conectas a tu contenedor**. No importa si apagas, eliminas o actualizas el contenedor; tus datos seguirán a salvo en ese "disco externo", listos para ser conectados a otro contenedor cuando lo necesites.

## Los 3 Tipos de Almacenamiento en Docker

Docker nos ofrece tres formas principales de manejar los datos. Es crucial entender sus diferencias para saber cuándo usar cada una.

| Tipo         | Descripción                                                                                                                              | Ventajas                                                                                                        | Desventajas                                                                                                            |
| :----------- | :--------------------------------------------------------------------------------------------------------------------------------------- | :-------------------------------------------------------------------------------------------------------------- | :--------------------------------------------------------------------------------------------------------------------- |
| Volúmenes    | La forma preferida y oficial de Docker. Los datos se guardan en una carpeta del host gestionada por Docker (`/var/lib/docker/volumes/`). | Son portables, fáciles de respaldar, más seguros y su gestión no depende de la estructura de archivos del host. | Requieren comandos específicos de Docker para su manejo.                                                               |
| Bind Mounts  | Mapean directamente una carpeta o archivo del host dentro del contenedor. Es como crear un "espejo".                                     | Tienes control total y es ideal para desarrollo (ej: reflejar cambios en tu código al instante).                | Dependen de la estructura de carpetas del host, lo que los hace menos portables. Pueden generar problemas de permisos. |
| tmpfs Mounts | Montan los datos directamente en la memoria RAM del host, nunca se escriben en el disco.                                                 | Son extremadamente rápidos. Perfectos para datos temporales o sensibles que no quieres que dejen rastro.        | No son persistentes. Si el contenedor se detiene, los datos se desvanecen.                                             |

## ¿Cómo Usar Volúmenes? Manos a la Obra

Gestionar volúmenes es muy sencillo. Aquí tienes los comandos esenciales:

### 1\. Crear un volumen:

```bash
docker volume create mi_volumen_de_datos
```

### 2\. Listar los volúmenes existentes:

```bash
docker volume ls
```

### 3\. Inspeccionar un volumen para ver sus detalles:

```bash
docker volume inspect mi_volumen_de_datos
```

## Ejemplo práctico: MySQL con datos persistentes

Vamos a crear un contenedor de MySQL. Usando la opción `-v`, le decimos a Docker que conecte nuestro volumen `mi_volumen_de_datos` a la carpeta `/var/lib/mysql` dentro del contenedor, que es donde MySQL guarda sus datos.
```bash
docker run -d \
  --name mi_base_de_datos \
  -e MYSQL_ROOT_PASSWORD=mi_contraseña_secreta_para_root \
  -v mi_volumen_de_datos:/var/lib/mysql \
  mysql:latest
```
¡Y listo\! Ahora puedes detener, eliminar y volver a crear el contenedor `mi_base_de_datos` cuantas veces quieras. Mientras sigas conectando el mismo volumen, tus tablas y datos estarán siempre ahí.

## Casos de Uso Comunes para los Volúmenes

Los volúmenes son increíblemente versátiles. Aquí algunos usos típicos:

* **Bases de datos:** Es el caso de uso por excelencia. Para que los datos de PostgreSQL, MySQL, MongoDB, etc., no se pierdan.
* **Compartir datos entre contenedores:** Varios contenedores pueden montar el mismo volumen para leer y escribir datos en un espacio compartido.
* **Respaldos y migraciones:** Es muy fácil hacer un backup montando el volumen en un contenedor temporal que comprima los datos.
* **Separar la lógica de los datos:** Permite actualizar la imagen de tu aplicación sin tocar ni poner en riesgo los datos del usuario.

## Volúmenes con Docker Compose

Cuando trabajas con aplicaciones multi-contenedor, Docker Compose simplifica enormemente la gestión. Así se define un volumen en un archivo `docker-compose.yml`:

```yml
version: "3.9"

services:
  db:
    image: postgres:15
    restart: always
    environment:
      - POSTGRES_PASSWORD=mi_contraseña_secreta
    # Paso 1: Conecta el volumen "datos\_postgres" a la carpeta de datos de postgres
    volumes:
      - datos_postgres:/var/lib/postgresql/data

# Paso 2: Declara el volumen para que Docker Compose lo gestione
volumes:
  datos_postgres:
```
Con esta configuración, al ejecutar `docker-compose up`, Docker se encargará de crear el volumen `datos_postgres` (si no existe) y conectarlo automáticamente al contenedor del servicio `db`. ¡Magia\! ✨

### Problemas Frecuentes y Cómo Solucionarlos

1. **Volúmenes Huérfanos:** A veces, al eliminar contenedores, los volúmenes que usaban quedan olvidados, ocupando espacio en disco.
   * **Solución:** Ejecuta periódicamente un comando de limpieza.

```bash
docker volume prune
```

2. **Confusión con Bind Mounts:** Recuerda que los *bind mounts* dependen de una ruta específica en tu máquina. Si te llevas tu `docker-compose.yml` a otro servidor, esa ruta podría no existir. **Prioriza siempre los volúmenes en producción**.
3. **Conflictos de Permisos:** Es común que los archivos creados dentro del volumen pertenezcan al usuario `root`. Esto puede causar problemas de permisos si tu aplicación dentro del contenedor no se ejecuta como `root`.
   * **Solución:** Puedes especificar el usuario con el que corre el contenedor (con el flag `--user` en `docker run` o `user:` en Docker Compose) o ajustar los permisos manualmente como parte del script de inicio de tu contenedor.

## Un ejemplo completo

Usaremos Docker Compose con MySQL en la version `8.0.36` (usa la version que tu quieras) para una practica con php, hay muchas formas de hacer esta conexión trabajando con php pero yo mostrare mi favorita hasta el momento. Empecemos por el `docker-compose.yml`.

```yml
version: '3.8'

services:
  mysql:
    image: mysql:8.0.36
    container_name: mysql-container
    environment:
      MYSQL_ROOT_PASSWORD: 123456       # Contraseña para root
      MYSQL_DATABASE: base_de_datos_php # Nombre de nuestra DB
      MYSQL_USER: angelcgar             # Nuestro usuario
      MYSQL_PASSWORD: password          # La contraseña para nuestro usuario
    ports:
      - "3306:3306"
    volumes:
      - mysql_data:/var/lib/mysql       # El volumen que usara la imagen de mysql
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql # Un helper mínimo
    restart: unless-stopped

volumes:                                # Información sobre el volumen
  mysql_data:
    driver: local
```

Ahora crea un archivo init.sql en la misma carpeta para tener una tabla lista:

```sql
USE base_de_datos_php;

CREATE TABLE IF NOT EXISTS usuarios (
    id INT AUTO_INCREMENT PRIMARY KEY,
    nombre VARCHAR(100) NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    fecha_creacion TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO usuarios (nombre, email) VALUES
('Angel Garcia', 'angelcgar@example.com'),
('Usuario Demo', 'demo@example.com');
```

### Algunas aclaraciones

La variable `MYSQL_PASSWORD=password` establece la contraseña para el usuario personalizado que se crea con `MYSQL_USER=angelcgar`.

En tu en `docker-compose.yml`:

```yml
   - MYSQL_ROOT_PASSWORD=123456 # Contraseña del usuario root

   - MYSQL_USER=angelcgar # Crea el usuario angelcgar

   - MYSQL_PASSWORD=password # Contraseña del usuario angelcgar
```

Así que tendrás dos usuarios:


- root con contraseña 123456

- angelcgar con contraseña password

### Para usar el Docker Compose

Ejecuta:

```bash
docker-compose up -d
```

### URL para conectarse

Por si te interesa usar la URL, la tienes aquí:

```bash
mysql://angelcgar:password@localhost:3306/base_de_datos_php
```

## Conclusión

Los volúmenes son una herramienta fundamental en el ecosistema de Docker que todo desarrollador o sysadmin debe dominar. Nos dan la tranquilidad de que los datos de nuestras aplicaciones están seguros, separados de la lógica del contenedor y listos para ser respaldados o migrados.

Entender cómo y cuándo usarlos te ahorrará muchísimos dolores de cabeza, especialmente cuando trabajes con aplicaciones críticas o bases de datos en producción.

👉 **¿Todavía no los usas en tus proyectos?** ¡Hoy es un gran día para empezar\! Prueba a levantar una base de datos con un volumen y experimenta por ti mismo lo fácil que te hace la vida.
