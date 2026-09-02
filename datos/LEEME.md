# Carpeta de datos · contrato

Aquí van los ficheros. **Ninguno de estos ficheros se versiona en git** (ver
`.gitignore`): salen de aplicativos del hospital y no tienen por qué acabar en un
historial. Lo único que git guarda de aquí es este documento y los ejemplos
anonimizados.

## Dónde va cada cosa

```
datos/
├── entrada/          # crudo, tal como sale del aplicativo. INMUTABLE
│   ├── inventario/   # ubicación, mín/máx, consumo medio, existencias
│   ├── movimientos/  # histórico. De aquí salen el consumo diario y las roturas
│   ├── ofertas/      # código BDM, CN, laboratorio, presentación, precio
│   └── compras/      # el informe de compras: dice qué oferta está vigente
├── catalogos/        # los que mantenemos nosotros, no salen de ningún aplicativo
│                     #   → reenvasados.xlsx (se reenvasa / se etiqueta / ya en unidosis)
├── intermedio/       # normalizado por el código. GENERADO: se puede borrar entero
├── salida/           # propuestas por dispensador y fecha. GENERADO
└── ejemplos/         # muestras pequeñas y anonimizadas, para tests. SÍ se versiona
```

## Las tres reglas de la carpeta

1. **`entrada/` no se toca a mano.** El fichero se deja tal cual lo descargas, con su
   nombre original. Si trae una columna mal, una fila de más o una cabecera en la línea
   4, eso lo arregla el código. Un arreglo manual no se puede repetir el mes que viene y
   nadie recuerda haberlo hecho.
2. **`intermedio/` y `salida/` son desechables.** Todo lo que hay ahí se regenera
   ejecutando de nuevo. Si algo solo existe ahí, está mal puesto.
3. **Nada con datos de paciente.** El código comprueba cada carga y aborta si detecta
   columnas identificadoras, pero la comprobación es la segunda barrera, no la primera.
   Al pedir una extracción, se pide sin esas columnas.

## Cómo dejar un fichero nuevo

Déjalo en la subcarpeta que le toque con su nombre original y dime qué es. No hace falta
que lo renombres ni lo limpies: describir de dónde sale y qué representa cada columna es
más útil que dejarlo bonito.
