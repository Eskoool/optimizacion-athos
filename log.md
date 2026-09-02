# Log de optimizacion-athos

Append only. No se reescribe nunca, no se lee entero: se busca.
Responde a "qué pasó y cuándo", incluidas las decisiones que en su momento parecieron menores.

Formato de entrada:

```
## AAAA-MM-DD · Sesión N · [título]
**Hecho:** ...
**Decidido:** ...
**Desviaciones del PRD:** ninguna | [cuál y cómo se resolvió]
**Pendiente:** ...
```

---

## 2026-09-02 · Sesión 0 · Andamiaje y bloque B6 de la entrevista

**Hecho:** creada la estructura del proyecto con Forja PRD. Reescrito `CLAUDE.md` con los
invariantes reales (la plantilla venía con reglas de aplicación web —navegador, RLS,
servidor— que no aplican a un piloto local en Python). Definida la arquitectura de
`datos/` con su contrato en `datos/LEEME.md` y `.gitignore` que mantiene fuera del
historial cualquier dato real del hospital.

**Decidido:**
- **La v1 es un piloto metodológico**, no el producto. Claude Code + scripts de Python +
  Excel, ejecutado localmente. El objetivo es demostrar que el método de optimización
  funciona, no entregar herramienta.
- **La v2 —producto acabado para farmacéuticos no técnicos— queda declarada** como fase
  del roadmap, a petición expresa del usuario de que entre en el ciclo de vida del
  proyecto y no se quede en una intención.
- **Restricción de arquitectura derivada de lo anterior:** núcleo de cálculo en funciones
  puras, separado de la lectura/escritura de Excel. Es lo que evita que la v2 sea una
  reescritura.
- **Datos semilla: empezar limpio.** No se importa nada de Frello, pese a que el módulo
  FarDosys existe allí con 47 dispensadores y 6.414 pactos de stock.
- **Entradas del piloto:** inventario (ubicación de medicamentos, mín/máx, consumo medio,
  histórico de movimientos) y un catálogo de ofertas a crear (laboratorio, CN, precio,
  unidad, y más campos que aportará el usuario). **Quedan fuera** la extracción de calidad
  y las prescripciones no disponibles (IND-1).
- **Método:** reglas deterministas y auditables, descartados el enfoque predictivo y el
  uso de un LLM para redactar las propuestas.
- **Reenvasados es un catálogo**, no un registro de producción: por medicamento, se
  reenvasa / se etiqueta / ya viene en unidosis, aplicable solo a comprimidos y cápsulas.
  No aplica a ampollas, viales ni inyectables.

**Desviaciones del PRD:** ninguna respecto al PRD (todavía no existe). Sí una desviación
del orden de `forja-prd`: el andamiaje se ha creado en el paso 6 antes del checkpoint del
paso 5, a petición del usuario, que necesita la carpeta para dejar ficheros. `PRD.md`
queda como esqueleto con todas las secciones en `[PENDIENTE]` y no se rellena hasta que la
entrevista termine.

**Nota (misma fecha, sesión 1):** varias de estas decisiones se corrigieron horas después,
en la misma sesión. Ver la entrada siguiente. Se conservan aquí porque el log no se
reescribe: saber qué se creyó y durante cuánto explica decisiones posteriores.

**Pendiente:** bloques B7–B10 de la entrevista, redactar el PRD, checkpoint, y solo
entonces fase 1. Tres `[PENDIENTE]` abiertos en `memoria.md`: Unibot vs. Unibox/Unipack,
el contrato de columnas de entrada, y para qué sirve el catálogo de ofertas.

## 2026-09-02 · Sesión 1 · PRD v1.0 completo, y el motor reescrito dos veces

**Hecho:** entrevista de `forja-prd` cerrada (B0–B10). Redactado `PRD.md` v1.0 con 46
requisitos funcionales en notación EARS y 11 no funcionales con umbral, todos trazados a
fase. Escritos `docs/anexos/prompts-fases.md` (instrucción literal de las nueve fases) y
`docs/anexos/arquitectura.md`. Rúbrica de auditoría de la skill pasada al documento: 28/28
tras corregir seis fallos, uno de ellos propio —§11 afirmaba "sin huérfanos" mientras
NFR-006 y NFR-009 no tenían fase asignada—, y ese error queda escrito dentro del propio PRD.

