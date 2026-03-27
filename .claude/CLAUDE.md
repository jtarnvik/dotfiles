# Global Coding Preferences

## About me

I have been a programmer for more than 30 years, the last 20 of them programming in Java. I mostly work with
microservices using Spring Boot. I have also some experience with writing React web applications.
I have been creating web applications for 7 years using React and TypeScript for about 50% of my
time at work. But I have probably approached the React/TypeScript worlds from the Java/Spring Boot
way of doing things so it is very possible I write web apps more in a Java way. I don't mind pointers
when that is the case.

I work in an enterprise environment and my personal projects are mostly for my own use, including for a few friends.
I want to play around with newer technologies and experiment with programming-for-fun.
This is also an attempt at learning to code with AI assistance. I am quite convinced that I could solve all
issues on my own, but it would take some time and effort.

## Code Style

- Prefer readability over brevity.
- Prefer maintainability over performance.
- Prefer simplicity over complexity.
- If statements and all other control flow constructs should use curly braces even if they are followed by just a single statement.
- I prefer the imports section sorted in the following order:
  - React (for frontend projects)
  - Third-party
  - Local
  - css files

## File Management

When a new file is created and the user has approved it, stage it in git with `git add <file>`.

## This file

After making any edit to this file, immediately stage, commit, and push it:
```bash
cd ~/dotfiles && git add .claude/CLAUDE.md && git commit -m "Update global Claude preferences" && git push
```

## AI Collaboration

- Implement only the current step unless told otherwise.
- Future steps may be read for context and direction, but treat them as provisional — they may change.
- Where a preferred approach is given, use it unless there is a clearly better alternative — in that case, propose it before implementing.
- If the current step requires an implementation decision that a future step would influence, use that step as guidance.

## Java Style Guide

The base package is defined per-project as `{{BASE_PACKAGE}}`.

### Dependency injection
Prefer constructor injection over field injection. Injected dependencies should be declared `private final`. For `@Configuration` classes that also inject `@Value` properties, annotate the `@Value` parameters directly on the constructor parameters.

### Lombok
Prefer lombok annotations over explicit accessors and constructors.

### Scheduled Jobs

Classes containing `@Scheduled` methods are treated as incoming triggers and live in `{{BASE_PACKAGE}}.port.incoming.scheduled`, on the same level as `port/incoming/rest`. Annotate with `@Component` and use `@Transactional` on methods that modify the database. Use cron expressions for time-based scheduling (e.g. `"0 0 0 * * *"` for midnight daily).

### Controller tests

Use `@WebMvcTest` for controller-layer tests. Spring Boot 4 splits security autoconfiguration across more modules than Boot 3 — to fully disable security in a `@WebMvcTest` slice, exclude all five classes:

```java
@WebMvcTest(controllers = MyController.class,
            excludeAutoConfiguration = {
              SecurityAutoConfiguration.class,
              SecurityFilterAutoConfiguration.class,
              ServletWebSecurityAutoConfiguration.class,
              OAuth2ClientAutoConfiguration.class,
              OAuth2ClientWebSecurityAutoConfiguration.class
            })
```

Tests that need security (e.g. verifying that an endpoint requires a role) should instead use `@WithMockUser` and not exclude security.

### REST Controllers
Shall be concerned with verifying parameters. Lives in the package `{{BASE_PACKAGE}}.port.incoming.rest`.

Use Bean Validation for parameter validation — annotate DTOs with `@NotBlank`, `@NotNull`, etc. and use `@Valid` on `@RequestBody` parameters. Do not use manual null/blank checks in controller methods for input that can be validated this way. The `spring-boot-starter-validation` dependency is included for this purpose.

Admin-only endpoints must be annotated with `@PreAuthorize("hasRole('ADMIN')")`. `@EnableMethodSecurity` is already enabled in `SecurityConfig`.

### DTO naming convention

DTOs are named with a `Request` or `Response` suffix based on direction:
- Inbound (request body): `XxxRequest`
- Outbound (response body): `XxxResponse`

