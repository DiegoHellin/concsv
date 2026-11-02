# Documentación de CONCSV

## Qué es este repositorio

CONCSV es la aplicación de la CAIB que permite verificar y recuperar un
documento electrónico a partir de su Código Seguro de Verificación (CSV). La
aplicación localiza el documento en el Archivo Digital o en la Custodia Antigua,
muestra sus metadatos y firmantes, permite descargar el original cuando está
permitido y genera una copia auténtica imprimible con CSV, QR y datos de
verificación.

## Línea base documentada

- Rama: `concsv-2.0`
- Versión: `2.1.7`
- Commit: `b75ab0073a0a80e502bc2c3cceaec62b9c08a96e`
- Entregable: `concsv-ear/target/concsv.ear`
- Runtime: JBoss EAP 7.2.7, Java 11, Java EE 8
- Frontend: React/TypeScript/Vite incluido en el WAR público

No deben atribuirse a esta versión las funcionalidades del backoffice React que
sólo existen en `concsv-dev` o `concsv-dev-2.2.2`.

## Índice de documentación

| Documento | Contenido |
| --- | --- |
| [tecnica/README.md](tecnica/README.md) | Índice detallado y alcance |
| [tecnica/01-arquitectura-y-modulos.md](tecnica/01-arquitectura-y-modulos.md) | Arquitectura, módulos y artefactos |
| [tecnica/02-flujos-e-integraciones.md](tecnica/02-flujos-e-integraciones.md) | Consulta, descarga, imprimible, ENI, caché e integraciones |
| [tecnica/03-apis-y-frontend.md](tecnica/03-apis-y-frontend.md) | APIs, errores, seguridad y React |
| [tecnica/04-datos-configuracion-y-operacion.md](tecnica/04-datos-configuracion-y-operacion.md) | Oracle, propiedades, compilación, JBoss y Docker |
| [tecnica/05-calidad-riesgos-y-pruebas.md](tecnica/05-calidad-riesgos-y-pruebas.md) | Pruebas, riesgos y deuda técnica |
| [tecnica/06-evolucion-y-guia-de-mantenimiento.md](tecnica/06-evolucion-y-guia-de-mantenimiento.md) | Issues, ramas y mantenimiento |

## Resumen técnico

- El EAR contiene un EJB JAR y dos WAR: `/concsvfront` y
  `/concsvapi/interna`.
- El WAR público ofrece la aplicación React y la API Spring MVC en
  `/concsvfront/api`.
- La API JAX-RS interna ofrece descargas, metadatos, ENI, salud, estadísticas y
  logs.
- `HashService` orquesta la consulta en Archivo Digital y Custodia Antigua.
- La persistencia Oracle se usa sólo para estadísticas de explotación
  (`concsvPU`); los documentos no se almacenan en esa base de datos.
- La caché es local al filesystem y opcional (`cache.activa`).
- La aplicación mantiene estado local: caché, exclusiones, métricas y salud.
  No es stateless y no debe escalarse horizontalmente sin una estrategia
  explícita.

## Hallazgos principales

1. La API documental interna no tiene autenticación en `web.xml`; la restricción
   `CSV_REST` está comentada. El 2026-11-02, `csv.caib.es` devolvía un 404 de
   Apache para `/concsvapi/interna`, por lo que no estaba publicada en ese
   hostname. El riesgo es de defensa en profundidad y debe vigilarse.
2. La clave de caché del original no incluye `packedFile`; puede mezclar el
   original empaquetado y el no empaquetado.
3. `lang` se usa sin sanear al construir rutas de caché; con caché activa puede
   permitir path traversal.
4. La salud de base de datos está simulada con 1 ms, aunque existe
   `EstadisticaHelper.checkDatabaseLatency()` sin cablear.
5. El issue #16 sigue abierto en 2.1.7. El fix `9026fc1` de 2.2.2 evita cachear
   documentos con firmante `Certificate Error`, pero no corrige las otras
   debilidades de caché.
6. `GlobalExceptionHandler` puede devolver stack traces con `?trace=` porque
   `reflectoring.trace` vale `true` por defecto.
7. `HashServiceTest` no tiene pruebas activas; no hay tests de controladores,
   frontend ni despliegue EAR.
8. `docker-compose.yml` referencia `concsv:2.1.6` y el POM es `2.1.7`; el README
   usa `es.caib.concsv.files` en un ejemplo, pero el código espera
   `es.caib.concsv.fitxers`.

## Correcciones aplicadas en esta revisión

- Se corrigieron las rutas React: `view/:csv`, `hash/:csv`, `view.xhtml`,
  `index.xhtml`, `a11y` y fallback 404.
- Se aclaró que `PersistenceConfig` de `concsv-persistence` es legado y no carga
  en este EAR; la unidad activa es `concsvPU`.
- Se precisó el tratamiento 404/500 de `GenericServiceException` y la fuga de
  stack traces.
- Se acotó el alcance del fix `9026fc1` del issue #16.
- Se documentó la comprobación externa de la API interna y se rebajó de P0 a
  riesgo de defensa en profundidad.
- Se marcaron las pruebas deshabilitadas y las dependencias de entorno.
- Se corrigió el valor por defecto de retención de estadísticas a 365 días.
- Se documentó que la caché negativa es ineficaz y que la salud de BD tiene un
  método real sin usar.

## Evidencia y limitaciones

La revisión se hizo sobre el código de `concsv-2.0`, los descriptores de
despliegue, las plantillas de configuración, los scripts SQL, el historial Git y
las issues del milestone. No se encontraron documentos previos de otros modelos
en `doc/` ni en las ramas remotas; la documentación existente se contrastó
directamente con el código y con la comprobación HTTP del 2026-11-02.

No se pudo ejecutar `mvn` porque no está instalado y el proyecto no incluye
wrapper. Tampoco se dispone de Oracle, JBoss ni las integraciones corporativas
para una prueba completa. Las afirmaciones sobre comportamiento en ejecución se
basan en lectura de código y en la evidencia HTTP indicada.

## Uso recomendado

Para un cambio de código, empezar por [tecnica/06-evolucion-y-guia-de-mantenimiento.md](tecnica/06-evolucion-y-guia-de-mantenimiento.md),
localizar el área en el capítulo correspondiente y revisar los riesgos de
[tecnica/05-calidad-riesgos-y-pruebas.md](tecnica/05-calidad-riesgos-y-pruebas.md).
Para operación, usar [tecnica/04-datos-configuracion-y-operacion.md](tecnica/04-datos-configuracion-y-operacion.md).