**Decidido, y estas son las que cambiaron el producto:**

- **La clave natural NO es el código nacional.** Es el código interno del Servicio Canario
  de Salud, proyecto BDM, cuyo prefijo clasifica el artículo: `V` medicamento (`V00210`),
  `Y` fórmula magistral interna (`Y80879`), `DM` material y otros (`dm000116`, en
  minúsculas), `T` medicamento extranjero (`T80502`). El CN vive un nivel por debajo, en las
  ofertas, y el **informe de compras** dice cuál está vigente. Se rehízo el modelo de datos
  entero: `MEDICAMENTOS` pasó a `ARTICULOS`, nació `OFERTAS`, y `HUECOS`, `REENVASADOS` y
  las tablas en memoria se reindexaron.
  ⚠ Antes de esto, el agente leyó "código V T DM Y" como un acrónimo, "VTDM", y llegó a
  escribirlo así en el PRD. Lo corrigió el usuario con cuatro ejemplos reales.

- **El motor no optimiza stock: optimiza el trabajo del operario.** El dispensador se repone
  cada 24 h pase lo que pase. Lo que se decide es cuántas referencias hay que tocar en cada
  visita: cada una debería aguantar una semana, y el objetivo son 8-10 referencias por
  visita. Eso convirtió la frecuencia de reposición de variable deducida en constante del
  proceso, y añadió **M4** como cuarta métrica de éxito.

- **El mínimo cubre 3-4 días, no las 24 h de la visita.** Corrección del usuario sobre la
  primera versión de la fórmula: atar el mínimo al plazo del operario deja el armario sin
  margen si sube el consumo medio. El mínimo protege contra la demanda, no contra el
  retraso. Consecuencia asumida y escrita: para aguantar la semana una referencia necesita
  que en su hueco quepan ~10-11 días de consumo, así que **muchas más líneas chocarán contra
  la capacidad**, y la base de huecos pasa de accesorio a pieza que decide si el piloto es
  interpretable.

- **El armario lleva medicamentos y solo medicamentos.** Entran `V`, `Y` y `T`. **`DM` sale
  del alcance entero**, no solo del reenvasado. Fuera también sueros y grandes volúmenes.

- **UFA no es una exclusión, es una clasificación.** El usuario la retiró de la lista de
  exclusiones y pidió una propiedad de catálogo, `ambito`: `onco_hemato` | `ufa` | `general`.
  Clasificar conserva la información y permite segmentar; excluir la tira.

- **Objetivo de roturas: cero.** No llegar nunca a cajetín 0. El −80 % es el umbral con el
  que se declara éxito, no la meta. El informe da siempre el número absoluto y lista las
  roturas que quedan.

- **Sin fatiga de alertas (NFR-011).** El umbral de movimientos mínimos queda **desactivado**
  por decisión del usuario: con datos escasos el sistema no marca, aplica el `k` más
  conservador (0,6) y protege. Y ninguna marca que afecte a más del 25 % de las líneas de un
  dispensador se pinta fila a fila: se resume como un problema del armario. Marcar 180 de
  220 filas en rojo entrena a ignorar el rojo.

- Umbrales confirmados: M1 100 % · M2 ≥ 90 % · M3 −80 % · M4 8-10 refs/visita ·
  `k` 0,2/0,4/0,6 · `C_minimo` 3-4 días · `C_objetivo` 7 días.
- Salida: **un libro Excel por unidad**, con los datos crudos en pestañas, `ACCIONES`,
  `ESTADISTICA` y `VALIDACION`.
- Ritmo: **un fichero por fase, secuencial**, sin pasar al siguiente hasta validar.

**Desviaciones del PRD:** ninguna. El PRD nació en esta sesión.

**Pendiente:** aprobación del usuario. Nueve `[PENDIENTE]` de dominio abiertos en
`memoria.md`; ninguno bloquea las fases 1 y 2. El usuario aportará el catálogo de ofertas y
el informe de compras en `datos/entrada/`.
