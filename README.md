<img alt="Logo concsv" height="150" src="https://github.com/GovernIB/concsv/raw/main/assets/logo_concsv.png"/>

# CONCSV
Consulta de documents de l'arxiu de la CAIB donat el seu CSV

## Documentació tècnica
La documentació revisada de la versió 2.1.7 (branca `concsv-2.0`) és a
[doc/README.md](doc/README.md). L'índex detallat és a
[doc/tecnica/README.md](doc/tecnica/README.md).

## Compilar
Executar la següent comanda:

```
$ mvn clean install -DskipTests
```
La compilació del projecte crea els següents fitxers per a desplegar l'aplicació:
- concsv-ear/target/concsv.ear: fitxer per a executar l'aplicació sobre un servidor JBoss EAP 7.2.

## Execució sobre JBoss EAP 7.2

### Configuració de les propietats
Modificar o afegir la secció `<system-properties>` del fitxer /standalone/configuration/standalone.xml amb el següent contingut:

```
    <system-properties>
        <property name="es.caib.concsv.properties" value="JBOSS_PROPS_PATH/jboss.properties"/>
        <property name="es.caib.concsv.system.properties" value="JBOSS_PROPS_PATH/jboss_system.properties"/>
    </system-properties>
```
En aquest contingut d'exemple s'han de substituir les següents variables:
- JBOSS_PROPS_PATH: carpeta a on es troben els fitxers de propietats.

Exemple de contingut del fitxer jboss.properties:

```
es.caib.concsv.new.digital.archive.organization=DIGITAL_ARCHIVE_ORG
es.caib.concsv.new.digital.archive.version=DIGITAL_ARCHIVE_VERSION
es.caib.concsv.new.digital.archive.traces=DIGITAL_ARCHIVE_TRACES

es.caib.concsv.consult.oldSafeKeeping=OLD_SAFEKEEPING_ACTIVE
es.caib.concsv.consult.newDigitalArchive=DIGITAL_ARCHIVE_ACTIVE

es.caib.concsv.performance=PERFORMANCE_ACTIVE
es.caib.concsv.cache.ttl.minuts=CACHE_TTL_MINUTS

es.caib.concsv.query.url=http://APP_HOST/concsvfront/view.xhtml?hash=

es.caib.concsv.front.preview.enabled=PREVIEW_ENABLED
```
Exemple de contingut del fitxer jboss_system.properties:

```
es.caib.concsv.files=FILES_PATH
es.caib.concsv.logo.path=LOGO_PATH
es.caib.concsv.optionalLabelMetadata.path=LABEL_METADATA_PATH
es.caib.concsv.old.savekeeping.endpoint=OLD_SAFEKEEPING_ENDPOINT
es.caib.concsv.new.digital.archive.endpoint=DIGITAL_ARCHIVE_ENDPOINT
es.caib.concsv.new.digital.archive.app.client=DIGITAL_ARCHIVE_APP
es.caib.concsv.new.digital.archive.username=DIGITAL_ARCHIVE_USERNAME
es.caib.concsv.new.digital.archive.password=DIGITAL_ARCHIVE_PASSWORD
es.caib.concsv.plugins.validatesignature.afirmacxf.endpoint=VALIDATESIGNATURE_AFIRMA_ENDPOINT
es.caib.concsv.plugins.validatesignature.afirmacxf.applicationID=VALIDATESIGNATURE_AFIRMA_APP_ID
es.caib.concsv.plugins.validatesignature.afirmacxf.authorization.method=VALIDATESIGNATURE_AFIRMA_AUTH_METHOD
es.caib.concsv.plugins.validatesignature.afirmacxf.authorization.username=VALIDATESIGNATURE_AFIRMA_AUTH_USERNAME
es.caib.concsv.plugins.validatesignature.afirmacxf.authorization.password=VALIDATESIGNATURE_AFIRMA_AUTH_PASSWORD
es.caib.concsv.plugins.validatesignature.afirmacxf.TransformersTemplatesPath=VALIDATESIGNATURE_AFIRMA_TEMPLATES_PATH
es.caib.concsv.plugins.validatesignature.afirmacxf.printxml=VALIDATESIGNATURE_AFIRMA_PRINTXML
es.caib.concsv.plugins.validatesignature.afirmacxf.debug=VALIDATESIGNATURE_AFIRMA_DEBUG
es.caib.concsv.plugins.documentconverter.openoffice.host=DOCUMENTCONVERTER_HOST
es.caib.concsv.plugins.documentconverter.openoffice.port=DOCUMENTCONVERTER_PORT

es.caib.concsv.front.api.url=FRONT_API_URL
es.caib.concsv.front.recaptcha.enabled=FRONT_RECAPTCHA_ENABLED
es.caib.concsv.front.recaptcha.sitekey=FRONT_RECAPTCHA_SITE_KEY
```
### Desplegament i execució
Per a desplegar l'aplicació s'ha de copiar el fitxer concsv-ear/target/concsv.ear generat amb la copilació de maven a la carpeta /standalone/deployments.
Per a iniciar el servidor jboss s'ha d'anar a la carpeta bin del servidor i executar la següent comanda:

```
$ ./standalone.sh
```
## Execució sobre JBoss emprant Docker
Crear la imatge de docker amb la següent comanda:

```
$ mvn -pl concsv-ear docker:build
```

Iniciar l'aplicació amb la següent comanda:

```
$ docker compose up
```
Aturar l'execució de l'aplicació amb la següent comanda:

```
$ docker compose down
```
Inspeccionar la imatge docker amb la següent comanda:

```
$ docker run --rm -it --entrypoint=/bin/sh concsv
```

## Canvi de versió

```
$ mvn versions:set -DnewVersion=X.Y.Z
$ mvn versions:commit
```
