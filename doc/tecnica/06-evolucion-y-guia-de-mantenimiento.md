# Evolución y guía de mantenimiento

## Línea base

Esta documentación corresponde a `concsv-2.0`, versión `2.1.7`, commit
`b75ab0073a0a80e502bc2c3cceaec62b9c08a96e`. Las ramas `concsv-dev` y
`concsv-dev-2.2.2` contienen decenas de commits posteriores y no representan el
despliegue descrito aquí.

El remoto `origin` es el fork `DiegoHellin/concsv`. Su rama `main` está en
`41ac658` (“Tancament ConCSV 2.1.7”), un commit distinto del `concsv-2.0`
analizado; `main` es una rama de distribución, no la línea base de este
documento.

Cambios conocidos posteriores:

- issues #3 y #17: backoffice React (`concsv-back`) en `concsv-dev` y
  `concsv-dev-2.2.2`, con pantallas de propiedades, exclusiones, caché, avisos,
  entidades e integraciones;
- issue #16: el fix `9026fc1` está sólo en `concsv-dev-2.2.2`; añade
  `isFirmaValida` y evita cachear `DocumentInfo` cuando hay un firmante
  `Certificate Error`, pero no corrige la clave de caché del original ni
  `packedFile`;
- otras mejoras 2.2.x deben revisarse mediante diff antes de retroportarlas.

No copiar una clase aislada desde 2.2.x sin revisar cambios de contratos,
dependencias, configuración, esquema y frontend asociados.

## Estado de issues a 2026-11-02

El milestone `concsv-2.0` tenía 22 issues: 13 abiertos y 9 cerrados.

| # | Estado | Tema | Prioridad / área |
| --- | --- | --- | --- |
| 22 | Abierto | Contraste de propiedades no editables | Normal / Web |
| 21 | Abierto | Información de documentos en caché | Normal / Web |
| 20 | Abierto | Renombrar sección de excluidos | Baja / Web |
| 19 | Abierto | Enlaces en lista de excluidos | Normal / Web |
| 18 | Abierto | No imprimir PDF Portfolio | Alta / Core |
| 17 | Cerrado | Diseño del backoffice React | Web; no está en 2.1.7 |
| 16 | Abierto | No cachear error de validación de firma | Alta / Core; fix posterior |
| 15 | Abierto | Separar Custodia en Comanda | Alta / WebServices |
| 14 | Cerrado | Seguir traslado a nuevo nodo sin CSV | Alta |
| 13 | Cerrado | Error de original por UUID | Alta |
| 12 | Cerrado | Estado incorrecto de subsistemas | Alta |
| 11 | Abierto | Reducir logo en imprimible | Normal |
| 10 | Cerrado | No tratar ausencia como error | Normal |
| 9 | Cerrado | Errores de dependencias en build | Sin etiqueta |
| 8 | Cerrado | Caché filesystem | Alta / Core |
| 7 | Cerrado | Desactivar TinyCaib | Alta / Core |
| 6 | Abierto | Adaptación global stateless | Alta / Core |
| 5 | Abierto | Sesión en entorno stateless | Alta / Core |
| 4 | Abierto | Análisis/desacoplamiento stateless | Alta / Core |
| 3 | Cerrado | Backoffice de administración | Alta / Web; no está en 2.1.7 |
| 2 | Abierto | Configuración multientidad | Normal / Core |
| 1 | Abierto | Exportación ENI desde Web | Alta / Web |

El estado “cerrado” de GitHub no demuestra que el cambio esté en esta rama. #3
y #17 son ejemplos explícitos: pertenecen a la evolución 2.2.x. Para cada
issue debe comprobarse el commit de cierre y su pertenencia al ancestro de la
versión desplegada.

## Dónde modificar cada comportamiento

| Necesidad | Punto inicial |
| --- | --- |
| Añadir operación documental | `HashServiceInterface`, `HashService`, EJB y controlador |
| Cambiar selección de repositorio | `HashService.checkHash*` |
| Cambiar Archivo Digital | `NewDigitalArchiveService` |
| Cambiar Custodia Antigua | `OldSaveKeepingService` |
| Cambiar imprimible/QR/metadatos | `PrintableUtils` y métodos PDF de `HashService` |
| Cambiar caché | `CacheHelper`, llamadas en `HashService` y timer EJB |
| Añadir propiedad | `PropertyConfig`, inyección, plantillas Docker y documentación |
| Cambiar API pública | controladores/configuración de `concsv-front` y React |
| Cambiar API interna | `FrontResource` y OpenAPI |
| Cambiar estadísticas | enums/modelos, `EstadisticaHelper`, entidades y SQL |
| Cambiar salud/Comanda | `ComandaService`, helpers y endpoints `/v1` |
| Cambiar despliegue | `concsv-ear/pom.xml`, Docker/JBoss y `docker-compose.yml` |

## Procedimiento para un cambio

1. Confirmar rama, versión y commit objetivo.
2. Localizar contrato, implementación, fachada EJB y todas las superficies HTTP.
3. Revisar configuración, serialización, datos persistidos y compatibilidad de
   entradas de caché existentes.
4. Añadir o adaptar pruebas unitarias y de contrato.
5. Ejecutar `mvn clean test` y `mvn clean install` con JDK 11.
6. Inspeccionar el EAR y desplegar en JBoss equivalente a 7.2.7.
7. Probar integraciones habilitadas y escenarios negativos.
8. Actualizar esta documentación, README, OpenAPI y plantillas de propiedades.
9. Si procede de 2.2.x, documentar explícitamente el backport y sus dependencias.

## Guía para agentes automáticos

Antes de editar:

- leer este índice y el capítulo del área afectada;
- ejecutar `git status`, `git branch --show-current` y revisar el POM raíz;
- no asumir que `main`, `concsv-dev` o `concsv-dev-2.2.2` equivalen a producción;
- buscar usos de una propiedad o método en todos los módulos;
- preservar cambios ajenos presentes en el workspace.

Antes de concluir:

- contrastar rutas y respuestas con controladores, no sólo con Swagger;
- revisar que una propiedad nueva llegue a despliegue manual y Docker;
- comprobar riesgos de seguridad, caché, concurrencia y multinodo;
- indicar cualquier validación no ejecutada por falta de Maven, JDK, red,
  Oracle, JBoss o servicios corporativos;
- dejar referencias de fichero y símbolo para que el siguiente mantenedor pueda
  verificar la afirmación.

## Decisiones que requieren diseño previo

No deben resolverse con un parche local sin acordar arquitectura:

- conversión a stateless y almacenamiento compartido;
- coordinación de timers en clúster;
- modelo de autenticación/autorización de APIs internas;
- semántica pública de 204/404/errores funcionales;
- persistencia o invalidación distribuida de caché;
- soporte multientidad y selección de logos/configuración;
- límites de tamaño, streaming y procesamiento asíncrono de documentos.
