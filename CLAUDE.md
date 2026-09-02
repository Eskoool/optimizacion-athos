# optimizacion-athos

Contrato invariante del proyecto. Se lee al inicio de cada sesión.
Si algo de aquí ha dejado de ser cierto, no lo edites en caliente: pasa por `/cambio`.

## Qué es esto

Piloto metodológico que propone, con reglas deterministas y auditables, los mín/máx de
cada dispensador ATHOS-Dosys a partir de extracciones de inventario, para que el
farmacéutico responsable valide o descarte cada propuesta.

**La v1 no es la herramienta: es la prueba de que el método funciona.** El producto para
farmacéuticos no técnicos es la v2, está decidido y vive en el roadmap (PRD §12), no en
la cabeza de nadie.

Documento de referencia: `PRD.md`. Estado actual: `memoria.md`. Historial: `log.md`.

## Stack fijado

Python 3 + pandas + openpyxl · entrada y salida en Excel · sin base de datos ni web en la
v1 (piloto local, ejecutado con Claude Code).

Motor de build: claude-code

## Reglas que no se negocian

1. **Sin datos de paciente. Nunca.** Ni reales, ni pseudonimizados, ni "solo para probar".
   Y no basta con no pedirlos: al leer cualquier fichero, el producto **detecta**
   columnas identificadoras, **alerta** y **no persiste nada** de esa carga. Es un
   requisito verificable, no una advertencia.
2. **El producto propone, no decide.** Ninguna salida es vinculante. Cada propuesta pasa
   por validación humana explícita antes de existir en el mundo real.
3. **Vive aparte del software de APD Solutions.** No lo sustituye y no escribe en él.
   El resultado del piloto es un fichero que una persona lee, no una integración.
4. **Núcleo puro, separado de la entrada/salida.** Las reglas de optimización son
   funciones que reciben tablas y devuelven propuestas, sin leer ni escribir ficheros.
   Esta regla existe por un motivo concreto: la v2 pone una interfaz encima del mismo
   núcleo en vez de reescribirlo. Meter `pd.read_excel` dentro de una regla rompe eso.
5. **Toda propuesta se explica número a número.** Si un farmacéutico pregunta "¿por qué
   18 y no 12?", la respuesta está en la salida, no en el código. Sin esto el piloto no
   convence a nadie y el método no se generaliza.
6. **`datos/entrada/` es inmutable.** Los ficheros llegan tal cual salen del aplicativo
   y no se editan a mano jamás. Toda corrección es código, para que sea repetible.
7. **Ningún dato real entra en el repositorio.** `datos/` está ignorado salvo su
   documentación y los ejemplos anonimizados.

## Cláusula de parada

Si una petición contradice el PRD, o pide algo que el PRD no contempla:

**para, dilo y pregunta antes de escribir código.**

No amplíes el alcance por iniciativa propia, aunque la ampliación parezca obvia y
pequeña. Las funcionalidades que nadie decidió meter son las que nadie sabe mantener. La
vía correcta es `/cambio`.

## Comandos del repo

```
python -m venv .venv && .venv\Scripts\activate
pip install -r requirements.txt
python -m src.cli --help
pytest
```

Comandos de sesión: `/checkpoint` vuelca el estado a `memoria.md` y `log.md`, `/cambio`
tramita una desviación del PRD, `/auditar` pasa la rúbrica al documento y al código,
`/prompt` genera la instrucción pegable de una fase.

## Al cerrar sesión

Actualiza `memoria.md` (estado presente) y añade una entrada a `log.md` (qué se hizo, qué
se desvió). O invoca `/checkpoint` y hazlo de una vez.
