# PRD · optimizacion-athos

```
Versión: 1.1
Fecha: 2026-09-03
Estado: aprobado por el usuario el 2026-09-03, con las aclaraciones de dominio del Anexo
Motor de build: Claude Code
Autor: Yared González Pérez
```

---

## 1. Resumen ejecutivo · *capa QUÉ*

**Un piloto que propone, con reglas deterministas y auditables, los mín/máx de cada
dispensador ATHOS-Dosys, y comprueba después si esas propuestas funcionaron en el armario
real.**

Lo usan Yared González y Alfredo Montero como coordinadores del proceso, sobre 3-4 unidades
del HUNSC (unos 5-7 armarios). Lee las extracciones de inventario, movimientos y ofertas
que hoy se descargan a mano de los aplicativos internos, las cruza con un catálogo de
reenvasados y con la capacidad física de cada hueco, y entrega un libro Excel donde el
farmacéutico responsable acepta, modifica o rechaza cada propuesta línea a línea. Está
escrito en Python y no tiene interfaz: se ejecuta desde la línea de comandos.

Lo que lo distingue de abrir una hoja de cálculo es que **cada número viene con su
justificación numérica**, que **ninguna propuesta supera lo que cabe físicamente en el
hueco**, y que el resultado se mide después contra el histórico de movimientos para saber
si hubo roturas. Una hoja de cálculo produce una cifra; esto produce una cifra defendible y
un veredicto sobre si acertó.

**La v1 no es la herramienta: es la prueba de que el método funciona.** El producto para
farmacéuticos no técnicos es la v2, está decidido, y vive en §12.

---

## 2. Problema, usuario y éxito · *capa QUÉ*

### Problema

Los mín/máx de los dispensadores ATHOS-Dosys se fijaron en su día y se ajustan por
intuición y a demanda: cuando algo se rompe, se sube; cuando sobra, a veces se baja y a
veces no. Nadie tiene una vista de qué armario está mal configurado ni por qué. El coste se
paga tres veces: en roturas de stock que obligan a enfermería a pedir por el circuito
alternativo, en medicación inmovilizada en armarios que no la consumen, y en trabajo del
técnico que repone huecos que no hacía falta reponer.

Hay además un problema físico que ningún sistema recoge: **hay medicamentos que
simplemente no caben** en el hueco que tienen asignado. Ese dato vive en la cabeza del
técnico que repone, y por eso las propuestas de mín/máx hechas sobre papel se caen al
llegar al armario.

### Solución actual

Ajuste manual, sin registro del criterio, directamente en el software de APD Solutions. No
hay histórico de por qué un mínimo es 12 y no 18, así que cada revisión empieza de cero y
cada persona aplica su propio criterio.

### Usuario

- **Yared González Pérez**, farmacéutico hospitalario del HUNSC, coordinador de Fardosys.
  Es quien ejecuta el piloto: descarga las extracciones, corre los scripts y revisa la
  salida. Tiene criterio técnico y no le asusta la línea de comandos.
- **Alfredo Montero**, coordinador del proceso. Recibe y valida resultados; no ejecuta nada.
- **Farmacéutico responsable de cada armario**, destinatario de la salida. Recibe el libro
  Excel y decide línea a línea. **No es usuario del software en la v1**: es usuario del
  fichero que el software produce.
- **Técnico de reposición**, fuente de un dato y no usuario. Aporta dónde no cabe qué.

### Métricas de éxito

Las tres se miden sobre las unidades piloto, comparando el periodo anterior a aplicar las
propuestas con el posterior.

| # | Métrica | Objetivo | De dónde sale el dato |
|---|---|---|---|
| **M1** | Aplicabilidad física: propuestas que caben en su hueco | **100 %**, cero excepciones | Base de huecos (§7 M5). Una excepción es un fallo del motor, no del armario |
| **M2** | Reposición ejecutada sin ajuste manual sobre lo propuesto | **≥ 90 %** de las líneas | Se contrasta el histórico digital de reposiciones con la hoja de reposición que el técnico anota a mano al hacer el pedido. **Dato manual**, no lo calcula el software |
| **M3** | Roturas de stock en las unidades piloto | **Reducción ≥ 80 %** frente al periodo previo de igual duración | Histórico de movimientos: línea en la que la cantidad hallada en el cajetín es 0 |
| **M4** | Referencias a reponer por dispensador y visita | **Entre 8 y 10** | Calculado: `Σ CMD ÷ (máx − mín)` sobre las referencias del armario |

**Los cuatro umbrales están confirmados por el usuario** (02/09/2026). M1 y M4 son objetivos
de diseño, comprobables sobre el papel antes de aplicar nada; M2 y M3 solo se saben después.

> **Cómo se mide M2 de verdad, confirmado por el usuario el 03/09/2026.** No hay un campo
> digital que diga "se ajustó o no". La señal real vive en la hoja de reposición en papel que
> el técnico rellena al hacer el pedido del ATHOS: círculos, tachones y cantidades corregidas
> a mano sobre lo que proponía el sistema. M2 se obtiene comparando esa hoja con la propuesta,
> y es un trabajo manual de quien coordina el piloto — el software no lo genera solo.
>
> La misma hoja sirve además de **evidencia directa de M1**: una cantidad tachada porque "no
> cabe" es una propuesta que violó la capacidad del hueco. El usuario aportó un ejemplo real
> —dispensador ATHOS 3, LEVETIRACETAM 500 mg comp or/sonda (`V15084`), cantidad propuesta 78,
> anotación a mano "BASE J 2S3S pequeño para tantos comprimidos"— que es exactamente el caso
> que cubre REQ-041/REQ-059: el hueco J de la sección 2S3S no cabe físicamente esa cantidad,
> y el técnico lo corrige a mano porque el sistema de origen (APD) no conoce esa restricción.
> Es la prueba de campo de por qué la base de huecos (M5) no es un accesorio.

**El objetivo último de M3 es cero: que no se llegue nunca a cajetín 0.** El −80 % es el
umbral con el que se declara que el piloto funcionó, no la meta. Por eso el informe
**siempre da el número absoluto de roturas junto al porcentaje**, y por eso la lista de
roturas que quedan es parte del entregable: mientras haya una, hay algo que mirar.

> Dar siempre el absoluto resuelve además, sin necesidad de umbrales extra, el problema de
> los números pequeños: "de 5 roturas a 1" se lee tal cual y cada uno juzga si eso es una
> mejora o es ruido. Un "−80 %" a secas no se puede juzgar.

**La pregunta de los tres meses:** ¿cómo sabremos si funcionó? Porque en las unidades
piloto habrá menos líneas a cero en el histórico de movimientos que en el periodo anterior,
y porque el farmacéutico responsable habrá aceptado las propuestas sin reescribirlas.

---

## 3. Alcance · *capa QUÉ*

### Dentro de la v1

1. `[Must]` Guarda de datos de paciente: detección, aviso y aborto sin escribir nada.
2. `[Must]` Lectura y normalización de las extracciones de **inventario** (ubicación,
   mín/máx vigentes, consumo medio, existencias).
3. `[Must]` Lectura y normalización del **histórico de movimientos**.
4. `[Must]` **Maestro de artículos persistente** en Excel, que se actualiza en cada
   ejecución con lo detectado al cruzar datos: tipo de artículo, unidad, existencias,
   consumo.
4b. `[Must]` **Catálogo de ofertas e informe de compras**: vinculan cada artículo con su
   código nacional, su laboratorio y su precio, y determinan qué oferta está vigente.
5. `[Must]` **Catálogo de reenvasados**: por medicamento, se reenvasa / se etiqueta / ya
   viene en unidosis. Solo comprimidos y cápsulas.
6. `[Must]` **Base de huecos**: qué cantidad de qué medicamento no cabe en qué sección y
   hueco, usada como **restricción dura** del cálculo.
7. `[Must]` **Motor de optimización** de mín/máx con reglas deterministas, cada propuesta
   con su justificación numérica.
8. `[Must]` **Libro Excel de salida**: una pestaña por fichero crudo, una pestaña de
   acciones integrada, una de estadística y una de validación.
9. `[Must]` **Ingesta de la validación** devuelta por el farmacéutico, con registro de la
   decisión por línea.
10. `[Should]` **Medición de resultados**: detección de roturas en el histórico y
    comparación antes/después por dispensador.
11. `[Should]` Comparación del consumo medio del dispensador con el consumo global del
    artículo.
12. `[Could]` Valoración económica: importe inmovilizado por armario, vigente frente a
    propuesto.

### Fuera de alcance — lista contractual

| Fuera | Motivo |
|---|---|
| Cualquier interfaz web o gráfica | La v1 es un piloto que ejecuta una persona. La interfaz es el objeto de la v2 |
| Escribir en el software de APD Solutions | Decisión del usuario: el producto vive aparte y no toca el sistema de origen |
| Aplicar propuestas automáticamente | El producto propone; nada sale sin validación humana |
| Datos de pacientes, en cualquier forma | Regla dura del usuario. No es "pendiente", es prohibido |
| La extracción de calidad ATHOS y las prescripciones no disponibles (IND-1) | El usuario las dejó fuera del piloto. IND-1 es además la extracción con más riesgo de arrastrar identificadores |
| Importar datos de Frello / FarDosys | Decisión del usuario: empezar limpio, tomada ya sabiendo que allí hay 47 dispensadores y 6.414 pactos de stock |
| Multiusuario, autenticación, aislamiento | Usuario único en la v1. Todo eso es problema de la v2 |
| IA, correo, pagos, tareas programadas, API externas | No hacen falta: entradas locales y método determinista |
| Registro de producción de reenvasados | El usuario acotó reenvasados a catálogo. El registro de producción es v2 |
| Detección de oportunidades de cambio de proveedor | Eso es compras, no optimización de armario: otro proceso y otros interlocutores |
| Los 47 dispensadores | Contradice la decisión de validar sobre dispensadores concretos |
| **Material y otros (`DM`)** | No se pone en el dispensador. Fuera del alcance entero, no solo del reenvasado |
| **Sueros y grandes volúmenes** (p. ej. paracetamol intravenoso) | No van al armario |
| **Lectura automática de la hoja de reposición en papel** (OCR de anotaciones a mano) | M2 se contrasta a mano por quien coordina el piloto (§2). Automatizarlo es un cambio de arquitectura que no se ha pedido — la hoja es evidencia de campo, no una entrada del sistema |

**UFA no está aquí a propósito.** Se planteó como exclusión y el usuario lo retiró: los
medicamentos de UFA (Unidad de Farmacia Ambulatoria) y los de onco-hematología **entran en
el análisis** y se marcan con la propiedad `ambito` del catálogo, para poder segmentarlos.
Clasificar conserva la información; excluir la tira.

### Qué significa "la v1 está terminada"

Las nueve fases de §11 cerradas **y** un ciclo completo recorrido de punta a punta sobre al
menos una unidad piloto real: extracciones cargadas → libro generado → farmacéutico
responsable validado → decisiones ingestadas → medición ejecutada con dos periodos.

Ojo a la distinción, porque se confunden: **la v1 está terminada cuando el ciclo funciona;
el piloto ha tenido éxito cuando se cumplen M1 a M4 de §2.** Son dos cosas distintas y
pueden darse por separado. Es perfectamente posible entregar una v1 terminada cuyo veredicto
sea que el método no mejora nada — y ese también es un resultado válido del piloto, no un
fracaso del producto.

---

## 4. Decisiones técnicas · *capa CÓMO*

