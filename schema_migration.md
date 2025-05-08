# Schema Migration

## Schema Migration with Flyway

Flyway é uma ferramenta de migração de banco de dados de código aberto. Ela prioriza a simplicidade e a convenção em detrimento da configuração.

### Entidade

```java
package example.micronaut;

import io.micronaut.core.annotation.NonNull;
import io.micronaut.core.annotation.Nullable;
import io.micronaut.data.annotation.GeneratedValue;
import io.micronaut.data.annotation.Id;
import io.micronaut.data.annotation.MappedEntity;
import io.micronaut.data.annotation.Version;

import jakarta.validation.constraints.NotBlank;

@MappedEntity
public class Person {

    @Id
    @GeneratedValue
    private Long id;

    @Version
    private Long version;

    @NonNull
    @NotBlank
    private final String name;

    private final int age;

    public Person(@NonNull String name, int age) {
        this.name = name;
        this.age = age;
    }

    public int getAge() {
        return age;
    }


    @NonNull
    public String getName() {
        return name;
    }

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public Long getVersion() {
        return version;
    }

    public void setVersion(Long version) {
        this.version = version;
    }
}

```

### Migração de banco de dados com Flyway

```xml
<dependency>
    <groupId>io.micronaut.flyway</groupId>
    <artifactId>micronaut-flyway</artifactId>
    <scope>compile</scope>
</dependency>
```

```yml
flyway:
  datasources:
	default:
	  enabled: true
	  locations: classpath:db/migration
```
A migração do Flyway será acionada automaticamente antes do início do seu aplicativo Micronaut. O Flyway lerá os comandos de migração no resources/db/migration/diretório, os executará se necessário e verificará se a fonte de dados configurada é consistente com eles.

```sql

CREATE TABLE person(
    id   bigint primary key not null,
    name varchar(255)       not null,
    age  int                not null
)

```

#### Aplicações mudando o esquema

```java
    @Nullable
    private final Integer age;

    public Person(@NonNull String name,
                  @Nullable Integer age) {
        this.name = name;
        this.age = age;
    }

    @Nullable
    public Integer getAge() {
        return age;
    }
```
Adicione uma nova migração para remover a restrição nula:

```sql
ALTER TABLE person ALTER COLUMN age DROP NOT NULL;
```


### Flyway endpoint

Para habilitar o ponto de extremidade Flyway, adicione a managementdependência no seu classpath.

```xml

<dependency>
    <groupId>io.micronaut</groupId>
    <artifactId>micronaut-management</artifactId>
    <scope>compile</scope>
</dependency>

```

```yml
endpoints:
  flyway:
	enabled: true
	sensitive: false
```

```java
package example.micronaut;

import io.micronaut.core.type.Argument;
import io.micronaut.http.HttpRequest;
import io.micronaut.http.HttpResponse;
import io.micronaut.http.client.BlockingHttpClient;
import io.micronaut.http.client.HttpClient;
import io.micronaut.http.client.annotation.Client;
import io.micronaut.test.extensions.junit5.annotation.MicronautTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.util.List;

import static io.micronaut.http.HttpStatus.OK;
import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertNotNull;

@MicronautTest
public class FlywayEndpointTest {

    @Inject
    @Client("/")
    HttpClient httpClient;

    @Test
    void migrationsAreExposedViaAndEndpoint() {
        BlockingHttpClient client = httpClient.toBlocking();

        HttpResponse<List<FlywayReport>> response = client.exchange(
                HttpRequest.GET("/flyway"),
                Argument.listOf(FlywayReport.class));
        assertEquals(OK, response.status());

        List<FlywayReport> flywayReports = response.body();
        assertNotNull(flywayReports);
        assertEquals(1, flywayReports.size());

        FlywayReport flywayReport = flywayReports.get(0);
        assertNotNull(flywayReport);
        assertNotNull(flywayReport.getMigrations());
        assertEquals(2, flywayReport.getMigrations().size());
    }

    static class FlywayReport {
        private List<Migration> migrations;

        public void setMigrations(List<Migration> migrations) {
            this.migrations = migrations;
        }

        public List<Migration> getMigrations() {
            return migrations;
        }
    }

    static class Migration {
        private String script;

        public void setId(String script) {
            this.script = script;
        }

        public String getScript() {
            return script;
        }
    }
}

```
