# Memoria de optimizacion-athos

Estado presente del proyecto. Tope 200 líneas: al superarlo, compacta y baja lo antiguo a `log.md`.
Responde a "dónde estamos", no a "qué pasó".

Última actualización: 2026-09-03

## Fase actual

**PRD v1.1 aprobado el 03/09/2026.** El usuario validó el PRD y aportó cuatro aclaraciones
de dominio en la misma sesión (registradas en el Anexo de `PRD.md` y en el log). **Lista para
arrancar la fase 1** (esqueleto + guarda de datos de paciente): no necesita ningún fichero
real.

**Llegaron los dos primeros ficheros reales el 03/09/2026**, analizados a mano (sin código
de producto todavía, no hay lector construido): `ofertas_farmatools.xls` →
`datos/entrada/ofertas/` y `stock_21_agos.xls` → `datos/entrada/compras/` (ubicación
provisional, ver bloqueos). Ninguno de los dos es el fichero de **inventario por
dispensador** (con `dispensador`/`sección`/`hueco`/mín/máx vigentes) que necesita la fase 2 —
sigue faltando. Hallazgos del análisis abajo, en bloqueos.

PRD auditado con su propia rúbrica en la v1.0: 28/28. Los cuatro umbrales (M1-M4), los
parámetros del motor y las familias de artículo están confirmados por el usuario. De los
nueve `[PENDIENTE]` de dominio originales, dos se resolvieron en esta sesión (ver abajo);
ninguno de los que quedan impide seguir con la fase 1.

## Decisiones vigentes

| Decisión | Elección | Por qué |
|---|---|---|
| Qué es la v1 | **Piloto metodológico**, no producto | Primero se demuestra que el método de optimización funciona sobre dispensadores concretos; generalizar antes de eso es construir sobre una hipótesis |
| Producto final | **v2 declarada**, para farmacéuticos no técnicos | Decisión del usuario el 02/09/2026. No es una aspiración: es fase del roadmap y condiciona la arquitectura de la v1 |
| Motor de build | claude-code | Piloto local ejecutado por Yared; sin despliegue ni usuarios externos en la v1 |
| Stack | Python 3 + pandas + openpyxl, Excel de entrada y salida | Los datos llegan como extracciones de aplicativos internos; no hay API que consultar y no hay base de datos en la v1 |
| Arquitectura | **Núcleo puro separado de la E/S** | Es lo que permite que la v2 ponga una interfaz encima en vez de reescribir. Restricción, no recomendación |
| Método de cálculo | **Reglas deterministas y auditables** | Cada propuesta se defiende número a número ante el farmacéutico responsable. Un modelo que no se puede explicar no se valida |
| **Qué se optimiza de verdad** | **El trabajo del operario, no el stock** | El dispensador se repone cada 24 h siempre. El objetivo es que cada referencia aguante una semana sin tocarla y que se toquen **8-10 referencias por visita**. Confirmado por el usuario el 02/09/2026 |
| **Mínimo** | Cubre **3-4 días**, no las 24 h de la visita | Corrección expresa del usuario: atarlo a la visita deja el armario sin margen si sube el consumo medio. El mínimo protege contra la demanda, no contra el retraso del operario |
| **Máximo** | Mínimo + 7 días de consumo | Es el dial del trabajo: `(máx − mín) / CMD` son los días entre reposiciones de esa referencia |
| **Umbrales** | M2 ≥ 90 % · M3 −80 % · M4 8-10 refs/visita · `k` 0,2/0,4/0,6 | Confirmados por el usuario el 02/09/2026 |
| **Familias en el piloto** | `V` medicamentos, `Y` fórmulas magistrales, `T` extranjeros | **`DM` (material) fuera: no se pone en el armario.** Tampoco sueros ni grandes volúmenes. Si aparece otro prefijo, se pregunta |
| **Extranjeros (`T`)** | Sí tienen código de oferta (no CN nacional) y sí pueden reenvasarse | Confirmado por el usuario el 03/09/2026. Cierra los dos últimos `[PENDIENTE]` de la tabla de prefijos |
| **Cómo se mide M2** | Contrastando el histórico digital con la hoja de reposición en papel, anotada a mano por el técnico | Confirmado el 03/09/2026, con foto de una hoja real. Es dato manual: el software no lo calcula. Sirve también de evidencia de M1 |
| **UFA y onco-hemato** | **Propiedad `ambito` del catálogo, no exclusión** | El usuario retiró UFA de la lista de exclusiones el 02/09/2026. Clasificar conserva la información y permite segmentar; excluir la tira |
| **Objetivo de roturas** | **Cero: no llegar nunca a cajetín 0** | El −80 % es el umbral con el que se declara éxito, no la meta. El informe da siempre el número absoluto y lista las roturas que quedan |
| **Umbral de movimientos mínimos** | **Desactivado** (`min_movimientos` = 0) | Decisión del usuario: no quiere fatiga de alertas. Con datos escasos el sistema no marca, aplica el `k` más conservador y protege |
| **Sin fatiga de alertas** | NFR-011: una marca que supere el 25 % de las líneas se resume, no se pinta fila a fila | Con mínimo de 3-4 días saldrán muchísimas líneas recortadas por capacidad. 180 celdas rojas entrenan a ignorar el rojo |
| **Reenvasado** | Indexado por **CN**, la presentación concreta | Venir en unidosis depende del laboratorio adjudicado: si cambia la oferta, cambia la clasificación |
| Datos de paciente | **Detectar → alertar → no persistir** | Regla dura del usuario. Requisito verificable en §10, no advertencia de cumplimiento |
| Carácter de la salida | **No vinculante**, validación humana obligatoria | Vive aparte del software de APD Solutions; no lo sustituye ni escribe en él |
| Ámbito de validación | Dispensadores concretos, no agregado | El piloto valida la metodología antes de generalizar |
| Datos semilla | **Empezar limpio**, nada de Frello | Decisión del usuario tomada ya conociendo que FarDosys existe en Frello con 47 dispensadores y 6.414 pactos de stock |
| Alcance de reenvasados | **Catálogo**, no registro de producción | Se reenvasa / se etiqueta / ya viene en unidosis. Solo comprimidos y cápsulas, y solo de tipo medicamento |
| **Clave natural** | **Código interno del SCS, proyecto BDM** | Confirmado por el usuario el 02/09/2026. **No es el código nacional.** El prefijo clasifica: `V` medicamento (`V00210`) · `Y` fórmula magistral interna (`Y80879`) · `DM` material y otros (`dm000116`, en minúsculas) · `T` medicamento extranjero (`T80502`) |
| **CN y precio** | Viven en la entidad **Oferta**, no en el artículo | Cada artículo puede tener varias ofertas; el **informe de compras** dice cuál está vigente. Las fórmulas magistrales y el material no llegan a tener CN |
| Salida | **Un libro Excel por unidad** | Confirmado por el usuario el 02/09/2026 |
| Usuarios v1 | Usuario único (Yared ejecuta, Alfredo valida) | Sin autenticación ni aislamiento en la v1; todo eso es problema de la v2 |