| Decisión | Elección | Por qué | Alternativas descartadas |
|---|---|---|---|
| Motor de build | Claude Code | Piloto local; el usuario ejecuta y revisa. No hay despliegue ni usuarios externos | Lovable (no hay interfaz que construir en la v1); dejarlo por decidir |
| Lenguaje | Python 3.11+ | Es el lenguaje del análisis tabular, y lo que el usuario declaró | R (menos común en el entorno); VBA (no versionable ni testeable) |
| Manipulación de datos | pandas | Estándar de facto para cruces y agregación | Polars (más rápido, sin ventaja al volumen previsto y con menos ejemplos públicos) |
| Lectura/escritura Excel | openpyxl | Escribe libros multi-pestaña con formato, que es exactamente la salida pedida | xlsxwriter (no lee); pandas a secas (no controla el formato de las pestañas) |
| Interfaz | Línea de comandos (`argparse`) | Un solo usuario técnico. Cualquier interfaz en la v1 es trabajo que la v2 tira | Notebook Jupyter (no reproducible: el estado depende del orden de ejecución) |
| Persistencia | **Ninguna base de datos. Excel maestro único, actualizado en sitio** | **Decisión expresa del usuario:** todo abrible y editable sin herramientas nuevas | ⚠ **SQLite fue la recomendación técnica y el usuario la descartó conociendo el motivo:** un maestro editado en sitio no tiene historial y un fallo a mitad de escritura lo corrompe. **Mitigación acordada:** copia fechada antes de cada escritura (REQ-021) y negativa a ejecutar si el fichero está abierto (REQ-022). Supabase descartado: contradice "piloto local" y subiría datos del hospital a la nube antes de validar el método |
| Arquitectura | Núcleo puro separado de la E/S | Permite que la v2 ponga una interfaz encima en vez de reescribir el cálculo | Script monolítico: más rápido hoy, reescritura completa en la v2 |
| Método de cálculo | Reglas deterministas y auditables | Cada propuesta se defiende número a número ante el farmacéutico | Predictivo/series temporales (no defendible con pocos armarios); LLM para redactar la justificación (mete un tercero donde no hace falta y cambia el bloque de cumplimiento) |
| Autenticación | No aplica | Un solo usuario, ejecución local, sin red | — |
| Hosting | No aplica: el equipo del usuario | El producto no se despliega en la v1 | — |
| Secretos | **No hay.** Ninguna clave, ningún servicio externo | Ver §10 | — |
| Tests | pytest, obligatorios en el núcleo | El núcleo es lo único que la v2 hereda; sin pruebas no es reutilizable, es un borrador | Verificación manual (no repetible) |

---

## 5. Arquitectura y flujo · *capa CÓMO*

```
  datos/entrada/          ┌──────────────┐
  ┌──────────────┐        │   GUARDAS    │  ← corre SIEMPRE primero.
  │ inventario/  │───────▶│  detección   │    Si detecta identificador de
  │ movimientos/ │        │  de paciente │    paciente: ABORTA y no se
  │ ofertas/     │        └──────┬───────┘    escribe NADA, en ningún sitio
  │ compras/     │               │ pasa
  └──────────────┘               │
                                 ▼
                          ┌──────────────┐
                          │   LECTORES   │  perfiles de lectura declarados:
                          │ Excel → tabla│  qué columna de origen es qué campo
                          └──────┬───────┘
                                 │
        datos/catalogos/         ▼
        ┌──────────────┐  ┌──────────────┐
        │ maestro.xlsx │─▶│    TABLAS    │── rechazos ──▶ datos/salida/
        │ · ARTICULOS  │  │ normalizadas │                rechazos_<fecha>.xlsx
        │ · OFERTAS    │  └──────┬───────┘
        │ · REENVASADOS│         │
        │ · HUECOS     │         │
        │ · PROPUESTAS │         ▼
        └──────▲───────┘  ╔══════════════════════╗
               │          ║       NÚCLEO         ║  funciones puras
               │          ║  reglas de mín/máx   ║  · sin rutas
               │          ║  restricción de hueco║  · sin pd.read_*
               │          ║  justificación       ║  · misma entrada →
               │          ╚══════════┬═══════════╝    misma salida
               │                     │
               │                     ▼
               │              ┌─────────────┐
               │              │  INFORMES   │
               │              └──────┬──────┘
               │                     ▼
               │    datos/salida/propuestas_<unidad>_<fecha>.xlsx
               │    ├── CRUDO_<origen>  (una por fichero de entrada)
               │    ├── ACCIONES        ← lo que hay que hacer, integrado
               │    ├── ESTADISTICA     ← el resumen del armario
               │    └── VALIDACION      ← columnas vacías para el farmacéutico
               │                     │
               │                     ▼  el farmacéutico rellena y devuelve
               │              ┌─────────────┐
               └──────────────│  INGESTA DE │  decisión por línea:
                              │  VALIDACIÓN │  aceptada / modificada / rechazada
                              └─────────────┘
```

### Recorrido completo de la operación principal

Todo ocurre en la máquina del usuario. **No hay frontera cliente/servidor porque no hay
red:** ni una sola llamada sale del equipo. Esa es la razón de que §10 no tenga sección de
secretos.

1. El usuario descarga las extracciones de los aplicativos internos y las deja en
   `datos/entrada/<tipo>/`, sin renombrarlas ni editarlas.
2. Ejecuta `python -m src.cli procesar --unidad <unidad>`.
3. **Guardas.** Se inspeccionan cabeceras y una muestra de cada fichero. Si aparece un
   indicador de paciente, el proceso aborta aquí: no se escribe intermedio, ni salida, ni
   se toca el maestro. El aviso nombra fichero y columna, **nunca el contenido**.
4. **Lectores.** Cada fichero se normaliza según su perfil. Las filas no convertibles se
   apartan a un fichero de rechazos y se cuentan; no se descartan en silencio.
5. **Maestro.** Antes de tocarlo se copia a
   `datos/intermedio/copias/maestro_<fecha-hora>.xlsx`. Si está abierto en Excel, el
   proceso aborta y lo dice. Después se dan de alta los CN nuevos y se actualizan los
   campos cambiados.
6. **Núcleo.** Recibe tablas en memoria y devuelve propuestas: mínimo, máximo,
   justificación numérica y el resultado de aplicar la restricción de capacidad del hueco.
7. **Informes.** Se escribe el libro de salida con sus pestañas.
8. El usuario reparte el libro. El farmacéutico responsable rellena `VALIDACION` y lo
   devuelve.
9. `python -m src.cli validar --fichero <ruta>` registra las decisiones. **Aquí termina el
   ciclo del producto:** aplicar el cambio en el sistema de APD lo hace una persona, a
   mano, y el producto no se entera hasta verlo en el inventario del mes siguiente.
10. Pasado un ciclo, `python -m src.cli medir --unidad <u> --desde <d> --hasta <h>` compara
    roturas antes y después.

### Dónde viven los secretos

**En ninguna parte: no hay.** El producto no llama a ningún servicio externo, no tiene
claves de API y no necesita credenciales. Si durante el build aparece la necesidad de una
clave, eso es un cambio de arquitectura y pasa por `/cambio`.

---

## 6. Modelo de datos · *capa CÓMO*

No hay base de datos: el modelo se materializa en libros Excel y en tablas en memoria. Esta
especificación es tan vinculante como lo sería un `CREATE TABLE`, y los lectores la hacen
cumplir.

### Entidades y relaciones

```
  Unidad 1───n Dispensador 1───n Ubicación (sección, hueco)
                    │                    │
                    │                    └──0..1── RestricciónCapacidad
                    │                              (cuánto NO cabe)
                    n
              PactoVigente ──n──1── Artículo ──1──n── Oferta ──1──0..1── Reenvasado
                    │                  │              (cn)
                    │                  ├──n── Movimiento
                    │                  └──n── ConsumoGlobal
                    │
                    └────────▶ Propuesta ──▶ Validación

  Artículo se identifica por codigo_bdm. Su prefijo (V/Y/DM/T) fija tipo_articulo.
  Solo los que tienen oferta llegan al CN, y por tanto al precio.
```

### `maestro.xlsx` — persistente, se actualiza en sitio

Clave natural: **el código interno del Servicio Canario de Salud** (proyecto BDM) —
REQ-023. **No es el código nacional.**

El **prefijo del código clasifica el artículo**, y esa clasificación es dato, no adorno:

| Prefijo | Ejemplo | Tipo de artículo | ¿Tiene CN y oferta? | ¿Aplica reenvasado? |
|---|---|---|---|---|
| `V` | `V00210` | Medicamento | Sí | Si es comprimido o cápsula |
| `Y` | `Y80879` | Fórmula magistral interna | **No**: se elabora en el servicio | `[PENDIENTE]`: no tiene CN, y la pestaña se indexa por CN |
| `DM` | `dm000116` | Material y otros | **No** | **Fuera del piloto: no se pone en el armario** |
| `T` | `T80502` | Medicamento extranjero | **Sí** — código propio del proveedor extranjero, no un CN nacional registrado. Confirmado 03/09/2026 | **Sí**, si es comprimido o cápsula — igual que `V`. Confirmado 03/09/2026 |

**Entran en el piloto `V`, `Y` y `T`.** `DM` queda fuera. Si aparece un prefijo distinto de
esos cuatro, **se pregunta al usuario**: no se clasifica por parecido (REQ-026).

> **Por qué esto importa más de lo que parece.** Tres consecuencias que cambian el
> comportamiento del producto, no solo su vocabulario:
> 1. **Un armario no contiene solo medicamentos.** El material (`DM`) también ocupa hueco y
>    también se repone, así que también tiene mín/máx que optimizar — pero el catálogo de
>    reenvasados no le aplica en absoluto, igual que no aplica a un vial.
> 2. **Las fórmulas magistrales (`Y`) no tienen código nacional ni oferta**, luego no tienen
>    precio por esa vía. Cualquier cálculo de importe inmovilizado (REQ-091) las deja fuera
>    o las valora por otro camino — y el informe tiene que **decir cuántas quedaron fuera**,
>    no callarlo. Es exactamente el motivo por el que REQ-091 exige informar el porcentaje
>    de cobertura junto al importe.
> 3. **El caso `dm000116` viene en minúsculas** y los demás en mayúsculas. La clave se
>    normaliza a mayúsculas antes de comparar, o el mismo artículo aparecerá dos veces en el
>    maestro y nadie entenderá por qué.

El código nacional vive un nivel por debajo, en las ofertas: **cada artículo con oferta
tiene una o varias**, y es cada oferta la que lleva su CN, su laboratorio y su precio.

```
   codigo_bdm  ────1:n────▶  Oferta (cn, laboratorio, presentación, precio)
       │                            ▲
       │                            └── el CN vive AQUÍ, no en el maestro
       └── es lo que el inventario y los movimientos referencian
```

> ⚠ **Consecuencia sobre el reenvasado, confirmada por el usuario el 03/09/2026:** que algo
> venga ya en unidosis depende de **cómo lo envasa el laboratorio que ganó la oferta**, no
> del principio activo. Si cambia la oferta adjudicada —y un motivo real de cambio es una
> rotura de stock que obliga a cambiar de proveedor— el mismo artículo puede pasar de
> `ya_unidosis` a `se_reenvasa` sin que nadie haya tocado nada. Por eso `REENVASADOS` se
> indexa aquí por **CN** —la presentación concreta, no el código BDM— y por eso el informe de
> compras importa: es lo que dice qué oferta está viva ahora mismo. **Confirmado: el
> reenvasado sí cambia con el proveedor**, así que la clave por CN es definitiva, no una
> simplificación pendiente de validar.

**Pestaña `ARTICULOS`** — una fila por código BDM

| Campo | Tipo | Oblig. | Notas |
|---|---|---|---|
| `codigo_bdm` | texto | **sí** | Clave natural, normalizada a mayúsculas. Sin duplicados — REQ-023 |
| `tipo_articulo` | **valor cerrado** | sí | `medicamento` \| `formula_magistral` \| `material` \| `medicamento_extranjero`. **Derivado del prefijo**, no se teclea — REQ-026 |
| `descripcion` | texto | sí | |
| `principio_activo` | texto | no | Vacío en material y en buena parte de las fórmulas |
| `unidad` | texto | sí | Unidad de dispensación — base del redondeo de REQ-052 |
| `forma_farmaceutica` | valor cerrado | no | Determina si aplica reenvasado — REQ-031 |
| `ambito` | **valor cerrado** | sí | `onco_hemato` \| `ufa` \| `general`. **Clasifica, no excluye** — REQ-029 |
| `cn_vigente` | texto | no | CN de la oferta activa hoy. **Derivado** de `OFERTAS`; no se teclea |
| `precio_unitario_vigente` | decimal | no | Derivado de la oferta activa. Base de REQ-091 |
| `existencias_globales` | entero | no | Se actualiza en cada carga |
| `consumo_global_periodo` | entero | no | Base de la comparación de REQ-090 |
| `activo` | booleano | sí | `no` cuando el código deja de aparecer en las fuentes. **Nunca se borra la fila** — ver reglas de borrado |
| `primera_vez_visto` | fecha | sí | Se escribe al dar de alta y no se toca más |
| `ultima_actualizacion` | fecha-hora | sí | |

