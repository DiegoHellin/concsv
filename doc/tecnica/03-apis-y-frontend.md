# APIs y frontend

## API pública Spring MVC

Base: `/concsvfront/api`.

| Método y ruta | Resultado |
| --- | --- |
| `GET /api` | Índice HATEOAS |
| `GET /api/documents` | Índice de operaciones documentales |
| `GET /api/documents/{csv}` | `DocumentInfo` y enlaces |
| `GET /api/documents/{csv}/original?preview=` | Original; attachment salvo preview |
| `GET /api/documents/{csv}/printable?preview=&lang=` | PDF imprimible |
| `GET /api/documents/{csv}/preview?lang=` | Imprimible si es posible; original como fallback |
| `GET /sysenv` | Variables públicas de configuración del frontend |
| `GET /manifest` | Información del manifiesto |

`preview` controla `Content-Disposition`, no el contenido. El endpoint de
preview silencia cualquier excepción al generar el imprimible y cae al
original. Los nombres de archivo se insertan directamente en la cabecera; deben
normalizarse si el origen puede contener caracteres de control.

El controlador registra métricas incluso cuando la operación falla. Un 204 se
clasifica `INVALID`, un 2xx `OK` y el resto/excepción `ERROR`.

## API interna JAX-RS

Base: `/concsvapi/interna`; `@ApplicationPath("/")`.

| Método y ruta | Función |
| --- | --- |
| `GET /test` | Liveness simple con hora del sistema |
| `GET /printable/{csv}` | Imprimible por CSV |
| `GET /printable/uuid/{uuid}` | Imprimible por UUID y seguimiento de traslado |
| `GET /original/{csv}` | Original por CSV, sujeto a exclusiones |
| `GET /original/uuid/{uuid}` | Original por UUID, sujeto a exclusiones |
| `GET /enidoc/{csv}` | Paquete/documento ENI por CSV |
| `GET /enidoc/uuid/{uuid}` | ENI por UUID |
| `GET /metadata/{csv}` | Mapa de metadatos por CSV |
| `GET /metadata/uuid/{uuid}` | Mapa de metadatos por UUID |
| `GET /v1/salut` | Salud Comanda |
| `GET /v1/salut/info` | Información de aplicación e integraciones |
| `GET /v1/salutPerformance` | Performance/liveness textual |
| `GET /v1/estadistiques` | Estadísticas del último día |
| `GET /v1/estadistiques/info` | Catálogo de dimensiones e indicadores |
| `GET /v1/estadistiques/{dies}` | Estadísticas de N días |
| `GET /v1/estadistiques/of/{data}` | Estadísticas de una fecha |
| `GET /v1/estadistiques/from/{inicio}/to/{fin}` | Estadísticas de un rango |
| `GET /v1/logs` | Listado de logs |
| `GET /v1/logs/{nombre}` | Contenido/modelo del log |
| `GET /v1/logs/{nombre}/linies/{n}` | Últimas N líneas |

La clase completa `FrontResource` es la fuente para variantes adicionales de
rangos estadísticos y logs. Swagger se publica bajo
`/concsvapi/interna/swagger/`.

## Semántica HTTP y errores

- Documento ausente en API interna suele devolver `204 No Content`, no 404.
- Original excluido devuelve `403 Forbidden`.
- UUID sin CSV para imprimible/ENI devuelve `400 Bad Request`.
- Descargas internas usan normalmente `application/octet-stream` aunque el
  contenido tenga un MIME más específico.
- `GlobalExceptionHandler` convierte `DocumentNotExistException` en 404 y
  `RuntimeException` en 500. No declara manejador para `GenericServiceException`
  (excepción comprobada), por lo que ésta puede acabar en un 500 del contenedor.
  Es la causa de que “no encontrado” sea 404 o 500 según la ruta.
- `reflectoring.trace` vale `true` por defecto y `?trace=` añade el stack trace
  a la respuesta de error. Es una fuga de información si no se desactiva.
- Los mapeadores comunes convierten excepciones EJB, validación y procesamiento
  a `ErrorBean`.

La seguridad efectiva está en `concsv-api-interna/src/main/webapp/WEB-INF/web.xml`:

