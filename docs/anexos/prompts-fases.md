# Prompts literales por fase

Uno por fase de PRD §11. Se pegan en una sesión de Claude Code situada en la carpeta del
proyecto. **Todos son autocontenidos**: quien los pega no lleva encima nada de la
conversación en que se escribieron.

Preámbulo común a todas las fases (va antes del texto de cada una):

```text
Actúa como desarrollador senior de Python especializado en procesamiento de datos
tabulares y herramientas de análisis reproducibles.

Contexto: proyecto "optimizacion-athos", piloto local que propone mín/máx de stock para
dispensadores automatizados de medicamentos a partir de extracciones Excel. Python 3.11,
pandas, openpyxl, sin base de datos, sin red, sin servicios externos.

Lee CLAUDE.md, PRD.md y memoria.md antes de nada. El PRD manda: si algo de lo que te pido
lo contradice, para y dímelo en vez de resolverlo por tu cuenta.
```

Y el cierre común, después de cada fase:

```text
Fase <N> terminada. Antes de seguir:
1. Verifica la definición de terminado de PRD §11 punto por punto y dime qué falta.
2. Enumera lo que hiciste que NO estaba en el PRD, por pequeño que sea.
3. Ejecuta /checkpoint.
Si algo del punto 2 no es trivial, no sigas: pasa por /cambio.
```

---

## Fase 1 · Esqueleto y guarda de datos de paciente

```text
Tarea: construye el esqueleto del repositorio y el módulo de guardas.

1. Estructura: src/guardas/, src/lectores/, src/nucleo/, src/informes/, src/cli.py,
   config/, tests/. requirements.txt con pandas, openpyxl, pyyaml, pytest.
2. config/indicadores_paciente.yaml: lista editable de indicadores (NHC, CIP, DNI, NIE,
   historia, episodio, nombre, apellidos, fecha_nacimiento). El código no lleva la lista
   clavada.
3. src/guardas/: inspecciona cabeceras y una muestra de >=200 filas de un Excel y devuelve
   un veredicto. Cierra REQ-001, REQ-002, REQ-003 y NFR-002.
4. tests/ con datos sintéticos INVENTADOS en datos/ejemplos/: un fichero limpio y otro con
   una columna NHC.

Restricciones:
- Al detectar un indicador se aborta el proceso entero. No se escribe ningún fichero, no
  se toca el maestro, y el aviso nombra fichero y columna pero NUNCA un valor.
- El log de guardas no puede contener ninguna cadena procedente de los datos.
- Un falso positivo es aceptable; un falso negativo no. Ante la duda, aborta.
- REQ-002 lleva test obligatorio, incluida la comprobación de que no se escribió nada.

Formato: markdown con el plan, y espera mi OK antes de crear ficheros.
```

## Fase 2 · Lectura y normalización del inventario

```text
Tarea: el lector de ficheros de inventario.

1. config/perfiles/inventario.yaml: mapeo columna de origen → campo interno, fila de
   cabecera, columnas obligatorias.
2. src/lectores/inventario.py: aplica el perfil y devuelve la tabla normalizada
   inventario(dispensador, seccion, hueco, codigo_bdm, descripcion, min_vigente,
   max_vigente, existencias, consumo_medio, fecha_extraccion).
3. Fichero de rechazos en datos/salida/rechazos_<fecha>.xlsx con el motivo por fila.
4. Resumen de ejecución por pantalla: filas leídas, cargadas, rechazadas.

Cierra REQ-010, REQ-011, REQ-012, REQ-013, NFR-001, NFR-008.

Restricciones:
- Los ficheros de datos/entrada/ no se modifican, ni siquiera su fecha.
- Las filas malas se apartan y se cuentan; jamás se descartan en silencio.
- Si faltan columnas obligatorias, se aborta ese fichero nombrándolas.
- La guarda de la fase 1 corre antes que este lector, siempre.
- Casos que deben tener test: cabecera desplazada, número como texto con separador de
  miles, fila de totales al final, valor no convertible.

Formato: markdown con el plan, y espera mi OK.
```

## Fase 3 · Histórico de movimientos

```text
Tarea: el lector del histórico de movimientos y la detección de roturas.

1. config/perfiles/movimientos.yaml.
2. src/lectores/movimientos.py → movimientos(dispensador, codigo_bdm, fecha, tipo,
   cantidad, cantidad_hallada).
3. Función que cuente roturas: toda línea en la que cantidad_hallada sea 0 al reponer.
   Cierra REQ-080.
4. Serie de consumo DIARIO por dispensador y artículo. De ella salen CMD (media) y CV
   (coeficiente de variación). Las consumirá el núcleo en la fase 6.

Restricciones:
- CMD y CV son cálculo, así que viven en src/nucleo/, no en el lector. El lector entrega la
  tabla de movimientos; el núcleo la interpreta.
- CV necesita consumo DIARIO, no la media mensual. El consumo medio que trae el inventario
  da la media pero no la dispersión, y la dispersión es justo lo que fija el colchón de
  seguridad. Si te ves tentado de usar el consumo medio del inventario para esto, para y
  dímelo.
- La frecuencia de reposición NO se deduce del histórico: el dispensador se visita cada
  24 h siempre, por definición del proceso. Lo que se deduce del histórico es el consumo.

Formato: markdown con el plan, y espera mi OK.
```

