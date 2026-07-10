<a id="autenticacion-jwt"></a>

# 🧩 4. Autenticación con JWT

!!! warning "🚧 Contenido pendiente de desarrollo"
    Esta página todavía no tiene la teoría redactada. Usa el prompt de más abajo con
    `/improve-notes`, apoyándote en el proyecto **GameVault** adjunto, para generar el
    contenido definitivo.

---

## Prompt para `/improve-notes`

```text
Redacta el apartado de teoría "Autenticación con JWT" del Tema 2 (RA5 - Programación
segura) del módulo Programación de Servicios y Procesos (0490), semana real 10 del
calendario. Sigue las convenciones de estilo del README.md del repo.

Criterios de evaluación de RA5 que cubre este apartado (curriculum.md):
- f) Métodos para asegurar la información transmitida.
- g) Aplicaciones que utilizan comunicaciones seguras para la transmisión de
  información.

ESTRUCTURA — teoría primero: antes del JWT concreto, explica desde cero el problema
general de "recordar quién eres" en un protocolo sin estado como HTTP (cada petición
llega sola, sin memoria de la anterior) y las dos soluciones históricas comparadas: la
SESIÓN en el servidor (el servidor guarda quién eres y te da una cookie con el número de
sesión — cómo funciona y sus límites) frente al TOKEN autocontenido (el servidor no
guarda nada: te firma un "carné" con tus datos y tú lo presentas en cada petición).
Explica qué significa "firmado" retomando la criptografía de la semana anterior: firmar
no oculta el contenido, garantiza que no se ha modificado y quién lo emitió.

Contenido central: sustituir HTTP Basic (credenciales en cada petición) por JWT (un
token firmado que se obtiene una vez en el login y se presenta en las siguientes
peticiones). Este es el estado final de la referencia adjunta.

Apóyate en el proyecto GameVault (com.aleroig.gamevault, paquete `seguridad`) como
ejemplo real:
- Anatomía de un JWT con un token real de ejemplo: header.payload.signature en Base64,
  decodificable (¡el contenido NO va cifrado!) pero no falsificable (la firma HMAC con
  el secreto). Conecta con la criptografía de la semana anterior: firmar ≠ cifrar.
- JwtService.java: cómo se genera el token en el login (claims: subject, roles,
  expiración) usando el JwtEncoder.
- AuthController.java + LoginRequestDTO/LoginResponseDTO: el endpoint público
  POST /api/v1/auth/login — recibe credenciales, las verifica con el
  AuthenticationManager contra los usuarios BCrypt de la semana anterior, y devuelve el
  token.
- SecurityConfig.java, la parte JWT: `oauth2ResourceServer(oauth2 -> oauth2.jwt(...))`,
  los beans JwtEncoder/JwtDecoder con el secreto compartido
  (`@Value("${gamevault.jwt.secret}")` — principio de la semana 7: el secreto vive en la
  configuración, no en el código), el JwtAuthenticationConverter que convierte el claim
  "roles" en autoridades ROLE_*, `SessionCreationPolicy.STATELESS` (por qué con JWT no
  hay sesión en el servidor) y el `httpBasic(disable)` que retira oficialmente el
  mecanismo provisional de las semanas 8-9.
- AuthMeController.java (GET /api/v1/auth/me): el endpoint que devuelve quién eres según
  tu token — útil para probar y para entender qué información viaja dentro.
- HTTPS (criterio f y, sobre todo, g): el JWT evita reenviar la contraseña en cada
  petición, pero el canal en sí sigue viajando en claro si no hay TLS — la firma
  garantiza integridad del token, no confidencialidad del transporte. El criterio g
  pide "aplicaciones que UTILICEN comunicaciones seguras", no solo la mención de que
  existen: por eso esta página no puede quedarse en 3-4 frases teóricas sobre HTTPS.
  Incluye un paso práctico mínimo y real, no solo teoría: generar un certificado
  autofirmado (`keytool -genkeypair`, comando dado), configurar
  `server.ssl.key-store`/`server.ssl.key-store-password`/`server.ssl.key-store-type` en
  `application-dev.yaml` (o un perfil `https` aparte para no romper el resto del curso
  en HTTP), levantar GameVault sirviendo por `https://localhost:8443` y hacer una
  petición con curl (`-k` para aceptar el certificado autofirmado) o Postman contra el
  login y contra un endpoint protegido. Explica también, en 3-4 frases, por qué en
  producción real se usaría un certificado de una CA reconocida o TLS terminado en un
  proxy/gateway (Nginx, un balanceador), y por qué en el resto del curso se sigue
  trabajando en HTTP simple: para no complicar cada actividad con certificados, dejando
  claro que es una simplificación deliberada de entorno de aprendizaje, no la
  recomendación para producción.

Cierra recordando la conexión con AD: este Principal/JWT es el que usará el PUT de
reseñas con control de autoría (AD, semana real 16).
```
