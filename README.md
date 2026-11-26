# webapp-orchestrator

Skill para Claude Code que **analiza una aplicación web, crea un plan detallado por fases y lo ejecuta con un equipo jerárquico de subagentes especializados**, ajustando el trabajo al uso de tokens que le queda a tu plan.

Está inspirada en [REA (reverse-engineer-anything)](https://github.com/morluto/rea), en concreto en su forma de separar lo observado, lo inferido y lo desconocido. Su gran diferencia es que no se queda en el análisis: lo convierte en un plan y reparte el trabajo entre agentes especializados.

---

## ¿Qué hace?

Le pides algo amplio sobre una web ("analiza mi tienda y mejórala", "se ve fatal en el móvil", "hazme una web parecida a esta") y la skill:

1. **Entiende el objetivo** y elige el modo:
   - **Mejorar**: hay código del proyecto y se modifica.
   - **Reconstruir**: solo hay una URL y se crea un proyecto nuevo.
2. **Consulta el presupuesto** de tokens (límite de 5 horas y semanal) y a qué hora se reinicia.
3. **Analiza el código.** No hace capturas: el código contiene la misma información y cuesta mucho menos. El análisis se divide por áreas.
4. **Crea un plan por fases.** Cada tarea tiene un solo rol, una zona de archivos exclusiva, un coste estimado, el modelo que la hará y unos criterios de aceptación comprobables.
5. **Te pide aprobación** antes de tocar nada.
6. **Ejecuta fase a fase** con subagentes y comprueba la build y los tests antes de pasar a la siguiente.
7. **Cierra** con una verificación visual de QA (el único momento con capturas) y un informe final.

---

## El equipo

```
                 DIRECTOR (la conversación principal)
                 decide qué se hace con el uso que queda y reparte el trabajo
                /                         \
        VIGÍA                          ESPECIALISTAS
  solo informa del uso del plan    análisis · desarrollo · estilos · qa · calidad
  y de cuándo se reinicia          cada uno con su parte única del trabajo
```

| Agente | Función | Modelo por defecto |
|---|---|---|
| **Director** | Decide, reparte y verifica. Hace él mismo solo lo trivial. | el de tu sesión |
| **Vigía** | Lee el uso del plan y la hora de reinicio. Nada más. | Haiku, esfuerzo bajo |
| **Análisis** | Lee el código de su área (solo lectura). | Haiku |
| **Desarrollo** | Lógica, componentes, rutas, estado, APIs y bugs. | Sonnet |
| **Estilos** | CSS, diseño, responsive y tokens de diseño. | Sonnet |
| **Calidad** | Accesibilidad WCAG AA, SEO y rendimiento. | Haiku |
| **QA** | Tests y verificación final en el navegador. | Haiku |

### Semáforo de presupuesto

Antes de planificar y antes de cada fase, el director consulta al vigía y aplica el **peor** de los dos indicadores:

| Nivel | 5 h | Semanal | Qué hace |
|---|---|---|---|
| 🟢 Verde | < 60 % | < 75 % | Plan completo, con paralelismo. |
| 🟡 Ámbar | 60–85 % | 75–90 % | Solo tareas importantes, sin paralelismo y con modelos más baratos. |
| 🔴 Rojo | > 85 % | > 90 % | Termina la tarea en curso, guarda el estado y te dice cuándo se reinicia el límite. |

---

## Ahorro de tokens

- **Caveman.** Todos los subagentes se comunican con la skill [`caveman`](#requisitos) (frases cortas, sin relleno). Esto solo afecta a sus informes: el código, los textos de la web y los documentos se escriben con normalidad.
- **Cada agente lee solo su parte**: su archivo de análisis, su tarea y sus archivos exactos.
- **Lectura precisa**: buscar antes de leer y leer fragmentos, no archivos enteros.
- **Tests dirigidos** durante el trabajo. La batería completa se ejecuta solo al final.
- **Sin capturas** hasta la verificación final, y a media resolución.
- **Modelos baratos** para las tareas sencillas.
- **Sin agentes innecesarios**: las tareas pequeñas se agrupan y las correcciones se piden al mismo agente.

---

## Archivos que genera

Todo el trabajo queda documentado en `.orquesta/`, en la raíz del proyecto:

```
.orquesta/
├── resumen.md          # stack, comandos y carpetas (lo comparten todos)
├── analisis/
│   ├── desarrollo.md
│   ├── estilos.md
│   ├── calidad.md
│   └── qa.md
├── plan.md             # fases, tareas, estado y registro de decisiones
└── informe-final.md
```

---

## Instalación

**Desde el archivo `.skill`:** ábrelo en la app de Claude y pulsa **Save skill**.

**A mano (Claude Code):** copia la carpeta `webapp-orchestrator/` en `~/.claude/skills/` (para tu usuario) o en `.claude/skills/` dentro de un proyecto.

```
webapp-orchestrator/
├── SKILL.md                      # flujo completo
├── README.md
└── references/
    ├── briefs-agentes.md         # encargos de cada rol
    └── plantillas.md             # plantillas de .orquesta/
```

## Requisitos

- **Claude Code** con herramienta de subagentes. Sin ella, el director hace cada tarea él mismo siguiendo el encargo de su rol.
- **Skill `caveman`** (plugin `caveman`). Si no está disponible, los agentes escriben igualmente en estilo telegráfico.
- **Navegador integrado** de la app de escritorio, para la verificación final y para el modo Reconstruir.
- **Herramienta de uso del plan** (`get_usage`, en la app de escritorio). Si no existe, la skill trabaja en ámbar por precaución.

## Ejemplos de uso

```
analiza mi web de la tienda y mejórala, quiero que quede profesional y sin errores
```
```
mis clientes dicen que la web se ve fatal en el móvil, arregla solo eso
```
```
he visto esta web http://ejemplo.com, hazme una parecida para mi negocio en la carpeta nueva-web
```

## Resultados de las pruebas (iteración 1)

Con 3 casos de prueba sobre una tienda de ejemplo con fallos sembrados:

| Caso | Con skill | Sin skill |
|---|---|---|
| Mejora completa | 9/9 | 4/9 |
| Solo móvil | 6/6 | 4/6 |
| Reconstruir desde URL | 6/6 | 4/6 |

Con la skill, en los tres casos consultó el presupuesto, analizó sin capturas, separó el análisis por áreas y pidió aprobación antes de tocar código. Estas pruebas llegan hasta el plan: la ejecución completa por fases todavía no se ha evaluado.
