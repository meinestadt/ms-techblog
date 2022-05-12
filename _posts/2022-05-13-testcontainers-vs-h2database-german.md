---
title: Testcontainers vs. H2 Datenbank
excerpt_separator: "<!--more-->"
categories:
  - IT 
tags:
  - Testcontainer
  - H2 Datenbank
published: true
---
![Testcontainer vs. H2 Datenbank](/assets/images/blog/002_testcontainer_vs_h2database/blog_testcontainer_vs_h2database_1.png)

Eine hohe Qualität von Quellcode ist bei zunehmender Komplexität von Software unverzichtbar. 
Der große, oft vernachlässigte Bereich des Software-Testings stellt hierbei viele Möglichkeiten zur Verfügung, um die Qualität zu steigern.

Dieser Blogbeitrag wird ein genaues Auge auf das Testen mit Datenbanken legen und somit einen Fokus auf Integrationstests setzen. 
Dabei werden zwei unterschiedliche Verfahren verglichen. 
Auf der einen Seite steht hierbei das Testen mit einer In Memory H2 Datenbank und auf der anderen Seite das Testen mit Testcontainers. 

### Übersicht über die Datenbanksysteme
Wir, bei meinestadt.de, verwenden beim Testen unserer Software, sowohl Testcontainers als auch das H2 Datenbanksystem. 
Um einen besseren Überblick zu bekommen, werden diese im folgenden Abschnitt kurz vorgestellt:

#### H2
Im Allgemeinen ist eine H2 (Hypersonic 2) Datenbank ein in Java entwickeltes relationales Datenbankmanagementsystem. 
Dieses kann entweder in Java-Anwendungen direkt oder als Server bereitgestellt werden. Das Open Source System bietet eine schlanke In-Memory Lösung an, 
sodass die Daten nicht auf der Festplatte gespeichert werden müssen. Eine H2 wird oft dann hinzugezogen, wenn die Applikation keine großen Systeme mit ausgiebiger Funktionsvielfalt benötigt. 
Aus diesem Grunde ist sie für das Testen einer Applikation sehr beliebt, da hierfür schnelle und einfach zu errichtende Testfälle integriert werden können.

#### Testcontainers
Testcontainers ist eine Java Library, mit der man programmatisch Docker Images erzeugen und den Lebenszyklus von Containern steuern kann. 
Sie benötigen keine aufwändigen Konfigurationen und sind deswegen einfach zu nutzen. Testcontainers können alles abbilden, was in einem Docker Container laufen kann. 
Es kann daher in vielen Fällen das Datenbanksystem genommen werden, welches ebenfalls im Produktiv-Code verwendet wird. 
Testcontainers stellen sehr viele Werkzeuge zur Verfügung, mit denen es sehr einfach ist, Tests zu schreiben und schnell hinzuzufügen.
Dadurch ist es möglich, schnell und einfach Tests zu schreiben, da kaum Aufwand für Voreinstellungen anfällt.

### Konfiguration
Bei meinestadt.de sind die meisten Services in Java geschrieben und basieren auf Spring Boot und Maven. 
Im Produktiv-Code wird häufig eine Postgres Datenbank verwendet, die vorher mit einem passenden Schema angelegt wird. 
Um die verschiedenen Interaktionen mit der Datenbank durch den Quellcode zu überprüfen, werden Tests geschrieben.
Hierbei fallen unterschiedliche Konfigurationen an, abhängig davon, ob mit einer H2 Datenbank oder Testcontainers getestet wird.
Diese Unterschiede werden im Folgenden erläutert:

#### Dependencies
In der pom.xml finden sich die ersten relevanten Unterschiede wieder. 
Für die H2 Datenbank und für Testcontainers müssen die jeweiligen Abhängigkeiten hinzugefügt werden.

> H2
>```
><dependency>
>    <groupId>com.h2database</groupId>
>    <artifactId>h2</artifactId>
>    <version>2.1.212</version>
>    <scope>test</scope>
></dependency>
>```

>Testcontainers
>```
><dependency>
>    <groupId>org.springframework.boot</groupId>
>    <artifactId>spring-boot-starter-test</artifactId>
>    <scope>test</scope>
></dependency>
><dependency>
>    <groupId>org.testcontainers</groupId>
>    <artifactId>junit-jupiter</artifactId>
>    <scope>test</scope>
></dependency>
><dependency>
>    <groupId>org.testcontainers</groupId>
>    <artifactId>postgresql</artifactId>
>    <scope>test</scope>
></dependency>
>```