Exception: if the `Request` suffix produces a confusing or redundant name (e.g. `AccessRequestRequest`), use `Dto` suffix instead.

DTOs live in `{{BASE_PACKAGE}}.port.incoming.rest.dto`, not as inner classes in controllers.

### Services
Shall be concerned with business logic. Typically called from restcontroller and other services.
Lives in the package `{{BASE_PACKAGE}}.service`.

### Entity-to-DTO mapping (MapStruct)

Use MapStruct for all entity-to-DTO conversions. Do not write manual field-by-field mapping in controllers or services.

- Mappers live in `{{BASE_PACKAGE}}.port.incoming.rest.mapper`
- Declare mappers as `interface` (not abstract class)
- MapStruct is wired as a Spring bean via `componentModel = "spring"` — inject mappers with constructor injection like any other bean
- For custom type conversions (e.g. `LocalDateTime` → `String`), use a `@Component` converter class with a method annotated with a MapStruct `@Qualifier` annotation, and reference it via `uses` + `qualifiedBy`. **Do not use `expression = "java(...)"` — it embeds Java code in a string and breaks IDE refactoring.**
- `DateConverter` (in the mapper package) handles `LocalDateTime → String` formatting using the `@DateFormat` qualifier. Use it like this:
  ```java
  @Mapper(componentModel = "spring", uses = DateConverter.class)
  public interface FooMapper {
    @Mapping(target = "createDate", source = "createDate", qualifiedBy = DateFormat.class)
    FooResponse toResponse(Foo foo);
  }
  ```
- Annotation processor ordering: Lombok runs before MapStruct via `lombok-mapstruct-binding` in the maven-compiler-plugin `annotationProcessorPaths`. Do not reorder these entries.

### Requests to other systems
Shall be named Providers. Typically calls to the Qwerty API, implemented in the QwertyProvider class.
Providers are sorted by first type and then service name. For example, a REST provider to the Qwerty service lives in
`{{BASE_PACKAGE}}.port.outgoing.rest.qwerty`. A Kafka provider to the Foo service would be
`{{BASE_PACKAGE}}.port.outgoing.kafka.foo`. Providers should hide as much implementation detail as possible,
and only expose the logical API.

## Database Development

### Timestamp columns
All tables should have the following two columns.

```java
@CreationTimestamp
@Column(name = "create_date")
private LocalDateTime createDate;

@UpdateTimestamp
@Column(name = "latest_update")
private LocalDateTime latestUpdate;
```
The only exceptions are tables matching concepts owned by outside libraries, e.g. `spring_session`.

### Package structure

```
Base package: {{BASE_PACKAGE}}
model/
  domain/
    entity/     // Database entities
    repository/ // Repositories
    dao/        // Rarely used, but if used, they live here
```

### Entity IDs
A typical entity id should be a long, generated as such:
```java
@TableGenerator(name = "id_generator_<TABLE_NAME>", table = "id_gen", pkColumnName = "gen_name", valueColumnName = "gen_value",
pkColumnValue = "<TABLE_NAME>_gen", initialValue = 10000, allocationSize = 1)
@GeneratedValue(strategy = GenerationType.TABLE, generator = "id_generator_<TABLE_NAME>")
@Column(name = "id")
@Id
private Long id;
```

Every new entity that uses `@TableGenerator` must also have a corresponding row seeded in `id_gen` via a Liquibase changeset. The `id_gen` table has no auto-increment on its own primary key, so Hibernate cannot insert the row itself. Use `gen_value = initialValue` (typically 10000). Pick the next available integer for the `id` column.

```xml
<insert tableName="id_gen">
    <column name="id" valueNumeric="<NEXT_INT>"/>
    <column name="gen_name" value="<TABLE_NAME>_gen"/>
    <column name="gen_value" valueNumeric="10000"/>
</insert>
```
Tables should normally have an id column.

### Repositories
Prefer JpaRepository over CrudRepository.