## Fase 4 · Maestro de artículos, ofertas e informe de compras

```text
Tarea: el maestro persistente, el lector de ofertas y el lector del informe de compras.

1. src/lectores/ofertas.py y src/lectores/compras.py, con sus perfiles.
2. src/informes/maestro.py: crea maestro.xlsx si no existe (pestañas ARTICULOS, OFERTAS,
   REENVASADOS, HUECOS, EXCLUSIONES, PROPUESTAS_HIST, CAMBIOS con sus cabeceras), y lo
   actualiza: alta de códigos nuevos, actualización de campos cambiados, registro en
   CAMBIOS.
3. EXCLUSIONES(codigo_bdm, motivo, detalle, fecha): lo que NO va al armario y por tanto no
   se optimiza. motivo es lista cerrada: material | gran_volumen | otro.
   Entran al piloto V, Y y T. DM queda FUERA del alcance entero, no solo del reenvasado.
   Fuera también los sueros y grandes volúmenes (p. ej. paracetamol intravenoso).
   Un artículo excluido que aparezca en el armario NO se omite en silencio: se reporta
   como hallazgo (REQ-028).
4. ARTICULOS lleva además la propiedad ambito: onco_hemato | ufa | general (REQ-029).
   OJO: UFA NO es una exclusión. Es una clasificación. Un medicamento de UFA entra en el
   cálculo como cualquier otro; la marca solo permite segmentar la estadística. Excluir
   tira información, clasificar la conserva.

Cierra REQ-020 a REQ-029 y NFR-003.

Contexto imprescindible sobre la codificación, porque no es lo que uno espera:
- La clave natural es el CÓDIGO INTERNO del Servicio Canario de Salud (proyecto BDM), NO
  el código nacional.
- El prefijo del código clasifica el artículo y de él se DERIVA tipo_articulo, que nunca
  se teclea:
    V  → medicamento              (ej. V00210)
    Y  → fórmula magistral interna (ej. Y80879)   · no tiene CN ni oferta
    DM → material y otros          (ej. dm000116) · no tiene CN, no se reenvasa
    T  → medicamento extranjero    (ej. T80502)
- El código nacional vive en OFERTAS, no en ARTICULOS. Un artículo puede tener varias
  ofertas, cada una con su CN, laboratorio, presentación y precio.
- El INFORME DE COMPRAS es lo que determina qué oferta está vigente. Sin él hay precios
  pero no se sabe cuál aplicar.

Restricciones, todas obligatorias y con test:
- ANTES de escribir, copia el maestro a datos/intermedio/copias/maestro_<fecha_hora>.xlsx.
- Si el maestro está abierto o bloqueado, aborta sin escribir nada y dilo nombrando el
  fichero.
- El código BDM se normaliza a MAYÚSCULAS antes de comparar: dm000116 y DM000116 son el
  mismo artículo. Si no lo haces, el maestro se duplica en silencio.
- Dos filas con el mismo código abortan la escritura.
- Un prefijo no reconocido NO se asigna por parecido: se da de alta con tipo sin
  determinar y se señala en el resumen.
- Si las compras de un artículo no casan con ninguna oferta conocida, cn_vigente queda
  vacío y se señala. No elijas una oferta al azar.
- primera_vez_visto se escribe una vez y no se toca nunca más.
- Conserva al menos las 30 últimas copias.

Esta fase implementa la mitigación de una decisión de diseño arriesgada tomada a
conciencia por el usuario (Excel actualizado en sitio, sin base de datos). No la relajes.

Formato: markdown con el plan, y espera mi OK.
```

## Fase 5 · Reenvasados y base de huecos

