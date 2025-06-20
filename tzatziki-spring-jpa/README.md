Tzatziki Spring JPA Library
======

## Description

This module provides the base dependencies to start testing your Spring App if you use JPA

## Get started with this module

You need to add this dependency to your project:

```xml
<dependency>
    <groupId>com.decathlon.tzatziki</groupId>
    <artifactId>tzatziki-spring-jpa</artifactId>
    <version>1.0.x</version>
    <scope>test</scope>
</dependency>
```

Please note that if you are using JSONB in your entities, you need to exclude a transitive dependency depending on which persistence API you're using in your application.
If using Java Persistence (@Table is taken from javax.persistence):
```xml
<exclusions>
    <exclusion>
        <artifactId>jackson-datatype-hibernate5-jakarta</artifactId>
        <groupId>com.fasterxml.jackson.datatype</groupId>
    </exclusion>
</exclusions>
```
If using Jakarta (@Table is taken from jakarta.persistence):
```xml
<exclusions>
    <exclusion>
        <artifactId>jackson-datatype-hibernate5</artifactId>
        <groupId>com.fasterxml.jackson.datatype</groupId>
    </exclusion>
</exclusions>
```
It will prevent com.vladmihalcea.hibernate-types-* / io.hypersistence.hypersistence-utils-hibernate-* mapper from having conflict on which Module to use for (de)serialization

## Adding the datasource configuration

