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

## 2026-09-03 · Sesión 2 · PRD v1.1: validación con cuatro aclaraciones de dominio

**Hecho:** el usuario aprobó el PRD v1.0 ("con estos cambios valido el PRD") aportando en el
mismo turno cuatro aclaraciones de dominio, procesadas por el modo CAMBIO de `forja-prd`.
Actualizado `PRD.md` a v1.1 (estado: aprobado), con entrada en el Anexo · Registro de
cambios por cada una. Compactadas las decisiones vigentes y los bloqueos en `memoria.md`.

**Decidido:**

- **M2 se mide contrastando el histórico digital de reposiciones con la hoja de reposición
  en papel** que el técnico anota a mano al hacer el pedido (círculos, tachones, cantidades
  corregidas). No es un campo estructurado: es un trabajo manual de quien coordina el
  piloto. El usuario adjuntó una foto real —dispensador ATHOS 3, LEVETIRACETAM 500 mg comp
  (`V15084`), cantidad propuesta 78, nota a mano "BASE J 2S3S pequeño para tantos
  comprimidos"— que sirve a la vez de evidencia de M1 fallando por capacidad de hueco.
  Consecuencia añadida al PRD: la lectura automática (OCR) de esa hoja queda explícitamente
  **fuera de alcance**, para que la foto no se lea como pie a construirlo.
- **Los medicamentos extranjeros (`T`) sí tienen código de oferta** (no es CN nacional
  registrado, pero cumple la misma función de clave en `OFERTAS`) **y sí pueden
  reenvasarse**. Cierra los dos últimos `[PENDIENTE]` de la tabla de prefijos de §6.
- **El reenvasado sí cambia con el proveedor** —una rotura de stock puede forzar el cambio—,
  lo que confirma indexar `REENVASADOS` por CN y no por código BDM: ya no es una
  simplificación pendiente de validar.
- **El hospital opera dos centros, HUNSC y "Sur".** El usuario dictó la lista de columnas
  reales de un fichero que llama "inventario de compras": unidades pendientes de recibir,
  existencias por ubicación (almacén, UFA, Kardex, carrusel horizontal/vertical) y consumo,
  cada uno con variante por centro y variante agregada. **Resuelve REQ-090** (de dónde sale
  el consumo global del artículo): la columna `consumed_ad00`, consumo del artículo en todas
  las máquinas ATHOS de HUNSC. Queda `[PENDIENTE]` qué fichero es exactamente este —puede ser
  el "informe de compras" ya previsto en el PRD o uno distinto— y qué columnas `EXIST*` usa
  el piloto; se resuelve al ver el fichero en fase 4.

**Desviaciones del PRD:** ninguna — es precisamente el mecanismo de control de cambios que
el PRD preveía para esto (modo CAMBIO).

**Pendiente:** el usuario aún no ha dejado ningún fichero real en `datos/entrada/`. La fase 1
(esqueleto + guarda de datos de paciente) no los necesita y puede arrancar ya. Quedan siete
`[PENDIENTE]` de dominio en `memoria.md`, ninguno bloqueante para la fase 1.

## 2026-09-03 · Sesión 3 · Llegan los dos primeros ficheros reales; análisis manual de columnas

**Hecho:** el usuario aportó `ofertas_farmatools.xls` (9.215 filas, 34 columnas) y
`stock_21_agos.xls` (2.914 filas, 26 columnas), pidiendo identificar qué es cada variable.
No hay lector construido todavía (fase 1 no ha arrancado), así que el análisis fue manual:
copia local de los `.xls` (formato OLE2/BIFF genuino, `xlrd` de pandas no pudo parsearlos
—"OLE2 inconsistency"—, así que se convirtieron con LibreOffice headless a `.xlsx` para
inspeccionarlos con `openpyxl`), inspección de cabeceras con cribado de indicadores de
paciente (regla dura 4/M1), y verificación cruzada de columnas sobre el dataset completo,
no solo una muestra. Copiados los originales `.xls`, intactos, a `datos/entrada/ofertas/` y
`datos/entrada/compras/` (esta segunda ubicación, provisional). Los ficheros temporales de
trabajo (copia local y conversión) se borraron al terminar.

**Decidido:** ninguna decisión de modelo todavía — todo lo que salió del análisis quedó
como `[PENDIENTE]` en `memoria.md` en vez de asumirse, porque hay demasiadas lecturas
plausibles para cada campo ambiguo. Resumen de lo encontrado:

- **Ninguno de los dos ficheros es el inventario por dispensador** que necesita la fase 2:
  son catálogo/stock a nivel de hospital (unas 20 ubicaciones: almacén, UFA, Kardex,
  carruseles, oncología, curas, estupefacientes…), no por armario ATHOS. Puede que ese dato
  solo exista en el software de APD y no tenga extracción digital — coherente con que la
  evidencia de M2 sea la hoja de reposición en papel.
- **`stock_21_agos.xls` trae exactamente las columnas de dos centros** que el usuario había
  dictado el 03/09/2026 (`ud_pte_rec*`, `EXIST1..68`, `consumed*`, `ubica`), pero no encaja
  del todo con el "informe de compras" que define REQ-027 (fija la oferta vigente): no tiene
  columna de oferta vigente. Podría ser una tercera entidad del modelo.
- **`existencia` y `consumed` no son la suma de sus columnas desglosadas** (67,6 % y 50,7 %
  de las filas cuadran respectivamente, comprobado sobre las 2.914 filas completas, no una
  muestra) — hay ubicaciones y canales de consumo no cubiertos por las columnas con nombre.
  `ud_pte_rec = ud_pte_rec_1 + ud_pte_rec_18` sí cuadra al 100 % (2.913/2.914).
- **Prefijo `EC` no contemplado en el PRD** (`V`/`Y`/`DM`/`T` son los cuatro conocidos):
  aparece en ambos ficheros, son medicamentos de ensayos clínicos (protocolos ATHENEA,
  GEM2017FIT, GEM21MENOS65). Por REQ-026 se pregunta, no se clasifica por parecido.
- **Verificado sin datos de paciente:** las cabeceras que sonaban a alerta (`nombre`,
  `nombre.1`, `nombre_proveedor`) son laboratorio/proveedor, no personas. Los códigos `PAC` y
  `NOGUIA` que aparecían en el catálogo resultaron ser placeholders genéricos ("medicamento
  que aporta paciente", "no guía farmacoterapéutica"), no identificadores de nadie.
- Mapeos que sí quedaron razonablemente confirmados (cruce sobre el dataset completo, no
  adivinados): `dospresen` (ofertas) y `upe` (stock) coinciden en el 89,6 % de los códigos
  comunes → unidades por envase. `tipo` en ofertas correlaciona 1:1 con `descripcio` → parece
  canal de adjudicación de la oferta (concurso / envase normal / envase clínico / …), no tipo
  de artículo.

**Desviaciones del PRD:** ninguna. No se ha escrito ningún lector ni tocado el modelo de
datos de `PRD.md`: los hallazgos ambiguos se dejan como `[PENDIENTE]` hasta que el usuario
los confirme, tal y como exige la regla de "sin respaldo, no lo sé" del proyecto.

**Pendiente:** cinco preguntas nuevas de dominio en `memoria.md` (inventario por
dispensador, encaje de `stock_21_agos` en el modelo, fiabilidad de `existencia`/`consumed`,
prefijo `EC`, significado de `codigo_asociado` y `tipo`). Ninguna bloquea arrancar la fase 1
(esqueleto + guarda de paciente), que sigue sin empezar.
