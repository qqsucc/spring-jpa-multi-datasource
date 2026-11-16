# Spring JPA Multi-Datasource

A minimal, copy-ready example of running **two PostgreSQL databases in one Spring Boot application** with Spring Data JPA. Each database gets its own `DataSource`, `EntityManagerFactory` and `TransactionManager`, and repositories are routed to the right database with a **marker annotation** instead of package conventions.

![Java](https://img.shields.io/badge/Java-17-orange) ![Spring Boot](https://img.shields.io/badge/Spring%20Boot-3.0-6db33f) ![PostgreSQL](https://img.shields.io/badge/PostgreSQL-2%20datasources-336791) ![Liquibase](https://img.shields.io/badge/Liquibase-migrations-2962ff)

## How it works

```text
@PrimaryDataBaseRepository ──► PrimaryDataBaseDatabaseConfig ──► primary DataSource   (@Primary)
@SecondaryRepository       ──► SecondaryDatabaseConfig       ──► secondary DataSource
```

Each config class enables JPA repositories for the whole application package but only **includes** the ones carrying its annotation:

```java
@EnableJpaRepositories(
    basePackageClasses = MultipleDbApplication.class,
    entityManagerFactoryRef = "primaryDataBaseEntityManagerFactory",
    transactionManagerRef  = "primaryDataBaseTransactionManager",
    includeFilters = @ComponentScan.Filter(
        type = FilterType.ANNOTATION, classes = PrimaryDataBaseRepository.class)
)
```

So moving a repository between databases is a one-line change:

```java
@SecondaryRepository
public interface SecondaryDBEntityRepository extends JpaRepository<SecondaryDBEntity, UUID> { }
```

The primary datasource beans are marked `@Primary`, so anything that injects an unqualified `DataSource` or `EntityManager` gets the primary database. The primary config also builds its own `SpringLiquibase` bean, so migrations run against a specific datasource rather than whatever Spring Boot autoconfigures.

## Getting started

**Requirements:** Java 17, two reachable PostgreSQL databases (Gradle wrapper included).

1. Fill in the placeholders in [`application.yml`](src/main/resources/application.yml) (or pass them as properties/environment variables):

   ```yaml
   db:
     primary:   { host: localhost:5432, database: app_main,  username: postgres, password: secret }
     secondary: { host: localhost:5433, database: app_audit, username: postgres, password: secret }
   ```

2. Run:

   ```bash
   ./gradlew bootRun
   ```

Need two throwaway databases?

```bash
docker run -d --name pg-primary   -e POSTGRES_PASSWORD=secret -e POSTGRES_DB=app_main  -p 5432:5432 postgres:16
docker run -d --name pg-secondary -e POSTGRES_PASSWORD=secret -e POSTGRES_DB=app_audit -p 5433:5432 postgres:16
```

## Transactions

With more than one `TransactionManager`, name the one you want in `@Transactional`; otherwise the `@Primary` one is used, which silently does nothing for the secondary database.

```java
@Transactional("primaryDataBaseTransactionManager")
public void deleteById(UUID id) {
    primaryDBEntityRepository.deleteById(id);
}

@Transactional("secondaryDataBaseTransactionManager")
public void deleteSecondaryById(UUID id) {
    secondaryDBEntityRepository.deleteById(id);
}
```

A single `@Transactional` method cannot atomically span both databases. For that you need XA/JTA or an outbox/saga pattern.

## Project layout

```text
src/main/java/com/shutuper/springdatajpamultipledb
├── config/db
│   ├── PrimaryDataBaseDatabaseConfig.java   # primary DataSource, EMF, TM, Liquibase
│   ├── SecondaryDatabaseConfig.java         # secondary DataSource, EMF, TM
│   └── filter/                              # @PrimaryDataBaseRepository, @SecondaryRepository
└── example                                  # entities, repositories, services per DB
src/main/resources
├── application.yml
└── changelog/                               # Liquibase changelogs
```

## Tech stack

Spring Boot 3.0 · Spring Data JPA · Hibernate · Liquibase · PostgreSQL · Lombok · Gradle
