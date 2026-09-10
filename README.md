# Vikunja - Task Management Stack

Despliegue de [Vikunja](https://vikunja.io/) mediante Docker Compose, gestionado a través de Portainer e integrado con Traefik como Reverse Proxy. Este servicio utiliza una base de datos MariaDB centralizada preexistente en el homelab.

## 🏗️ Arquitectura de Red

El stack se conecta a dos redes externas existentes en Portainer:
- **`traefik`:** Conecta el contenedor de la aplicación con el Reverse Proxy para la salida web y certificados SSL.
- **`service` (Interna):** Red backend utilizada para conectar aplicaciones a la base de datos MariaDB central sin exponerla.

## ⚙️ Requisito Previo: Base de Datos

Antes de desplegar este stack, debes crear la base de datos y el usuario en tu contenedor de MariaDB central. Entra a la consola de tu MariaDB y ejecuta:

```sql
CREATE DATABASE vikunja;
CREATE USER 'vikunja'@'%' IDENTIFIED BY 'tu_password_seguro';
GRANT ALL PRIVILEGES ON vikunja.* TO 'vikunja'@'%';
FLUSH PRIVILEGES;