### Application Properties
In Spring-Boot können die Property-Dateien genutzt werden, um weitere Konfigurationen einbauen zu können.
Diese werden genutzt, damit der Produktiv-Scope als auch der Test-Scope von verschiedenen Einstellungen konsumieren.
Zu der Applikation werden daher drei Dateien hinzugefügt, die im Folgenden vorgestellt werden:

#### application.properties
Die application.properties sind global gültig und gelten für alle Scopes. Daher dürfen hier nur Einstellungen enthalten sein, 
die sowohl für den produktiven Scope, als auch den Test-Scope gültig sind. Dadurch gibt es hier keine Unterschiede.
Unter Verwendung einer Postgres im Produktiv-Code, sieht die Datei dann wie folgt aus:
```
spring.jpa.properties.hibernate.default_schema={DATENBANK_SCHEMA}
spring.jpa.hibernate.ddl-auto=update
spring.datasource.driver-class-name=org.postgresql.Driver
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.PostgreSQLDialect
```
#### application-development.properties
In der application-development.properties werden Konfigurationen getroffen, die ausschließlich für die lokale Entwicklung gelten. 
Diese Einstellungen werden aktiv, indem in der IDE oder in Spring Boot der Parameter **Active Profiles** auf **development** eingestellt wird.
Auch hierbei sind keine Unterschiede aufgrund der Verwendung einer H2 Datenbank oder Testcontainers zu erkennen.
Unter Verwendung einer Postgres im Produktiv-Code, sieht die Datei wie folgt aus:
```
spring.datasource.url=jdbc:postgresql://localhost:5432/{DATENBANK_NAME}
spring.datasource.username=
spring.datasource.password=
```

#### application-test.properties
Die application-test.properties definieren die Einstellungen für den Test-Scope und sind damit nun von großer Bedeutung, wenn es um 
unterschiedliche Konfigurationen bei Verwendung einer H2 Datenbank oder Testcontainers geht. 
Damit Spring Boot diese Einstellungen nutzen kann, muss im jeweiligen Test die Annotation **@Profile(“test”)** gesetzt sein. 
Damit werden diese Properties in den Application Context geladen. 
Im Falle der Verwendung von Testcontainers bleibt diese Datei leer. Testcontainers stellt eigene Methoden zur Verfügung, 
mit denen die Einstellungen gesetzt werden können. Daher liegt ein größeres Augenmerk darauf, wie diese Datei unter Verwendung der H2 Datenbank aussieht.

Für die H2 Datenbank muss in der application-test.properties Folgendes ergänzt werden:
```
spring.datasource.driver-class-name=org.h2.Driver
spring.datasource.url=jdbc:h2:mem:exampleh2db;INIT=RUNSCRIPT FROM 'classpath:init.sql';DB_CLOSE_DELAY=-1;
spring.datasource.username=
spring.datasource.password=
spring.jpa.database-platform=org.hibernate.dialect.H2Dialect
```

Hierbei sind drei Einstellungen besonders hervorzuheben, die dem Wert von **spring.datasource.url** zugeordnet werden. 
Als Erstes wird durch **jdbc:h2:mem:{DATENBANK_NAME}** gewährleistet, dass eine In Memory H2 Datenbank angelegt wird. Der Datenbank Name ist dabei variabel.  
Des Weiteren wird im Test-Scope unter resources eine SQL-Datei manuell angelegt, das ein Schema anlegt. 
```sql
CREATE SCHEMA IF NOT EXISTS exampleh2db;
```
Da eine In-Memory Datenbank direkt nach dem Erstellen wieder verworfen wird, muss **DB_CLOSE_DELAY=-1;** gesetzt werden.
Damit stehen die Daten so lange zur Verfügung, bis der gesamte Test durchgelaufen ist und die Ergebnisse geprüft werden konnten.

