# Documentación técnica de CONCSV 2.0

## Propósito y alcance

Esta carpeta describe la aplicación desplegada desde la rama `concsv-2.0`. La
referencia analizada es la versión Maven `2.1.7`, commit
`b75ab0073a0a80e502bc2c3cceaec62b9c08a96e`.

La documentación se ha obtenido del código, descriptores de despliegue,
configuración, scripts SQL, pruebas e historial del repositorio. Cuando el
comportamiento observado no coincide con nombres, comentarios o documentación
OpenAPI, se considera autoritativo el código de esta rama.

No deben atribuirse a producción funcionalidades que sólo estén en
`concsv-dev` o `concsv-dev-2.2.2`. En particular, el backoffice React asociado a
los issues #3 y #17 no forma parte del artefacto `2.1.7`, aunque exista trabajo
posterior en esas ramas.

## Resumen y estado de revisión

El resumen ejecutivo, los hallazgos principales y las correcciones aplicadas
están en [../README.md](../README.md). La revisión se cerró el 2026-11-02
contrastando el código de esta rama con la comprobación HTTP de
`csv.caib.es`. No se encontraron documentos previos de otros modelos en `doc/`
ni en las ramas remotas.

## Mapa de lectura

| Documento | Contenido |
| --- | --- |
| [01-arquitectura-y-modulos.md](01-arquitectura-y-modulos.md) | Arquitectura, dependencias, módulos y artefactos |
| [02-flujos-e-integraciones.md](02-flujos-e-integraciones.md) | Consulta, descarga, imprimible, ENI, caché e integraciones |
| [03-apis-y-frontend.md](03-apis-y-frontend.md) | Superficies HTTP, errores, seguridad y React |
| [04-datos-configuracion-y-operacion.md](04-datos-configuracion-y-operacion.md) | Oracle, propiedades, compilación, Docker y operación |
| [05-calidad-riesgos-y-pruebas.md](05-calidad-riesgos-y-pruebas.md) | Pruebas existentes, riesgos y deuda técnica |
| [06-evolucion-y-guia-de-mantenimiento.md](06-evolucion-y-guia-de-mantenimiento.md) | Issues, ramas y procedimiento para futuros cambios |

## Resumen ejecutivo

CONCSV consulta documentos por CSV o UUID en dos repositorios externos: el
Archivo Digital nuevo y la Custodia Antigua. Expone:

- una aplicación pública React y una API Spring MVC en `/concsvfront`;
- una API JAX-RS de integración y monitorización en `/concsvapi/interna`;
- servicios EJB/CDI que encapsulan consulta, descarga, generación de copias
  imprimibles, exportación ENI, caché y métricas;
- persistencia Oracle exclusivamente para estadísticas de explotación.

El entregable es un EAR para JBoss EAP 7.2.7. El código se compila con Java 11,
Spring Boot 2.7.18 y APIs Java EE 8. El frontend React/TypeScript se construye
durante el empaquetado del WAR.

## Diagrama de contexto

```mermaid
flowchart LR
    U[Ciudadanía / navegador] --> F[concsv-front WAR]
    C[Clientes internos / Comanda] --> A[concsv-api-interna WAR]
    F --> E[concsv-ejb]
    A --> E
    E --> S[concsv-service]
    S --> AD[Archivo Digital]
    S --> CA[Custodia Antigua SOAP]
    S --> AF[@firma]
    S --> OO[OpenOffice]
    S --> FS[Caché filesystem]
    S --> DB[(Oracle concsvDS)]
```

## Fuentes principales

- `pom.xml`: versión, plataforma y reactor Maven.
- `concsv-ear/pom.xml`: composición y context roots del EAR.
- `concsv-service/.../HashService.java`: orquestación documental.
- `concsv-api-interna/.../FrontResource.java`: API interna.
- `concsv-front/.../DocumentController.java`: API pública.
- `concsv-service-intf/.../PropertyConfig.java`: catálogo base de propiedades.
- `concsv-ear/src/main/docker`: configuración de imagen y JBoss.
- `scripts/bbdd/2.0/oracle`: esquema de estadísticas.

## Cómo mantener estos documentos

Cada cambio funcional debe actualizar el capítulo correspondiente. Las rutas,
propiedades y respuestas HTTP deben contrastarse con código ejecutable, no sólo
con anotaciones OpenAPI. Al publicar una versión nueva debe actualizarse la
referencia de rama, versión y commit de este archivo y revisarse el documento de
evolución.
