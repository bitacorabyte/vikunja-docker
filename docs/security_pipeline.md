# Arquitectura del Pipeline de Seguridad CI/CD (DevSecOps)

Este documento detalla los controles de seguridad automatizados integrados en el flujo de GitHub Actions para el despliegue de stacks Docker Compose.

---

## 🎯 Objetivos de Seguridad

1. **Shift-Left Security:** Identificar vulnerabilidades y malas configuraciones en la fase de desarrollo e integración, antes de que Portainer despliegue el stack en producción.
2. **Prevención de Fugas de Credenciales (Zero Secret Leakage):** Impedir que tokens, claves de aplicación de correo o credenciales de MariaDB se consoliden en el historial de Git.
3. **Gestión de Postura de Seguridad en Contenedores (SCA):** Conocer la superficie de exposición y CVEs presentes en las dependencias de la imagen.

---

## 🔍 Fases del Pipeline

### 1. Detección de Secretos (`TruffleHog`)
- **Mecanismo:** Analiza la totalidad del historial `.git` y el árbol de trabajo mediante análisis heurístico y búsqueda de firmas de entropía alta.
- **Acción:** Detecta credenciales reales de proveedores conocidos (como SMTP de Gmail, tokens de API o contraseñas) para evitar fugas catastróficas.

### 2. Validación Estática de IaC (`Docker Compose Linting`)
- **Mecanismo:** Utiliza el binario de Docker para renderizar y validar la configuración declarativa del archivo `docker-compose.yml`.
- **Estrategia de Mocking:** Clona temporalmente el archivo `.env.example` como `.env` para garantizar que la interpolación de variables `${VAR}` sea válida sin exponer valores reales.

### 3. Análisis de Vulnerabilidades y Formato SARIF (`Trivy`)
- **Mecanismo:** Analiza la imagen base de Vikunja (`vikunja/vikunja`) cruzando los paquetes instalados con bases de datos de vulnerabilidades (NVD, CVE).
- **Formato SARIF:** Genera un archivo `.sarif` (estándar OASIS para intercambio de resultados de análisis estático).
- **Integración con GitHub:** Mediante la acción `upload-sarif`, los hallazgos se cargan de forma interactiva en la pestaña **Security > Code scanning alerts** del repositorio, permitiendo auditar el riesgo por nivel de severidad (`CRITICAL`, `HIGH`).

---

## 🚦 Códigos de Salida (Exit Codes) en CI/CD

El comportamiento del pipeline ante vulnerabilidades se rige por códigos de salida Unix:

| Exit Code | Modo | Comportamiento en GitHub Actions |
| :--- | :--- | :--- |
| `0` | **Informativo / Auditoría** | Registra los hallazgos en la pestaña Security pero mantiene el pipeline en verde. Recomendado para entornos de monitorización. |
| `1` | **Bloqueante / Quality Gate** | Rompe el pipeline en rojo si se detectan vulnerabilidades no parcheadas con severidad `CRITICAL` o `HIGH`, impidiendo el merge o despliegue. |