# Datos, configuración y operación

## Persistencia

La unidad JPA `concsvPU` es JTA y usa
`java:jboss/datasources/concsvDS`, dialecto `Oracle10gDialect` y SQL oculto.
Las entidades representan un esquema estrella mínimo de explotación:

| Entidad | Responsabilidad |
| --- | --- |
| `ExplotTempsEntity` | Día de agregación |
| `ExplotDimensioEntity` | Tipo de operación y origen |
| `ExplotFetsEntity` | Contadores y tiempo medio |

Los scripts están en `scripts/bbdd/2.0/oracle`, separados en tablespace,
tablas, secuencias, índices y grants. Deben ejecutarse en ese orden y adaptarse
a usuarios/tablespaces del entorno. JPA no está configurado para crear o
actualizar el esquema.

`concsv-persistence` también contiene un `PersistenceConfig` de Spring que
declara la unidad `main` y paquetes `es.caib.concsv.persist.entity`. No es la
configuración activa: `ConcsvFrontApp` sólo escanea `es.caib.concsv.front`, y
`EstadisticaHelper` usa `@PersistenceContext(unitName = "concsvPU")`. Ese
`PersistenceConfig` debe considerarse legado o código a limpiar, no fuente de
verdad para el despliegue.

## Fuentes de configuración

`PropertyFileConfigSource` carga dos ficheros indicados por propiedades de
sistema:

- `es.caib.concsv.properties`: configuración funcional;
- `es.caib.concsv.system.properties`: rutas, credenciales y endpoints.

La carga es fail-fast: el bloque estático de `PropertyFileConfigSource` lanza
`RuntimeException` si falta cualquiera de las dos propiedades de sistema o si
no puede leer el fichero. El WAR público también las importa con
`SystemPropertiesConfig` para los controladores Spring.

En Docker se copian plantillas a `$JBOSS_USERHOME/apps/concsv` y
`jboss_config.sh` adapta JBoss. También pueden inyectarse propiedades Java con
el nombre completo mediante variables de entorno, como hace
`docker-compose.yml`.

## Propiedades de aplicación

| Propiedad | Uso | Valor por defecto conocido |
| --- | --- | --- |
| `es.caib.concsv.fitxers` | Raíz de ficheros y caché | Obligatoria para caché |
| `es.caib.concsv.logo.path` | Logo de copias imprimibles | Cadena vacía |
| `es.caib.concsv.query.url` | URL codificada en verificación/QR | Sin defecto |
| `es.caib.concsv.performance` | Instrumentación de rendimiento | Según entorno |
| `es.caib.concsv.logs.location` | Directorio de logs consultable | `standalone/log/` |
| `es.caib.concsv.cache.activa` | Activa caché local | `false` |
| `es.caib.concsv.cache.ttl.minuts` | TTL deslizante | `30`; `0` no expira |
| `es.caib.concsv.amagar.boto.original` | Oculta original si hay imprimible | `false` |
| `es.caib.concsv.arxiu.documents.exclosos.path` | CSV sin descarga original | Vacío |
| `es.caib.concsv.estadisticas.dias.conservar` | Retención de estadísticas | `365` |
| `es.caib.concsv.optionalLabelMetadata.path` | Etiquetas de metadatos | Sin defecto |
| `es.caib.concsv.consult.oldSafeKeeping` | Habilita Custodia Antigua | Se espera `S` |
| `es.caib.concsv.consult.newDigitalArchive` | Habilita Archivo Digital | Se espera `S` |

## Propiedades de integración

Archivo Digital usa `new.digital.archive.endpoint`, `organization`,
`app.client`, `username`, `password`, `version` y `traces`. Custodia Antigua
usa `old.savekeeping.endpoint` y `old.savekeeping.timeout`.

@firma usa el prefijo
`es.caib.concsv.plugins.validatesignature.afirmacxf` para endpoint,
`applicationID`, método y credenciales de autorización, ruta de plantillas,
`printxml` y `debug`. OpenOffice usa
`plugins.documentconverter.openoffice.host` y `.port`.

El frontend usa `front.api.url`, `front.preview.enabled`,
`front.recaptcha.enabled` y `front.recaptcha.sitekey`.

No registrar ni versionar valores reales de passwords, secretos o tokens. Las
plantillas del repositorio contienen marcadores, no valores de producción.

## Compilación

Requisitos: JDK 11, Maven compatible, acceso a Maven Central, repositorios
Red Hat/JBoss y `https://governib.github.io/maven/maven/`. No es necesario un
Node global para el build Maven: el plugin instala Node 16.17.0 dentro de
`concsv-front/target`. La compilación documentada es:

```bash
mvn clean install -DskipTests
```

El resultado desplegable es `concsv-ear/target/concsv.ear`. Para validar cambios
de lógica debe omitirse `-DskipTests`. El `buildnumber-maven-plugin` obtiene
rama y revisión SCM durante `validate`.

## Despliegue JBoss

1. Crear el datasource JNDI `java:jboss/datasources/concsvDS`.
2. Definir las propiedades de sistema `es.caib.concsv.properties` y
   `es.caib.concsv.system.properties` con las rutas absolutas de los dos
   ficheros, en `standalone.xml` o mediante el script de imagen. Sin ambas, la
   aplicación no arranca.
3. Instalar recursos: logo, etiquetas opcionales, truststore y plantillas de
   @firma.
4. Copiar `concsv.ear` a `$JBOSS_HOME/standalone/deployments`.
5. Verificar `/concsvapi/interna/test`, `/v1/salutPerformance`, Swagger y el
   acceso a `/concsvfront/home`.
6. Verificar conectividad real con Oracle, Archivo Digital, Custodia Antigua,
   @firma y OpenOffice según las funciones habilitadas.

## Docker

`mvn -pl concsv-ear docker:build` usa `docker-maven-plugin` y la imagen base
`280421/jboss-eap-caib-7.2`. Copia configuración a `$JBOSS_USERHOME`, recursos a
`$JBOSS_USERHOME/apps/concsv` y el EAR a deployments.

`docker-compose.yml` publica 8080, 8787 y 9990 y actualmente referencia
`concsv:2.1.6`, mientras el POM es `2.1.7`. Debe alinearse antes de usarlo como
evidencia de una prueba de la versión actual. El fichero también contiene hosts
de red corporativa específicos de entorno.

## Lista de comprobación operativa

- Espacio y permisos de escritura en `${fitxers}/cache`.
- Hora y zona horaria coherentes para TTL, manifiesto y agregación diaria.
- Retención y permisos de lectura del directorio de logs.
- Certificados/truststore válidos para endpoints HTTPS.
- Timeout y disponibilidad de integraciones externas.
- Jobs de estadísticas y limpieza ejecutándose una sola vez por clúster o con
  una estrategia explícita.
- Fichero de exclusiones presente y legible cuando se configure.
- Monitorización externa independiente de `/v1/salut`, porque su chequeo de BD
  no ejecuta una consulta real.
- Comprobar desde fuera que el proxy no publica `/concsvapi/interna` ni los
  puertos de JBoss. El 2026-11-02 `csv.caib.es` devolvía 404 de Apache para ese
  context root.
- Revisar que `reflectoring.trace` esté desactivado o que `?trace=` no exponga
  stack traces en producción.