we will assume that you followed the [readme from the spring module](https://github.com/Decathlon/tzatziki/tree/main/tzatziki-spring)

The only thing you need to do is to add the test container instance and the datasource configuration to your test Steps.

```java
@CucumberContextConfiguration
@SpringBootTest(webEnvironment = RANDOM_PORT, classes = Application.class)
@ContextConfiguration(initializers = ApplicationSteps.Initializer.class)
public class ApplicationSteps {

    private static final PostgreSQLContainer<?> postgres =
            new PostgreSQLContainer<>("postgres:12").withTmpFs(Maps.of("/var/lib/postgresql/data", "rw"));

    static class Initializer implements ApplicationContextInitializer<ConfigurableApplicationContext> {

        public void initialize(ConfigurableApplicationContext configurableApplicationContext) {
            postgres.start();
            TestPropertyValues.of(
                    "spring.datasource.url=" + postgres.getJdbcUrl(),
                    "spring.datasource.username=" + postgres.getUsername(),
                    "spring.datasource.password=" + postgres.getPassword()
            ).applyTo(configurableApplicationContext.getEnvironment());
        }
    }
}
```

*Note: The code mostly uses Spring classes to manipulate the database so this should work with most vendors.*

## Inserting data

You will need a CrudRespository<YourEntity> for it work. If you don't have one for your production code,
you can still create one locally for your test code, this will work just fine. 

Assuming the following entity class:
```java
@NoArgsConstructor
@Getter
@Entity
@Table(name = "users")
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    Integer id;

    @Column(name = "first_name")
    String firstName;

    @Column(name = "last_name")
    String lastName;
}
```

You can now insert rows with:
```gherkin
Given that the users table will contain:
  | firstName | lastName |
  | Darth     | Vader    |

# or alternatively
Given that the User entities will contain:
  """
  - firstName: Darth
    lastName: Vader
  """
# single rows work as well  
Given that the users table will contain:
  """
  firstName: Darth
  lastName: Vader
  """
# or Json  
Given that the UserRepository repository will contain:
  """
  [
    {
      "firstName": "Darth",
      "lastName": "Vader"
    }
  ]
  """
```

Adding `only` to the step will also empty the table before inserting the data:
```gherkin
But when the users table will contain only:
  | firstName | lastName |
  | Han       | Solo     |
```

If your entity contains generated data like IDs, creation dates ..., you can get back the data you just inserted in its new state in a variable, using the following steps:

**Using repository**
```gherkin
Given that the UserDataSpringRepository repository will contain the following data available in users_list:
"""yml
      - firstName: Darth
        lastName: Vader
      """

Then users_list contains:
"""
    - id: 1
      firstName: Darth
      lastName: Vader
    """
```
**Using table**
```gherkin
Given that the users table will contain the following data available in users_list:
"""yaml
      - firstName: Darth
        lastName: Vader
      """

Then users_list contains:
"""yaml
    - id: 1
      firstName: Darth
      lastName: Vader
    """

```
**Using entities**
```gherkin
Given that the User entities will contain the following data available in users_list:
"""yaml
      - firstName: Darth
        lastName: Vader
      """

Then users_list contains:
"""yaml
    - id: 1
      firstName: Darth
      lastName: Vader
    """
```

### Important Note on Generated IDs

Starting with Hibernate 6.6+ and Spring Boot 3.4+, you can no longer manually specify values for IDs that are generated by the framework.

#### Example
Suppose you have an entity mapped as follows:

```java
@Getter
@Entity
@NoArgsConstructor
@Table(name = "users")
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Integer id;

    @Column(name = "first_name")
    private String firstName;
}
```

In this example, Hibernate **will not allow you** to insert data with explicit `id` values like this:

```gherkin
Given the users table will contain:
 | id | firstName | lastName |
 | 42 | Darth     | Vader    |
```

The test above fails with the following exception:

```
org.springframework.orm.ObjectOptimisticLockingFailureException: Row was updated or deleted by another transaction (or unsaved-value mapping was incorrect)
```
This exception occurs because Hibernate expects the database to assign the primary key when `GenerationType.IDENTITY` is used. 
Providing an explicit `id` value conflicts with Hibernate’s assumptions, leading to a failure.

#### Correct Approach

To avoid this issue, ensure that the input data does not include the `id` column, as shown below:

```gherkin
Given the users table will contain:
 | firstName | lastName |
 | Darth     | Vader    |
```

### Inserting Data into Parent and Child Tables
When working with parent and child tables where the child table has a foreign key referencing the parent table, you may need to ensure correct key mapping during data insertion.

When the primary key of the parent table is auto-generated, you may not know its exact value at the time of insertion into the child table. 

However, since the primary key values in the parent table follow a predictable sequence starting at 1, you can infer the foreign key values.

This behavior is consistent because Tzatziki resets the sequence at the beginning of each scenario.

The following code demonstrates an example where the `User` entity references the `Group` entity:

```java
@NoArgsConstructor
@Getter
@Entity
@Table(name = "groups")
public class Group {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    Integer id;

    @Column(name = "name")
    String name;

    @OneToMany(mappedBy = "group")
    List<User> users;
}
```
```java
@NoArgsConstructor
@Getter
@Entity
@Table(name = "users")
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    Integer id;

    @Column(name = "first_name")
    String firstName;

    @Column(name = "last_name")
    String lastName;

    @ManyToOne
    @JoinColumn(name = "group_id")
    Group group;
}

```
The example below inserts the data properly:

```gherkin
    Given that the groups table will contain:
      | name   |
      | admins |
      | guests |
    And the users table will contain:
      | firstName | lastName | group.id |
      | Chuck     | Norris   | 1        |
      | Uma       | Thurman  | 2        |
      | Jackie    | Chan     | 2        |
```

## Asserting data

You can assert the content of the database with:
```gherkin
Then the users table contains:
  | id | firstName | lastName |
  | 1  | Darth     | Vader    |

# or alternatively
Then the User entities contains:
  """
  id: 1
  firstName: Darth
  lastName: Vader
  """
# many rows, in Json or Yaml or Table, like for the insert.
Then the UserRepository repository contains:
  ...
```

To assert that a table is empty:
```gherkin
Then the users table contains nothing
```

Consistent with the other steps in the Core and Http module, you can assert that the table contains `only` `exactly` 
or `at least` and possibly `in order` the given rows, as a table, a YAML or a Json.

The library will also attempt to look up a table, an entity or a repository matching your step. 

By default, the triggers will be disabled when you insert your rows, so that you can specify even the generated values.
However, if you wish you can re-enable them with:

```gherkin
But if the triggers are enabled
```

## Getting data

You can get the content of the database with:
```gherkin
Then content is the users table content

# or alternatively
Then content is the User entities
```

The `content` variable created will be a List created from `org.springframework.data.repository.CrudRepository#findAll` 

For both you can order the List fetched from the database (direction is not mandatory and defaults to ascending)
```gherkin
Then content is the users table content ordered by date_of_birth desc and date_of_death

# or alternatively
Then content is the User entities ordered by date_of_birth desc and date_of_death
```

The `content` variable created will be a List created from `org.springframework.data.jpa.repository.JpaRepository#findAll(Sort)`

## Resetting the database between tests

The library will automatically reset the database between each test. 
It will run `TRUNCATE <table> RESTART IDENTITY CASCADE` for each table and re-enable the triggers if you have disabled them.
If you need more, feel free to autowire the DataSource in your local steps. 

if you do not want to reset the database between the tests, or you prefer to do it yourself, you can alter this with:

```java
SpringJPASteps.autoclean = false; // to turn it off completely
SpringJPASteps.schemasToClean = List.of("your-schema1","your-schema2"); // the default is 'public'
DatabaseCleaner.addToTablesNotToBeCleaned("config"); // this will prevent the config table to be cleaned
DatabaseCleaner.resetTablesNotToBeCleanedFilter(); // to purge the previously added tables
```

## Handling Bidirectional JPA Relationships

If you are using `tzatziki-spring-jpa` with a bidirectional mapping between two JPA entities, you may encounter issues such as infinite recursion or serialization problems.
 
Please refer to the [hint on handling bidirectional relationships in the tzatziki-core README](../tzatziki-core/README.md#bidirectional-relationships) for guidance on how to address these issues.

# More examples

For more examples you can have a look at the tests:
https://github.com/Decathlon/tzatziki/blob/main/tzatziki-spring-jpa/src/test/resources/com/decathlon/tzatziki/steps/spring-jpa.feature
