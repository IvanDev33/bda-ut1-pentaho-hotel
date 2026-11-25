# Proyecto PDI: cierre de cobros de hoteles

## 1. Datos generales

- **Autor:** POR RELLENAR: nombre y apellidos del alumno.
- **Código personal asignado:** `A001`.
- **Conjunto de referencia:** `cierre_aula_01`.
- **Versión de asignación:** `1`.
- **Fecha y hora de generación de los resultados:** POR RELLENAR: anotar la del log de la ejecución final correcta de `cierre_hotel.kjb` (la del 25/11/2026 00:15 corresponde a una ejecución fallida de `validar_sin_cobro`).
- **Versión de Pentaho Data Integration (PDI):** 9.4.0.0-343 (según el log de Spoon).
- **Versión de Java utilizada:** POR RELLENAR: versión exacta; se obtiene con `java -version` o en *Ayuda > Acerca de* de Spoon.
- **Sistema operativo:** POR RELLENAR: nombre y versión de Windows (las rutas del log indican Windows).
- **Repositorio:** https://github.com/IvanDev33/bda-ut1-pentaho-hotel (POR RELLENAR: confirmar que es la URL que se entregará).
- **Commit de entrega:** POR RELLENAR: identificador (hash) del último commit tras el `git push`.

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

Los tres CSV son los ficheros originales incluidos en el paquete personal asignado. No se deben editar para la entrega básica. Procedencia concreta del paquete (profesor/aula y fecha de distribución): **POR RELLENAR**.

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

CSV es un formato sencillo e interoperable que permite revisar los resultados con LibreOffice Calc, Excel, un editor de texto o herramientas de análisis. Se especifican separador, codificación y formato decimal para evitar interpretaciones ambiguas.

## 5. Resumen para gerencia

El cierre `cierre_aula_01` resume la información de las reservas y los cobros presentes en los ficheros recibidos. Según los controles de referencia del encargo, **15 reservas tienen cobro asociado**, que suman **3.965,50 €**, y **5 reservas** (identificadores 4, 9, 11, 15 y 18) no tienen una fila de cobro asociada en el archivo recibido. Estas cinco reservas requieren revisión; su ausencia en el fichero de cobros no demuestra por sí sola que exista un impago. El informe no permite calcular ocupación: faltan datos de habitaciones disponibles y habitaciones ocupadas por periodo. Para responder a esa pregunta habría que solicitar, para cada hotel y periodo, el inventario de habitaciones disponibles y el número de habitaciones ocupadas (con fechas de inicio y fin y reglas para habitaciones fuera de servicio).

Los importes y recuentos anteriores son **valores esperados de referencia del enunciado**. El resultado obtenido y la fecha/hora de la ejecución real deben anotarse aquí después de ejecutar y revisar los CSV: **POR RELLENAR** (reservas con cobro, total en € y reservas sin cobro obtenidos, y si coinciden con lo esperado).

## 6. Validaciones automáticas

- `validar_informe.ktr` lee el CSV provisional, resume el número de grupos, la suma de `n_reservas` y la suma de `cobrado_total`, y exige 13 grupos, 15 reservas y 3.965,50 €. Su `Group by` sin claves tiene activada la devolución de una fila resumen incluso si el CSV solo contiene cabecera; así el caso vacío produce recuentos/sumas que no superan el filtro y llega a `Abort`.
- `validar_sin_cobro.ktr` exige que los identificadores sean exactamente los esperados (4, 9, 11, 15 y 18), ordena por `id_reserva`, agrupa por identificador y comprueba que cada grupo tenga una sola fila. Finalmente cuenta los identificadores únicos y exige cinco. El resumen final también devuelve una fila cuando la entrada está vacía, por lo que un CSV vacío falla.

Los controles fijos son específicos del conjunto `cierre_aula_01`; no deben trasladarse sin cambios a cierres reales con otros datos.

## 7. Ejecución desde una copia recién descargada

1. Instalar PDI y Java compatibles. Versiones utilizadas en este trabajo: las indicadas en el apartado 1.
2. Descargar o clonar el repositorio y mantener su estructura de carpetas.
3. Confirmar que existen los tres CSV en `entrada/` y que `asignacion.json` corresponde al paquete recibido.
4. Crear las carpetas `temporal/` y `salida/` si no existen. El job no elimina los informes publicados al comenzar.
5. Desde Linux, ejecutar Kitchen (ajustando la ruta a la instalación de PDI):

   ```bash
   /ruta/a/data-integration/kitchen.sh \
     /file="/ruta/al/repositorio/procesos/cierre_hotel.kjb" \
     /level:Basic
   ```

   En Windows se puede utilizar `kitchen.bat` con la ruta equivalente. El comando ejecuta el job una vez; **no lo programa** para una hora determinada. La ejecución nocturna requeriría configurar el programador de tareas del sistema (por ejemplo, cron en Linux o el Programador de tareas de Windows).

6. Los resultados provisionales se escriben en `temporal/`. Solo después de superar las dos validaciones, el job copia `informe_hotel_canal.csv` y `reservas_sin_cobro.csv` a `salida/`, sustituyendo los anteriores.

Para probar una transformación individual se puede utilizar Pan:

```bash
/ruta/a/data-integration/pan.sh \
  /file="/ruta/al/repositorio/procesos/informe_cobros.ktr" \
  /level:Basic
```

Se puede repetir con `reservas_sin_cobro.ktr`, `validar_informe.ktr` y `validar_sin_cobro.ktr`. Ajustar las rutas a cada instalación.