- `/v1/salut/*`, estadísticas y logs exigen el rol `CSV_COM` mediante Keycloak;
- `/v1/salut`, `/v1/salutPerformance`, Swagger y las operaciones documentales
  se declaran sin autenticación;
- la restricción `CSV_REST` de printable, original, ENI y metadata está
  comentada explícitamente por problemas de rendimiento de autenticación;
- el WAR público declara todo `/*` como no autenticado, aunque mantiene el
  `login-config` Keycloak.

Las anotaciones OpenAPI hablan de `basic-auth`, mientras el descriptor usa
Keycloak, por lo que la documentación publicada no representa exactamente el
mecanismo real. Los métodos de servicio están mayoritariamente marcados
`@PermitAll`; hay anotaciones `@RolesAllowed` comentadas.

Comprobación externa del 2026-11-02: `https://csv.caib.es/concsvapi/interna/test`,
`/swagger/` y `/metadata/...` devuelven un 404 generado por Apache, no por
JBoss. Por tanto ese context root no está publicado en ese hostname; la falta de
autenticación del WAR queda compensada por la topología de proxy/red, pero sigue
siendo una debilidad de defensa en profundidad. No se debe publicar
`/concsvapi/interna` sin autenticación ni exponer directamente los puertos de
JBoss.

## CORS e idioma

`concsv-commons` contiene filtros CORS para servlet y JAX-RS y un filtro de
locale. El frontend usa catalán y castellano. La API pública acepta `lang` con
valor por defecto `ca`; la interna aplica `ca` cuando no se informa.

## Frontend React

Fuentes: `concsv-front/src/main/resources/concsv-front`.

| Elemento | Responsabilidad |
| --- | --- |
| `src/main.tsx` | Bootstrap de React |
| `src/App.tsx` | Aplicación raíz |
| `src/AppRoutes.tsx` | Rutas públicas |
| `src/pages/Home.tsx` | Formulario de consulta de CSV |
| `src/pages/View.tsx` | Información y acciones del documento |
| `src/components/Preview.tsx` | Previsualización embebida |
| `src/i18n/*` | Traducciones CA/ES |
| `src/theme.ts` | Tema Material UI |
| `lib/*` | Componentes y utilidades compartidas |
| `public/sysenv` y `public/manifest` | Placeholders sustituidos/servidos por backend |

Las rutas de `AppRoutes.tsx` son: índice `Home`; `view/:csv`, `hash/:csv`,
`view.xhtml` e `index.xhtml` apuntando a `View`; `a11y`; y fallback 404. La
página física `view.xhtml` conserva compatibilidad con URLs antiguas
`?hash=<csv>` y redirige a `view/<csv>`; sin hash redirige a `home`.

`View.tsx` muestra metadatos ENI y firmantes, pero no ofrece una acción de
exportación ENI; el issue #1 solicita añadirla. `Home.tsx` incorpora reCAPTCHA
cuando `VITE_RECAPTCHA_ENABLED` está activo, y `View.tsx` usa
`VITE_PREVIEW_ENABLED` para decidir la previsualización inicial.

El frontend obtiene la URL base del API y flags de preview/reCAPTCHA desde
`SysenvController`. `GET /sysenv?format=vite` (o `reactapp`) devuelve
`window.__RUNTIME_CONFIG__` con las claves `VITE_*`/`REACT_APP_*` y un conjunto
explícito de propiedades de `PropertyConfig`; sin `format` devuelve vacío.
`GET /manifest` devuelve `window.__MANIFEST__`. Ambos están fuera de `/api`.

## Construcción del frontend

`concsv-front/pom.xml` y `package.json` son la referencia del ciclo exacto. El
`frontend-maven-plugin` instala Node 16.17.0, ejecuta `npm install` y
`npm run build-front`; Vite genera `dist`, que Maven incorpora en `static` del
WAR. Para trabajo aislado se usan los scripts npm, pero el entregable soportado
sigue siendo el WAR dentro del EAR.

No confundir este frontend público con el backoffice React de ramas 2.2.x. En
2.1.7 no hay pantallas de administración de propiedades, exclusiones o caché.