**Pestaña `OFERTAS`** — una fila por oferta. Es donde el artículo se une al mundo económico

| Campo | Tipo | Oblig. | Notas |
|---|---|---|---|
| `codigo_bdm` | texto | **sí** | → `ARTICULOS.codigo_bdm` |
| `cn` | texto | **sí** | Código nacional de esta presentación, o —si el artículo es `medicamento_extranjero`— el código propio que trae el archivo de ofertas del proveedor extranjero. No es un CN registrado en España, pero cumple la misma función de clave dentro de `OFERTAS`. Confirmado 03/09/2026 |
| `laboratorio` | texto | sí | |
| `presentacion` | texto | sí | Tamaño de envase y formato |
| `precio` | decimal | sí | |
| `unidades_por_envase` | entero | sí | Necesario para llevar el precio a precio por unidad |
| `vigente` | booleano | sí | Qué oferta está viva. **Lo determina el informe de compras**, no el catálogo de ofertas |
| `fecha_alta` | fecha | sí | |

Clave natural: `(codigo_bdm, cn)`. Un artículo con varias ofertas es lo normal, no una
anomalía.

**Pestaña `REENVASADOS`** — REQ-030

| Campo | Tipo | Oblig. | Notas |
|---|---|---|---|
| `cn` | texto | **sí** | → `OFERTAS.cn`. Es la presentación concreta lo que se reenvasa o no |
| `codigo_bdm` | texto | sí | Redundante a propósito: permite leer la pestaña sin cruzar |
| `tratamiento` | **valor cerrado** | sí | `se_reenvasa` \| `se_etiqueta` \| `ya_unidosis`. No hay un cuarto valor |
| `maquina` | texto | no | `[PENDIENTE: Unibot vs. Unibox/Unipack]` |
| `observaciones` | texto | no | |

> **Ámbito del campo**, y ahora tiene dos condiciones, no una:
> 1. Solo **comprimidos y cápsulas**. Para ampollas, viales e inyectables **no aplica** — no
>    es que esté vacío pendiente de rellenar.
> 2. Solo artículos de tipo `medicamento` (prefijo `V`) **y también** `medicamento_extranjero`
>    (`T`) — confirmado 03/09/2026: los extranjeros sí pueden reenvasarse. El **material
>    (`DM`) queda fuera por definición**: no es medicamento, no se reenvasa. Las **fórmulas
>    magistrales (`Y`)** no tienen CN, así que no pueden indexarse aquí — `[PENDIENTE: ¿se
>    reenvasan las fórmulas? Si sí, esta pestaña necesita aceptar también el código BDM como
>    clave alternativa.]`
>
> El validador no exige el campo fuera de ese ámbito y la estadística no cuenta esos
> artículos como sin clasificar (REQ-031).

**Pestaña `HUECOS`** — la base de huecos. REQ-040

| Campo | Tipo | Oblig. | Notas |
|---|---|---|---|
| `dispensador` | texto | **sí** | |
| `seccion` | texto | **sí** | |
| `hueco` | texto | **sí** | |
| `codigo_bdm` | texto | **sí** | El artículo concreto. Incluye material (`DM`): también ocupa hueco |
| `cn` | texto | no | La presentación con la que se midió. La capacidad depende del envase: si cambia la oferta, este dato puede haber caducado |
| `cantidad_maxima` | entero | **sí** | Lo que cabe de verdad. **Restricción dura** — REQ-041 |
| `informado_por` | texto | sí | Quién lo dijo. Sin esto el dato no se puede contrastar |
| `fecha` | fecha | sí | |

Clave natural: `(dispensador, seccion, hueco, codigo_bdm)`. Sin duplicados.

> Guardar el `cn` con el que se midió no es redundancia: si la oferta cambia a un envase
> distinto, **la capacidad medida deja de ser fiable** y el sistema puede avisarlo en vez de
> seguir usando un número que ya no vale.

**Pestaña `EXCLUSIONES`** — lo que no va al armario y por tanto no se optimiza. REQ-028

| Campo | Tipo | Oblig. | Notas |
|---|---|---|---|
| `codigo_bdm` | texto | **sí** | |
| `motivo` | **valor cerrado** | sí | `material` \| `gran_volumen` \| `otro` |
| `detalle` | texto | no | Obligatorio si `motivo` es `otro` |
| `fecha` | fecha | sí | |

> **Qué va al armario, según el usuario (02/09/2026):** el armario **siempre lleva
> medicamentos**. Quedan fuera:
> - **Material y otros (`DM`)** — no se ponen en el dispensador. Esto **retira la familia
>   `DM` del alcance del piloto**, no solo del catálogo de reenvasados.
> - **Sueros y grandes volúmenes** — paracetamol intravenoso citado como ejemplo.
>
> ⚠ **UFA ya no es una exclusión.** El usuario lo retiró de esta lista el 02/09/2026: pasa a
> ser una **propiedad del artículo** (`ambito`), junto con onco-hemato. La diferencia no es
> cosmética: excluir significa que el artículo desaparece del análisis; clasificar significa
> que **sigue dentro y se puede mirar por separado**. Lo segundo permite responder "¿cómo se
> comportan los de onco-hemato frente al resto?", que la exclusión hacía imposible.
>
> `[PENDIENTE: cómo se reconoce en los datos que un artículo es suero o gran volumen. Si hay
> un campo que lo diga, se deriva solo; si no lo hay, esta pestaña se rellena a mano una vez
> y se mantiene.]`
>
> **El criterio de seguridad mientras tanto:** solo se optimiza lo que **aparece en la
> extracción de inventario de un dispensador del piloto**. Si algo no está en el armario, no
> sale en esa extracción y el problema no se plantea. `EXCLUSIONES` existe para el caso
> contrario: algo que aparece y no debería estar ahí — que es en sí mismo un hallazgo que el
> informe debe señalar.

**Pestaña `PROPUESTAS_HIST`** — histórico de lo propuesto y lo decidido. REQ-072

Ciclo de vida de una propuesta, con las **únicas** transiciones legales:

```
  generada ──▶ aceptada ─────▶ aplicada ──▶ verificada
     │                             ▲
     ├───────▶ modificada ─────────┘
     │
     └───────▶ rechazada          (terminal)

  generada ──▶ sin_datos          (terminal: no había base para proponer)
```

Prohibidas de forma explícita: `rechazada → aceptada` —se genera una propuesta nueva, no se
resucita la vieja— y cualquier salto que se salte `aplicada` para llegar a `verificada`.
Una propuesta no verificada no cuenta para las métricas de §2.

**Pestaña `CAMBIOS`** — REQ-024: `cn`, `campo`, `valor_anterior`, `valor_nuevo`,
`fecha_hora`.

### Tablas normalizadas en memoria (no persisten)

- `inventario(dispensador, seccion, hueco, codigo_bdm, descripcion, min_vigente,
  max_vigente, existencias, consumo_medio, fecha_extraccion)`
- `movimientos(dispensador, codigo_bdm, fecha, tipo, cantidad, cantidad_hallada)` —
  `cantidad_hallada` es el campo que vale 0 en una rotura (REQ-080)
- `ofertas(codigo_bdm, cn, laboratorio, presentacion, precio, unidades_por_envase, …)`
- `compras(codigo_bdm, cn, fecha, cantidad, importe, …)` — del informe de compras; es lo que
  determina qué oferta está vigente

Los campos exactos de `ofertas` y `compras` se fijan al ver los dos ficheros que el usuario
va a aportar. Lo que **no** está pendiente es la relación entre ellos.

> **Campos reales de un fichero de inventario/compras, aportados por el usuario el
> 03/09/2026**, registrados aquí para que la fase 4 no los redescubra. El servicio opera
> **dos centros** —HUNSC y "Sur"— y varias columnas de este fichero son la suma de ambos; el
> PRD, hasta ahora, solo hablaba de HUNSC. **Esto no cambia el alcance del piloto** (§3: sigue
> validándose solo sobre unidades de HUNSC), pero sí exige leer las columnas por centro y no
> las agregadas:
>
> | Columna | Significado |
> |---|---|
> | `ud_pte_rec` | Unidades pendientes de recibir, HUNSC + Sur |
> | `ud_pte_rec_1` | Unidades pendientes de recibir, solo HUNSC |
> | `ud_pte_rec_18` | Unidades pendientes de recibir, solo Sur |
> | `consumed` | Consumo medio, HUNSC + Sur |
> | `consumed_9000` | Consumo de farmacia, HUNSC |
> | `consumed_ad00` | **Consumo de todas las máquinas carrusel ATHOS en HUNSC** — señalado por el usuario como el dato importante |
> | `consumed_9001` | Consumo de farmacia, Sur |
> | `consumed_ad01` | Consumo de máquinas, Sur |
> | `EXIST1` | Existencias, almacén de farmacia HUNSC |
> | `EXIST2` | Existencias, robot UFA |
> | `EXIST18` | Existencias, almacén de farmacia Sur |
> | `EXIST50` | Existencias, UFA Sur |
> | `EXIST58` / `EXIST59` | Existencias, Kardex 1 / Kardex 2 |
> | `exist60` / `exist61` | Existencias, carrusel vertical / carrusel horizontal |
> | `exist68` | Existencias, carrusel Sur |
> | `ubica` | Ubicación principal del medicamento |
>
> **Esto resuelve REQ-090** (de dónde sale el consumo global del artículo, antes
> `[PENDIENTE]`): `consumed_ad00` es exactamente ese dato — el consumo del artículo en todas
> las máquinas ATHOS de HUNSC, no solo en el dispensador piloto.
>
> `[PENDIENTE, dos cosas antes de la fase 4]`:
> 1. **Qué fichero es exactamente este.** El usuario lo llamó "inventario de compras"; puede
>    ser el `informe de compras` ya previsto (el que fija la oferta vigente, REQ-027) o un
>    fichero distinto. Se confirma al ver el fichero real, no antes.
> 2. **Qué columnas usa el piloto.** Con dos centros en el mismo fichero, hay que decidir de
>    forma explícita si `existencias_globales` de `ARTICULOS` suma todas las columnas
>    `EXIST*` de HUNSC o solo un subconjunto (p. ej. si Kardex no aplica a los dispensadores
>    piloto). No se decide por parecido: se pregunta al ver el fichero.

### Reglas de borrado y ciclo de vida por entidad

No hay `DELETE` en este producto salvo donde se diga. El histórico es la mitad del valor: un
piloto que borra no puede explicar después por qué propuso lo que propuso.

| Entidad | Al desaparecer de la fuente | Arrastra a |
|---|---|---|
| **Artículo** (`ARTICULOS`) | **No se borra.** Se marca `activo = no` y conserva su fila y su `primera_vez_visto` | Nada. Sus propuestas históricas siguen siendo válidas y legibles |
| **Oferta** (`OFERTAS`) | **No se borra:** pasa a `vigente = no`. El histórico de precios explica importes calculados en su día | Actualiza `cn_vigente` y `precio_unitario_vigente` del artículo |
| **Reenvasado** | Se borra solo por orden expresa. Un CN sin oferta vigente conserva su clasificación, por si la oferta vuelve | Nada |
| **Hueco / capacidad** | **No se borra:** se anota una fila nueva con fecha posterior y gana la más reciente | Nada. El histórico de capacidades explica propuestas antiguas que hoy parecerían raras |
| **Propuesta** | **Nunca se borra ni se reescribe.** Solo avanza por las transiciones legales | — |
| **Dispensador que sale del piloto** | Deja de procesarse; sus datos y propuestas permanecen | Nada |
| **Fichero de `datos/intermedio/` y `datos/salida/`** | Borrable entero en cualquier momento | Nada: es todo regenerable, salvo las copias del maestro, que se conservan según NFR-003 |

### Datos semilla

**Empezar limpio, por decisión del usuario.** No se importa nada de Frello. Para poder
probar sin esperar a un fichero real, `datos/ejemplos/` contiene libros **sintéticos
inventados** —no extracciones reales anonimizadas— con 2 dispensadores, 30 artículos y 90
días de movimientos, incluyendo a propósito: una fila con valor no numérico, el mismo código
en mayúsculas y minúsculas, un artículo de cada uno de los cuatro prefijos (`V`, `Y`, `DM`,
`T`), un artículo con tres ofertas de las que solo una está vigente, uno con consumo cero y
otro con solo 2 movimientos.

---

## 7. Especificación funcional · *capa QUÉ*

### M1 · Guarda de datos de paciente