### Data Source
Das Zusammenbauen der Data Source ist ebenfalls bei beiden Varianten unterschiedlich. 
Für das Verfahren mit der H2 Datenbank wurden alle Konfigurationen mit dem Befüllen der application-test.properties durchgeführt.
Bei Testcontainers wird ein anderes Verfahren gewählt. Unter Verwendung der Library, wird durch Objekte und Funktionen der passende Container mit dem gewünschten Datenbanksystem hochgefahren. Anschließend kann in diesem Container die Datasource gebaut werden.
In der Initialisierung des Containers ist wichtig, sowohl das gewünschte Docker Image, als auch die Parameter wie **Datenbankname** und **Datenbankschema** mit anzugeben. 
Das Schema kann durch Testcontainers erstellt werden, indem ein einfaches SQL Skript abhilfe schafft.  
Darüber hinaus können in diesem SQL-Skript auch weitere Voreinstellungen für die Test-Datenbank getroffen werden, abhängig davon, was benötigt wird.
```sql
CREATE SCHEMA IF NOT EXISTS exampletestcontainerdb;
```
Für das Erstellen der Datasource in Testcontainers, muss somit folgende Datei im Test-Scope angelegt werden.
```java
@Configuration
@Profile("test")
public class DbConfiguration {
private PostgreSQLContainer<?> postgreSQLContainer;
        @Bean
        public PostgreSQLContainer<?> getPostgreSQLContainer() {
            if (postgreSQLContainer == null) {
                DockerImageName postgres = DockerImageName.parse("postgres:13.1");
                postgreSQLContainer = new PostgreSQLContainer<>(postgres)
                    .withDatabaseName("exampletestcontainerdb")
                    .withInitScript("database/init.sql");
                this.postgreSQLContainer.start();
            }

            return postgreSQLContainer;
        }

        @Bean
        public DataSource dataSource(JdbcDatabaseContainer<?> postgresqlTestContainer) {
            DriverManagerDataSource dataSource = new DriverManagerDataSource();
            dataSource.setSchema("exampletestcontainerdb");
            dataSource.setDriverClassName("org.hibernate.dialect.PostgreSQLDialect");

            dataSource.setUrl(postgresqlTestContainer.getJdbcUrl());
            dataSource.setUsername(postgresqlTestContainer.getUsername());
            dataSource.setPassword(postgresqlTestContainer.getPassword());

            return dataSource;
        }
}
```

### Zusammenfassung und Fazit
Testcontainers ist ein sehr gutes Werkzeug um komplexen Code zu testen und kann dabei auf einige Vorteile verweisen.
Es müssen keine lokalen Instanzen erstellt werden und überall wo Docker läuft, kann Testcontainers verwendet werden. Die Tests sind isoliert von einander, was den großen Vorteil hat,
dass sich die Tests nicht gegenseitig beeinflussen.
Durch Testcontainers werden alle Werkzeuge für das Testen auf einer echten Datenbank bereits zur Verfügung gestellt, womit durch wenige Codezeilen ein vollständiger Test geschrieben werden kann.
Wir können den Lebenszyklus der Datenbank selber steuern und können die gleiche Version der Datenbank im Test verwenden, die wir auch im Produktiv-Code gewählt haben.
Ein großer Nachteil ist jedoch, dass für jedes Ausführen der Tests ein Dockercontainer hochgefahren werden muss. Das macht Testcontainers in vielen Fällen sehr langsam.

Gegenüber dem großen Performance-Nachteil von Testcontainers ist eine H2 - In Memory Datenbank sehr schlank und schnell.
Dennoch überwiegen hier die Nachteile.  
In unserem Fall wäre es für einen Testfall üblich, durch Integrationstests zu testen, ob Operationen mit der Datenbank richtig durchgeführt werden. 
Bei der Verwendung einer H2 kann es vorkommen, dass unser Code anders reagiert, da im Produktiv-Code ein anderes Datenbanksystem eingesetzt wurde.
Ebenso verhält es sich bei Updates. Würde aufgrund eines Updates des Datenbanksystems im Produktiv-Code ein Fehler auftreten, würde dieser nicht erkannt werden,
da eine H2 benutzt wird.
Des Weiteren erlangt eine H2 Datenbank die Schlankheit gerade durch das Minimieren von Funktionen. 
Dadurch fehlen allerdings mögliche Werkzeuge, um umfassende Tests zu schreiben, die den Code ganzheitlich überprüfen können.

Mit der Zeit hat sich meinestadt.de daher von den vielen Features von Testcontainers überzeugt und verwendet diese vor allen Dingen bei den neuen Services flächendeckend.
Auch wenn bei Testcontainers das Ausführen von Tests eine deutlich längere Zeit in Anspruch nimmt, sind die Vorteile deutlich übergewichtet.
Testcontainers bietet eine moderne Möglichkeit, komplexen Code vollständig mit Tests abzudecken und dabei die Qualität des Codes zu gewährleisten.
Eine H2 hingegen bleibt eine H2 und in den meisten Fällen eben nicht das Datenbanksystem, mit dem der Produktiv-Scope arbeitet.


> #### Quellen
> https://www.h2database.com/html/main.html  
> https://medium.com/miq-tech-and-analytics/testcontainers-the-modern-way-of-writing-database-tests-ed49554856e5  
> https://www.testcontainers.org/  
> https://www.spring-boot-blog.de/blog/spring-boot-tests-mit-h2/  
> http://thecodist.com/article/write-your-own-database-again-an-interview-with-the-author-of-h2  
