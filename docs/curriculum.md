# Currículo: Módulo 0490 — Programación de Servicios y Procesos

**Equivalencia ECTS:** 5  
**Duración:** 40 horas

---

## Resultados de Aprendizaje y Criterios de Evaluación

### RA1 — Programación Multiproceso
> Desarrolla aplicaciones compuestas por varios procesos reconociendo y aplicando principios de programación paralela.

- a) Se han reconocido las características de la programación concurrente y sus ámbitos de aplicación.
- b) Se han identificado las diferencias entre programación paralela y programación distribuida, sus ventajas e inconvenientes.
- c) Se han analizado las características de los procesos y de su ejecución por el sistema operativo.
- d) Se han caracterizado los hilos de ejecución y descrito su relación con los procesos.
- e) Se han utilizado clases para programar aplicaciones que crean subprocesos.
- f) Se han utilizado mecanismos para compartir información con los subprocesos iniciados.
- g) Se han utilizado mecanismos para sincronizar y obtener el valor devuelto por los subprocesos iniciados.
- h) Se han desarrollado aplicaciones que gestionen y utilicen procesos para la ejecución de varias tareas en paralelo.
- i) Se han depurado y documentado las aplicaciones desarrolladas.

### RA2 — Programación Multihilo
> Desarrolla aplicaciones compuestas por varios hilos de ejecución analizando y aplicando librerías específicas del lenguaje de programación.

- a) Se han identificado situaciones en las que resulte útil la utilización de varios hilos en un programa.
- b) Se han reconocido los mecanismos para crear, iniciar y finalizar hilos.
- c) Se han programado aplicaciones que implementen varios hilos.
- d) Se han identificado los posibles estados de ejecución de un hilo y programado aplicaciones que los gestionen.
- e) Se han utilizado mecanismos para compartir información entre varios hilos de un mismo proceso.
- f) Se han desarrollado programas formados por varios hilos sincronizados mediante técnicas específicas.
- g) Se ha establecido y controlado la prioridad de cada uno de los hilos de ejecución.
- h) Se han depurado y documentado los programas desarrollados.
- i) Se ha analizado el contexto de ejecución de los hilos.
- j) Se han analizado librerías específicas del lenguaje de programación que permiten la programación multihilo.
- k) Se han reconocido los problemas derivados de la compartición de información entre los hilos de un mismo proceso.

### RA3 — Comunicación en Red mediante Sockets
> Programa mecanismos de comunicación en red empleando sockets y analizando el escenario de ejecución.

- a) Se han identificado escenarios que precisan establecer comunicación en red entre varias aplicaciones.
- b) Se han identificado los roles de cliente y de servidor y sus funciones asociadas.
- c) Se han reconocido librerías y mecanismos del lenguaje de programación que permiten programar aplicaciones en red.
- d) Se ha analizado el concepto de socket, sus tipos y características.
- e) Se han utilizado sockets para programar una aplicación cliente que se comunique con un servidor.
- f) Se ha desarrollado una aplicación servidor en red y verificado su funcionamiento.
- g) Se han desarrollado aplicaciones que utilizan sockets para intercambiar información.
- h) Se han utilizado hilos para posibilitar la comunicación simultánea de varios clientes con el servidor.
- i) Se han caracterizado los modelos de comunicación más usuales en las arquitecturas de aplicaciones distribuidas.
- j) Se han depurado y documentado las aplicaciones desarrolladas.

### RA4 — Servicios en Red
> Desarrolla aplicaciones que ofrecen servicios en red, utilizando librerías de clases y aplicando criterios de eficiencia y disponibilidad.

- a) Se han identificado diferentes protocolos estándar de comunicación para la implementación de servicios en red.
- b) Se han reconocido las ventajas de la utilización de protocolos estándar para la comunicación entre aplicaciones y procesos.
- c) Se han analizado librerías que permitan implementar servicios en red utilizando protocolos estándar de comunicación.
- d) Se han desarrollado y probado servicios de comunicación en red.
- e) Se han utilizado clientes de comunicaciones para verificar el funcionamiento de los servicios.
- f) Se han incorporado mecanismos para posibilitar la comunicación simultánea de varios clientes con el servicio.
- g) Se ha verificado la disponibilidad del servicio.
- h) Se han depurado y documentado las aplicaciones desarrolladas.

### RA5 — Programación Segura
> Protege las aplicaciones y los datos definiendo y aplicando criterios de seguridad en el acceso, almacenamiento y transmisión de la información.

- a) Se han identificado y aplicado principios y prácticas de programación segura.
- b) Se han analizado las principales técnicas y prácticas criptográficas.
- c) Se han definido e implantado políticas de seguridad para limitar y controlar el acceso de los usuarios a las aplicaciones desarrolladas.
- d) Se han utilizado esquemas de seguridad basados en roles.
- e) Se han empleado algoritmos criptográficos para proteger el acceso a la información almacenada.
- f) Se han identificado métodos para asegurar la información transmitida.
- g) Se han desarrollado aplicaciones que utilicen comunicaciones seguras para la transmisión de información.
- h) Se han depurado y documentado las aplicaciones desarrolladas.

---

## Contenidos Básicos

### Programación multiproceso (→ RA1)
- Ejecutables. Procesos. Servicios.
- Estados de un proceso. Planificación de procesos.
- Hilos.
- Programación concurrente.
- Programación paralela y distribuida.
- Comunicación entre procesos.
- Gestión de procesos. Herramientas de monitorización.
- Sincronización entre procesos.
- Programación de aplicaciones multiproceso.

### Programación multihilo (→ RA2)
- Contexto de ejecución de los hilos. Recursos compartidos.
- Estados de un hilo. Cambios de estado.
- Librerías y clases.
- Gestión de hilos. Prioridades.
- Sincronización de hilos.
- Compartición de información entre hilos. Problemas.
- Programación de aplicaciones multihilo.

### Programación de comunicaciones en red (→ RA3)
- Comunicación entre aplicaciones. Modelos.
- Roles cliente y servidor.
- Librerías y clases.
- Sockets. Tipos. Características.
- Creación de sockets.
- Enlazado y establecimiento de conexiones.
- Utilización de sockets para la transmisión y recepción de información.
- Programación de aplicaciones cliente y servidor.
- Utilización de hilos para la implementación de comunicaciones simultáneas con el servidor.

### Generación de servicios en red (→ RA4)
- Protocolos estándar de comunicación en red a nivel de aplicación. Ventajas de su utilización.
- Servicios web.
- Librerías de clases y componentes.
- Programación de servidores.
- Establecimiento y finalización de conexiones.
- Transmisión de información.
- Implementación de comunicaciones simultáneas.
- Utilización de aplicaciones clientes.
- Monitorización del servicio. Herramientas.

### Utilización de técnicas de programación segura (→ RA5)
- Prácticas de programación segura.
- Criptografía de clave pública y clave privada.
- Principales aplicaciones de la criptografía.
- Protocolos criptográficos.
- Política de seguridad. Roles.
- Programación de mecanismos de control de acceso.
- Encriptación de información.
- Protocolos seguros de comunicaciones.
- Programación de aplicaciones con comunicaciones seguras.