## Deuda técnica abierta

_(vacío — no hay código todavía)_

## Bloqueos

- **[PENDIENTE] Unibot vs. Unibox/Unipack.** El usuario nombra "Unibot"; el wiki tiene
  registrados Unibox (reenvasado) y Unipack (corte y reenvasado). No se unifican sin
  confirmar: puede ser la misma máquina con otro nombre de uso, o dos distintas.
- **[PENDIENTE] Sigue faltando el inventario por dispensador.** Ni `ofertas_farmatools.xls`
  ni `stock_21_agos.xls` traen `dispensador`/`sección`/`hueco`/mín-máx vigentes: son
  catálogos y stock **a nivel de hospital** (almacén, UFA, Kardex, carruseles, oncología,
  curas, estupefacientes…), no por armario ATHOS. La `ubica` de `stock_21_agos` tiene ~20
  valores y ninguno parece un dispensador piloto. **Hipótesis a confirmar con el usuario:**
  ese dato solo existe en el software de APD (de ahí que la evidencia de M2 sea la hoja de
  reposición en papel, no un fichero) — si es así, el "inventario" de la fase 2 puede no
  tener extracción digital y hay que replantear cómo se alimenta REQ-010.
- **[PENDIENTE] Dónde encaja `stock_21_agos.xls` en el modelo.** Confirmado el 03/09/2026:
  trae exactamente las columnas de dos centros que el usuario dictó (`ud_pte_rec*`,
  `EXIST1..68`, `consumed*`, `ubica`) — es el fichero al que se refería. Colocado
  provisionalmente en `datos/entrada/compras/`, pero **no encaja del todo** con lo que el
  PRD llama "informe de compras" (fija la oferta vigente, REQ-027): no tiene columna de
  oferta vigente ni de fecha de compra por línea. Puede ser una tercera entidad
  ("inventario global", distinta de "inventario por dispensador" y de "informe de compras")
  y el modelo de §6 necesitaría una tabla más. Decidir antes de fase 4.
