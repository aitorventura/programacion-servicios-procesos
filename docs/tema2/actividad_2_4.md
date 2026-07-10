# 🧪 Actividad 2.4: Login con JWT

!!! warning "🚧 Contenido pendiente de desarrollo"
    Esta actividad todavía no está redactada. Usa el prompt de más abajo con
    `/improve-notes`, apoyándote en el proyecto **GameVault** adjunto, para generar el
    enunciado definitivo.

---

## Prompt para `/improve-notes`

```text
Redacta la Actividad 2.4 del Tema 2 (RA5 - Programación segura) del módulo Programación
de Servicios y Procesos (0490), semana real 10 del calendario. Si necesita
plantilla/solución en .docx, crea antes la skill de plantilla de PSP clonando
/actividad-plantilla-acceso-a-datos con la paleta marrón/ámbar.

IMPORTANTE — enfoque: es una PRÁCTICA GUIADA, no un reto. El alumnado trabaja sobre su
propia copia de GameVault. El enunciado debe guiar paso a paso, mostrando el código y
explicando cada decisión; solo se deja sin guiar, como mini-reto, lo que repita un
patrón idéntico ya mostrado.

Objetivo (RA5, criterios f, g): que el alumnado implemente en su GameVault el login con
JWT tal y como existe en la referencia (paquete com/aleroig/gamevault/seguridad/),
sustituyendo el HTTP Basic provisional de las actividades 2.2-2.3.

Estructura sugerida de pasos guiados:
1. El secreto JWT en la configuración (`gamevault.jwt.secret` en application-dev.yaml,
   con nota sobre por qué no se sube un secreto real al repositorio) y los beans
   JwtEncoder/JwtDecoder en SecurityConfig, guiados con el código de la referencia
   mostrado y explicado.
2. JwtService guiado: generación del token con sus claims (subject, roles, expiración) —
   código mostrado, cada claim explicado.
3. AuthController + LoginRequestDTO/LoginResponseDTO guiados: el endpoint público
   POST /api/v1/auth/login que verifica credenciales con el AuthenticationManager y
   devuelve el token — recordar abrir la ruta en la política de SecurityConfig
   (permitAll solo para el login).
4. El cambio de modo en SecurityConfig, guiado: oauth2ResourceServer con el
   JwtAuthenticationConverter (claim "roles" → ROLE_*), sesión STATELESS, y retirar
   httpBasic — explicando qué sustituye a qué respecto a la actividad anterior.
5. Prueba guiada del flujo completo con curl (comandos dados): login → copiar el token →
   petición autenticada con `Authorization: Bearer ...` → misma petición sin token (401).
   Decodificar juntos el payload del token (comando dado) para ver los claims.
6. Mini-reto (repite el patrón del paso 5): probar GET /api/v1/auth/me con el token
   (añadiendo antes AuthMeController de la referencia si no lo tienen) y explicar qué
   devuelve y de dónde sale.
7. Pregunta de comprensión: el payload del JWT se puede decodificar sin conocer el
   secreto — ¿por qué entonces el servidor confía en él? ¿Qué detectaría si alguien
   modificara el claim "roles" a mano?
```