```text
Tarea: las pestañas REENVASADOS y HUECOS del maestro, y su validación.

1. REENVASADOS(cn, tratamiento, maquina, observaciones). tratamiento es lista cerrada:
   se_reenvasa | se_etiqueta | ya_unidosis. Cualquier otro valor se rechaza.
2. HUECOS(dispensador, seccion, hueco, codigo_bdm, cn, cantidad_maxima, informado_por,
   fecha), clave natural (dispensador, seccion, hueco, codigo_bdm), sin duplicados. El cn
   guarda con qué presentación se midió: si cambia la oferta, la capacidad puede caducar.
3. Validación de datos de Excel en las columnas de valor cerrado, para que quien rellene a
   mano no pueda escribir cualquier cosa.
4. Listado de comprimidos y cápsulas sin tratamiento asignado.

Cierra REQ-030, REQ-031, REQ-032, REQ-040, REQ-042.

Restricciones:
- El tratamiento de reenvasado SOLO aplica a comprimidos y cápsulas. Para ampollas, viales
  e inyectables el campo NO APLICA: no se exige, no se cuenta como pendiente y no aparece
  entre los sin clasificar. No es lo mismo que estar vacío.
- Las dos pestañas se rellenan a mano por personas: si el fichero se puede romper
  rellenándolo, está mal hecho.

Formato: markdown con el plan, y espera mi OK.
```

## Fase 6 · Núcleo: motor de optimización

```text
Tarea: el motor de cálculo de mín/máx. Es la pieza que la v2 reutilizará sin reescribir.

Contexto del proceso, que es lo que explica la fórmula y no es evidente:
- El dispensador se repone CADA 24 H, siempre. La visita es diaria y no depende de nada.
- Cada referencia debería necesitar reposición como mucho UNA VEZ POR SEMANA.
- El objetivo operativo es tocar 8-10 referencias por dispensador y visita.
- El mínimo NO se ata a las 24 h de la visita: cubre 3-4 días, para que una subida del
  consumo medio no produzca rotura. Protege contra la demanda, no contra el operario.

Implementa en src/nucleo/, con funciones puras, las reglas de PRD §7 M6:
  CMD        = consumo del periodo / días del periodo
  CV         = desviación típica / media del consumo DIARIO
  k          = 0,2 si CV<0,3 · 0,4 si 0,3<=CV<0,6 · 0,6 si CV>=0,6
  C_minimo   = 3 días (ajustable 3-4)
  C_objetivo = 7 días

  minimo = max(1, techo(CMD × C_minimo × (1+k)))
  maximo = minimo + techo(CMD × C_objetivo)
  maximo = min(maximo, cantidad_maxima_del_hueco)
  ambos redondeados al alza al múltiplo de la unidad de dispensación

  dias_entre_reposiciones     = (maximo − minimo) / CMD            objetivo >= 7
  reposiciones_diarias_espera = Σ CMD_i / (max_i − min_i)          objetivo 8-10

Cierra REQ-041, REQ-050 a REQ-059, NFR-004, NFR-007.

Restricciones que no se negocian:
- NI UNA lectura o escritura de fichero dentro de src/nucleo/. Ni una ruta. Ni un
  pd.read_excel. Recibe tablas, devuelve propuestas. Si te resulta cómodo saltártelo,
  para y dímelo.
- k, los cortes de CV, C_minimo, C_objetivo y min_movimientos se leen de config/, no se
  clavan.
- min_movimientos vale 0 por defecto: el umbral de 'datos insuficientes' está DESACTIVADO
  por decisión del usuario, que no quiere fatiga de alertas. Con pocos movimientos NO se
  marca la línea: se aplica el k más conservador (0,6) y se propone igual. Ante datos
  escasos el sistema protege, no avisa.
- Ningún máximo propuesto puede superar la capacidad del hueco (REQ-041, test
  obligatorio).
- Con un mínimo de 3-4 días, para aguantar la semana una referencia necesita que en su
  hueco quepan unos 10-11 días de consumo. Van a salir MUCHAS líneas recortadas por
  capacidad. Eso no es un fallo del cálculo: es el armario diciendo la verdad, y hay que
  reportarlo como hallazgo de ubicación (REQ-057, REQ-059), no esconderlo bajando el
  mínimo por tu cuenta.
- Si un artículo no cabe ni para el colchón mínimo, la acción propuesta es "revisar
  ubicación", no un mín/máx (REQ-059).
- Sin capacidad registrada, la propuesta se emite marcada capacidad_no_verificada; no se
  omite.
- Consumo cero en el periodo → acción "retirar", no un mínimo de 0.
- Menos de 5 movimientos → estado sin_datos y NO se propone.
- Cada propuesta lleva CMD, F, CV, k, el valor antes de restricciones y cuál fue la
  restricción activa.
- Reproducibilidad exacta: misma entrada y mismos parámetros dan la misma salida
  (NFR-004, test obligatorio).
- Cobertura de test del núcleo >= 80%.

Antes de dar la fase por terminada, genera propuestas sobre un dispensador real y
revísalas conmigo: C_minimo, C_objetivo y k son parámetros de proceso, no
decisiones del usuario, y esta fase es donde se corrigen.

Formato: markdown con el plan, y espera mi OK.
```

## Fase 7 · Libro Excel de salida

