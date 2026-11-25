# Proyecto PDI: cierre de cobros de hoteles

## 1. Datos generales

- **Autor:** Iván Burgos Maccolm
- **Código personal asignado:** `A001`.
- **Conjunto de referencia:** `cierre_aula_01`.
- **Versión de asignación:** `1`.
- **Fecha y hora de generación de los resultados:** 01:37:38 | 25/11/2026
- **Versión de Pentaho Data Integration (PDI):** 9.4.0.0-343.
- **Versión de Java utilizada:** OpenJDK version 11.0.33
- **Sistema operativo:** Ubuntu 22.04 y Windows 10. Gran parte del proyecto se realizó en clase con Ubuntu, pero se finalizó con Windows 10 en casa.
- **Repositorio:** https://github.com/IvanDev33/bda-ut1-pentaho-hotel
- **Commit de entrega:** 80f3fa4. (Penúltimo, después de este hay uno más en el que se actualiza el README)

Este proyecto prepara un cierre de cobros a partir de tres ficheros CSV. Las transformaciones generan resultados provisionales en `temporal/`; el job los valida y solo entonces los copia a `salida/`.

## 2. Estructura del repositorio

```text
bda-ut1-pentaho-hotel/
├── asignacion.json
├── entrada/
│   ├── reservas_pdi.csv
│   ├── cobros_pdi.csv
│   └── hoteles_pdi.csv
├── procesos/
│   ├── informe_cobros.ktr
│   ├── reservas_sin_cobro.ktr
│   ├── validar_informe.ktr
│   ├── validar_sin_cobro.ktr
│   └── cierre_hotel.kjb
├── temporal/                 # resultados provisionales; no se versiona
├── salida/                   # resultados publicados
├── evidencias/
└── README.md
```

## 3. Entradas y procedencia

Los tres CSV son los ficheros originales incluidos en el paquete personal asignado. Se consiguieron a partir del Moodle en la tarea de Pentaho correspondiente.

| Archivo | Separador | Cabecera y tipos principales |
|---|---:|---|
| `reservas_pdi.csv` | Coma `,` | `id_reserva` entero; `id_hotel` entero; `canal` texto; `noches` entero; `importe` decimal con punto |
| `cobros_pdi.csv` | Punto y coma `;` | `id_reserva` entero; `medio` texto; `cobrado` decimal con punto |
| `hoteles_pdi.csv` | Punto y coma `;` | `id_hotel` entero; `hotel`, `comarca` texto; `id_cadena` entero |

`asignacion.json` identifica el paquete como `A001` / `cierre_aula_01`, versión 1, e incluye las huellas SHA-256 de los tres CSV. Para comprobar que no se han alterado, se pueden calcular sus hashes y compararlos con ese archivo.

## 4. Recorrido de los datos y decisiones de diseño

### Informe de cobros

`informe_cobros.ktr` lee reservas y cobros, ordena ambas ramas por `id_reserva` y las une mediante `INNER JOIN`. El conjunto de referencia tiene 20 reservas y 15 cobros, por lo que la unión debe producir 15 filas. Después incorpora el nombre del hotel mediante otro `INNER JOIN` por `id_hotel`; las dos entradas se ordenan previamente y deben seguir existiendo 15 filas.

El resultado se ordena por `hotel` y `canal` y se agrupa exclusivamente por esos dos campos. `cobrado_total` es la suma de `cobrado` y `n_reservas` cuenta las reservas del grupo. Los importes monetarios se expresan en **euros**, con punto decimal y dos decimales. El CSV utiliza punto y coma, UTF-8 y cabecera, con las columnas en este orden: `hotel;canal;n_reservas;cobrado_total`.

En estos datos hay como máximo un cobro por reserva, por lo que cada reserva aparece una sola vez tras la unión. Si existieran varios pagos por reserva, primero habría que agrupar los cobros por `id_reserva` y sumar sus importes; después se uniría ese resumen con reservas. Así no se multiplicaría el número de reservas al contar. Si se conservaran las transacciones individuales, el recuento tendría que ser de identificadores de reserva distintos, no un recuento simple de filas.

### Reservas sin cobro asociado

`reservas_sin_cobro.ktr` conserva las 20 reservas mediante un `LEFT OUTER JOIN`, con reservas a la izquierda y cobros a la derecha. La rama de cobros renombra la clave como `id_reserva_cobro`. El filtro selecciona `id_reserva_cobro IS NULL`: no se usa `cobrado = 0`, porque un cobro de importe cero no es lo mismo que la ausencia de una fila de cobro.