Cubre REQ-001, REQ-002, REQ-003.

Es lo primero que corre y lo único que puede detener todo el proceso.

- **REQ-001** `[Must]` — *Ubicuo.* El sistema DEBERÁ inspeccionar las cabeceras y una
  muestra de al menos 200 filas de todo fichero de entrada, antes de cargarlo, buscando
  indicadores de identificación de paciente: NHC, CIP, DNI/NIE, número de historia, número
  de episodio, nombre y apellidos, fecha de nacimiento.
  > Dado un fichero de inventario limpio, cuando se ejecuta el procesado, entonces la
  > guarda registra "pasó" y el proceso continúa.
  > Dado un fichero con una columna `NHC`, cuando se ejecuta el procesado, entonces la
  > guarda lo detecta antes de que se lea una sola fila de datos.

- **REQ-002** `[Must]` — *No deseado.* SI se detecta un indicador de identificación de
  paciente, ENTONCES el sistema DEBERÁ abortar la carga completa, no escribir ningún
  fichero intermedio ni de salida, no modificar el maestro, y emitir un aviso que nombre el
  fichero y la columna **sin reproducir su contenido**.
  > Dado un fichero con una columna `Paciente`, cuando se procesa, entonces el proceso
  > termina con código de error, `datos/intermedio/` y `datos/salida/` quedan sin cambios y
  > el maestro conserva su fecha de modificación anterior.
  > Dado ese mismo fichero, cuando se lee el aviso, entonces aparece el nombre de la
  > columna y ningún valor de esa columna.

- **REQ-003** `[Must]` — *Ubicuo.* El sistema DEBERÁ registrar cada ejecución de la guarda
  en `datos/salida/guardas.log` con fichero, resultado y fecha-hora, y **nunca** valores de
  las columnas inspeccionadas.
  > Dado un aborto por detección, cuando se inspecciona el log, entonces contiene el nombre
  > del fichero y la palabra "abortado", y ninguna cadena procedente de los datos.

**Casos límite:** fichero vacío (aborta con mensaje propio, no como detección); cabecera en
la fila 4 (la guarda localiza la cabecera antes de decidir); una columna `NOMBRE` que en
realidad es el nombre del medicamento — **falso positivo aceptado a propósito**: es
preferible parar de más que colar un identificador. La lista de indicadores es un fichero
de configuración editable, no está clavada en el código.

### M2 · Lectura y normalización de ficheros

Cubre REQ-010 a REQ-013.

Cada tipo de fichero tiene un **perfil de lectura declarado** —un YAML que dice qué columna
de origen es qué campo interno, dónde está la cabecera y qué columnas son obligatorias—.
Toda la suciedad de los ficheros reales vive aquí y en ningún otro sitio.

- **REQ-010** `[Must]` — *Evento.* CUANDO se ejecuta la carga de un fichero, el sistema
  DEBERÁ normalizarlo a su tabla interna aplicando el perfil de lectura correspondiente.
  > Dado un inventario con las columnas en distinto orden que el mes pasado, cuando se
  > carga, entonces la tabla normalizada es idéntica, porque el perfil mapea por nombre.
  > Dado un fichero cuyo perfil no existe, cuando se carga, entonces el sistema lo dice y no
  > intenta adivinar el mapeo.

- **REQ-011** `[Must]` — *No deseado.* SI el fichero no contiene todas las columnas
  obligatorias del perfil, ENTONCES el sistema DEBERÁ abortar la carga de ese fichero
  nombrando las que faltan.
  > Dado un inventario sin la columna de consumo, cuando se carga, entonces el error nombra
  > exactamente `consumo_medio` y no se genera ninguna propuesta.

- **REQ-012** `[Must]` — *No deseado.* SI una fila contiene un valor no convertible en una
  columna numérica, ENTONCES el sistema DEBERÁ apartarla a
  `datos/salida/rechazos_<fecha>.xlsx`, contabilizarla y continuar con el resto.
  > Dado un fichero con 3 filas donde el mínimo pone "N/D", cuando se carga, entonces la
  > tabla tiene 3 filas menos, el fichero de rechazos las contiene con su motivo, y el
  > resumen dice "3 filas rechazadas".

- **REQ-013** `[Must]` — *Ubicuo.* El sistema DEBERÁ dejar todo fichero de `datos/entrada/`
  sin modificar, incluida su fecha de modificación.
  > Dado un fichero de entrada, cuando termina un procesado completo, entonces su hash y su
  > fecha de modificación son idénticos a los de antes de ejecutar.
  > Dado un procesado que aborta a mitad, cuando se comprueban los ficheros de entrada,
  > entonces siguen igualmente intactos.

**Casos límite:** cabecera desplazada; celdas combinadas; números guardados como texto con
separador de miles; fila de totales al final que no es un dato; el mismo medicamento dos
veces en el mismo hueco; fichero de 100 MB (NFR-008); libro con varias hojas donde la buena
no es la primera.

### M3 · Maestro de artículos persistente

Cubre REQ-020 a REQ-026.

El maestro es el único estado que sobrevive entre ejecuciones. Todo lo demás es regenerable.

- **REQ-020** `[Must]` — *Evento.* CUANDO termina una carga válida, el sistema DEBERÁ
  actualizar `maestro.xlsx`: alta de los códigos BDM no vistos antes y actualización de los
  campos que hayan cambiado en los ya conocidos.
  > Dado un maestro con 300 artículos y una carga con 12 códigos nuevos, cuando se procesa,
  > entonces el maestro tiene 312 filas y las 300 anteriores conservan su
  > `primera_vez_visto`.
  > Dado un medicamento cuyo precio ha cambiado, cuando se procesa, entonces el maestro
  > refleja el precio nuevo y `ultima_actualizacion` con la fecha de hoy.

- **REQ-021** `[Must]` — *Ubicuo.* El sistema DEBERÁ copiar `maestro.xlsx` a
  `datos/intermedio/copias/maestro_<AAAA-MM-DD_HHMM>.xlsx` **antes** de escribir sobre él.
  > Dado un maestro existente, cuando se ejecuta el procesado, entonces existe una copia
  > nueva fechada, idéntica al maestro previo a la escritura.

- **REQ-022** `[Must]` — *No deseado.* SI `maestro.xlsx` está abierto o bloqueado por otro
  proceso, ENTONCES el sistema DEBERÁ abortar antes de escribir nada e indicar que hay que
  cerrarlo.
  > Dado el maestro abierto en Excel, cuando se ejecuta el procesado, entonces el proceso
  > termina con un mensaje que nombra el fichero, y ni el maestro ni la salida cambian.

- **REQ-023** `[Must]` — *Ubicuo.* El sistema DEBERÁ usar el **código BDM normalizado a
  mayúsculas** como clave natural del maestro, y DEBERÁ rechazar la escritura si detecta dos
  filas con el mismo código.
  > Dado un fichero donde el mismo artículo aparece como `dm000116` y como `DM000116`,
  > cuando se procesa, entonces se reconocen como el mismo y el maestro tiene una sola fila.
  > Dado un fichero con el mismo código repetido con descripciones distintas, cuando se
  > procesa, entonces se rechaza la escritura nombrando el código duplicado.

- **REQ-024** `[Should]` — *Evento.* CUANDO el maestro se actualiza, el sistema DEBERÁ
  anotar en la pestaña `CAMBIOS` qué código y qué campos cambiaron, con valor anterior y
  nuevo.
  > Dado un cambio de precio, cuando se consulta `CAMBIOS`, entonces hay una fila con el
  > código, el campo, el valor viejo y el nuevo.

- **REQ-026** `[Must]` — *Ubicuo.* El sistema DEBERÁ derivar `tipo_articulo` del prefijo del
  código BDM: `V` → `medicamento`, `Y` → `formula_magistral`, `DM` → `material`, `T` →
  `medicamento_extranjero`. El campo **no se teclea nunca**.
  > Dado el código `dm000116`, cuando se da de alta, entonces `tipo_articulo` es `material`
  > sin que nadie lo haya escrito.
  > Dado un código con un prefijo que no está en la lista, cuando se procesa, entonces el
  > artículo se da de alta con `tipo_articulo` sin determinar y se señala en el resumen, en
  > vez de asignarle un tipo por parecido.

- **REQ-029** `[Must]` — *Ubicuo.* El sistema DEBERÁ mantener por artículo la propiedad
  `ambito` con valor de la lista cerrada `onco_hemato`, `ufa`, `general`, y DEBERÁ permitir
  segmentar por ella la pestaña de estadística. **Clasificar no es excluir:** un artículo de
  UFA entra en el cálculo como cualquier otro.
  > Dado un armario con 12 artículos de onco-hemato, cuando se abre `ESTADISTICA`, entonces
  > sus indicadores se pueden leer por separado del resto.
  > Dado un artículo de UFA, cuando se genera `ACCIONES`, entonces tiene su propuesta de
  > mín/máx como cualquier otro: la marca clasifica, no filtra.

- **REQ-028** `[Must]` — *No deseado.* SI un artículo figura en `EXCLUSIONES`, o su
  `tipo_articulo` es `material`, ENTONCES el sistema DEBERÁ excluirlo del cálculo de mín/máx
  y **señalar su presencia en el armario como hallazgo**, en vez de omitirlo en silencio.
  > Dado un artículo de material que aparece en la extracción de un dispensador piloto,
  > cuando se procesa, entonces no se le calcula mín/máx y la estadística lo lista como
  > "presente en el armario pese a estar excluido".
  > Dado un suero excluido por `gran_volumen`, cuando se genera `ACCIONES`, entonces su fila
  > existe con la acción "excluido · gran volumen" y sin propuesta, para que quien revise
  > vea que se tuvo en cuenta y se descartó a propósito.

- **REQ-027** `[Must]` — *Evento.* CUANDO se carga el catálogo de ofertas y el informe de
  compras, el sistema DEBERÁ poblar la pestaña `OFERTAS` y marcar como `vigente` la oferta
  que respalde el informe de compras.
  > Dado un artículo con tres ofertas y un informe de compras que solo refleja una, cuando
  > se procesa, entonces solo esa queda `vigente` y `cn_vigente` del artículo apunta a ella.
  > Dado un artículo cuyas compras no casan con ninguna oferta conocida, cuando se procesa,
  > entonces se señala en el resumen y `cn_vigente` queda vacío, en vez de elegir una al
  > azar.

- **REQ-025** `[Should]` — *Evento.* CUANDO se solicita retirar el nombre de una persona, el
  sistema DEBERÁ sustituirlo por `(retirado)` en todas las pestañas del maestro **y en las
  copias fechadas**, conservando el dato técnico asociado.
  > Dado un técnico que pide que su nombre no figure, cuando se ejecuta
  > `anonimizar --persona`, entonces ninguna pestaña ni ninguna copia contiene su nombre, y
  > las capacidades de hueco que informó siguen ahí.
  > Dado ese mismo caso, cuando se revisa una copia fechada anterior a la operación,
  > entonces tampoco aparece el nombre — si no, el borrado sería ficticio.

**Casos límite:** primer arranque sin maestro (se crea vacío con sus pestañas y cabeceras —
es el estado vacío de §8); maestro corrupto o ilegible (aborta y señala la copia más
reciente); el mismo CN con dos descripciones distintas en dos ficheros (gana el más
reciente y se anota en `CAMBIOS`); anonimizar a alguien que no figura (no es un error: lo
dice y no toca nada).

### M4 · Catálogo de reenvasados

Cubre REQ-030, REQ-031, REQ-032.

- **REQ-030** `[Must]` — *Ubicuo.* El sistema DEBERÁ mantener por medicamento un tratamiento
  de reenvasado con valor de la lista cerrada `se_reenvasa`, `se_etiqueta`, `ya_unidosis`, y
  DEBERÁ rechazar cualquier otro valor.
  > Dado un catálogo con el valor "reenvasar" escrito a mano, cuando se carga, entonces la
  > fila se rechaza nombrando los tres valores admitidos.

- **REQ-031** `[Must]` — *No deseado.* SI el `tipo_articulo` no es `medicamento` ni
  `medicamento_extranjero`, **o** la forma farmacéutica no es comprimido ni cápsula, ENTONCES
  el sistema DEBERÁ tratar el campo como **no aplicable**: no exigirlo, no contarlo como
  pendiente y no mostrarlo entre los sin clasificar.
  > Dado un vial sin tratamiento asignado, cuando se genera la estadística, entonces no
  > aparece entre los sin clasificar.
  > Dado un artículo `DM000116` de material, cuando se genera la estadística, entonces
  > tampoco aparece: el material no se reenvasa, y pedirle una clasificación sería ruido que
  > nadie puede resolver.

