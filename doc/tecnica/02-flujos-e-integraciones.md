# Flujos e integraciones

## Consulta por CSV

Entrada principal: `HashService.checkHash(hash)`.

1. Si la caché está activa, busca `INFO/CSV_<id-saneado>`.
2. Si está habilitado el Archivo Digital, lo consulta primero.
3. Sólo si no encuentra resultado consulta Custodia Antigua.
4. Custodia Antigua debe devolver un documento con custodia correcta.
5. Calcula si el formato es imprimible: PDF/JPG directamente u otros formatos
   admitidos por el conversor OpenOffice.
6. Marca exclusión de original y visibilidad del botón original.
7. Si la caché está activa, guarda `DocumentInfo` y devuelve el resultado.

En 2.1.7 el guardado no comprueba si la validación de firma falló. El documento
se devuelve con un firmante `Certificate Error`, pero se cachea como una
consulta correcta. El commit `9026fc1` de `concsv-dev-2.2.2` añade
`isFirmaValida` para no cachear en ese caso; no forma parte de esta versión.

Un “no encontrado” de Custodia Antigua termina convertido en
`GenericServiceException` no notificable, no siempre en `null`. En la API
pública eso puede acabar en 404 si el controlador recibe `null`, o en 500 si el
servicio lanza la excepción comprobada. La API interna suele devolver 204. Esta
diferencia es relevante al modificar el contrato de errores.

## Consulta por UUID

`HashService.checkHashFromUUID(uuid)` sólo usa Archivo Digital. No cae a
Custodia Antigua. Completa la exclusión según el CSV devuelto y usa un espacio
de caché separado, `INFO/UUID_<id-saneado>`.

Los endpoints internos de imprimible siguen `nuevaLocalizacion` mientras el
documento no tenga CSV, consultando sucesivos UUID. Este seguimiento no está
centralizado en `HashService` y no se aplica de forma idéntica en todas las
operaciones.

## Descarga de original

`HashService.getDocument(info, packedFile)`:

- usa la clave `documentInfo.codi` y tipo `ORIGINAL` en caché;
- en Custodia Antigua recupera por CSV y normaliza el nombre a `.pdf`;
- en Archivo Digital recupera por código documental y transmite `packedFile`;
- `packedFile=true` permite que firmas detached se entreguen empaquetadas;
- registra el subsistema `ORI` y encapsula fallos en
  `GenericServiceException`.

La clave de caché usa `DocumentInfo.getCodi()` (hash o UUID) y no incorpora
`packedFile`. Una descarga con `packedFile=false` puede poblar la misma entrada
que otra con `true`, aunque el contenido esperado sea distinto. Debe corregirse
antes de confiar en esa caché para documentos con firma detached.

## Generación de imprimible

`HashService.getPrintableDocument(info, lang)`:

1. Busca `IMPRIMIBLE/<codi>_<lang>` en caché.
2. Obtiene el original sin empaquetar.
3. Para PDF, rechaza documentos marcados como mal formados y estampa el PDF.
4. Para JPG, genera un PDF y lo estampa.
5. Para otros formatos admitidos, convierte con OpenOffice y estampa el PDF.
6. Añade página/bloques de metadatos, CSV, URL/QR y firmantes.
7. Devuelve `application/pdf` con sufijo `_imprimible.pdf`.

Los PDF protegidos con contraseña generan una excepción funcional no
notificable y marcan `DocumentInfo.hasPassword`. Los PDF Portfolio no reciben
tratamiento especial en esta versión; es el objeto del issue #18.

## Documento ENI

`getEniDocument` sólo admite documentos del Archivo Digital. Custodia Antigua
produce “No es un document ENI”. La API interna expone variantes por CSV y UUID.
La aplicación pública no ofrece esta exportación en su interfaz; el issue #1
solicita esa acción.

## Metadatos opcionales

`OptionalMetadataBlock` carga etiquetas por idioma desde la ruta configurada y
selecciona valores presentes en el mapa de metadatos. Los recursos Docker
incluyen `optionalMetadata_ca.properties` y `optionalMetadata_es.properties`.
Un fallo devuelve una lista vacía y registra el subsistema `MET`.

## Caché filesystem

Raíz: `${es.caib.concsv.fitxers}/cache`.

```text
cache/
  INFO/CSV_<id>
  INFO/UUID_<id>
  ORIGINAL/<codi>
  IMPRIMIBLE/<codi>_<lang>
```

Los valores son serialización Java de `DocumentInfo` o `DocumentContent`. El
TTL se expresa en minutos y usa expiración deslizante: cada lectura válida
actualiza `lastModifiedTime`. `0` significa sin expiración. Una entrada
caducada, corrupta o incompatible se elimina y se trata como miss.

El identificador se sanea, pero `lang` se concatena sin sanear. No debe pasarse
entrada no confiable sin validación porque puede alterar la ruta. No hay
bloqueos ni escritura atómica; accesos simultáneos pueden observar archivos
parciales, que después se eliminan como corruptos.

## Integraciones externas

| Integración | Uso | Configuración principal |
| --- | --- | --- |
| Archivo Digital | Consulta CSV/UUID, original y ENI | endpoint, organización, cliente, usuario, contraseña, versión |
| Custodia Antigua | Consulta CSV y original | endpoint y timeout |
| @firma | Validación de firmas y datos de firmantes | endpoint, application ID, autenticación y plantillas |
| OpenOffice | Conversión de formatos no PDF/JPG | host y puerto |
| Comanda | Contrato de salud, estadísticas y logs | endpoints JAX-RS y librería cliente |
| Oracle | Estadísticas persistidas | datasource JNDI `concsvDS` |

La salud de Comanda informa las integraciones `ARX`, `SIG` y `VFI`; no existe
una integración separada para Custodia Antigua. Es el objeto del issue #15.

## Estadísticas y salud

Cada endpoint registra una dimensión `TIPO:ORIGEN`, resultado
`OK/INVALID/ERROR` y duración. El acumulador en memoria se vuelca a las tablas
de explotación. Los indicadores publicados son correctos, código inválido,
errores y tiempo medio correcto.

La salud global combina una llamada HTTP al endpoint de performance, estado de
subsistemas e integraciones. `EstadisticaHelper.checkDatabaseLatency()` sí
ejecuta una consulta real, pero `ComandaService.checkDatabase()` no la usa:
devuelve una latencia fija de 1 ms y siempre `UP`. El cliente HTTP creado para
la comprobación de performance tampoco se cierra explícitamente.

La caché también escribe entradas nulas cuando no encuentra documento, pero
`CacheHelper.getInfo` las convierte en `Optional.empty`; no se reaprovechan como
caché negativa. Sólo ocupan espacio hasta que las elimina el TTL o la limpieza.