```text
Tarea: el generador del libro de propuestas, que es la única interfaz que verá el
farmacéutico responsable.

Un libro por unidad, llamado propuestas_<unidad>_<AAAA-MM-DD>.xlsx, que no sobrescribe
uno existente. Pestañas:
- CRUDO_<origen>: una por fichero de entrada, con los datos tal como se leyeron.
- ACCIONES: una fila por (dispensador, medicamento) con mín/máx vigente, propuesto,
  acción, justificación numérica y las marcas limitado_por_capacidad,
  capacidad_no_verificada, sin_datos, retirar.
- ESTADISTICA: nº de líneas, cuántas cambian y en qué sentido, importe inmovilizado
  vigente y propuesto con su porcentaje de cobertura, comprimidos y cápsulas sin
  clasificar, líneas limitadas por capacidad.
- VALIDACION: la clave de ACCIONES más decision, valor_modificado y comentario, vacías.

Cierra REQ-060 a REQ-064, NFR-010, NFR-011.

Restricciones:
- La columna decision lleva validación de Excel limitada a acepto | modifico | rechazo.
- El color NUNCA es el único portador de información: cada marca tiene su columna de
  texto. Va a acabar impreso en blanco y negro.
- SIN FATIGA DE ALERTAS (NFR-011), y es requisito expreso del usuario. Con el mínimo
  cubriendo 3-4 días es previsible que MUCHÍSIMAS líneas salgan recortadas por capacidad.
  Si una marca afecta a más del 25% de las líneas de un dispensador, NO se pinta fila a
  fila: se resume en ESTADISTICA como un problema del armario, con su recuento y con la
  lista corta de líneas donde el ajuste individual sí cambia algo. Marcar 180 de 220 filas
  en rojo entrena a quien lo lee para ignorar el rojo.
- Ninguna fila de ACCIONES puede tener la justificación vacía, ni siquiera las sin_datos.
- Si la unidad no tiene propuestas, el libro se genera igual con una nota que explique por
  qué. Un libro sin pestañas parece un fallo.
- Un crudo de más de 200.000 filas se trunca con aviso visible y el completo queda en
  datos/intermedio/.

Formato: markdown con el plan, y espera mi OK.
```

## Fase 8 · Ingesta de la validación

```text
Tarea: leer el libro devuelto por el farmacéutico y registrar las decisiones.

Comando: python -m src.cli validar --fichero <ruta>

Cierra REQ-070, REQ-071, REQ-072.

Restricciones:
- Si la clave de una línea devuelta no casa con ninguna propuesta generada, se rechaza el
  FICHERO ENTERO nombrando la primera línea discordante. Nunca se registran decisiones a
  medias.
- Las líneas sin decisión se quedan en estado generada y el resumen dice cuántas faltan.
- Transiciones legales, y ninguna más:
    generada → aceptada → aplicada → verificada
    generada → modificada → aplicada → verificada
    generada → rechazada (terminal)
    generada → sin_datos (terminal)
  rechazada → aceptada está PROHIBIDA: se genera una propuesta nueva.

Formato: markdown con el plan, y espera mi OK.
```

## Fase 9 · Medición de resultados

```text
Tarea: el informe que dice si el piloto funcionó.

Comando: python -m src.cli medir --unidad <u> --desde <d> --hasta <h>

1. Cuenta roturas por dispensador y medicamento en cada periodo (cantidad_hallada = 0).
2. Compara antes y después, en absoluto y en porcentaje.
3. Informe con las cuatro métricas de PRD §2: M1 aplicabilidad física (objetivo 100%),
   M2 reposición sin ajuste manual (>=90%), M3 reducción de roturas (>=80%), M4
   referencias por visita (8-10).
   El objetivo ÚLTIMO de M3 es CERO: no llegar nunca a cajetín 0. El -80% es el umbral con
   el que se declara éxito, no la meta. Da SIEMPRE el número absoluto junto al porcentaje
   y LISTA las roturas que quedan, con artículo y fecha: mientras haya una, hay algo que
   mirar. Si no queda ninguna, dilo explícitamente.
4. Comparación del consumo del dispensador con el consumo global del artículo.

Cierra REQ-081, REQ-082, REQ-090, REQ-091.

Restricciones:
- Solo cuentan las propuestas en estado verificada. Una propuesta que nunca se aplicó no
  puede figurar como éxito ni como fracaso.
- REQ-090 tiene un PENDIENTE sin resolver en el PRD: de dónde sale el consumo global del
  artículo. NO lo resuelvas por tu cuenta. Pregunta antes de implementarlo.
- El importe inmovilizado se informa siempre junto al porcentaje de líneas con precio
  conocido. Una cifra sin su cobertura engaña.
- Si no hay ninguna propuesta aplicada, dilo con una frase clara en vez de devolver ceros.

Formato: markdown con el plan, y espera mi OK.
```
