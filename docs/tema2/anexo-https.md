<a id="anexo-https"></a>

# 📎 Anexo: HTTPS en desarrollo

En [Autenticación con JWT](autenticacion-jwt.md) viste la distinción clave: la firma de un JWT garantiza **integridad** (nadie lo ha modificado), pero no **confidencialidad** — si el canal no está cifrado, cualquiera que intercepte el tráfico puede leer tanto las credenciales del login como el propio token, tal cual viajan. HTTPS es lo que cierra ese hueco, cifrando la conexión completa con TLS. Aquí ves cómo se monta, paso a paso.

## 🔐 De HTTP a HTTPS: qué cambia

HTTP manda los datos en texto plano: cualquiera con acceso a la red —un proxy, un router intermedio, alguien en la misma red wifi— puede leer el contenido de cada petición y respuesta. **TLS** (*Transport Layer Security*, el protocolo detrás de HTTPS) añade una capa de cifrado entre el cliente y el servidor: antes de que viaje ni un byte de tu petición HTTP, cliente y servidor negocian una clave de cifrado compartida, válida solo para esa conexión. A partir de ahí, todo el tráfico —cabeceras, cuerpo, cookies, tokens— va cifrado; alguien que intercepte los paquetes solo ve datos ilegibles.

Para que el cliente confíe en que está hablando con el servidor correcto —y no con un impostor haciéndose pasar por él—, el servidor presenta un **certificado**: un documento digital que asocia una clave pública con una identidad (normalmente un dominio), firmado por alguien de confianza. Ese "alguien de confianza" es una **autoridad certificadora** (*CA*, *Certificate Authority*) — entidades reconocidas cuyo certificado raíz ya viene preinstalado en tu sistema operativo y en tu navegador. Cuando visitas una web con el candado, tu navegador ha comprobado que el certificado del servidor está firmado por una CA que ya conoce y en la que confía.

## 🔏 Certificado autofirmado: la versión de desarrollo

En desarrollo no tiene sentido pedir un certificado a una CA real: cuestan dinero, y exigen demostrar que controlas un dominio público, algo que no tienes en `localhost`. La alternativa es un **certificado autofirmado**: generas tú mismo el par de claves y firmas el certificado, sin que ninguna CA lo avale. Cliente y servidor cifran la conexión exactamente igual que con un certificado real, pero el navegador (o `curl`) no tiene forma de verificar que ese certificado es de fiar —nadie reconocido lo ha firmado—, así que muestra un aviso de "conexión no segura". Es la señal correcta y esperada para un certificado autofirmado, no un error de configuración.

`keytool` es la herramienta de gestión de claves y certificados que viene incluida con el propio JDK — no hace falta instalar nada aparte:

```bash
keytool -genkeypair -alias miapp -keyalg RSA -keysize 2048 \
  -storetype PKCS12 -keystore keystore.p12 -validity 365
```

| Opción | Qué indica |
|---|---|
| `-genkeypair` | Genera un par de claves (pública/privada) y un certificado autofirmado que las acompaña |
| `-alias miapp` | Nombre con el que identificar esta entrada dentro del almacén de claves — puede haber varias |
| `-keyalg RSA -keysize 2048` | Algoritmo y tamaño de la clave — RSA de 2048 bits es el mínimo razonable hoy |
| `-storetype PKCS12` | Formato del almacén de claves — PKCS12 es el estándar, soportado nativamente por Java |
| `-keystore keystore.p12` | Fichero donde se guardan el par de claves y el certificado |
| `-validity 365` | Días que el certificado será válido antes de caducar |

El comando pide una contraseña para proteger el fichero `keystore.p12` (el **keystore**, el almacén donde vive la clave privada) y algunos datos de identidad (nombre, organización...) que en desarrollo no importan demasiado.

## ⚙️ Configurar Spring Boot para servir por HTTPS

Con el `keystore.p12` ya generado, Spring Boot solo necesita saber dónde está y cómo abrirlo:

```yaml
server:
  ssl:
    key-store: classpath:keystore.p12
    key-store-password: tu-contraseña
    key-store-type: PKCS12
  port: 8443
```

`key-store` apunta al fichero (aquí, dentro del classpath, normalmente `src/main/resources/`); `key-store-password` es la misma contraseña que le diste a `keytool`; `key-store-type` tiene que coincidir con el formato que generaste (`PKCS12`). El puerto `8443` es una convención —podrías dejarlo en `8080`—, pero es habitual reservar `8443` para HTTPS y `8080` para HTTP: así el propio número ya indica qué protocolo esperar.

!!! warning "Con esta configuración, el servidor deja de responder por HTTP"
    En cuanto activas `server.ssl`, el único puerto que arranca es el HTTPS: el servidor embebido no sirve HTTP y HTTPS a la vez por defecto, a menos que configures explícitamente un segundo conector (algo más avanzado, fuera de este anexo). Cualquier petición a `http://` deja de tener respuesta desde ese momento; hay que usar `https://` en todas partes.

## ✅ Probarlo

Arranca la aplicación y prueba el login por HTTPS:

```bash
curl -k -X POST https://localhost:8443/api/v1/auth/login \
  -H "Content-Type: application/json" -d '{"username":"admin","password":"admin123"}'
```

`-k` (o `--insecure`) le dice a `curl` que acepte el certificado autofirmado sin intentar validarlo contra ninguna CA conocida. Sin `-k`, `curl` rechazaría la conexión con un error de certificado, porque no reconoce quién lo ha firmado — exactamente el mismo motivo por el que un navegador muestra su aviso de "conexión no segura" y obliga a aceptarlo a mano antes de dejarte continuar.

## 🌍 Y en producción

Un certificado autofirmado nunca es aceptable en producción: cualquier usuario real vería el aviso de "conexión no segura", indistinguible a simple vista de un ataque real. Ahí se usa uno de estos dos caminos:

- Un certificado emitido por una **autoridad certificadora reconocida** —hoy en día, a menudo gratis y automatizado, con servicios como Let's Encrypt—, que se instala igual que el autofirmado pero sin generar ningún aviso en el navegador, porque esa CA ya es de confianza para todo el mundo.
- Delegar el TLS a un **proxy o gateway** que se coloca delante de la aplicación (Nginx, un balanceador de carga, un servicio gestionado en la nube): ese proxy termina la conexión cifrada con el cliente, y le habla a la aplicación por HTTP simple dentro de una red ya considerada de confianza. Es el patrón más habitual en despliegues reales — la propia aplicación ni siquiera llega a gestionar certificados.
