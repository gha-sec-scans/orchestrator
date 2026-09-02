# Guía de despliegue local de DefectDojo y conexión con GitHub Actions

Esta guía describe paso a paso cómo levantar una instancia local de **OWASP DefectDojo** mediante Docker Compose, recuperar las credenciales iniciales de administrador, generar el token de la API v2, exponer el servicio a Internet de forma segura mediante **Cloudflare Tunnel** y configurar los secretos requeridos en los repositorios de GitHub para que el orquestador pueda enviar los reportes de vulnerabilidades y evaluar el Quality Gate.

---

## Tabla de contenidos

- [1. Introducción y arquitectura](#1-introducción-y-arquitectura)
- [2. Requisitos previos](#2-requisitos-previos)
- [3. Despliegue de DefectDojo con Docker Compose](#3-despliegue-de-defectdojo-con-docker-compose)
- [4. Obtención de la contraseña de administrador](#4-obtención-de-la-contraseña-de-administrador)
- [5. Acceso a la interfaz web y generación de la API v2 Key](#5-acceso-a-la-interfaz-web-y-generación-de-la-api-v2-key)
- [6. Exposición pública con Cloudflare Tunnel (cloudflared)](#6-exposición-pública-con-cloudflare-tunnel-cloudflared)
- [7. Configuración de secretos en GitHub](#7-configuración-de-secretos-en-github)
- [8. Comprobación de conectividad](#8-comprobación-de-conectividad)
- [9. Parada y reanudación del servicio](#9-parada-y-reanudación-del-servicio)

---

## 1. Introducción y arquitectura

Los runners de GitHub Actions se ejecutan en máquinas virtuales efímeras en la nube de GitHub. Para que las acciones del orquestador (`defectdojo-upload` y `quality-gate`) puedan interactuar con tu DefectDojo local, es imprescindible establecer un canal de comunicación HTTPS público.

```text
[Runner de GitHub Actions]
       │
       │ HTTPS REST API
       ▼
[Cloudflare Tunnel (trycloudflare.com)]
       │
       │ Tráfico local seguro
       ▼
[Docker: DefectDojo (http://localhost:8080)]
```

---

## 2. Requisitos previos

* **Docker Desktop** instalado y en ejecución (en Windows, con backend WSL2 activado).
* **Git** instalado.
* Conexión a Internet activa.

---

## 3. Despliegue de DefectDojo con Docker Compose

1. Abre un terminal (PowerShell o Bash) y clona el repositorio oficial de DefectDojo:
   ```bash
   git clone https://github.com/DefectDojo/django-DefectDojo.git
   cd django-DefectDojo
   ```

2. Levanta los contenedores con Docker Compose:
   ```bash
   docker compose up -d
   ```


3. Comprueba que todos los contenedores estén levantados y saludables:
   ```bash
   docker compose ps
   ```
   El servicio web quedará expuesto por defecto en el puerto **8080** de tu máquina (`http://localhost:8080`).

---

## 4. Obtención de la contraseña de administrador

Durante el primer arranque, el contenedor `initializer` genera una contraseña aleatoria y segura para el usuario `admin`.

Para recuperarla, ejecuta el siguiente comando en el terminal dentro de la carpeta `django-DefectDojo`:

### En Windows (PowerShell):
```powershell
docker compose logs initializer | Select-String "Admin password:"
```

### En Linux / macOS / Git Bash:
```bash
docker compose logs initializer | grep "Admin password:"
```

La salida mostrará una línea similar a:
```text
Admin password: a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6
```

> **Nota:** Guarda esta contraseña. El usuario por defecto siempre es **`admin`**.

---

## 5. Acceso a la interfaz web y generación de la API v2 Key

1. Abre tu navegador web y entra a:
   ```text
   http://localhost:8080
   ```
2. Inicia sesión con:
   * **Username:** `admin`
   * **Password:** *(La contraseña obtenida en el paso anterior)*

3. Para obtener el token de API:
   * En la esquina superior derecha, haz clic en el icono de tu perfil de usuario (**admin**).
   * En el menú desplegable, selecciona **API v2 Key** (o navega directamente a `http://localhost:8080/api/key-v2`).
   * Verás una cadena alfanumérica de 40 caracteres (por ejemplo: `9a8b7c6d5e4f3a2b1c0d9e8f7a6b5c4d3e2f1a0b`).
   * Si no aparece ningún token, pulsa el botón **Generate API Key**.
   * **Copia este token**, ya que será el valor del secreto `DEFECTDOJO_TOKEN`.

---

## 6. Exposición pública con Cloudflare Tunnel (cloudflared)

Para que los runners de GitHub Actions puedan enviar reportes a tu servidor local sin necesidad de abrir puertos en tu router ni configurar IP dinámica, utiliza **Cloudflare Tunnel** (*Quick Tunnels* gratuito).


```bash
docker run --rm -it --net=host cloudflare/cloudflared:latest tunnel --url http://localhost:8080
```

### Obtención de la URL pública:
En la consola verás una línea similar a la siguiente:
```text
+--------------------------------------------------------------------------------------------+
|  Your quick Tunnel has been created! Visit it at (it may take some time to be reachable):  |
|  https://random-generated-subdomain.trycloudflare.com                                      |
+--------------------------------------------------------------------------------------------+
```

Copia esa URL.

---

## 7. Configuración de secretos en GitHub

Los secretos **únicamente deben registrarse en el repositorio consumidor** (desde donde se invoca el pipeline, por ejemplo `webgoat` o `juice-shop`) o a nivel de **Organización de GitHub**.

### Dónde añadirlos:
1. Ve a tu repositorio consumidor en GitHub (ej. `webgoat` o `juice-shop`).
2. Entra en la pestaña **Settings**.
3. En la barra lateral izquierda, despliega **Secrets and variables** > selecciona **Actions**.
4. Pulsa el botón **New repository secret**.

### Nombres y valores exactos:

| Nombre del Secreto | Obligatorio | Valor que debes introducir | Ejemplo de valor |
| :--- | :---: | :--- | :--- |
| **`DEFECTDOJO_URL`** | **Sí** | La URL HTTPS pública generada por Cloudflare Tunnel (**sin barra al final**). | `https://alpha-bravo-charlie.trycloudflare.com` |
| **`DEFECTDOJO_TOKEN`** | **Sí** | El token de la API v2 obtenido en el perfil de admin de DefectDojo. | `9a8b7c6d5e4f3a2b1c0d9e8f7a6b5c4d3e2f1a0b` |
| **`SONAR_TOKEN`** | Opcional | Token de autenticación de SonarCloud (solo si se utiliza el análisis de Sonar). | `sqp_1234567890abcdef...` |


---

## 8. Comprobación de conectividad

Antes de lanzar el pipeline de GitHub Actions, puedes verificar desde cualquier terminal que el túnel y la API de DefectDojo respondan correctamente.

Ejecuta este comando sustituyendo tu URL y tu Token:

```bash
curl -i -H "Authorization: Token TU_DEFECTDOJO_TOKEN" "https://TU_URL_CLOUDFLARE.trycloudflare.com/api/v2/products/"
```

* **Respuesta esperada:** Un código `HTTP/2 200 OK` (o `HTTP/1.1 200 OK`) con un JSON que lista los productos existentes:
  ```json
  {"count":0,"next":null,"previous":null,"results":[]}
  ```
* Si recibes `401 Unauthorized`, revisa que el token copiado sea exacto y que no contenga espacios.
* Si recibes `502 Bad Gateway`, comprueba que DefectDojo esté corriendo localmente en el puerto 8080.

---

## 9. Parada y reanudación del servicio

* **Para detener DefectDojo sin perder datos:**
  ```bash
  cd django-DefectDojo
  docker compose stop
  ```
  Los volúmenes de Docker mantendrán intactas las bases de datos, los hallazgos cargados y los usuarios.

* **Para volver a iniciarlo en cualquier momento:**
  ```bash
  cd django-DefectDojo
  docker compose start
  ```
  Y vuelve a levantar el comando de `cloudflared` para obtener la URL pública (si la URL cambia, actualiza únicamente el secreto `DEFECTDOJO_URL` en GitHub).
