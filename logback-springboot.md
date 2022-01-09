# Logback avec Spring Boot

Différence entre **SLF4J**, **Logback** et **Log4j2**
- SLF4J (*Simple Logging Facade for Java*) est une **interface** de framework de journalisation
- java.util.logging, Logback et Log4j2 sont des **implémentations** de cette interface

Rappel sur Spring Boot :
- par défaut, Spring Boot utilise le framework de log **Logback**
- pour utiliser **Log4j2** à la place, il faut modifier la configuration du projet


# Configuration avec Log4j2

- modification du POM pour exclure la bibliothèque de log par défaut et utiliser log4j2 à la place

```xml
<!-- Indique à Spring Boot que l'on utilise log4j2 et pas logback qui est proposé par défaut -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-logging</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-log4j2</artifactId>
</dependency>
```

- il faut ensuite créer un fichier de configuration de la log en XML, généralement placé dans `src/main/resources`
- il faut ensuite indiquer dans une property pour que le CEI puisse surcharger ce fichier

```properties
# en local
logging.config=classpath:log4j2.xml
# surcharge CEI
logging.config=/etc/tomcat9/log4j2-cei.xml
```

# Configuration avec Logback

- pour utiliser Logback, il n'est donc plus nécessaire de modifier le POM car logback est le framework de log par défaut
- fonctionne directement sans rien configurer en écrivant dans la console

## Configuration plus fine pour les modules web
- possibilité de configurer la log via un fichier XML comme pour log4j2, mais souvent pas nécessaire
- configuration directement par property plus simple et rapide
    - possibilité de modifier les niveaux de logs selon les packages
    - possibilité d'écrire la log dans un fichier en plus de la console avec la property `logging.file.name`
- pour les modules web, le flux de sortie standard s'écrit dans le fichier `catalina.out` sur le tomcat. On écrit un fichier de log explicitement qui sera accessible sur applishare après configuration par le CEI

dans le projet :
```properties
# Gestion des logs avec LOGBACK
logging.level.root=INFO
logging.level.fr.insee=DEBUG
logging.file.name=./logs/toto.log
logging.logback.rollingpolicy.file-name-pattern=${LOG_FILE}.%d{yyyy-MM-dd}.%i.log
# taille maximum d'un fichier de log
logging.file.max-size=10MB
# coloration syntaxique des logs
spring.output.ansi.enabled=ALWAYS
# encodage du fichier de log
logging.charset.file=UTF8
```

surcharge CEI :
```properties
logging.level.root=INFO
logging.level.fr.insee=DEBUG
# écriture du fichier de log sur applishare
logging.file.name=/mnt/applishare3/sirhwsp/dv/logs/apirh.log
# ne pas activer ailleurs qu'en local, crée des caractères spéciaux dans le fichier catalina.out si jamais vous l'exploitez
spring.output.ansi.enabled=NEVER
```

## Configuration plus fine pour les modules batchs
- pour les modules batchs, il n'est pas nécessaire d'écrire dans un fichier en plus de la console car le CEI redirige le flux de sortie standard directement sur applishare en écrivant un fichier avec le nom du batch et son heure d'exécution, par exemple `pdbatdivlst02_batch_UP_APIRH_ALIMENTATION_2021-10-20_14-33-00.log`

dans le projet :
```properties
# Gestion des logs avec LOGBACK
logging.level.root=INFO
logging.level.fr.insee=DEBUG
# coloration syntaxique des logs
spring.output.ansi.enabled=ALWAYS
```

surcharge CEI :
```properties
# Gestion des logs avec LOGBACK
logging.level.root=INFO
logging.level.fr.insee=TRACE
# coloration syntaxique des logs
spring.output.ansi.enabled=NEVER
```

## Migration de log4j2 à logback
- la migration est assez simple dans le code source, cependant cela demande des actions côté intégrateur, il faut donc passer par un ticket Siamoi `Mise à jour d'un module applicatif en production`
- il faut supprimer le fichier `log4j2.xml.erb` dans le contrat puppet du module web et du module batch
- pour la partie web, il faut demander à l'intégrateur de supprimer la property `logging.config` des properties surchargées en prod, et ajouter celles présentées juste avant, et également le fichier log4j2.xml présent sur le tomcat
- pour la partie batch, il faut également faire cela, et en plus, il faut demander à enlever de la ligne de commande java la property `-Dlog4j.configuration=file:/opt/insee/sirhwsp/pd/properties/log4j.xml`
    - avant : `java11 -Xms1024m -Xmx1024m -Dlog4j.configuration=file:/opt/insee/sirhwsp/pd/properties/log4j.xml -classpath '/opt/insee/sirhwsp/pd/lib/*' -Dproperties.path=/opt/insee/sirhwsp/pd/properties fr.insee.sirhwsp.batch.Lanceur BATCH_APIRH_ALIMENTATION`
    - après : `java11 -Xms1024m -Xmx1024m -classpath '/opt/insee/sirhwsp/pd/lib/*' -Dproperties.path=/opt/insee/sirhwsp/pd/properties fr.insee.sirhwsp.batch.Lanceur BATCH_APIRH_ALIMENTATION`


# Utilisation de la log

- importance d'utiliser les imports de slf4j (interface) en lieu et place de des imports de log4j2/logback pour ne pas avoir à les changer lors du passage d'une implémentation à une autre

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

// ajouter dans la classe TestController
private static final Logger log = LoggerFactory.getLogger(TestController.class);
// ajouter dans une méthode de la classe
log.info("hello");
```
