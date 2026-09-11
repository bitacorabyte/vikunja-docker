# Vikunja con Docker & Traefik

Despliegue automatizado y seguro de Vikunja (plataforma integral de gestión de tareas y proyectos) utilizando Docker Compose, conectado a una base de datos MariaDB externa y enrutado mediante Traefik como proxy reverso en red local.

---

## 📋 Precondiciones

Antes de desplegar este contenedor, asegúrate de cumplir con los siguientes requisitos en tu host Docker / Proxmox:

- **Docker y Docker Compose** instalados y operativos en el sistema.
- **Redes externas creadas:** Este `compose.yaml` utiliza dos redes externas que deben existir previamente (creadas desde la terminal o mediante Portainer):
  - `service`: Red privada interna backend para la comunicación aislada y segura entre Vikunja y el contenedor central de base de datos (MariaDB).
  - `traefik`: Red compartida a través de la cual Traefik gestiona y redirige el tráfico HTTP.
  
  Puedes crearlas ejecutando:
  ```bash
  docker network create service
  docker network create traefik

* **Base de datos externa preparada:** Vikunja genera sus tablas y migraciones en el primer arranque, pero **requiere que la base de datos vacía y el usuario existan previamente** en tu MariaDB central. Accede a la consola de tu MariaDB y ejecuta:
```sql
CREATE DATABASE vikunja_db;
CREATE USER 'vikunja_user'@'%' IDENTIFIED BY 'tu_contraseña_segura';
GRANT ALL PRIVILEGES ON vikunja_db.* TO 'vikunja_user'@'%';
FLUSH PRIVILEGES;
```


* **Directorio de datos y gestión de permisos:** El contenedor corre internamente bajo un usuario sin privilegios (`UID 1000`). Crea el directorio local y ajusta la propiedad antes de levantar el stack para evitar errores de permisos de escritura (`storage validation failed`):
```bash
sudo mkdir -p /opt/docker/vikunja/files
sudo chown -R 1000:1000 /opt/docker/vikunja/files

```


* **Proxy Reverso Traefik:** Tener operativo un contenedor de Traefik escuchando en la red `traefik`.

---

## 📂 Volúmenes y Persistencia

El contenedor mapea el siguiente directorio local persistente en el host:

* `/opt/docker/vikunja/files` (montado en `/app/vikunja/files` dentro del contenedor): Almacena archivos adjuntos a las tareas, avatares, exportaciones y multimedia. Al residir fuera del ciclo de vida del contenedor, garantiza persistencia ante actualizaciones o recreaciones del stack.

---

## 🔒 Consideraciones de Seguridad

Al desplegar una plataforma de productividad interna, se aplican buenas prácticas SecDevOps:

* **Inmutabilidad de la Imagen:** Uso de imagen anclada por digest SHA256 (`image: vikunja/vikunja:${VIKUNJA_VERSION}`) para garantizar reproducibilidad exacta y blindar el entorno contra cambios no verificados en el registro Docker.
* **Secreto de Servicio (`VIKUNJA_SERVICE_SECRET`):** Asegúrate de generar una cadena aleatoria y robusta para cifrar tokens de sesión y JWT. Puedes generarla con:
```bash
openssl rand -base64 48

```


* **Uso de Variables de Entorno (.env):** Nunca expongas credenciales de base de datos, contraseñas de aplicación ni secretos directamente en el `compose.yaml`. En flujos GitOps (Portainer + GitHub), inyecta estas variables desde el panel de Portainer o mantén un archivo `.env` con permisos restringidos (`chmod 600 .env`) excluido del control de versiones (`.gitignore`).
* **Aislamiento en Redes:**
* La comunicación con MariaDB discurre de forma privada y exclusiva por la red `service`.
* El puerto interno `3456` de Vikunja no está expuesto en el host (`ports:` no se declara), forzando a que todo el tráfico transite obligatoriamente a través del proxy reverso.


* **Validación CORS (`VIKUNJA_SERVICE_PUBLICURL`):** Vikunja requiere definir explícitamente su URL pública para validar el origen de las peticiones (por ejemplo, `http://vikunja.home.arpa/`).

---

## ⚙️ Configuración de Etiquetas (Labels) de Traefik

El servicio incluye las siguientes etiquetas de Docker para el descubrimiento y balanceo dinámico:

* `traefik.enable=true`: Habilita la gestión de este contenedor por parte de Traefik.
* `traefik.docker.network=traefik`: **Crítico.** Fuerza a Traefik a utilizar su red compartida para alcanzar el contenedor, previniendo errores de enrutamiento y *Gateway Timeout (504)* provocados por la red interna multi-homed.
* `traefik.http.routers.vikunja.rule=Host(`${DOMAIN}`)`: Define el dominio/subdominio local por el cual se accederá al gestor de tareas (ej: `vikunja.home.arpa`).
* `traefik.http.routers.vikunja.entrypoints=web`: Especifica el punto de entrada HTTP (puerto 80) configurado en Traefik.
* `traefik.http.services.vikunja.loadbalancer.server.port=3456`: Indica a Traefik que redirija las peticiones hacia el puerto web interno por defecto de Vikunja (`3456`).

---

## 🚀 Puesta en Marcha

1. Clona este repositorio o estructura los archivos en tu servidor / Portainer (App Delivery).
2. Crea el archivo `.env` (o define las variables en la pestaña de variables de entorno de tu Stack en Portainer) basándote en la plantilla `.env.example`:
* Configura la versión con hash inmutable de Vikunja.
* Introduce el secreto de aplicación y la URL pública (`http://vikunja.home.arpa/`).
* Define el host de MariaDB (`DB_HOST`) con el nombre exacto de tu contenedor de base de datos dentro de `service`.
* Rellena las credenciales SMTP de Gmail (utilizando tu contraseña de aplicación de 16 caracteres sin espacios).


3. Asegúrate de tener creadas las redes externas (`service`, `traefik`), el directorio persistente con permisos `1000:1000` y la base de datos lista.
4. Despliega la aplicación ejecutando:
```bash
docker compose up -d

```


*(o pulsa en **Deploy the stack** desde la UI de Portainer).*
5. Accede a tu navegador mediante la URL configurada en Traefik (ej: `http://vikunja.home.arpa`) para registrar tu cuenta de usuario e iniciar la gestión de tareas.

---

*Otros archivos `docker-compose.yml` en mi perfil para completar este o ampliar con más herramientas y/o servicios.*