- **[PENDIENTE] `existencia` y `consumed` NO son la suma de sus columnas desglosadas.**
  Comprobado sobre las 2.914 filas de `stock_21_agos.xls`: `existencia` = suma de
  `exist1..exist68` solo en el 67,6 % de las filas (hay más ubicaciones de las que cubren
  esas 9 columnas); `consumed` = suma de las 4 columnas de consumo por centro/máquina solo en
  el 50,7 %. **Consecuencia:** `consumed_ad00` (consumo en máquinas ATHOS de HUNSC) es útil
  como dato específico, pero **no puede tratarse como "el resto" de `consumed`** — hay que
  usar `existencia` y `consumed` tal cual como totales autoritativos, no reconstruirlos.
  `ud_pte_rec = ud_pte_rec_1 + ud_pte_rec_18` sí cuadra exacto (2.913 de 2.914 filas).
- **[PENDIENTE] Prefijo `EC` no contemplado.** Aparece en ambos ficheros (14 filas en
  ofertas, 34 en stock): son medicamentos de ensayos clínicos/protocolos (ATHENEA,
  GEM2017FIT, GEM21MENOS65 — protocolos de mieloma del grupo GEM/PETHEMA). El PRD (§6) solo
  cubre `V`/`Y`/`DM`/`T` — por REQ-026 esto se pregunta, no se clasifica por parecido. Además
  `cod_nac` está vacío en el 64,7 % de estas filas en `stock_21_agos`, coherente con que un
  fármaco en ensayo clínico no tenga CN.
- **[PENDIENTE] Significado de `codigo_asociado` (stock) y `tipo`/`codigo` (ofertas).**
  `codigo_asociado` casi siempre repite el propio código BDM, pero en 1.043 de 2.914 filas
  trae valores categóricos: `DPE`, `DPA`, `PAC`, `SUER 30 00`, etc. — parecen un canal o
  grupo de dispensación, no otro código de producto (confirmado: no hay dato de paciente
  real detrás — los códigos `PAC`/`NOGUIA` del catálogo son placeholders genéricos,
  "medicamento que aporta paciente" y "no guía farmacoterapéutica", no identifican a nadie).
  Y `tipo` en `ofertas_farmatools` (`EN`/`EC`/`SF`/`FO`/`SA`/`PR`/`HE`/`MS`) correlaciona 1:1
  con la columna `descripcio` (envase normal / envase clínico / concurso / fórmulas /
  sanidad / material sanitario / productos / hemoderivados) — parece el canal de
  adjudicación de la oferta, no el tipo de artículo (que ya da el prefijo del código). Ni uno
  ni otro se usan en el modelo hasta que el usuario confirme qué significan.
- **[PENDIENTE] Reenvasado de las fórmulas magistrales (`Y`).** No tienen CN, y la pestaña
  `REENVASADOS` se indexa por CN. Si se reenvasan, necesita clave alternativa.
- **[PENDIENTE] Cómo se reconoce en los datos un suero o gran volumen.** Si hay un campo
  que lo diga, se deriva solo; si no, la pestaña `EXCLUSIONES` se rellena a mano una vez.
- **[PENDIENTE] De dónde sale el valor de `ambito`** (onco-hemato / UFA / general): ¿viene
  en alguna extracción o se marca a mano en el catálogo?
- **[PENDIENTE] ¿Puede un artículo ser de onco-hemato y de UFA a la vez?** Si sí, son dos
  marcas independientes y no un campo de valor único.
- **[PENDIENTE] Visto bueno de sistemas/seguridad** antes de generalizar más allá del
  piloto (PRD §10).
- **[PENDIENTE] Dónde se respalda el maestro.** Hoy vive en un único equipo: si se pierde,
  se pierde el piloto (PRD §10).

Resueltos el 03/09/2026: si los extranjeros (`T`) tienen oferta/CN y si se reenvasan (sí a
ambas), y de dónde sale el consumo global del artículo para REQ-090 (`consumed_ad00`, a
falta de confirmar el fichero exacto — ver arriba).

## Siguiente paso

**Arrancar la fase 1** (esqueleto del repo + guarda de datos de paciente, REQ-001 a
REQ-003): no necesita ningún fichero real, se puede hacer ya. En paralelo, el usuario aporta
los ficheros reales de `datos/entrada/` (inventario primero, es lo que abre la fase 2). El
contrato de entrada exacto —columnas de inventario, ofertas e "inventario de compras"— se
fija al ver esos ficheros, no antes.
