# Guía de integración del orquestador de seguridad - Repositorios clientes

[![Security Pipeline](https://img.shields.io/badge/Security-Orchestrator_Integrated-brightgreen?logo=githubactions)](#)
[![ASOC](https://img.shields.io/badge/Vulnerability_Management-DefectDojo-FF5722?logo=owasp)](#)
[![DevSecOps](https://img.shields.io/badge/DevSecOps-Shift--Left-blueviolet)](#)
[![Multi-Stack](https://img.shields.io/badge/Stacks-Java%20%7C%20Node.js-blue)](#)

Este documento detalla cómo consumir el pipeline de seguridad automatizado y reutilizable provisto por el repositorio maestro [`gha-sec-scans/orchestrator`](https://github.com/gha-sec-scans/orchestrator).

El orquestador es **completamente agnóstico del stack tecnológico**, con soporte nativo para aplicaciones **Java / Maven** (ej. WebGoat) y **Node.js / Fullstack** (ej. OWASP Juice Shop), integrando análisis estático, dinámico, escaneo de dependencias, imágenes y control de calidad (*Security Quality Gate*) previo al despliegue.

---

## Tabla de contenidos

- [Cómo funciona la integración](#cómo-funciona-la-integración)
- [Paso a paso: Configurar el workflow](#paso-a-paso-configurar-el-workflow)
  - [Ejemplo 1: Proyecto Java / Maven (Spring Boot / WebGoat)](#ejemplo-1-proyecto-java--maven-spring-boot--webgoat)
  - [Ejemplo 2: Proyecto Node.js / TypeScript / Fullstack (Juice Shop)](#ejemplo-2-proyecto-nodejs--typescript--fullstack-juice-shop)
- [Referencia completa de parámetros (with)](#referencia-completa-de-parámetros-with)
- [Secretos requeridos (secrets)](#secretos-requeridos-secrets)
- [Security Quality Gate (Control de despliegue)](#security-quality-gate-control-de-despliegue)
- [Mecanismo de autenticación dinámica DAST (auth-hook.sh)](#mecanismo-de-autenticación-dinámica-dast-auth-hooksh)
  - [Ejemplo A: Cookie de sesión formulario (WebGoat)](#ejemplo-a-cookie-de-sesión-formulario-webgoat)
  - [Ejemplo B: Token Bearer / JWT (APIs REST / Juice Shop)](#ejemplo-b-token-bearer--jwt-apis-rest--juice-shop)
- [Requisitos y restricciones del proyecto](#requisitos-y-restricciones-del-proyecto)
- [Visualización de resultados en DefectDojo](#visualización-de-resultados-en-defectdojo)
- [Solución de problemas frecuentes](#solución-de-problemas-frecuentes)

---

## Cómo funciona la integración

El repositorio cliente no necesita instalar herramientas ni configurar complejos scripts de seguridad. Mediante la directiva `uses` de GitHub Actions (*Reusable Workflows*), el repositorio consumidor delega la ejecución en el orquestador:

1. **Smart Setup & Detección:** Detecta automáticamente el lenguaje de la aplicación (`pom.xml` para Java, `package.json` para Node.js).
2. **Pruebas Unitarias:** Ejecuta `mvn test` o `npm test`.
3. **Análisis Estático (SAST):** Semgrep, CodeQL y SonarCloud en paralelo.
4. **Análisis de Dependencias (SCA):** Trivy FS.
5. **Inventario de Software (SBOM):** Generación CycloneDX (`sbom.json`) e importación directa en DefectDojo.
6. **Detección de Secretos:** Gitleaks.
7. **Auditoría de Infraestructura como Código (IaC):** Checkov y KICS.
8. **Compilación y Construcción Docker:** Compilación de backend y frontend (Angular/React) y publicación de la imagen en GitHub Container Registry (GHCR).
9. **Escaneo de Contenedores:** Trivy Image Scanning sobre el paquete publicado en GHCR.
10. **Análisis Dinámico (DAST):** Despliegue efímero del contenedor, autenticación opcional con `auth-hook.sh` y escaneo web con OWASP ZAP y Nikto.
11. **Security Quality Gate:** Consulta en tiempo real a la API de DefectDojo para verificar que las vulnerabilidades críticas y altas no superen los umbrales autorizados antes de permitir el despliegue.
12. **Centralización ASOC:** Consolidación, deduplicación y triaje de todos los hallazgos en DefectDojo.

---

## Paso a paso: Configurar el workflow

Crea el archivo `.github/workflows/call_orchestrator.yaml` en la raíz de tu repositorio consumidor.

### Ejemplo 1: Proyecto Java / Maven (Spring Boot / WebGoat)

```yaml
name: Security Pipeline Call

on:
  push:
    branches: [ main, master, feature/** ]
  pull_request:
    branches: [ main, master ]
  workflow_dispatch:

jobs:
  security-pipeline:
    uses: gha-sec-scans/orchestrator/.github/workflows/main.yaml@main
    with:
      app-port: '8080'
      app-path: '/WebGoat'
      defectdojo_product_name: 'WebGoat-TFG'
      
      # Parámetros opcionales de SonarCloud
      sonar_organization: 'gha-sec-scans'
      sonar_project_key: 'gha-sec-scans_webgoat'
      
      # Versión de Java
      java-version: '25'
      
      # Quality Gate estricto: 0 críticas y 0 altas permitidas
      max_critical: '0'
      max_high: '0'
      
    secrets: inherit
```

### Ejemplo 2: Proyecto Node.js / TypeScript / Fullstack (Juice Shop)

```yaml
name: Security Pipeline Call

on:
  push:
    branches: [ main, master, feature/** ]
  pull_request:
    branches: [ main, master ]
  workflow_dispatch:

jobs:
  security-pipeline:
    uses: gha-sec-scans/orchestrator/.github/workflows/main.yaml@main
    with:
      app-port: '3000'
      app-path: '/'
      defectdojo_product_name: 'Juice-Shop-TFG'
      
      # Versión de Node.js (Juice Shop requiere Node >= 22)
      node-version: '22'
      
      # Umbrales adaptados para proyectos con deuda técnica o intencionalmente vulnerables
      max_critical: '10'
      max_high: '200'
      fail_on_security_gate: true
      
    secrets: inherit
```

---

## Referencia completa de parámetros (with)

| Parámetro | Tipo | Obligatorio | Por Defecto | Descripción |
| :--- | :---: | :---: | :---: | :--- |
| **`app-port`** | `string` | **Sí** | - | Puerto TCP expuesto por el contenedor Docker de la aplicación (ej. `'8080'`, `'3000'`). |
| **`app-path`** | `string` | No | `'/'` | Ruta base de la aplicación para el sondeo de salud (*healthcheck*) y DAST (ej. `'/WebGoat'`, `'/'`). |
| **`defectdojo_product_name`** | `string` | **Sí** | - | Nombre del Producto en DefectDojo donde se consolidarán todos los reportes (ej. `'WebGoat-TFG'`). |
| **`sonar_organization`** | `string` | No | `''` | Organización en SonarCloud. Si se omite o está vacío, el job de Sonar se omite. |
| **`sonar_project_key`** | `string` | No | `''` | Clave del proyecto (*Project Key*) en SonarCloud. |
| **`node-version`** | `string` | No | `'22'` | Versión de Node.js utilizada en proyectos Node (`'18'`, `'20'`, `'22'`). |
| **`java-version`** | `string` | No | `'25'` | Versión del JDK utilizada en proyectos Java (`'17'`, `'21'`, `'25'`). |
| **`max_critical`** | `string` | No | `'0'` | Número máximo de vulnerabilidades Críticas activas permitidas por el Quality Gate. |
| **`max_high`** | `string` | No | `'0'` | Número máximo de vulnerabilidades Altas activas permitidas por el Quality Gate. |
| **`fail_on_security_gate`** | `boolean` | No | `true` | Si es `true`, bloquea el despliegue cuando se superan los umbrales. Si es `false`, solo advierte. |

---

## Secretos requeridos (secrets)

Configura los siguientes secretos en el repositorio consumidor (**Settings** > **Secrets and variables** > **Actions**):

| Secreto | Requerido | Descripción |
| :--- | :---: | :--- |
| **`DEFECTDOJO_URL`** | **Sí** | URL pública de la instancia de DefectDojo (ej. túnel Cloudflare: `https://xxxx.trycloudflare.com` **sin barra final**). |
| **`DEFECTDOJO_TOKEN`** | **Sí** | Token API v2 de DefectDojo (en DefectDojo: *User Profile* > *API v2 Key*). |
| **`SONAR_TOKEN`** | Condicional | Token de SonarCloud (obligatorio solo si se configuran los inputs de Sonar). |
| **`GITHUB_TOKEN`** | Automático | Proporcionado automáticamente por GitHub Actions para publicar imágenes en GHCR. |

> **Guía detallada:** Para aprender a levantar DefectDojo localmente con Docker, recuperar la contraseña inicial de admin, generar el token y exponerlo mediante Cloudflare Tunnel, consulta la guía [DEFECTDOJO_SETUP.md](DEFECTDOJO_SETUP.md).

---

## Security Quality Gate (Control de despliegue)

Antes de llegar al paso final de despliegue (`deploy`), el orquestador evalúa la postura de seguridad del proyecto consultando la API de DefectDojo:

1. Realiza una llamada a `/api/v2/findings/` filtrando por producto, vulnerabilidades activas y no duplicadas.
2. Compara el total de vulnerabilidades **Critical** y **High** contra `max_critical` y `max_high`.
3. Publica una tabla visual interactiva en el **Job Summary** de GitHub Actions:

```text
### DefectDojo Security Quality Gate
Product: Juice-Shop-TFG

| Severity | Active Findings | Allowed Threshold | Status |
| :--- | :---: | :---: | :---: |
| 🔴 Critical | 7 | 10 | ✅ OK |
| 🟠 High | 166 | 200 | ✅ OK |
| 🟡 Medium | 45 | - | ℹ️ Info |
| 🔵 Low | 12 | - | ℹ️ Info |

## ✅ Security Quality Gate: PASSED
The product meets all security quality gate thresholds. Deployment approved.
```

* **Para aplicaciones en producción:** Mantén `max_critical: '0'` y `max_high: '0'` para garantizar tolerancia cero a vulnerabilidades graves.
* **Para aplicaciones en migración o con deuda técnica conocida:** Puedes ajustar los límites numéricos para permitir el paso (`max_critical: '10'`) o usar `fail_on_security_gate: false` para auditoría informativa no bloqueante.

---

## Mecanismo de autenticación dinámica DAST (auth-hook.sh)

Si tu aplicación contiene zonas protegidas por login, añade un script ejecutable llamado **`auth-hook.sh`** en la raíz del repositorio cliente.

El orquestador invocará automáticamente este script pasando como argumentos el host y puerto local del contenedor (`./auth-hook.sh 127.0.0.1 <PUERTO>`). El script debe devolver por salida estándar (`stdout`) la cabecera HTTP formateada como:

```text
NOMBRE_CABECERA|VALOR_CABECERA
```

### Ejemplo A: Cookie de Sesión Formulario (WebGoat)

```bash
#!/bin/bash
HOST=$1
PORT=$2

RESPONSE=$(curl -s -i -X POST "http://$HOST:$PORT/WebGoat/login" \
  -d "username=guest&password=guest")

COOKIE=$(echo "$RESPONSE" | grep -i "Set-Cookie" | grep -o "JSESSIONID=[^;]*")

if [ -n "$COOKIE" ]; then
  echo "Cookie|$COOKIE"
else
  echo "DummyHeader|DummyValue"
fi
```

### Ejemplo B: Token Bearer / JWT (APIs REST / Juice Shop)

```bash
#!/bin/bash
HOST=$1
PORT=$2

# Autenticación vía API REST
RESPONSE=$(curl -s -X POST "http://$HOST:$PORT/rest/user/login" \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@juice-sh.op","password":"admin"}')

TOKEN=$(echo "$RESPONSE" | jq -r '.authentication.token // empty')

if [ -n "$TOKEN" ]; then
  echo "Authorization|Bearer $TOKEN"
else
  echo "DummyHeader|DummyValue"
fi
```

---

## Requisitos y restricciones del proyecto

1. **Estructura del Proyecto:**
   * **Java:** Debe incluir un archivo `pom.xml` válido en la raíz.
   * **Node.js:** Debe incluir un archivo `package.json` en la raíz. Admite monorepos o SPAs en subcarpetas (ej. `frontend/package.json`), compilando automáticamente Angular/React.
2. **`Dockerfile` en la Raíz:**
   * Imprescindible para empaquetar la aplicación, generar el contenedor de prueba en GHCR y ejecutar los análisis DAST y de imágenes.
3. **Respuesta en el Endpoint de Salud (*Healthcheck*):**
   * Al levantarse, el contenedor debe responder con un código HTTP válido (`2xx`, `3xx` o `4xx`) en la URL `http://127.0.0.1:<app-port><app-path>` dentro de los primeros 120 segundos. Si el contenedor falla al arrancar, el orquestador volcará automáticamente los logs de Docker para facilitar la depuración.

---

## Visualización de resultados en DefectDojo

Una vez concluida la pipeline en GitHub Actions:

1. **Gestión Unificada de Vulnerabilidades:**  
   Accede a DefectDojo > **Products** > `[Tu defectdojo_product_name]`.  
   Verás los *Engagements* creados automáticamente para cada disciplina (`semgrep`, `trivy`, `gitleaks`, `kics`, `checkov`, `CodeQL`, `zap`, `nikto`, etc.).
2. **Inventario de Componentes y Licencias (SBOM):**  
   Accede a la pestaña **Components** dentro de tu producto en DefectDojo para auditar todas las dependencias directas y transitivas indexadas a partir de CycloneDX.

---

## Solución de problemas frecuentes

* **El Quality Gate falla por vulnerabilidades existentes:** Configura `max_critical` y `max_high` en tu `call_orchestrator.yaml` con valores superiores al número actual de hallazgos, o desactiva el bloqueo con `fail_on_security_gate: false`.
* **Error de conexión al subir a DefectDojo:** Comprueba que `DEFECTDOJO_URL` sea accesible públicamente y **no termine en barra diagonal `/`**.
* **Timeout en el sondeo de salud DAST:** Comprueba en el log del job si el contenedor ha caído al iniciar y revisa que `app-port` y `app-path` coincidan exactamente con la configuración de tu aplicación.
* **Error 403 en SonarCloud:** Verifica que el secreto `SONAR_TOKEN` esté configurado y coincida con el proyecto registrado en SonarCloud.
