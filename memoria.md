# Memoria de optimizacion-athos

Estado presente del proyecto. Tope 200 líneas: al superarlo, compacta y baja lo antiguo a `log.md`.
Responde a "dónde estamos", no a "qué pasó".

Última actualización: 2026-09-02

## Fase actual

**Fase 0 · PRD v1.0 redactado, pendiente de aprobación.** Entrevista de `forja-prd`
completa (B0–B10). **No se escribe código hasta el OK explícito del usuario.**

PRD auditado con su propia rúbrica: 28/28. Los cuatro umbrales (M1-M4), los parámetros del
motor y las familias de artículo están confirmados por el usuario. Lo que queda abierto son
nueve `[PENDIENTE]` de dominio, ninguno de los cuales impide arrancar la fase 1.

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
- **[PENDIENTE] Contrato de entrada.** No hay todavía ningún fichero real en
  `datos/entrada/`. Las columnas exactas del inventario, del catálogo de ofertas y del
  informe de compras se fijan al ver los ficheros, no antes. El usuario aportará los dos
  últimos.
- **[PENDIENTE] Reenvasado de las fórmulas magistrales (`Y`).** No tienen CN, y la pestaña
  `REENVASADOS` se indexa por CN. Si se reenvasan, necesita clave alternativa.
- **[PENDIENTE] Los medicamentos extranjeros (`T`)**: si tienen oferta y CN, y si se
  reenvasan.
- **[PENDIENTE] Cómo se reconoce en los datos un suero o gran volumen.** Si hay un campo
  que lo diga, se deriva solo; si no, la pestaña `EXCLUSIONES` se rellena a mano una vez.
- **[PENDIENTE] De dónde sale el valor de `ambito`** (onco-hemato / UFA / general): ¿viene
  en alguna extracción o se marca a mano en el catálogo?
- **[PENDIENTE] ¿Puede un artículo ser de onco-hemato y de UFA a la vez?** Si sí, son dos
  marcas independientes y no un campo de valor único.
- **[PENDIENTE] De dónde sale el consumo global del artículo** (PRD REQ-090). No está en la
  lista de extracciones acordada. Decidir antes de la fase 9.
- **[PENDIENTE] Visto bueno de sistemas/seguridad** antes de generalizar más allá del
  piloto (PRD §10).
- **[PENDIENTE] Dónde se respalda el maestro.** Hoy vive en un único equipo: si se pierde,
  se pierde el piloto (PRD §10).

## Siguiente paso

Que el usuario apruebe el PRD. Los nueve `[PENDIENTE]` se pueden resolver sobre la marcha:
ninguno bloquea la fase 1 (esqueleto y guarda de datos de paciente) ni la fase 2
(inventario). El primero que hará falta de verdad es el contrato de entrada, en la fase 2,
y se resuelve viendo el fichero.