Después se incorpora el nombre del hotel, se conservan únicamente `id_reserva`, `hotel`, `canal` e `importe`, y se añade el motivo indicado en el enunciado. En este fichero, `importe` es el **precio de la reserva en euros**, no una cantidad cobrada. La ausencia de cobro asociado en el archivo recibido no demuestra por sí sola un impago. El CSV resultante contiene `id_reserva;hotel;canal;importe;motivo`.

### Por qué se entrega CSV

CSV es un formato sencillo que permite revisar los resultados con Excel, editor de texto, etc... Se especifican separador, codificación y formato decimal para evitar errores al momento de lectura. También se tendrá que tener en cuenta si se abre con un Excel, cuál es el separador que utilizará, porque podría separar los decimales de un número como una columna aparte.

## 5. Resumen para gerencia

El cierre `cierre_aula_01` es la revisión de cobros de los cinco hoteles a partir de los ficheros recibidos: contrasta cada reserva con los pagos registrados y resume lo cobrado por hotel y canal de venta. Esta versión del informe no analiza la ocupación, ni la evolución en el tiempo, ni los pagos por reserva en detalle.

De las 20 reservas registradas, 15 tienen algún cobro y suman **3.965,50 €**. Las 5 restantes (4, 9, 11, 15 y 18) no aparecen en el fichero de cobros. El precio conjunto de esas cinco reservas es de **639,90 €**. Es dinero pendiente de revisar, pero no un impago confirmado: el cobro podría haberse registrado en otro sistema o no haberse enviado en este fichero. Conviene confirmarlo con cada hotel antes de reclamar nada.

Con los CSV que tenemos no nos sirve para calcular la ocupación de los hoteles, deberíamos de tener por ejemplo, un nuevo dato de hoteles que nos diga cuántas habitaciones tienen disponibles, y en las reservas, especificar cuándo comienza y cuando termina. Podríamos realizar un cálculo con una fecha sobre un hotel, contando cuántas reservas se encuentran dentro de esta fecha de inicio y final para comprobar la ocupación del hotel en ese periodo.

## 6. Validaciones y control de errores

- `validar_informe.ktr`: exige 13 grupos, 15 reservas y 3.965,50 €. Un CSV vacío falla.
- `validar_sin_cobro.ktr`: exige exactamente los identificadores 4, 9, 11, 15 y 18, sin duplicados. Un CSV vacío falla.
- `cierre_hotel.kjb`: comprueba los tres CSV de entrada, genera ambos informes, comprueba los provisionales, valida y publica en `salida/`. Si algo falla, aborta antes de publicar y conserva los ficheros ya publicados.
- La copia de los dos ficheros no es atómica: si falla la segunda, el job avisa de una posible publicación parcial. Repetir el job con las mismas entradas da los mismos resultados, sin acumular filas.

## 7. Cómo ejecutarlo en tu PC

### Requisitos

- **Java 11** (`java -version`). Si Kitchen no lo encuentra, definir `JAVA_HOME`.
- **Pentaho Data Integration 9.4** (Community), descomprimido en una carpeta, por ejemplo `C:\pdi\data-integration`.
- **Git**, o descargar el repositorio como ZIP.

### Pasos

1. Descargar el repositorio sin cambiar su estructura de carpetas:

   ```bash
   git clone https://github.com/IvanDev33/bda-ut1-pentaho-hotel.git
   ```

2. Comprobar que `entrada/` contiene los tres CSV y crear `temporal/` y `salida/` si no existen.
3. Ejecutar el job con Kitchen, ajustando las rutas a cada equipo.

   **Windows:**

   ```bat
   cd /d C:\pdi\data-integration
   Kitchen.bat /file:"C:\ruta\bda-ut1-pentaho-hotel\procesos\cierre_hotel.kjb" /level:Basic
   ```

   **Linux:**

   ```bash
   /ruta/data-integration/kitchen.sh /file="/ruta/bda-ut1-pentaho-hotel/procesos/cierre_hotel.kjb" /level:Basic
   ```

   También puede abrirse `cierre_hotel.kjb` en Spoon y pulsarse **Run**.


## 8. Fuentes y ayudas

- Enunciado de la práctica integradora de Pentaho.
- Asistencia de ChatGPT, Claude para revisar errores en las transformaciones y jobs. Guías para realizar ciertas tareas como ejecución de Kitchen en Windows o ayuda en errores con Pentaho. Ayuda en el Markdown de este README para darle formato, y comprobar que no faltan detalles o hay errores.