- **REQ-032** `[Should]` — *Ubicuo.* El sistema DEBERÁ listar en la pestaña de estadística
  los comprimidos y cápsulas presentes en los dispensadores piloto que aún no tienen
  tratamiento asignado.
  > Dado un armario con 12 comprimidos sin clasificar, cuando se abre `ESTADISTICA`,
  > entonces aparecen los 12 con su CN y descripción.

### M5 · Base de huecos y restricción de capacidad

Cubre REQ-040, REQ-041, REQ-042.

Este módulo es la diferencia entre una propuesta aplicable y una que se cae al llegar al
armario. **El dato nace incompleto y el producto tiene que decirlo, no disimularlo.**

- **REQ-040** `[Must]` — *Ubicuo.* El sistema DEBERÁ registrar, por
  `(dispensador, sección, hueco, cn)`, la cantidad máxima que cabe físicamente, con quién lo
  informó y cuándo.
  > Dado que el técnico informa de que en la sección B, hueco 14 solo caben 20 unidades de
  > un CN, cuando se anota en `HUECOS`, entonces queda registrado con su informante y su
  > fecha, y el motor lo respeta en el siguiente cálculo.
  > Dado un intento de anotar dos veces la misma combinación de dispensador, sección, hueco
  > y CN, cuando se guarda, entonces se rechaza por duplicado en vez de crear dos verdades.

- **REQ-041** `[Must]` — *Ubicuo.* El sistema DEBERÁ garantizar que ningún máximo propuesto
  supera la `cantidad_maxima` registrada para ese hueco y medicamento.
  > Dado un hueco con capacidad 20 y un cálculo que sugiere 28, cuando se genera la
  > propuesta, entonces el máximo propuesto es 20 y la justificación dice que la capacidad
  > física fue la restricción activa.
  > Dado ese mismo caso, cuando se revisa `ESTADISTICA`, entonces el medicamento aparece
  > señalado como limitado por capacidad, porque eso es una conversación pendiente sobre la
  > ubicación, no un éxito del cálculo.

- **REQ-042** `[Must]` — *No deseado.* SI no hay capacidad registrada para ese hueco y
  medicamento, ENTONCES la propuesta DEBERÁ marcarse `capacidad_no_verificada` y decirlo en
  la columna de justificación.
  > Dado un medicamento sin dato de capacidad, cuando se genera la propuesta, entonces la
  > propuesta existe y lleva la marca, en vez de omitirse en silencio.

### M6 · Motor de optimización de mín/máx

Cubre REQ-050 a REQ-059 y REQ-041. **Es el núcleo puro: no lee ni escribe ficheros.**

#### El objetivo real no es el stock: es el trabajo del operario

Dos hechos del proceso, declarados por el usuario, que determinan toda la fórmula:

1. **El dispensador se repone cada 24 h, siempre.** La visita es diaria y no depende de
   nada. Lo que varía es **qué referencias hay que tocar** en cada visita.
2. **Cada referencia debería necesitar reposición como mucho una vez por semana**, y el
   objetivo operativo es tocar **8-10 referencias por dispensador y visita**.

De ahí salen las dos consecuencias que cambian el cálculo respecto a un modelo de stock
clásico:

- **El mínimo es un colchón de 3-4 días, no el plazo de reposición.** Aunque el operario
  pase cada 24 h, atar el mínimo a esas 24 h dejaría el armario sin margen: basta con que
  el consumo medio suba —un ingreso más, un cambio de protocolo— para que la reposición
  llegue tarde. **Decisión expresa del usuario el 02/09/2026**, y es la correcta: el
  mínimo protege contra la variación de la demanda, no contra el retraso del operario.
- **El máximo es lo que compra una semana sin tocar esa referencia.** No es "un ciclo
  más": es exactamente el dial del trabajo del operario.

#### Definiciones

Calculadas por dispensador y artículo sobre el periodo analizado:

- `CMD` = consumo medio diario = consumo del periodo ÷ días del periodo.
- `C_minimo` = **3 días** por defecto, ajustable en el rango 3-4. Colchón que cubre el
  mínimo.
- `C_objetivo` = **7 días**: cobertura que se quiere entre reposiciones de una misma
  referencia, para no tocarla más de una vez por semana.
- `CV` = coeficiente de variación del consumo diario (desviación típica ÷ media).
- `k` = factor de seguridad según `CV`: `CV < 0,3 → k = 0,2` · `0,3 ≤ CV < 0,6 → k = 0,4` ·
  `CV ≥ 0,6 → k = 0,6`. **Confirmados por el usuario el 02/09/2026.**
- El dispensador se visita cada 24 h, siempre. Eso **no entra en la fórmula**: entra en el
  indicador agregado de trabajo por visita.

```
  minimo_propuesto = max( 1, techo( CMD × C_minimo × (1 + k) ) )
  maximo_propuesto = minimo_propuesto + techo( CMD × C_objetivo )
  maximo_propuesto = min( maximo_propuesto, cantidad_maxima_del_hueco )   ← REQ-041
  ambos se redondean al alza al múltiplo de la unidad de dispensación     ← REQ-052

  dias_entre_reposiciones = (maximo_propuesto − minimo_propuesto) ÷ CMD   ← REQ-057
      objetivo: >= 7. Menos que eso significa tocar esa referencia más de
      una vez por semana, y el motivo casi siempre es que no cabe.

  reposiciones_diarias_esperadas = Σ ( CMD_i ÷ (max_i − min_i) )          ← REQ-058
      sobre todas las referencias del dispensador. Objetivo: 8-10 por visita.
```

#### Por qué esto convierte los huecos en la pieza central

Con un artículo que consume 2 unidades al día, `C_minimo` = 3 y `k` = 0,2:

| | Sin límite de hueco | Hueco de capacidad 10 |
|---|---|---|
| Mínimo | techo(2×3×1,2) = **8** | **8** |
| Máximo | 8 + 14 = **22** | **10** (recortado) |
| Días entre reposiciones | (22−8)/2 = **7,0** ✓ | (10−8)/2 = **1,0** ✗ |
| Consecuencia | Se toca una vez por semana | **Se toca todos los días** |

> **La consecuencia incómoda del colchón de 3 días, dicha antes de construir:** para que una
> referencia aguante la semana, su hueco tiene que caber **unos 10-11 días de consumo**
> (3-4 de colchón más 7 de ciclo). Es una exigencia física alta, y va a hacer que **muchas
> más líneas choquen contra la capacidad** de las que chocarían con un mínimo de un día.
>
> Eso no es un problema del cálculo: es el armario diciendo la verdad. Pero significa que
> **la base de huecos deja de ser un accesorio y pasa a ser el dato que decide si el piloto
> funciona.** Si no se rellena, casi todas las líneas saldrán marcadas
> `capacidad_no_verificada` y el resultado no será interpretable.

El segundo caso de la tabla no es un fallo: es **un hallazgo de ubicación**. Ese artículo
está en un hueco demasiado pequeño para su rotación, y eso es lo que hay que poner delante
de quien decide dónde va cada cosa. Un armario con muchas líneas así es un armario que
genera trabajo diario evitable, y el producto puede decir **cuáles son y cuántas visitas al
año cuestan**.

> ⚠ `C_minimo` = 3-4 días, `C_objetivo` = 7 días y el rango 8-10 referencias por visita
> **vienen del usuario**. Los tres van en `config/`, como `k`, para afinarlos en la fase 6
> con datos reales sin tocar código.

- **REQ-050** `[Must]` — *Ubicuo.* El sistema DEBERÁ calcular el mínimo propuesto como la
  cobertura de `C_minimo` días de consumo más el colchón de seguridad, con los parámetros
  leídos de configuración y no clavados en el código.
  > Dado CMD=2, `C_minimo`=3 y CV=0,2 (`k`=0,2), cuando se calcula, entonces el mínimo
  > propuesto es 8 (2×3×1,2 = 7,2 → 8).
  > Dado un artículo con CMD=0,1, cuando se calcula, entonces el mínimo es 1 y no 0: un
  > mínimo de cero significa que el hueco se vacía antes de disparar la reposición.

- **REQ-051** `[Must]` — *Ubicuo.* El sistema DEBERÁ calcular el máximo propuesto como el
  mínimo más la cobertura de `C_objetivo` días, acotado por la capacidad física del hueco.
  > Dado el caso anterior sin límite de hueco y `C_objetivo`=7, cuando se calcula, entonces
  > el máximo es 22 (8 + 2×7).
  > Dado ese mismo artículo en un hueco de capacidad 10, cuando se calcula, entonces el
  > máximo es 10 y la justificación dice que la capacidad fue la restricción activa.

- **REQ-057** `[Must]` — *Ubicuo.* El sistema DEBERÁ calcular por línea los días entre
  reposiciones que implica la propuesta, y DEBERÁ señalar las que queden por debajo del
  objetivo de una reposición semanal.
  > Dado un artículo con máximo 22, mínimo 8 y CMD 2, cuando se calcula, entonces los días
  > entre reposiciones son 7,0 y la línea no se señala.
  > Dado ese mismo artículo recortado a máximo 10 por capacidad, cuando se calcula, entonces
  > es 1,0 día, la línea se marca `rotacion_alta_por_capacidad`, y la estadística la cuenta
  > entre las que generan trabajo diario evitable.

- **REQ-058** `[Must]` — *Ubicuo.* El sistema DEBERÁ calcular por dispensador el número
  esperado de referencias a reponer por visita, y compararlo con el objetivo de 8-10.
  > Dado un dispensador de 63 referencias cuyas propuestas dan todas 7 días de cobertura,
  > cuando se calcula el indicador, entonces sale 9,0 referencias por visita y se marca
  > dentro de objetivo.
  > Dado un dispensador cuyo indicador sale 15,4, cuando se lee la estadística, entonces
  > figuran **las líneas concretas que lo empujan al alza y por qué**, en vez de un número
  > suelto que nadie sabe cómo bajar.

- **REQ-059** `[Should]` — *No deseado.* SI una referencia no cabe físicamente ni para
  `C_minimo` días de consumo, ENTONCES el sistema DEBERÁ marcarla como **problema de
  ubicación, no de configuración**, y proponer revisar su hueco en vez de un mín/máx.
  > Dado un artículo con CMD=5 en un hueco de capacidad 8, cuando se calcula, entonces la
  > acción propuesta es "revisar ubicación" y la justificación dice que ni el colchón
  > mínimo cabe: ningún mín/máx arregla eso.

- **REQ-052** `[Must]` — *Ubicuo.* El sistema DEBERÁ redondear al alza al múltiplo de la
  unidad de dispensación del medicamento.
  > Dado un medicamento que se dispensa en blísteres de 10 y un cálculo de 23, cuando se
  > redondea, entonces la propuesta es 30 y no 23.
  > Dado un redondeo que hace superar la capacidad del hueco, cuando se aplica REQ-041,
  > entonces gana la capacidad: se baja al múltiplo inmediatamente inferior que quepa, y la
  > justificación lo dice.

- **REQ-053** `[Must]` — *Ubicuo.* El sistema DEBERÁ acompañar cada propuesta de su
  justificación numérica: CMD, F, CV, k, el valor calculado antes de restricciones y cuál
  fue la restricción activa si la hubo.
  > Dado cualquier propuesta de la salida, cuando un farmacéutico pregunta por qué ese
  > número, entonces la respuesta está en su fila, sin abrir el código.
  > Dado un máximo recortado por capacidad, cuando se lee la justificación, entonces figuran
  > el valor calculado y el final, no solo el final.

- **REQ-054** `[Must]` — *No deseado.* SI el consumo del periodo es cero, ENTONCES el
  sistema DEBERÁ proponer la **retirada del medicamento del armario** en vez de un mín/máx,
  e indicar cuántos días lleva sin movimiento.
  > Dado un medicamento con 0 consumo en 90 días, cuando se genera la propuesta, entonces la
  > acción es "retirar" y no un mínimo de 0.

