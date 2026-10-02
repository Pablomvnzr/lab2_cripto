# Laboratorio 2 — Criptografía y Seguridad en Redes

**Alumno:** Pablo Muñoz R  
**Sección:** 4  
**Fecha:** Octubre de 2026  

## Descripción

Este laboratorio utiliza **Damn Vulnerable Web Application (DVWA)** para estudiar ataques de diccionario sobre un formulario de autenticación y comparar el tráfico HTTP generado por **Burp Suite, cURL e Hydra**.

Las pruebas se realizaron en un entorno local con Docker, utilizando el módulo `/vulnerabilities/brute/` de DVWA con el nivel de seguridad **Low**.

## Herramientas utilizadas

- Docker Desktop y Docker Compose.
- DVWA y MariaDB.
- Burp Suite Community Edition.
- cURL.
- Hydra 9.7.
- Wireshark.
- Firefox y el navegador integrado de Burp.

## Actividades realizadas

### 1. Despliegue de DVWA

Se levantaron los servicios de DVWA y MariaDB mediante Docker Compose:

```bash
cd DVWA
docker compose up -d
docker compose ps
```

La aplicación se publicó con la siguiente configuración:

```yaml
ports:
  - 127.0.0.1:4280:80
```

Esto permite acceder a DVWA en `http://127.0.0.1:4280`, limitando el acceso al equipo local.

### 2. Ataque de diccionario con Burp Suite

Se capturó una petición del formulario Brute Force y se envió a Intruder. Se seleccionaron los parámetros `username` y `password` como posiciones de payload y se utilizó **Cluster bomb** con dos diccionarios de diez entradas.

Se comprobaron al menos dos pares válidos:

| Usuario | Contraseña |
|---|---|
| admin | password |
| gordonb | abc123 |

El éxito se confirmó mediante el mensaje de bienvenida del HTML. El código HTTP 200 y la longitud de la respuesta se utilizaron como información complementaria, no como prueba suficiente de autenticación.

### 3. Reproducción de peticiones con cURL

Se obtuvo el comando mediante **Copy as cURL** desde las herramientas de desarrollador del navegador. Se ejecutaron un acceso válido y otro inválido, conservando las cabeceras y la cookie de sesión.

Las respuestas se compararon mediante:

```bash
diff -u invalido.html valido.html
wc -c valido.html invalido.html
```

Se observaron diferencias en:

- El mensaje del resultado.
- Las etiquetas HTML utilizadas.
- La presencia de la imagen del usuario.
- El tamaño del documento: **4547 bytes** para el acceso válido y **4509 bytes** para el inválido.

Ambos intentos devolvieron HTTP 200.

### 4. Ataque de diccionario con Hydra

Se utilizó el módulo `http-get-form`, los mismos diccionarios y una cookie de sesión vigente. Se definió el mensaje de bienvenida como condición de éxito.

Hydra reportó cinco pares válidos:

| Usuario | Contraseña |
|---|---|
| admin | password |
| gordonb | abc123 |
| 1337 | charley |
| pablo | letmein |
| smithy | password |

Se validaron manualmente dos de estos pares en el formulario de DVWA.

### 5. Análisis de tráfico

Se capturó el tráfico de cada herramienta por separado en la interfaz `lo0` de Wireshark.

**Filtro de captura:**

```text
tcp port 4280
```

**Filtro de visualización:**

```text
http.request && tcp.port == 4280
```

Las peticiones observadas presentaron estas diferencias:

| Característica | cURL | Burp Intruder | Hydra |
|---|---|---|---|
| Versión HTTP | HTTP/1.1 | HTTP/1.1 | HTTP/1.0 |
| User-Agent | Firefox, definido en el comando | Conservado de Chromium | Mozilla/5.0 (Hydra) |
| Accept-Encoding | deflate, gzip | gzip, deflate, br | Ausente |
| Referer | Formulario con parámetros anteriores | Ruta del formulario | Ausente |
| Sec-Fetch-* | Presente | Presente | Ausente |

Estas características corresponden a las ejecuciones del laboratorio. Las cabeceras pueden modificarse, por lo que no permiten identificar con certeza una herramienta por sí solas.

## Exploración del directorio de imágenes

Después de abrir la imagen de un acceso válido, se eliminó `admin.jpg` de la URL para acceder a `/hackable/users/`. El servidor respondió con **403 Forbidden**, ya que el listado automático estaba deshabilitado.

Se habilitó el listado mediante `Options +Indexes` en Apache, utilizando acceso administrativo al contenedor. Esto fue un **cambio de configuración**, no un bypass remoto.

El listado permitió observar cinco nombres candidatos a usuarios. Utilizarlos con las diez contraseñas reduciría el espacio de búsqueda de 100 a 50 combinaciones. Las ejecuciones documentadas utilizaron la lista original de diez usuarios.

## Archivos principales

| Archivo o carpeta | Contenido |
|---|---|
| `DVWA/` | Repositorio de la aplicación utilizada en el laboratorio. |
| `usuarios.txt` | Diccionario de diez usuarios candidatos. |
| `passwords.txt` | Diccionario de diez contraseñas. |
| `resultados_hydra.txt` | Resultados guardados por Hydra. |
| `informe_lab2/informe.tex` | Fuente LaTeX del informe. |
| `informe_lab2/imagenes/` | Capturas utilizadas como evidencia. |
| `Informe_Lab2_Overleaf.zip` | Proyecto del informe para importar en Overleaf. |

## Compilación del informe

Para compilar localmente con Tectonic:

```bash
cd informe_lab2
tectonic informe.tex
```

También se puede importar `Informe_Lab2_Overleaf.zip` en Overleaf y seleccionar `informe.tex` como documento principal.

## Conclusiones

La experiencia mostró que es necesario revisar el contenido de las respuestas para distinguir accesos válidos e inválidos. También permitió comprobar que las sesiones deben mantenerse vigentes para que las herramientas alcancen el formulario vulnerable.

El análisis con Wireshark evidenció que HTTP transmite las credenciales y cookies sin cifrado, y que la identificación de software requiere combinar varios indicios en lugar de depender únicamente del User-Agent.

## Referencias

- [DVWA](https://github.com/digininja/DVWA)
- [Burp Suite Community Edition](https://portswigger.net/burp/communitydownload)
- [cURL](https://curl.se/docs/)
- [THC Hydra](https://github.com/vanhauser-thc/thc-hydra)
- [Wireshark](https://www.wireshark.org/docs/)