## 8. Publicación y comportamiento ante errores

El job sigue este orden: Start; comprobación de los tres CSV de entrada; generación del informe; generación de reservas sin cobro; comprobación de existencia de ambos provisionales; validación de ambos; copia de los dos ficheros a `salida/`; mensaje final de éxito.

Si falta una entrada, falla una transformación, falta una salida provisional o falla una validación, el job termina con error antes de comenzar la publicación y conserva las versiones ya publicadas. Las copias sustituyen los archivos de `salida/` únicamente después de validar ambos CSV.

La sustitución de dos archivos no es atómica. Si se copia correctamente el primer CSV y falla la segunda copia, `salida/` puede quedar con un archivo nuevo y otro antiguo o ausente. En ese caso el job termina con error, registra el aviso de posible publicación parcial y no registra éxito. No se intenta deshacer automáticamente la primera copia.

Al repetir el job con las mismas entradas, los resultados deben ser iguales: las salidas provisionales se sobrescriben y las copias a `salida/` también sobrescriben los archivos anteriores, sin acumular filas.

## 9. Comprobaciones manuales y pruebas

Registrar en `evidencias/` una tabla con **esperado, obtenido y conclusión**, y contrastar cada grupo del informe con la tabla de referencia del taller 2. La comprobación final se hace sobre los archivos completos escritos, no solo sobre vistas previas de Spoon.

Valores de referencia:

| Comprobación | Esperado |
|---|---:|
| Reservas / cobros / hoteles leídos | 20 / 15 / 5 |
| Claves duplicadas en cada origen | 0 |
| Unión reservas-cobros | 15 filas |
| Unión con hoteles | 15 filas |
| Grupos del informe | 13 |
| Suma de `n_reservas` | 15 |
| Suma de `cobrado_total` | 3.965,50 € |
| Laredo, canal web | 2 reservas; 496,50 € |
| Reservas sin cobro | 5: 4, 9, 11, 15 y 18 |
| Reconciliación | 15 + 5 = 20 |

Documentar estas seis pruebas con capturas o logs comentados:

1. **Primera ejecución:** ejecutar Kitchen sin informes anteriores; deben publicarse los dos CSV y finalizar correctamente.
2. **Repetición:** ejecutar de nuevo con las mismas entradas; deben mantenerse grupos, filas y totales sin duplicados.
3. **Falta una entrada:** cambiar temporalmente la ruta de un CSV en una copia de trabajo; el job debe fallar antes de publicar.
4. **Salida incompleta:** dejar solo la cabecera en una copia del CSV provisional y ejecutar directamente su validador; debe fallar.
5. **Total incorrecto:** alterar un importe en una copia del informe provisional y ejecutar directamente `validar_informe.ktr`; debe fallar.
6. **Recuperación:** restaurar los ficheros originales y ejecutar el job completo; debe generar, validar y publicar de nuevo los resultados correctos.

Para las pruebas 4 y 5 se ejecuta directamente el validador sobre el fichero alterado: si se ejecuta antes el job completo, este regeneraría el provisional y eliminaría la alteración. Registrar también el log de los fallos de preparación/validación y, si se prueba, de fallo durante la segunda copia.

**Estado de las evidencias:** POR RELLENAR; la carpeta debe contener los registros reales de ejecución y las seis pruebas. No se deben presentar resultados esperados como si fueran resultados ya ejecutados.

## 10. Fuentes y adaptación

- Enunciado de la práctica de cierre hotelero, apartados 4 a 9.
- Taller 2 de Pentaho Data Integration: **POR RELLENAR**: título o referencia exacta del material.
- Documentación de Pentaho Data Integration: [referencia de entradas de job](https://docs.pentaho.com/pdia-data-integration/pdi-job-entries-reference-overview) y [File Exists (Job Entry)](https://docs.pentaho.com/pdia-data-integration/pdi-job-entries-reference-overview/file-exists-job-entry).
- Asistencia de ChatGPT para revisar XML de transformaciones, validaciones, job y documentación, y de Claude (Anthropic) para corregir un hop desactivado en `validar_sin_cobro.ktr`, generar `cierre_hotel.kjb` y revisar este README; el alumno debe revisar, abrir en Spoon, ejecutar y comprender cada paso antes de entregar.
- Adaptaciones realizadas por el alumno: **POR RELLENAR**: qué pasos se reutilizaron del taller 2 y qué configuraciones se modificaron.

## 11. Lista final antes de entregar

- [ ] Sustituir todos los **POR RELLENAR** (autor, versión de Java, sistema operativo, fecha/hora de ejecución, resultado obtenido, procedencia, taller 2, adaptaciones y commit).
- [ ] Abrir en Spoon los cinco procesos (cuatro transformaciones y el job); revisar que no aparezcan pasos desconocidos ni errores de metadatos.
- [ ] Ejecutar las transformaciones con Pan y el job con Kitchen.
- [ ] Ejecutar el job para generar los dos CSV provisionales y publicarlos en `salida/` (ahora mismo no se incluyen resultados de una ejecución PDI comprobada).
- [ ] Revisar los dos CSV publicados fuera de Spoon y contrastar todos los grupos.
- [ ] Completar `evidencias/` con la tabla esperado/obtenido/conclusión y las seis pruebas.
- [ ] Añadir una nota en cada lienzo de transformación con el nombre del alumno y la pregunta que resuelve.
- [ ] Comprobar `git status`, hacer commit, `git push` y anotar el identificador del commit evaluable.
- [ ] Entregar en Moodle la URL del repositorio y el identificador del commit.