- **REQ-055** `[Must]` — *No deseado.* SI hay menos de `min_movimientos` movimientos de
  salida en el periodo, ENTONCES el sistema DEBERÁ marcar la línea como `sin_datos` y **no
  proponer**. **`min_movimientos` vale 0 por defecto: el umbral queda desactivado**, por
  decisión expresa del usuario el 02/09/2026 para evitar fatiga de alertas.
  > Dado `min_movimientos` = 0 y un artículo con 3 movimientos, cuando se genera la
  > propuesta, entonces **se propone** con los datos que hay, no se marca `sin_datos`.
  > Dado `min_movimientos` = 5 en configuración y ese mismo artículo, cuando se genera la
  > propuesta, entonces figura con estado `sin_datos` y las columnas vacías.

  > **Cómo se sostiene el cálculo sin ese umbral, que era su motivo de existir.** Con pocos
  > movimientos, el `CV` no se puede calcular de forma fiable. En ese caso **no se marca la
  > línea: se aplica el `k` más conservador (0,6)**. Es decir, ante datos escasos el sistema
  > no avisa — **protege**, dando el colchón más grande. Va en la dirección de que no se
  > llegue nunca a cajetín 0, que es lo que el usuario ha pedido, y no genera una marca más
  > que leer.

- **REQ-056** `[Must]` — *Ubicuo.* El sistema DEBERÁ tratar toda propuesta como no
  vinculante: ninguna se aplica automáticamente ni se escribe en ningún sistema externo.
  > Dado un procesado completo, cuando se inspecciona qué ha escrito el sistema, entonces
  > solo hay ficheros bajo `datos/`: ninguna llamada de red, ninguna escritura fuera de la
  > carpeta del proyecto.
  > Dado una propuesta aceptada en la ingesta, cuando se consulta el estado del armario real,
  > entonces sigue con su mín/máx anterior hasta que una persona lo cambie a mano en el
  > sistema de APD.

**Casos límite:** periodo de menos de 30 días (el `CMD` es poco fiable y se avisa); artículo
con consumo muy bajo pero no nulo, del tipo 1 unidad al mes (el mínimo cae al suelo de 1 y
el máximo apenas sube: la propuesta es válida pero conviene señalar que la cobertura de 7
días le sobra con mucho); `CMD` mayor que la capacidad del hueco (REQ-059: es un problema de
ubicación, no de configuración); dispensador en el que **ninguna** línea tiene capacidad
registrada — el indicador de referencias por visita se calcula igual, pero se marca entero
como no verificado, porque un 9,2 calculado sobre máximos que quizá no caben no significa
nada.

### M7 · Libro Excel de salida

Cubre REQ-060 a REQ-064.

**Un libro por unidad piloto.** Estructura fijada por el usuario:

| Pestaña | Contenido |
|---|---|
| `CRUDO_<origen>` | Una por fichero de entrada, con los datos tal como se leyeron. Es lo que hace auditable el resto sin salir del fichero |
| `ACCIONES` | La integración: una fila por (dispensador, medicamento) con mín/máx vigente, propuesto, la acción, la justificación numérica y las marcas `limitado_por_capacidad`, `capacidad_no_verificada`, `sin_datos`, `retirar` |
| `ESTADISTICA` | Resumen del armario: **referencias esperadas por visita frente al objetivo 8-10 (M4)**, **líneas que no llegan a los 7 días entre reposiciones y por qué**, nº de líneas, cuántas cambian y en qué sentido, importe inmovilizado vigente y propuesto con su cobertura, comprimidos y cápsulas sin clasificar, líneas limitadas por capacidad, artículos excluidos presentes en el armario |
| `VALIDACION` | La misma clave que `ACCIONES` más tres columnas vacías: `decision`, `valor_modificado`, `comentario` |

- **REQ-060** `[Must]` — *Evento.* CUANDO se genera la salida, el sistema DEBERÁ incluir una
  pestaña con los datos crudos de cada fichero de entrada utilizado.
  > Dado un procesado con inventario, movimientos y ofertas, cuando se abre el libro,
  > entonces hay tres pestañas `CRUDO_*`.

- **REQ-061** `[Must]` — *Evento.* CUANDO se genera la salida, el sistema DEBERÁ producir la
  pestaña `ACCIONES` con una fila por par (dispensador, medicamento) y su justificación.
  > Dado un dispensador con 120 medicamentos, cuando se genera el libro, entonces `ACCIONES`
  > tiene 120 filas y ninguna con la justificación vacía.
  > Dado un medicamento en estado `sin_datos`, cuando se lee su fila, entonces la
  > justificación explica que faltan movimientos, en vez de quedar en blanco.

- **REQ-062** `[Must]` — *Evento.* CUANDO se genera la salida, el sistema DEBERÁ producir la
  pestaña `ESTADISTICA` con los indicadores de la tabla anterior.
  > Dado un armario donde 30 de 120 líneas cambian, cuando se abre `ESTADISTICA`, entonces
  > figura ese recuento desglosado entre subidas y bajadas.

- **REQ-063** `[Must]` — *Evento.* CUANDO se genera la salida, el sistema DEBERÁ producir la
  pestaña `VALIDACION` con las columnas de decisión vacías y con validación de datos en
  `decision` limitada a `acepto` / `modifico` / `rechazo`.
  > Dado el libro recién generado, cuando se abre `VALIDACION`, entonces `decision` solo
  > admite esos tres valores y no texto libre.

- **REQ-064** `[Should]` — *Ubicuo.* El sistema DEBERÁ nombrar el libro
  `propuestas_<unidad>_<AAAA-MM-DD>.xlsx` y no sobrescribir uno existente.

**Casos límite:** unidad sin ninguna propuesta (el libro se genera igual, con sus pestañas y
una nota explicando el motivo — ver §8); más de 200.000 filas en un crudo (se trunca esa
pestaña con aviso visible y el fichero completo queda en `datos/intermedio/`, porque un
libro que Excel no puede abrir no sirve de nada).

### M8 · Ingesta de la validación

Cubre REQ-070, REQ-071, REQ-072.

- **REQ-070** `[Must]` — *Evento.* CUANDO se ejecuta la ingesta de un libro devuelto, el
  sistema DEBERÁ registrar en `PROPUESTAS_HIST` la decisión de cada línea con su fecha y su
  autor.
  > Dado un libro con 40 aceptadas, 5 modificadas y 2 rechazadas, cuando se ingesta,
  > entonces el histórico registra 47 decisiones y ninguna de esas propuestas queda en
  > `generada`.
  > Dado un libro parcialmente rellenado, cuando se ingesta, entonces las líneas sin decisión
  > permanecen en `generada` y el resumen dice cuántas faltan.

- **REQ-071** `[Must]` — *No deseado.* SI la clave de una línea devuelta no coincide con
  ninguna propuesta generada, ENTONCES el sistema DEBERÁ rechazar el fichero completo
  indicando la primera línea discordante.
  > Dado un libro donde alguien insertó una fila a mano, cuando se ingesta, entonces se
  > rechaza entero y se nombra la fila, en vez de registrar decisiones a medias.

- **REQ-072** `[Must]` — *Ubicuo.* El sistema DEBERÁ admitir únicamente las transiciones de
  estado declaradas en §6 y rechazar cualquier otra.
  > Dado una propuesta en estado `rechazada`, cuando se intenta pasarla a `aceptada`,
  > entonces el sistema lo impide y explica que hay que generar una propuesta nueva.

### M9 · Medición de resultados

Cubre REQ-080, REQ-081, REQ-082, REQ-090, REQ-091.

- **REQ-080** `[Must]` — *Ubicuo.* El sistema DEBERÁ identificar como rotura de stock toda
  línea del histórico de movimientos en la que la cantidad hallada en el cajetín al reponer
  sea 0.
  > Dado un histórico con 14 líneas a cero en el periodo, cuando se ejecuta la medición,
  > entonces se contabilizan 14 roturas, desglosadas por dispensador y medicamento.

- **REQ-081** `[Must]` — *Evento.* CUANDO se ejecuta la medición con dos periodos, el
  sistema DEBERÁ comparar roturas antes y después por dispensador, **siempre en absoluto y
  además en porcentaje**, y DEBERÁ listar las roturas que quedan con su artículo y fecha.
  > Dado 14 roturas antes y 2 después, cuando se compara, entonces el informe dice "de 14 a
  > 2 (−86 %)" contra el objetivo M3 de −80 %, **y lista las 2 que quedan**.
  > Dado un periodo posterior sin ninguna rotura, cuando se lee el informe, entonces lo dice
  > explícitamente: es el objetivo declarado del usuario y merece verse, no ser un hueco en
  > una tabla.

- **REQ-082** `[Should]` — *Evento.* CUANDO se cierra el piloto, el sistema DEBERÁ generar
  un informe con las métricas que puede calcular de §2 —**M1, M3 y M4**, del histórico
  digital— y DEBERÁ dejar una casilla para introducir **M2 a mano**, porque esa métrica sale
  de contrastar la hoja de reposición en papel con lo propuesto, no de ningún fichero
  digital (§2).

- **REQ-090** `[Should]` — *Ubicuo.* El sistema DEBERÁ comparar el consumo medio de cada
  medicamento en el dispensador con su consumo global, y señalar las desviaciones
  relevantes.
  > **Resuelto el 03/09/2026** (§6, nota sobre el fichero de inventario/compras): el consumo
  > global sale de la columna `consumed_ad00` — consumo del artículo en todas las máquinas
  > carrusel ATHOS de HUNSC. Queda `[PENDIENTE]` confirmar el nombre exacto del fichero que
  > la trae, al verlo en fase 4.

- **REQ-091** `[Could]` — *Ubicuo.* El sistema DEBERÁ calcular el importe inmovilizado por
  dispensador con los mín/máx vigentes y con los propuestos, usando el precio de la oferta
  vigente.
  > Dado un dispensador con precio conocido en el 80 % de sus líneas, cuando se calcula el
  > importe, entonces se informa el importe **y** el porcentaje de cobertura, en vez de dar
  > una cifra que parece completa y no lo es.
  > Dado un armario con fórmulas magistrales (`Y`) y material sin oferta, cuando se calcula
  > el importe, entonces esas líneas se excluyen y el informe dice cuántas y de qué tipo,
  > porque un importe que ignora en silencio una parte del armario induce a error a quien
  > lo lea.

### Requisitos no funcionales

| # | MoSCoW | Requisito | Umbral |
|---|---|---|---|
| **NFR-001** | `[Must]` | Un fichero de inventario DEBERÁ procesarse completo | < 60 s para 50.000 filas en un portátil de gama media |
| **NFR-002** | `[Must]` | La guarda de paciente DEBERÁ ejecutarse siempre antes de cualquier escritura | Sin excepción; coste < 5 s por fichero |
| **NFR-003** | `[Must]` | El sistema DEBERÁ conservar copias fechadas del maestro | Al menos las 30 últimas; borrado de las más viejas solo por orden explícita |
| **NFR-004** | `[Must]` | El cálculo DEBERÁ ser reproducible | Misma entrada + mismos parámetros = misma salida, celda a celda |
| **NFR-005** | `[Must]` | Ningún dato real DEBERÁ entrar en el repositorio git | 0 ficheros; verificado por `.gitignore` y por revisión antes del primer commit |
| **NFR-006** | `[Must]` | Coste de explotación | 0 €/mes. Cualquier servicio de pago es un cambio de arquitectura |
| **NFR-007** | `[Should]` | Cobertura de test del núcleo (`src/nucleo/`) | ≥ 80 % de líneas. Fuera del núcleo no se exige |
| **NFR-008** | `[Should]` | Tamaño máximo de fichero de entrada admitido | 100 MB; por encima, aviso y aborto controlado |
| **NFR-009** | `[Must]` | Retención y ubicación de los datos | Solo en el equipo del usuario. Nada se sube a ningún servicio |
| **NFR-010** | `[Should]` | Un ciclo completo (procesar → libro) para una unidad | < 5 minutos, para que quepa en una sesión de trabajo |
| **NFR-011** | `[Must]` | **Sin fatiga de alertas.** Ninguna categoría de marca debería afectar a la mayoría de las líneas de un dispensador | Si una marca supera el **25 %** de las líneas, la estadística la presenta como **un** problema del armario con su recuento, no como N problemas individuales fila a fila |

---

## 8. Interfaz · *capa UX*

No hay interfaz gráfica. Las "pantallas" son **tres comandos y un libro Excel**, y sus
estados se especifican con el mismo detalle que se especificaría una pantalla, porque son lo
único que el usuario ve.

