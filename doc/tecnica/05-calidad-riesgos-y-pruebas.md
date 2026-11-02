# Calidad, riesgos y pruebas

## Pruebas existentes

La mayor parte de las pruebas está en `concsv-service/src/test`:

| Prueba | Estado | Área |
| --- | --- | --- |
| `HashServiceTest` | Sin `@Test` activo; el único está comentado y `@BeforeClass` usa rutas `/opt/webapps/...` | Orquestación documental |
| `CacheHelperTest` | 8 pruebas activas | Caché, expiración y corrupción |
| `PrintableUtilsTest` | 1 activa; escribe `printable.pdf` en el directorio de trabajo | Transformación imprimible |
| `PdfUtilsTest` | 1 activa y 1 comentada | Casos PDF y recursos binarios |
| `EstadisticaHelperTest` | 5 activas | Acumulación/persistencia estadística |
| `ComandaServiceTest` | 9 activas | Adaptación de salud y estadísticas |
| `IntegracionsHelperTest` | 3 activas | Salud de integraciones |
| `SubsistemesHelperTest` | 3 activas | Salud de subsistemas |

El frontend Java sólo contiene un esqueleto `Test.java` sin pruebas; no se han
encontrado tests TypeScript ni pruebas de contrato HTTP completas. Tampoco hay
una suite de integración que despliegue el EAR contra JBoss, Oracle y servicios
externos. La clase que concentra la lógica documental, `HashService`, no tiene
pruebas activas.

## Validación recomendada

```bash
mvn clean test
mvn clean install
```

Después del build deben probarse manualmente al menos: CSV existente y ausente,
UUID trasladado, original excluido, PDF normal, PDF con contraseña, PDF
malformado, JPG, formato ofimático, firma detached, caché hit/miss/caducada,
integración caída y consulta de estadísticas.

## Riesgos críticos o altos

| Riesgo | Evidencia | Impacto |
| --- | --- | --- |
| Estado local incompatible con stateless | Caché filesystem, acumuladores, exclusiones y salud en memoria | Respuestas divergentes y datos perdidos al escalar |
| Colisión de caché del original | Clave `getCodi()` no incluye `packedFile`; aplica con caché activa | Puede devolverse ZIP/original equivocado para firmas detached |
| `lang` no saneado en ruta | `CacheHelper.resolveContentPath`; aplica con caché activa | Path traversal, sobrescritura o borrado de ficheros accesibles por JBoss |
| Salud de BD simulada | `ComandaService.checkDatabase()` fija 1 ms; `checkDatabaseLatency()` existe pero no se usa | Falso positivo operativo |
| API documental interna sin autenticación | Restricción `CSV_REST` comentada en `web.xml`; `csv.caib.es` devuelve 404 de Apache el 2026-11-02 | Defensa en profundidad: riesgo si se publica el contexto o se accede desde la red interna |
| Errores de firma cacheables | Issue #16 abierto; el fix `9026fc1` de 2.2.2 evita cachear si hay `Certificate Error` | Persistencia temporal de resultados inválidos hasta expirar la caché |
| Stack traces en errores públicos | `GlobalExceptionHandler` usa `reflectoring.trace=true` y `?trace=` | Divulgación de rutas, clases y detalles internos |

## Riesgos medios

- Escritura de caché no atómica y sin coordinación entre hilos/nodos.
- Serialización Java sensible a cambios de clase; se recupera eliminando la
  entrada, pero provoca misses masivos tras cambios incompatibles.
- Semántica inconsistente de “no encontrado”: `null`, 204 o excepción
  encapsulada según ruta y repositorio.
- Seguimiento de `nuevaLocalizacion` implementado en endpoints concretos y sin
  límite/ciclo explícito.
- `Content-Disposition` construido sin codificación robusta del nombre.
- Clientes JAX-RS creados en salud sin cierre explícito.
- `SimpleDateFormat` se guarda en campos de beans `@ApplicationScoped` en las
  dos integraciones; no es thread-safe y puede corromper fechas bajo carga.
- Carga completa de documentos en `byte[]`; PDFs grandes elevan memoria y GC.
- Conversión y estampado son síncronos dentro de la petición HTTP.
- Métricas diarias en memoria pueden perderse al reiniciar antes del flush.
- Jobs en varios nodos podrían duplicar flush o limpieza.
- `docker-compose.yml` usa 2.1.6 y el POM 2.1.7.
- El README usa en un ejemplo `es.caib.concsv.files`, pero el código espera
  `es.caib.concsv.fitxers`.
- Las entradas nulas de caché se escriben, pero `CacheHelper` las lee como miss;
  la caché negativa es ineficaz y sólo genera ficheros.
- `HashService.checkHashFromUUID` no cae a Custodia Antigua ni aplica el
  seguimiento de `nuevaLocalizacion`; sólo los endpoints de imprimible lo hacen.
- `PersistenceConfig` de `concsv-persistence` declara la unidad `main` y paquetes
  que no coinciden con las entidades reales; es código legado que confunde.
- `HashServiceTest` está deshabilitado, precisamente en la clase con más lógica
  de negocio.

## Deuda de diseño

- `HashService` mezcla selección de repositorio, caché, transformación PDF,
  metadatos y observabilidad; es una clase de alta responsabilidad.
- Los contratos de integración filtran excepciones y modelos concretos hacia
  capas HTTP.
- Hay configuración booleana representada unas veces por `boolean` y otras por
  texto `S`/`N` o `String`.
- `StartupServiceBean` no valida configuración aunque sus comentarios lo
  sugieren.
- La documentación OpenAPI no describe de forma completa respuestas, MIME y
  seguridad real.

## Prioridad de remediación

1. Corregir claves/saneamiento/escritura de caché y cubrir concurrencia.
2. Desactivar stack traces en producción y definir la exposición de la API
   interna (red, autenticación y autorización por operación).
3. Evitar cachear `Certificate Error` y retroportar `9026fc1` con pruebas.
4. Sustituir salud simulada por `checkDatabaseLatency()` con timeout y cierre de
   clientes.
5. Unificar contrato de documento ausente y traslado por UUID.
6. Hacer explícita la estrategia de estado y timers para despliegue multinodo.
7. Añadir pruebas de `HashService`, controladores, API, frontend y despliegue EAR.

## Regla para cambios de riesgo

Un cambio en `HashService`, `CacheHelper`, `PrintableUtils`, controladores o
configuración JBoss requiere pruebas de regresión del flujo completo, no sólo
unitarias. Los cambios de modelos serializables deben asumir que existen
entradas antiguas en disco y verificar su invalidación segura.
