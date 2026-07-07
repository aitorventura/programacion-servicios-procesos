# Plantilla base — sitio MkDocs Material para un módulo

Esta plantilla contiene todo lo imprescindible para arrancar un nuevo módulo con la misma
base técnica y visual (índigo/teal, MkDocs Material) que el proyecto original. La idea es
que a partir de aquí solo tengas que ir añadiendo temas y contenido, sin tocar la configuración.

## Puesta en marcha

```bash
pip install -r requirements.txt
mkdocs serve
```

Abre `http://127.0.0.1:8000` y verás la portada de ejemplo.

1. Cambia `site_name` en `mkdocs.yml` por el nombre real del módulo.
2. Sustituye `docs/curriculum.md` por el currículo oficial (ver instrucciones dentro del fichero).
3. Edita `docs/index.md` con la portada real.
4. Renombra/edita `docs/tema1/` con el contenido del primer tema, y añade `tema2/`, `tema3/`... a
   medida que avances, replicando siempre la misma estructura de carpetas.
5. Cambia el nombre y el año en `overrides/partials/copyright.html`.

## Qué incluye

```
mkdocs.yml              — configuración completa (tema, extensiones markdown, plugins)
requirements.txt         — mkdocs + mkdocs-material + mkdocs-pdf
.gitignore               — excluye soluciones del profesor y la carpeta site/
overrides/partials/      — pie de página con licencia CC BY-NC-SA
docs/
  index.md               — portada, con placeholders
  curriculum.md           — plantilla del currículo (RA + criterios + contenidos básicos)
  css/extra.css           — estilos: pestañas coloreadas (.tabs-colored), footer, marco de imágenes
  tema1/
    index.md              — índice de tema (RA, criterios, contenidos, actividades)
    ejemplo-apartado.md    — plantilla de página de teoría
    actividad_1_1.md       — plantilla de actividad
    plantillas/            — aquí van los .docx que descargan los alumnos
    img/                   — capturas e imágenes de este tema
    diapositivas/          — PDF (y opcionalmente PPTX) de las diapositivas de cada apartado
```

## Estructura por tema (repítela para cada tema nuevo)

Cada tema es una carpeta `docs/temaN/` con:

- `index.md` — cabecera con el RA, lista de criterios de evaluación (✅ una por línea) y el
  índice de apartados + actividades, en el orden en que se deben estudiar.
- Un `.md` por apartado de teoría, con el embed del PDF de diapositivas al principio.
- Un `actividad_N_M.md` por actividad, intercalado en el `nav` de `mkdocs.yml` justo después
  del apartado de teoría al que corresponde.
- `plantillas/`, `img/`, `diapositivas/` — mismos nombres en todos los temas, para que todo
  el sitio sea uniforme.

Recuerda añadir cada página nueva al bloque `nav:` de `mkdocs.yml`, o no aparecerá en el menú
lateral aunque el fichero exista.

## Convenciones de estilo (para que el contenido nuevo case con el resto)

**Pestañas:** envuélvelas siempre en `<div class="tabs-colored" markdown>` — sin el wrapper
no tienen color:

```markdown
<div class="tabs-colored" markdown>

=== "🔵 Opción A"
    Contenido A.

=== "🟢 Opción B"
    Contenido B.

</div>
```

**Diagramas Mermaid:** usa `flowchart LR/TD` con nodos simples + emoji. No uses `classDef` ni
colores personalizados en los nodos: rompe la consistencia visual del sitio.

**Admonitions:** `!!! info` para definiciones, `!!! tip` para matices, `!!! warning` para
errores frecuentes, `!!! example` para casos concretos. No satures: 2-3 seguidos como máximo.

**Densidad:** no más de 4-5 líneas de texto seguido sin un elemento visual (tabla, diagrama,
admonition, código o pestañas). Tampoco encadenes elementos visuales sin una frase que los
introduzca.

**Formas verbales:** pretérito perfecto compuesto ("ha coincidido", "has fallado"), no
pretérito indefinido ("coincidió", "fallaste"), salvo hechos históricos con fecha concreta.

**Soluciones del profesor:** nunca van en `docs/` (se publicarían en la web). Genera los
`.docx`/`.pptx` de soluciones y los scripts que los generan en una carpeta de trabajo aparte,
fuera del repo (por ejemplo `C:\Users\TuUsuario\docxgen`), y usa el patrón de nombre
`Actividad_X_Y_Solucion.docx`. Las plantillas en blanco que sí ve el alumno van dentro del
repo, en `docs/temaX/plantillas/`.

**Actividades a prueba de IA:** contexto personal e irrepetible, razonamiento explícito
obligatorio, preguntas de "qué pasaría si", errores deliberados para detectar, comparación
justificada de alternativas. El objetivo es que un alumno no pueda aprobar usando IA sin
entender la materia.