### Mapa de navegación

```
  python -m src.cli procesar --unidad <u>                      → genera el libro
  python -m src.cli validar  --fichero <f>                     → ingesta el devuelto
  python -m src.cli medir    --unidad <u> --desde <d> --hasta <h>  → informe de resultados
  python -m src.cli anonimizar --persona "<nombre>"            → retira un nombre (REQ-025)
```

### `procesar` — los cuatro estados

- **Con datos:** resumen en pantalla — ficheros leídos, filas cargadas, filas rechazadas,
  medicamentos nuevos en el maestro, propuestas generadas, ruta del libro. Una línea por
  cosa, sin adornos.
- **Vacío** (primer arranque, sin maestro y sin ficheros): **este es el estado del día uno.**
  El comando crea `maestro.xlsx` con sus pestañas y cabeceras, no procesa nada, y dice
  exactamente qué falta y dónde dejarlo: *"No hay ficheros en `datos/entrada/inventario/`.
  Deja ahí la extracción tal como sale del aplicativo y vuelve a ejecutar."* Un piloto que
  arranca con una traza de Python en vez de con esa frase pierde a su único usuario en el
  primer intento.
- **Cargando:** progreso por fichero. Todo lo que pase de 5 segundos dice qué está haciendo;
  un proceso mudo de dos minutos parece colgado.
- **Con error:** una línea que dice qué fichero, qué fila o columna, y qué hacer. Nunca una
  traza desnuda: las trazas van al log.

### `validar` y `medir` — estados

- `validar` **vacío**: no hay libros devueltos → dice dónde se esperan.
- `validar` **error**: fichero manipulado (REQ-071) → nombra la primera línea discordante y
  no registra nada.
- `medir` **vacío**: ninguna propuesta en estado `aplicada` → *"Todavía no hay nada que
  medir: ninguna propuesta ha llegado a aplicarse."*
- `medir` **error**: periodos solapados o invertidos → lo dice y no calcula.

### El libro Excel

Es la interfaz real para el farmacéutico responsable, que no verá jamás la línea de comandos.

- **Con datos:** las pestañas de M7, con `ACCIONES` congelando la primera fila y filtros
  activados. Las marcas van en color: rojo `limitado_por_capacidad`, ámbar
  `capacidad_no_verificada`, gris `sin_datos`.
- **Vacío:** unidad sin propuestas → el libro se genera con sus pestañas y una nota en
  `ACCIONES` que explica el motivo (sin datos suficientes, o sin cambios respecto a lo
  vigente). Un libro sin pestañas no se distingue de un fallo.
- **Accesibilidad:** el color **nunca** es el único portador de información — cada marca
  tiene además su columna de texto. Un farmacéutico dáltónico y una impresión en blanco y
  negro son el mismo caso, y el segundo va a pasar seguro.

### Principio de diseño: sin fatiga de alertas (NFR-011)

Requisito expreso del usuario, y va en contra de la tentación natural de este producto. Con
el mínimo cubriendo 3-4 días, **es previsible que muchísimas líneas salgan recortadas por
capacidad**. Marcar 180 de 220 filas en rojo no informa: entrena a quien lo lee para ignorar
el rojo, y a partir de ahí las 5 marcas que sí importaban tampoco se ven.

La regla, entonces: **cuando una marca afecta a más del 25 % de las líneas de un
dispensador, deja de ser una marca por fila y pasa a ser una frase en `ESTADISTICA`.**

> *"El 78 % de las líneas de este armario no caben para una semana. Esto no son 172
> problemas: es un problema de dimensionamiento del armario. Las 12 líneas donde el ajuste
> individual sí cambia algo son estas."*

Eso es un hallazgo accionable. Ciento setenta y dos celdas rojas no lo son.

### Sistema de diseño

No aplica: no hay interfaz gráfica. Para el libro Excel se fija lo mínimo — primera fila
congelada, filtros, ancho de columna ajustado, y las tres marcas de color con su equivalente
textual.

---

## 9. Integraciones, errores y coste · *capa CÓMO*

**No hay servicios externos.** Ni IA, ni correo, ni pagos, ni tareas programadas, ni API.
Nada sale del equipo del usuario. Por tanto no hay firma de webhooks, ni idempotencia de
eventos, ni contador de consumo, ni identificador de modelo que parametrizar.

**Coste total de explotación: 0 €/mes** (NFR-006).

La única "integración" real es el sistema de ficheros y el formato Excel, que falla más de
lo que la gente supone. Su camino de error es especificación, no pulido:

| Situación | Comportamiento | REQ |
|---|---|---|
| El maestro está abierto en Excel | Aborta antes de escribir; nombra el fichero y pide cerrarlo | REQ-022 |
| Fichero de entrada corrupto o ilegible | Aborta ese fichero, nombra el motivo, sigue con los demás y lo refleja en el resumen | REQ-011 |
| Faltan columnas obligatorias | Aborta ese fichero nombrando las que faltan | REQ-011 |
| Filas con valores no convertibles | Se apartan a rechazos, se cuentan, el proceso sigue | REQ-012 |
| El código BDM del inventario no aparece en el catálogo de ofertas | La línea queda sin precio; la estadística informa del porcentaje de cobertura en vez de dar un importe falsamente completo | REQ-091 |
| Las compras de un artículo no casan con ninguna oferta conocida | `cn_vigente` queda vacío y se señala en el resumen; no se elige una oferta por parecido | REQ-027 |
| El mismo código en mayúsculas y minúsculas | Se normaliza a mayúsculas y se tratan como el mismo artículo | REQ-023 |
| Prefijo de código no reconocido (ni `V`, `Y`, `DM` ni `T`) | Se da de alta con tipo sin determinar y se señala; no se asigna tipo por parecido | REQ-026 |
| Sin espacio en disco a mitad de escritura | El maestro previo sigue intacto gracias a la copia fechada; se avisa de que la escritura no se completó | REQ-021 |
| Fichero > 100 MB | Aviso y aborto controlado | NFR-008 |
| Indicador de paciente detectado | Aborto total: nada escrito en ningún sitio | REQ-002 |

---

## 10. Seguridad, privacidad y cumplimiento · *capa CÓMO*

### Salida del árbol de `cumplimiento.md`

1. **¿Datos de personas identificables?** Sí, pero **profesionales**: quién informó una
   capacidad de hueco (M5) y quién validó una propuesta (M8). No hay datos de pacientes.
2. **¿Datos de salud?** **No.** Es el eje del producto y está protegido por diseño, no por
   confianza: la guarda del módulo M1 inspecciona cada fichero y aborta. **El veto de datos
   clínicos no aplica** porque el producto no los trata.
   ⚠ **Riesgo residual, declarado:** el consumo por medicamento y unidad, en una unidad muy
   pequeña y con un medicamento muy específico, puede acercarse a ser reidentificable.
   Mitigación: el producto no guarda fechas de dispensación individuales, solo agregados por
   periodo, y no cruza consumo con ninguna otra fuente.
3. **¿Salen datos a terceros?** **No.** Sin red, sin nube, sin IA, sin correo.

### Datos personales tratados

| Dato | De quién | Finalidad | Base legal | Retención |
|---|---|---|---|---|
| Nombre de quien informa una capacidad | Técnico de reposición | Poder contrastar el dato con quien lo aportó | Interés legítimo en la gestión del servicio | Vida del piloto |
| Nombre de quien valida una propuesta | Farmacéutico responsable | Trazabilidad de quién decidió qué | Interés legítimo | Vida del piloto |

Minimización: no se guarda ningún otro dato de estas personas — sin correos, sin categoría
profesional, sin identificadores de empleado. **Iniciales en lugar de nombre completo si el
usuario lo prefiere**; es decisión suya y basta con decirlo.

**Borrado y portabilidad.** Si alguien pide que su nombre salga del maestro, se sustituye por
`(retirado)` en las pestañas `HUECOS` y `PROPUESTAS_HIST` mediante
`python -m src.cli anonimizar --persona "<nombre>"`, y el dato técnico —la capacidad del
hueco, la decisión tomada— se conserva, porque no es suyo: es del servicio. La copia fechada
previa a esa operación **también se anonimiza**, o el borrado sería ficticio. La portabilidad
se resuelve sola: todo el contenido ya está en ficheros Excel que la persona puede abrir.
Al cerrar el piloto, si no continúa en v2, se anonimizan todos los nombres de una vez.

### Secretos, aislamiento y validación

- **Secretos:** no hay. Ninguna clave de API, ningún `.env` con contenido real.
- **Aislamiento entre usuarios:** no aplica en la v1 — un solo usuario, ejecución local.
  Queda como requisito de la v2, donde sí habrá farmacéuticos viendo solo sus armarios.
- **Validación de entradas:** toda entrada externa (los ficheros) se valida en los lectores
  antes de tocar el núcleo.
- **Procesos programados:** no hay.
- **Sin datos reales en pruebas:** los tests corren contra `datos/ejemplos/`, que son datos
  **sintéticos inventados**, no extracciones reales anonimizadas (NFR-005).

### Contexto institucional

Aunque no toque datos de pacientes, esto lo va a usar un servicio de un hospital público, y
`cumplimiento.md` exige dejarlo escrito:

- **Responsable funcional:** Yared González Pérez y Alfredo Montero, como coordinadores del
  proceso.
- **Dónde se aloja:** el equipo del usuario. No hay servidor, no hay nube, no hay copia
  fuera del equipo.
- **¿Necesita visto bueno de sistemas o de seguridad?**
  `[PENDIENTE: decidirlo antes de generalizar más allá del piloto.]` Mi lectura, que es
  interpretación y no un hecho: mientras sea un piloto sobre datos de inventario, en el
  equipo del propio responsable del proceso, es análisis de trabajo ordinario. **En cuanto
  la v2 se despliegue para otros farmacéuticos, deja de serlo** y necesita interlocución
  formal. Conviene tener esa conversación antes de que el producto sea un hecho consumado
  que otros ya usan.
- **Copia de seguridad:** el equipo del usuario es el único sitio donde vive el maestro. Si
  ese equipo se pierde, se pierde el piloto. `[PENDIENTE: decidir dónde se respalda.]`

---

## 11. Plan de construcción · *capa ejecución*

**Ritmo fijado por el usuario: un fichero por fase, secuencial. No se pasa al siguiente
hasta haber validado el anterior.** Cada fase cabe en una sesión de trabajo.

Los prompts literales de cada fase están en `docs/anexos/prompts-fases.md`, para no inflar
esta sección.

| Fase | Objetivo | Cierra | Definición de terminado |
|---|---|---|---|
| **1** | Esqueleto del repo y guarda de datos de paciente | REQ-001, REQ-002, REQ-003, NFR-002, NFR-005, **NFR-006, NFR-009** | Se puede pasar un Excel cualquiera por la guarda y esta aborta ante una columna `NHC` sin haber escrito nada. Con tests que lo demuestran. Además: `requirements.txt` no contiene ninguna dependencia de pago ni de red (NFR-006) y no existe ninguna ruta de escritura fuera de la carpeta del proyecto (NFR-009) |
| **2** | Lectura y normalización del **inventario** | REQ-010, REQ-011, REQ-012, REQ-013, NFR-001, NFR-008 | Un fichero real de inventario entra y sale como tabla normalizada, con su fichero de rechazos y su resumen. **Validado con Yared antes de seguir** |
| **3** | Lectura del **histórico de movimientos** | REQ-080 | Se puede contar cuántas roturas hubo en un periodo. **Validado antes de seguir** |
| **4** | **Maestro de artículos**, **catálogo de ofertas** e **informe de compras** | REQ-020 a REQ-029, NFR-003 | El maestro se crea, se actualiza, guarda copia fechada y se niega a escribir si está abierto. Cada artículo tiene su tipo derivado del prefijo y, si la tiene, su oferta vigente con precio. Los excluidos quedan marcados como tales. `anonimizar` retira un nombre del maestro y de sus copias |
| **5** | **Reenvasados** y **base de huecos** | REQ-030, REQ-031, REQ-032, REQ-040, REQ-042 | Las pestañas existen, validan sus valores cerrados y se pueden rellenar a mano sin romper nada |
| **6** | **Núcleo:** motor de optimización | REQ-041, REQ-050 a REQ-059, NFR-004, NFR-007 | Dadas tablas de prueba salen propuestas con su justificación, ningún máximo supera la capacidad, dos ejecuciones dan resultado idéntico, y sale el indicador de referencias por visita frente al objetivo 8-10. **Aquí se revisan con Yared `C_minimo`, `C_objetivo` y `k` con datos reales delante** |
| **7** | **Libro Excel de salida** | REQ-060 a REQ-064, NFR-010, NFR-011 | Se genera un libro completo para una unidad piloto real y se puede repartir. Ninguna marca que afecte a más del 25 % de las líneas se pinta fila a fila: se resume en `ESTADISTICA` |
| **8** | **Ingesta de la validación** | REQ-070, REQ-071, REQ-072 | Un libro devuelto se ingesta, las decisiones quedan registradas, y una fila manipulada lo rechaza |
| **9** | **Medición de resultados** | REQ-081, REQ-082, REQ-090, REQ-091 | Se compara un periodo con otro y sale el informe de las cuatro métricas de §2 |

