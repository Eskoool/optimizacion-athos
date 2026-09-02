# Arquitectura · borrador

> **Borrador.** Se consolida en §5 del PRD cuando la entrevista termine. Se escribe ahora
> porque el usuario necesita saber dónde dejar los ficheros y cómo se va a organizar el
> código antes de aportarlos.

## La decisión que manda sobre todas las demás

La v1 es un piloto en Python que ejecuta una persona. La v2 es un producto que usan
farmacéuticos no técnicos. **Si el piloto se escribe como un script, la v2 es una
reescritura.** Para que no lo sea, el cálculo no puede saber de dónde vienen los datos.

```
   Excel de entrada ──▶ lectores ──▶ tablas normalizadas
                                            │
                                            ▼
                                     ┌──────────────┐
                                     │    NÚCLEO    │   funciones puras
                                     │  reglas de   │   sin E/S, sin rutas,
                                     │ optimización │   sin pandas.read_*
                                     └──────────────┘
                                            │
                                     propuestas + porqué
                                            │
                                            ▼
                              informes ──▶ Excel de salida
```

Los lectores y los informes son intercambiables. El núcleo no. En la v2 se sustituye la
columna izquierda por una base de datos y la derecha por una pantalla, y **el núcleo se
queda igual**. Esa es toda la razón de ser de esta separación.

## Estructura de código prevista

No se crea hasta que el PRD esté aprobado — regla de `productos/CLAUDE.md`: un producto no
empieza con código.

```
src/
├── guardas/     # detección de datos de paciente. Se ejecuta ANTES que nada
├── lectores/    # Excel crudo → tablas normalizadas. Aquí vive toda la suciedad
│                #   de los ficheros reales: cabeceras desplazadas, tipos, duplicados
├── nucleo/      # reglas de optimización. Funciones puras. Sin E/S. Sin excepciones
├── informes/    # tablas → Excel de salida, con la justificación de cada propuesta
└── cli.py       # el único sitio que orquesta y conoce rutas
tests/
├── nucleo/      # el grueso: cada regla con sus casos límite
└── lectores/    # contra los ficheros de datos/ejemplos/
```

**El orden importa:** `guardas` corre antes que `lectores`. Si detecta una columna
identificadora, el proceso aborta y no se escribe nada — ni intermedio, ni salida, ni log
con el contenido. Detectar después de haber cargado ya sería detectar tarde.

## Entidades previstas

Provisional hasta B5 completo y hasta ver el primer fichero real.

| Entidad | De dónde sale | Papel |
|---|---|---|
| Dispensador | inventario | La unidad de trabajo. El piloto opera sobre unos pocos, elegidos |
| **Artículo** | inventario | Clave natural: **código interno del SCS, proyecto BDM**. El prefijo fija el tipo: `V` medicamento · `Y` fórmula magistral interna · `DM` material y otros · `T` medicamento extranjero |
| Ubicación | inventario | Dónde vive el artículo dentro del armario. Base de la base de huecos |
| Pacto mín/máx vigente | inventario | El punto de partida contra el que se compara la propuesta |
| Consumo | inventario | El insumo del cálculo. Sin esto no hay optimización posible |
| Movimiento | histórico de movimientos | Permite ver el consumo **diario**: de aquí salen `CMD`, la variabilidad `CV` y las roturas |
| **Oferta** | catálogo de ofertas | `(codigo_bdm, cn)`. **Aquí y solo aquí vive el código nacional**, con laboratorio y precio. Un artículo puede tener varias |
| **Compra** | informe de compras | Determina **qué oferta está vigente**. Sin esto hay precios, pero no se sabe cuál se aplica |
| Reenvasado | catálogo mantenido a mano | Se reenvasa / se etiqueta / ya viene en unidosis. Solo comprimidos y cápsulas, y solo de tipo medicamento |
| Propuesta | generada | Mín/máx propuesto + justificación numérica + estado de validación |

> **El armario lleva medicamentos, y solo medicamentos.** Entran al piloto `V`
> (medicamentos), `Y` (fórmulas magistrales internas) y `T` (extranjeros). Quedan fuera:
> - **`DM`, material y otros** — no se pone en el dispensador.
> - **Sueros y grandes volúmenes** (p. ej. paracetamol intravenoso).
> - **Medicamentos de UFA** (Unidad de Farmacia Ambulatoria).
>
> Las fórmulas magistrales (`Y`) no tienen CN, luego tampoco precio por la vía de las
> ofertas: cualquier cálculo económico tiene que decir a cuántas líneas no llegó.

## Lo que se optimiza, en una frase

**No el stock: el trabajo del operario.** El dispensador se repone cada 24 h pase lo que
pase; lo que se decide es cuántas referencias hay que tocar en cada visita. El mínimo cubre
3-4 días —colchón contra subidas de consumo, no contra el retraso del operario— y el máximo
añade 7 días, para que cada referencia aguante la semana sin tocarla. Objetivo: 8-10
referencias por dispensador y visita.

Consecuencia directa: para aguantar la semana, una referencia necesita que en su hueco
quepan unos 10-11 días de consumo. **La base de huecos deja de ser un accesorio y pasa a
ser el dato que decide si el piloto es interpretable.**

## Lo que esta arquitectura deja fuera a propósito

- **No hay base de datos en la v1.** Los ficheros son la fuente. Meter SQLite "por si
  acaso" añade un esquema que mantener sin resolver ningún problema del piloto.
- **No hay interfaz.** La salida es un Excel que una persona abre. La interfaz es v2.
- **No hay integración con APD Solutions**, ni de lectura ni de escritura. El piloto no
  toca el sistema del que salen los datos.
