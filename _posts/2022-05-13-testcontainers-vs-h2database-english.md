---
title: Testcontainers vs. H2 Database
excerpt_separator: "<!--more-->"
categories:
  - IT 
tags:
  - Testcontainer
  - H2 Datenbank
published: true
---
![Testcontainer vs. H2 Datenbank](/assets/images/blog/002_testcontainer_vs_h2database/blog_testcontainer_vs_h2database_1.png)

A high quality of source code is indispensable with the increasing complexity of software.
The large, often neglected area of software testing provides many opportunities to increase quality.

This blog post will keep a closer look on testing with databases and focus on integration testing.
Two different methods will be compared.
On the one hand, there is testing with an in-memory H2 database and on the other hand, testing with testcontainers.

### Overview of the database systems
At meinestadt.de, we use both Testcontainers and the H2 database system to test our software.
To get a better overview, they are briefly introduced in the following section:

#### H2
In general, an H2 (Hypersonic 2) database is a relational database management system developed in Java.
This can be deployed either in Java applications directly or as a server. The open source system offers a lean in-memory solution,
so that the data does not have to be stored on the hard disk. An H2 is often called upon when the application does not require large systems with extensive functionality.
For this reason it is very popular for testing an application. It is quick and easy to set up test cases.

#### Testcontainers
Testcontainers is a Java library that allows you to programmatically create Docker images and control the lifecycle of containers.
They do not require complex configurations and are therefore easy to use. Testcontainers can map anything that can run in a Docker container.
Therefore, in many cases, the database system can be taken, which is also used in the production code.
Testcontainers provide many tools that make it very easy to write tests and add them quickly.
This makes it possible to write tests quickly and easily, since there is hardly any effort for presettings.

### Configuration
At meinestadt.de, most services are written in Java and are based on Spring Boot and Maven.
In the production code, a Postgres database is often used, which is created beforehand with a suitable schema.
Tests are written to check the various interactions with the database through the source code.
Here, different configurations are incurred depending on whether testing is done with an H2 database or Testcontainers.
These differences are explained below:

#### Dependencies
In the pom.xml the first relevant differences can be found.
For the H2 database and for Testcontainers the respective dependencies have to be added.

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
In Spring-Boot, the property files can be used to add further configurations.
These are used, so that the production scope as well as the test scope consume from different settings.
Therefore, three files are added to the application, which are presented below:

#### application.properties
The application.properties are globally valid and apply to all scopes. Therefore, only settings may be contained here,
which are valid for the productive scope as well as for the test scope. Moreover, there are no differences here.
Using a Postgres in the productive code, the file then looks like this:

```
spring.jpa.properties.hibernate.default_schema={DATABASE_SCHEMA}
spring.jpa.hibernate.ddl-auto=update
spring.datasource.driver-class-name=org.postgresql.Driver
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.PostgreSQLDialect
```
#### application-development.properties
In the application-development.properties, configurations are made, that apply exclusively to local development.
These settings become active by setting the **Active Profiles** parameter to **development** in the IDE or in Spring Boot.
Again, no differences are seen due to the use of an H2 database or Testcontainers.
Using a Postgres in production code, the file looks like this:
```
spring.datasource.url=jdbc:postgresql://localhost:5432/{DATABASE_NAME}
spring.datasource.username=
spring.datasource.password=
```

#### application-test.properties
The application-test.properties define the settings for the test scope and are therefore of great importance when it comes to
different configurations when using an H2 database or Testcontainers.
In order that Spring Boot can use these settings, the annotation **@Profile("test")** must be set in the respective test.
This loads these properties into the application context.
In case of using Testcontainers this file remains empty. Testcontainers provides its own methods,
with which the settings can be set. Therefore, more attention is paid to how this file looks when using the H2 database.

For the H2 database, the following must be added to application-test.properties:
```
spring.datasource.driver-class-name=org.h2.Driver
spring.datasource.url=jdbc:h2:mem:exampleh2db;INIT=RUNSCRIPT FROM 'classpath:init.sql';DB_CLOSE_DELAY=-1;
spring.datasource.username=
spring.datasource.password=
spring.jpa.database-platform=org.hibernate.dialect.H2Dialect
```
There are three settings in particular that are assigned to the value of **spring.datasource.url**.
First, **jdbc:h2:mem:{DATABASE_NAME}** ensures that an in-memory H2 database is created. The database name is variable.  
Furthermore, an SQL file is created manually in the test scope under resources, which creates a schema.
```sql
CREATE SCHEMA IF NOT EXISTS exampleh2db;
```
Since an in-memory database is discarded immediately after creation, **DB_CLOSE_DELAY=-1;** must be set.
This makes the data available until the entire test has run through and the results have been checked.

### Data Source
The assembly of the Data Source is also different for both variants.
For the procedure with the H2 database, all configurations were done by filling out the application-test.properties.
For Testcontainers, a different procedure is chosen. Under use of the Library, by objects and functions the suitable container with the desired data base system is started up. 
Afterwards the Datasource can be built in this container.
In the initialization of the container, it is important to specify both. The desired Docker image, as well as the parameters such as **database name** and **database schema**.
The schema can be created by Testcontainers by remedying a simple SQL script.  
In addition, other defaults for the test database can also be made in this SQL script, depending on what is needed.
```sql
CREATE SCHEMA IF NOT EXISTS exampletestcontainerdb;
```
To create the datasource in Testcontainers, the following file must be created in the test scope.
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

### Summary and conclusion
Testcontainers is a very good tool to test complex code and can point to several advantages.
There is no need to create local instances and anywhere Docker is running, Testcontainers can be used. The tests are isolated from each other, which has the great advantage,
that the tests do not influence each other.
Through Testcontainers, all the tools for testing on a real database are already provided, which means that a complete test can be written with just a few lines of code.
We can control the life cycle of the database ourselves and can use the same version of the database in the test, then that we chose in the production code.
However, a major drawback is that a Docker container needs to be booted up for each execution of the tests. This makes Testcontainers very slow in many cases.

Compared to the big performance disadvantage of Testcontainers, an H2 - In Memory database is very lean and fast.
Nevertheless, the disadvantages outweigh the advantages here.  
In our case, it would be common for a test case to use integration tests to test whether operations are performed correctly with the database.
When using an H2, our code may react differently because a different database system was used in the production code.
The same is true for updates. If an error were to occur in the production code due to an update of the database system, it would not be detected,
because an H2 is used.
Furthermore, an H2 database attains the slenderness straight by minimizing functions.
This lacks possible tools to write comprehensive tests that can check the code holistically.

Over time, meinestadt.de has convinced itself of the many good features of Testcontainers and uses them extensively, especially for the new services.
Even though running tests with Testcontainers takes a significantly longer time, the advantages are clearly outweighed.
Testcontainers offers a modern possibility to cover complex code completely with tests and guarantee the quality of the code.
An H2, on the other hand, remains an H2 and in most cases just not the database system that the production scope works with.


> #### Sources
> https://www.h2database.com/html/main.html  
> https://medium.com/miq-tech-and-analytics/testcontainers-the-modern-way-of-writing-database-tests-ed49554856e5  
> https://www.testcontainers.org/  
> https://www.spring-boot-blog.de/blog/spring-boot-tests-mit-h2/  
> http://thecodist.com/article/write-your-own-database-again-an-interview-with-the-author-of-h2  