**Orden:** estrictamente secuencial de la 1 a la 9, por decisión del usuario. Las fases 2, 3
y 4 podrían paralelizarse —son lectores independientes— pero **no se hace**: el criterio es
validar cada fichero antes de pasar al siguiente, y ese criterio manda sobre la eficiencia.

**Dependencia dura:** la fase 6 no puede empezar antes de la 3. El `CV` —la variabilidad del
consumo, que fija el colchón `k`— solo se puede calcular sobre el consumo **diario**, y eso
únicamente está en el histórico de movimientos. El consumo medio del inventario da la media,
no la dispersión, y tratar igual a un artículo que sale 2 al día todos los días y a otro que
sale 14 un martes es exactamente el error que `k` existe para evitar.

### Comprobación de trazabilidad

**Hacia delante** — todo REQ tiene fase: REQ-001/002/003 → F1 · REQ-010/011/012/013 → F2 ·
REQ-080 → F3 · REQ-020 a REQ-029 → F4 · REQ-030/031/032/040/042 → F5 · REQ-041/050-059 → F6 ·
REQ-060-064 → F7 · REQ-070/071/072 → F8 · REQ-081/082/090/091 → F9.

Los once NFR: NFR-002/005/006/009 → F1 · NFR-001/008 → F2 · NFR-003 → F4 · NFR-004/007 → F6 ·
NFR-010/011 → F7. **Sin huérfanos.**

> Nota de auditoría (02/09/2026): en la primera redacción, NFR-006 y NFR-009 no estaban
> asignados a ninguna fase pese a que este párrafo afirmaba lo contrario. Corregido
> asignándolos a la fase 1, que es donde se fija la estructura que los hace ciertos. Queda
> escrito porque un error de trazabilidad que se corrige en silencio vuelve a aparecer.

**Hacia atrás** — toda fase cierra al menos un REQ. **Ninguna tarea sin requisito.**

---

## 12. Setup externo y roadmap · *capa ejecución*

### Checklist de setup, en orden

1. Python 3.11 o superior en el equipo del usuario.
2. `python -m venv .venv` y activar.
3. `pip install -r requirements.txt` (pandas, openpyxl, pyyaml, pytest).
   **Las versiones se fijan al construir, no aquí:** un PRD que clava `pandas==2.1.3`
   caduca antes que el producto. En la fase 1 se genera el `requirements.txt` con las
   versiones vigentes en ese momento, pinneadas exactas, y esas mandan a partir de
   entonces. Python 3.11 sí es un mínimo firme.
4. Dejar la primera extracción de inventario en `datos/entrada/inventario/`.
5. `python -m src.cli procesar --unidad <unidad>` — el primer arranque crea el maestro vacío
   y dice qué falta.

**No hay cuentas que crear, ni servicios que contratar, ni claves que pedir.** Es
deliberado, y es una ventaja del piloto: no depende de la aprobación de nadie para empezar.

### Variables de entorno

**Ninguna.** No hay secretos. La configuración —parámetros `k`, `C_minimo`, `C_objetivo`, umbral de
movimientos mínimos, lista de indicadores de paciente— vive en `config/*.yaml`, versionada
en git, porque no es secreta: es criterio, y el criterio conviene que tenga historial.

### Roadmap

**v1.5 — cierre del piloto**
- Informe de resultados presentable a la supervisión de enfermería y al servicio.
- Ampliación a más unidades, si M1–M3 se cumplen.
- Registro de producción de reenvasados (lo que §3 dejó fuera).

**v2 — el producto para farmacéuticos no técnicos** · *decidido el 02/09/2026; no es una
aspiración, es fase declarada*
- Interfaz web sobre el **mismo núcleo de cálculo**, sin reescribirlo. Esa es la razón por la
  que la v1 separa el núcleo de la E/S.
- Base de datos real y multiusuario, con aislamiento por dispensador: cada farmacéutico ve
  solo sus armarios.
- Validación de propuestas dentro de la herramienta, sin Excel de ida y vuelta.
- Decisión pendiente y consciente: si la v2 se construye aparte o se integra en el módulo
  FarDosys que ya existe en Frello. En la v1 se decidió empezar limpio; **esa decisión no
  prejuzga la de la v2**.

**v3 — ideas aparcadas**
- Captura de la ocupación de huecos (IND-3) y su índice.
- Detección de oportunidades de compra: otro proceso y otros interlocutores.
- Integración de lectura con el sistema de APD, si alguna vez expone una vía.

---

## 13. Instrucción de arranque · *capa ejecución*

Copia este texto entero en una sesión nueva de Claude Code, situada en
`productos/optimizacion-athos`. Sustituye la marca por el contenido de este mismo fichero.

```text
Actúa como desarrollador senior de Python especializado en procesamiento de datos
tabulares y en herramientas de análisis reproducibles. Tienes experiencia construyendo
utilidades de línea de comandos que manejan ficheros Excel sucios del mundo real.

Contexto — vamos a construir "optimizacion-athos":

Es un piloto metodológico que propone, con reglas deterministas y auditables, los mínimos
y máximos de stock de cada dispensador automatizado de medicamentos (ATHOS-Dosys) de un
hospital, a partir de extracciones Excel de inventario, movimientos, ofertas y compras que
un farmacéutico descarga a mano. La salida es un libro Excel por unidad que otro
farmacéutico revisa y valida línea a línea. No hay interfaz web, no hay base de datos y no
hay red: todo se ejecuta en local con Python 3.11, pandas y openpyxl.

Dos cosas del dominio que hay que entender antes de planificar nada, porque determinan la
fórmula entera:

(a) El dispensador se repone CADA 24 H, siempre. Lo que se optimiza no es el plazo de
entrega: es cuántas referencias tiene que tocar el operario en cada visita. El objetivo es
que cada referencia aguante una semana sin ser tocada, y que se toquen 8-10 referencias por
dispensador y visita. El mínimo, en cambio, cubre 3-4 días, para que una subida del consumo
no produzca rotura. Consecuencia: la capacidad física del hueco es la restricción que más
veces va a morder, y por eso la base de huecos no es un accesorio.

(b) La clave de un artículo es el código interno del Servicio Canario de Salud (proyecto
BDM), NO el código nacional. Su prefijo clasifica el artículo — V medicamento, Y fórmula magistral interna,
DM material y otros, T medicamento extranjero — y de ahí se deriva su comportamiento: el
material no se reenvasa, y ni el material ni las fórmulas magistrales tienen código
nacional, luego tampoco precio. El código nacional vive en las ofertas, y el informe de
compras dice cuál está vigente.

El objetivo de esta v1 NO es entregar una herramienta acabada: es demostrar que el método
de optimización funciona sobre 3-4 unidades reales. Existe una v2 planificada, con
interfaz web y multiusuario, que reutilizará el núcleo de cálculo de esta v1 sin
reescribirlo.

PRD completo:
[PEGAR AQUÍ EL CONTENIDO DE PRD.md]

Paso 1 — NO escribas código todavía. Devuélveme:

1. Resumen del producto en 5 líneas: qué es, para quién y cuál es el flujo principal.
2. La lista de requisitos Must de §7, y cuáles son los TRES de mayor riesgo técnico, con
   tu razonamiento. Contrasta tu elección con estos cuatro, que yo considero los más
   delicados, y dime si coincides o no:
   - REQ-002 (abortar sin escribir NADA al detectar datos de paciente: el riesgo está en
     el orden de ejecución y en la atomicidad, no en la detección en sí)
   - REQ-041 (ningún máximo propuesto supera la capacidad física del hueco)
   - REQ-021 y REQ-022 (copia fechada y negativa a escribir si el maestro está abierto:
     son la única mitigación de una decisión de diseño arriesgada tomada a conciencia)
   - NFR-004 (reproducibilidad exacta del cálculo)
3. Plan por fases siguiendo §11, con dependencias y qué se podría paralelizar. El usuario
   ha decidido ir estrictamente secuencial; quiero saber a qué está renunciando.
4. Los supuestos que has tenido que hacer y las contradicciones que encuentres en el PRD.
   Si no encuentras ninguna, dilo explícitamente en vez de callarlo.
5. La estructura de ficheros que propones para src/, coherente con la separación
   núcleo/E-S descrita en §5 y en docs/anexos/arquitectura.md.

Restricciones que no se negocian:

- No inventes funcionalidades que no estén en §3. La lista de fuera de alcance es
  contractual: si crees que falta algo, dilo y para, no lo construyas.
- El núcleo (src/nucleo/) contiene funciones puras: recibe tablas, devuelve propuestas. Ni
  una sola lectura o escritura de fichero, ni una sola ruta, dentro del núcleo. Es lo que
  permite que la v2 reutilice el cálculo. Si en algún momento te resulta cómodo meter un
  pd.read_excel ahí dentro, para y dímelo.
- La guarda de datos de paciente (REQ-001 a REQ-003) se ejecuta ANTES que cualquier otra
  cosa, y su fallo aborta todo el proceso sin escribir ningún fichero. No es una
  validación más: es la regla dura del producto.
- Ningún dato real del hospital entra en el repositorio. Los tests usan datos sintéticos
  inventados, nunca extracciones reales anonimizadas.
- No hay servicios externos, ni claves de API, ni red. Si tu plan necesita alguno, el plan
  está mal.
- Los parámetros del cálculo (k, F por defecto, umbral de movimientos mínimos) viven en
  config/*.yaml, nunca clavados en el código.
- REQ-002, REQ-041 y NFR-004 llevan test obligatorio.
- Un commit por fase.

Formato: markdown, sin preámbulo ni cortesías. Espera mi OK antes de tocar un solo
fichero.
```

---

## Anexo. Registro de cambios

| Fecha | Cambio | Motivo | Secciones tocadas | Decisión |
|---|---|---|---|---|
| 2026-09-03 | M2 se mide contrastando el histórico digital con la hoja de reposición en papel anotada a mano por el técnico, no con un campo estructurado | El usuario aportó el mecanismo real, con foto de una hoja real (dispensador ATHOS 3, LEVETIRACETAM `V15084` como ejemplo de M1 fallando por capacidad) | §2, REQ-082, §3 (fuera de alcance) | Incorporado. Se añade explícitamente que la OCR de la hoja NO entra en el piloto |
| 2026-09-03 | Los extranjeros (`T`) sí tienen código de oferta (no CN nacional) y sí pueden reenvasarse | El usuario lo confirmó; resuelve los dos `[PENDIENTE]` de la fila `T` en §6 | §6 (tabla de prefijos, `OFERTAS.cn`, ámbito de `REENVASADOS`) | Incorporado |
| 2026-09-03 | El reenvasado sí cambia con el proveedor (una rotura de stock puede forzar el cambio); confirma indexar `REENVASADOS` por CN | El usuario lo confirmó; resuelve el `[PENDIENTE]` de si la clave podía simplificarse al código BDM | §6 (nota "Consecuencia sobre el reenvasado") | Incorporado. Ya no es una simplificación pendiente |
| 2026-09-03 | El hospital opera dos centros (HUNSC y Sur); se registran los campos reales de un fichero de inventario/compras con columnas por centro y agregadas. `consumed_ad00` resuelve el origen del consumo global (REQ-090) | El usuario aportó la lista de campos de un fichero real | §6 (tablas en memoria), REQ-090 | Incorporado parcialmente: los campos quedan documentados, pero **qué fichero es exactamente** y **qué columnas usa el piloto** (todas las `EXIST*` de HUNSC o un subconjunto) quedan `[PENDIENTE]` hasta ver el fichero real en fase 4 |
