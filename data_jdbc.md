# Data JDBC

## [Access a database with Micronaut Data JDBC](https://guides.micronaut.io/latest/micronaut-data-jdbc-repository-maven-java.html)

O aplicativo expõe alguns endpoints REST e armazena dados em um banco de dados MySQL usando o Micronaut Data JDBC.


### Configuração do banco de dados

```yaml
datasources:
  default:
	dialect: MYSQL
	driver-class-name: com.mysql.cj.jdbc.Driver
	url: jdbc:mysql://localhost:3306/micronaut
	username: root
	password: root
```

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


```sql
-- src/main/resources/db/migration/V1__create_person_table.sql
DROP TABLE IF EXISTS genre;

CREATE TABLE genre (
    id   BIGINT NOT NULL AUTO_INCREMENT UNIQUE PRIMARY KEY,
   name  VARCHAR(255) NOT NULL UNIQUE
);
```

### Entidade

```java
package example.micronaut.domain;

import io.micronaut.data.annotation.GeneratedValue;
import io.micronaut.data.annotation.Id;
import io.micronaut.data.annotation.MappedEntity;
import io.micronaut.serde.annotation.Serdeable;

import jakarta.validation.constraints.NotNull;

@Serdeable
@MappedEntity
public class Genre {

    @Id
    @GeneratedValue(GeneratedValue.Type.AUTO)
    private Long id;

    @NotNull
    private String name;

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    @Override
    public String toString() {
        return "Genre{" +
                "id=" + id +
                ", name='" + name + '\'' +
                '}';
    }
}

```

### Repositório

```java
package example.micronaut;

import example.micronaut.domain.Genre;
import io.micronaut.core.annotation.NonNull;
import io.micronaut.data.annotation.Id;
import io.micronaut.data.exceptions.DataAccessException;
import io.micronaut.data.jdbc.annotation.JdbcRepository;
import io.micronaut.data.model.query.builder.sql.Dialect;
import io.micronaut.data.repository.PageableRepository;

import jakarta.transaction.Transactional;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;

@JdbcRepository(dialect = Dialect.MYSQL)
public interface GenreRepository extends PageableRepository<Genre, Long> {

    Genre save(@NonNull @NotBlank String name);

    @Transactional
    default Genre saveWithException(@NonNull @NotBlank String name) {
        save(name);
        throw new DataAccessException("test exception");
    }

    long update(@NonNull @NotNull @Id Long id, @NonNull @NotBlank String name);
}
```

![alt text](image-4.png)


### Controlador

A validação do Micronaut é baseada na estrutura padrão JSR 380 , também conhecida como Validação de Beans 2.0. A Validação do Micronaut possui suporte integrado para validação de beans anotados com jakarta.validationanotações.

Para usar o Micronaut Validation, você precisa das seguintes dependências:

```xml
<!-- Add the following to your annotationProcessorPaths element -->
<path>
    <groupId>io.micronaut.validation</groupId>
    <artifactId>micronaut-validation-processor</artifactId>
</path>
<dependency>
    <groupId>io.micronaut.validation</groupId>
    <artifactId>micronaut-validation</artifactId>
    <scope>compile</scope>
</dependency>

```

```java

package example.micronaut;

import io.micronaut.serde.annotation.Serdeable;

import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;

@Serdeable
public class GenreUpdateCommand {
    @NotNull
    private final Long id;

    @NotBlank
    private final String name;

    public GenreUpdateCommand(Long id, String name) {
        this.id = id;
        this.name = name;
    }

    public Long getId() {
        return id;
    }

    public String getName() {
        return name;
    }

}

```

#### Controlador

```java

package example.micronaut;

import example.micronaut.domain.Genre;
import io.micronaut.data.exceptions.DataAccessException;
import io.micronaut.data.model.Pageable;
import io.micronaut.http.HttpHeaders;
import io.micronaut.http.HttpResponse;
import io.micronaut.http.HttpStatus;
import io.micronaut.http.annotation.Body;
import io.micronaut.http.annotation.Controller;
import io.micronaut.http.annotation.Delete;
import io.micronaut.http.annotation.Get;
import io.micronaut.http.annotation.Post;
import io.micronaut.http.annotation.Put;
import io.micronaut.http.annotation.Status;
import io.micronaut.scheduling.TaskExecutors;
import io.micronaut.scheduling.annotation.ExecuteOn;

import jakarta.validation.Valid;
import jakarta.validation.constraints.NotBlank;
import java.net.URI;
import java.util.List;
import java.util.Optional;

@ExecuteOn(TaskExecutors.BLOCKING)
@Controller("/genres")
public class GenreController {

    protected final GenreRepository genreRepository;

    public GenreController(GenreRepository genreRepository) {
        this.genreRepository = genreRepository;
    }

    @Get("/{id}")
    public Optional<Genre> show(Long id) {
        return genreRepository
                .findById(id);
    }

    @Put
    public HttpResponse update(@Body @Valid GenreUpdateCommand command) {
        genreRepository.update(command.getId(), command.getName());
        return HttpResponse
                .noContent()
                .header(HttpHeaders.LOCATION, location(command.getId()).getPath());
    }

    @Get("/list")
    public List<Genre> list(@Valid Pageable pageable) {
        return genreRepository.findAll(pageable).getContent();
    }

    @Post
    public HttpResponse<Genre> save(@Body("name") @NotBlank String name) {
        Genre genre = genreRepository.save(name);

        return HttpResponse
                .created(genre)
                .headers(headers -> headers.location(location(genre.getId())));
    }

    @Post("/ex")
    public HttpResponse<Genre> saveExceptions(@Body @NotBlank String name) {
        try {
            Genre genre = genreRepository.saveWithException(name);
            return HttpResponse
                    .created(genre)
                    .headers(headers -> headers.location(location(genre.getId())));
        } catch(DataAccessException e) {
            return HttpResponse.noContent();
        }
    }

    @Delete("/{id}")
    @Status(HttpStatus.NO_CONTENT)
    public void delete(Long id) {
        genreRepository.deleteById(id);
    }

    protected URI location(Long id) {
        return URI.create("/genres/" + id);
    }

    protected URI location(Genre genre) {
        return location(genre.getId());
    }
}

```

1. **É essencial que quaisquer operações de E/S de bloqueio** (como buscar dados do banco de dados) sejam descarregadas para um pool de threads separado que não bloqueie o loop de eventos.

2. **A classe é definida como um controlador** com a anotação [@Controller](https://docs.micronaut.io/latest/api/io/micronaut/http/annotation/Controller.html) mapeada para o caminho `/genres`.

3. **Use injeção de construtor** para injetar um bean do tipo `GenreRepository`.

4. **Mapeia uma `GET` solicitação para `/genres/{id}`**, que tenta mostrar um gênero. Isso ilustra o uso de uma variável de caminho de URL.

5. **Retornar um opcional vazio** quando o gênero não existe faz com que o framework Micronaut responda com 404 (não encontrado).

6. **Mapeia uma `PUT` solicitação para `/genres`**, que tenta atualizar um gênero.

7. **Adiciona `@Valid`** a qualquer parâmetro do método que requeira validação. Use um POJO fornecido como carga JSON na solicitação para preencher o comando.

8. **É fácil adicionar cabeçalhos personalizados** à resposta.

9. **Mapeia uma `GET` solicitação para `/genres/list`**, que retorna uma lista de gêneros. Este mapeamento ilustra parâmetros de URL sendo mapeados para um único POJO.

10. **Você pode vincular `Pageable`** como um argumento de método do controlador. Confira os exemplos na seção de testes a seguir e leia as opções [de configuração do Pageable](https://micronaut-projects.github.io/micronaut-data/latest/guide/configurationreference.html#io.micronaut.data.runtime.config.DataConfiguration.PageableConfiguration). Por exemplo, você pode configurar o tamanho padrão da página com a propriedade de configuração `micronaut.data.pageable.default-page-size`.

11. **Mapeia uma `POST` solicitação para `/genres`**, que tenta salvar um gênero.

12. **Mapeia uma `POST` solicitação para `/ex`**, o que gera uma exceção.

13. **Mapeia uma `DELETE` solicitação para `/genres/{id}`**, que tenta remover um gênero. Isso ilustra o uso de uma variável de caminho de URL.


### Testes

```java
package example.micronaut;

import example.micronaut.domain.Genre;
import io.micronaut.core.type.Argument;
import io.micronaut.http.HttpHeaders;
import io.micronaut.http.HttpRequest;
import io.micronaut.http.HttpResponse;
import io.micronaut.http.HttpStatus;
import io.micronaut.http.client.HttpClient;
import io.micronaut.http.client.annotation.Client;
import io.micronaut.http.client.exceptions.HttpClientResponseException;
import io.micronaut.test.extensions.junit5.annotation.MicronautTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertNotNull;
import static org.junit.jupiter.api.Assertions.assertThrows;

@MicronautTest
public class GenreControllerTest {

    @Inject
    @Client("/")
    HttpClient client;

    @Test
    public void testFindNonExistingGenreReturns404() {
        HttpClientResponseException thrown = assertThrows(HttpClientResponseException.class, () -> {
            client.toBlocking().exchange(HttpRequest.GET("/genres/99"));
        });

        assertNotNull(thrown.getResponse());
        assertEquals(HttpStatus.NOT_FOUND, thrown.getStatus());
    }

    @Test
    public void testGenreCrudOperations() {

        List<Long> genreIds = new ArrayList<>();

        HttpRequest<?> request = HttpRequest.POST("/genres", Collections.singletonMap("name", "DevOps"));
        HttpResponse<?> response = client.toBlocking().exchange(request);
        genreIds.add(entityId(response));

        assertEquals(HttpStatus.CREATED, response.getStatus());

        request = HttpRequest.POST("/genres", Collections.singletonMap("name", "Microservices"));
        response = client.toBlocking().exchange(request);

        assertEquals(HttpStatus.CREATED, response.getStatus());

        Long id = entityId(response);
        genreIds.add(id);
        request = HttpRequest.GET("/genres/" + id);

        Genre genre = client.toBlocking().retrieve(request, Genre.class);

        assertEquals("Microservices", genre.getName());

        request = HttpRequest.PUT("/genres", new GenreUpdateCommand(id, "Micro-services"));
        response = client.toBlocking().exchange(request);

        assertEquals(HttpStatus.NO_CONTENT, response.getStatus());

        request = HttpRequest.GET("/genres/" + id);
        genre = client.toBlocking().retrieve(request, Genre.class);
        assertEquals("Micro-services", genre.getName());

        request = HttpRequest.GET("/genres/list");
        List<Genre> genres = client.toBlocking().retrieve(request, Argument.of(List.class, Genre.class));

        assertEquals(2, genres.size());

        request = HttpRequest.POST("/genres/ex", Collections.singletonMap("name", "Microservices"));
        response = client.toBlocking().exchange(request);

        assertEquals(HttpStatus.NO_CONTENT, response.getStatus());

        request = HttpRequest.GET("/genres/list");
        genres = client.toBlocking().retrieve(request, Argument.of(List.class, Genre.class));

        assertEquals(2, genres.size());

        request = HttpRequest.GET("/genres/list?size=1");
        genres = client.toBlocking().retrieve(request, Argument.of(List.class, Genre.class));

        assertEquals(1, genres.size());
        assertEquals("DevOps", genres.get(0).getName());

        request = HttpRequest.GET("/genres/list?size=1&sort=name,desc");
        genres = client.toBlocking().retrieve(request, Argument.of(List.class, Genre.class));

        assertEquals(1, genres.size());
        assertEquals("Micro-services", genres.get(0).getName());

        request = HttpRequest.GET("/genres/list?size=1&page=2");
        genres = client.toBlocking().retrieve(request, Argument.of(List.class, Genre.class));

        assertEquals(0, genres.size());

        // cleanup:
        for (Long genreId : genreIds) {
            request = HttpRequest.DELETE("/genres/" + genreId);
            response = client.toBlocking().exchange(request);
            assertEquals(HttpStatus.NO_CONTENT, response.getStatus());
        }
    }

    protected Long entityId(HttpResponse<?> response) {
        String path = "/genres/";
        String value = response.header(HttpHeaders.LOCATION);
        if (value == null) {
            return null;
        }
        int index = value.indexOf(path);
        if (index != -1) {
            return Long.valueOf(value.substring(index + path.length()));
        }
        return null;
    }
}

```

## [Micronaut Data and Java Records](https://guides.micronaut.io/latest/micronaut-java-records-maven-java.html)

Aprenda como aproveitar os registros Java para configuração imutável, entidades mapeadas de dados Micronaut e DTOs de projeção


As classes de registro, que são um tipo especial de classe, ajudam a modelar agregados de dados simples com menos cerimônia do que as classes normais.

Uma declaração de registro especifica em um cabeçalho uma descrição de seu conteúdo; os métodos de acesso apropriados, construtor, equals, hashCode e toString são criados automaticamente. Os campos de um registro são finais porque a classe se destina a servir como um simples "portador de dados".

### Exemplo de registro

```java
package example.micronaut;

import io.micronaut.context.annotation.ConfigurationProperties;
import io.micronaut.core.annotation.NonNull;
import jakarta.validation.constraints.NotNull;
import java.math.BigDecimal;

@ConfigurationProperties("vat") //recuperar propriedades de configuração do arquivo application.yml
public record ValueAddedTaxConfiguration(
    @NonNull @NotNull BigDecimal percentage) {
}
```

```java

package example.micronaut;

import io.micronaut.context.BeanContext;
import io.micronaut.context.annotation.Property;
import io.micronaut.test.extensions.junit5.annotation.MicronautTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;
import java.math.BigDecimal;
import static org.junit.jupiter.api.Assertions.assertTrue;
import static org.junit.jupiter.api.Assertions.assertEquals;

@Property(name = "vat.percentage", value = "21.0")
@MicronautTest(startApplication = false)
class ValueAddedTaxConfigurationTest {

    @Inject
    BeanContext beanContext;

    @Test
    void immutableConfigurationViaJavaRecords() {
        assertTrue(beanContext.containsBean(ValueAddedTaxConfiguration.class));
        assertEquals(new BigDecimal("21.0"),
                beanContext.getBean(ValueAddedTaxConfiguration.class).percentage());
    }
}

```

## Migração de banco de dados com Liquibase

```xml
<dependency>
    <groupId>io.micronaut.liquibase</groupId>
    <artifactId>micronaut-liquibase</artifactId>
    <scope>compile</scope>
</dependency
```

```yaml
liquibase:
  enabled: true
  datasources:
    default:
      change-log: 'classpath:db/liquibase-changelog.xml'
```

```xml
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog
        xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
         http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-3.1.xsd">
    <include file="changelog/01-create-books-schema.xml" relativeToChangelogFile="true"/>
    <include file="changelog/02-insert-book.xml" relativeToChangelogFile="true"/>
</databaseChangeLog>
```

```xml
<?xml version="1.0" encoding="UTF-8"?>

<databaseChangeLog
        xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
         http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-3.1.xsd">
    <changeSet id="01" author="sdelamo">
        <createTable tableName="book" remarks="A table to contain all books">
            <column name="isbn" type="varchar(255)">
                <constraints nullable="false" unique="true" primaryKey="true"/>
            </column>
            <column name="title" type="varchar(255)">
                <constraints nullable="false"/>
            </column>
            <column name="price" type="NUMERIC">
                <constraints nullable="false"/>
            </column>
            <column name="about" type="LONGVARCHAR">
                <constraints nullable="true"/>
            </column>
        </createTable>
    </changeSet>
</databaseChangeLog>
```

```xml
<?xml version="1.0" encoding="UTF-8"?>

<databaseChangeLog
        xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
         http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-3.1.xsd">
    <changeSet id="02" author="sdelamo">
        <insert tableName="book">
            <column name="isbn">0321601912</column>
            <column name="title">Continuous Delivery</column>
            <column name="price">39.99</column>
            <column name="about">Winner of the 2011 Jolt Excellence Award! Getting software released to users is often a painful, risky, and time-consuming process. This groundbreaking new book sets out the principles and technical practices that enable rapid, incremental delivery of high quality, valuable new functionality to users.</column>
        </insert>
    </changeSet>
</databaseChangeLog>
```

### Entidade mapeadas com registros java

Criar uma entidade mapeada de dados do Micronaut

```java

package example.micronaut;

import io.micronaut.core.annotation.NonNull;
import io.micronaut.core.annotation.Nullable;
import io.micronaut.data.annotation.Id;
import io.micronaut.data.annotation.MappedEntity;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import jakarta.validation.constraints.Size;
import java.math.BigDecimal;

@MappedEntity
public record Book(@Id @NonNull @NotBlank @Size(max = 255) String isbn,
                   @NonNull @NotBlank @Size(max = 255) String title,
                   @NonNull @NotNull BigDecimal price,
                   @Nullable String about) {
}

```

### Projeções com registros Java

Crie um registro para projetar alguns dados da booktabela. Por exemplo, exclua o about campo.

```java

package example.micronaut;

import io.micronaut.core.annotation.NonNull;
import io.micronaut.core.annotation.Nullable;
import io.micronaut.data.annotation.GeneratedValue;
import io.micronaut.data.annotation.Id;
import io.micronaut.data.annotation.MappedEntity;
import io.micronaut.serde.annotation.Serdeable;

import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import jakarta.validation.constraints.Size;
import java.math.BigDecimal;

@Serdeable
public record BookCard(@NonNull @NotBlank @Size(max = 255) String isbn,
                       @NonNull @NotBlank @Size(max = 255) String title,
                       @NonNull @NotNull BigDecimal price) {
}


package example.micronaut;

import io.micronaut.data.jdbc.annotation.JdbcRepository;
import io.micronaut.data.model.query.builder.sql.Dialect;
import io.micronaut.data.repository.CrudRepository;
import java.util.List;

@JdbcRepository(dialect = Dialect.POSTGRES)
public interface BookRepository extends CrudRepository<Book, String> {
    List<BookCard> find();
}

```

### Serialização JSON com registros Java

Crie um registro Java para representar uma resposta JSON:

```java

package example.micronaut;

import io.micronaut.core.annotation.NonNull;
import io.micronaut.core.annotation.ReflectiveAccess;
import io.micronaut.serde.annotation.Serdeable;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import java.math.BigDecimal;

@Serdeable
@ReflectiveAccess//1
public record BookForSale(@NonNull @NotBlank String isbn,
                          @NonNull @NotBlank String title,
                          @NonNull @NotNull BigDecimal price) { }

```

1 - `ReflectiveAccess` - é necessário para permitir que o Micronaut gere código de serialização e desserialização em tempo de execução. Isso é necessário porque os registros Java são imutáveis e o Micronaut precisa acessar os campos do registro para serializá-los corretamente.

```java
package example.micronaut;

import io.micronaut.core.annotation.NonNull;
import io.micronaut.http.annotation.Controller;
import io.micronaut.http.annotation.Get;

import java.math.BigDecimal;
import java.math.RoundingMode;
import java.util.Collections;
import java.util.List;
import java.util.stream.Collectors;
import io.micronaut.scheduling.TaskExecutors;
import io.micronaut.scheduling.annotation.ExecuteOn;

@Controller("/books")
class BookController {

    private final BookRepository bookRepository;
    private final ValueAddedTaxConfiguration valueAddedTaxConfiguration;

    BookController(BookRepository bookRepository,
                   ValueAddedTaxConfiguration valueAddedTaxConfiguration) {
        this.bookRepository = bookRepository;
        this.valueAddedTaxConfiguration = valueAddedTaxConfiguration;
    }

    @ExecuteOn(TaskExecutors.BLOCKING)
    @Get
    List<BookForSale> index() {
        return bookRepository.find()
                .stream()
                .map(bookCard -> new BookForSale(bookCard.isbn(),
                    bookCard.title(),
                    salePrice(bookCard)))
                .collect(Collectors.toList());
    }

    @NonNull
    private BigDecimal salePrice(@NonNull BookCard bookCard) {
        return bookCard.price()
                .add(bookCard.price()
                    .multiply(valueAddedTaxConfiguration.percentage()
                        .divide(new BigDecimal("100.0"), 2, RoundingMode.HALF_DOWN)))
                .setScale(2, RoundingMode.HALF_DOWN);
    }
}

```

### Testes

```java
package example.micronaut;

import io.micronaut.context.annotation.Property;
import io.micronaut.core.type.Argument;
import io.micronaut.http.HttpRequest;
import io.micronaut.http.client.BlockingHttpClient;
import io.micronaut.http.client.HttpClient;
import io.micronaut.http.client.annotation.Client;
import io.micronaut.test.extensions.junit5.annotation.MicronautTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.math.BigDecimal;
import java.util.List;

import static org.junit.jupiter.api.Assertions.assertNotNull;
import static org.junit.jupiter.api.Assertions.assertEquals;

@Property(name = "vat.percentage", value = "21.0")
@MicronautTest(transactional = false)
class BookControllerTest {

    int booksInsertedInDbByMigration = 1;

    @Inject
    @Client("/")
    HttpClient httpClient;

    @Inject
    BookRepository bookRepository;

    @Test
    void recordsUsedForJsonSerialization() {
        String title = "Building Microservices";
        String isbn = "1491950358";

        String about = """
                        Distributed systems have become more fine-grained in the past 10 years, shifting from code-heavy monolithic applications to smaller, self-contained microservices. But developing these systems brings its own set of headaches. With lots of examples and practical advice, this book takes a holistic view of the topics that system architects and administrators must consider when building, managing, and evolving microservice architectures.

                        Microservice technologies are moving quickly. Author Sam Newman provides you with a firm grounding in the concepts while diving into current solutions for modeling, integrating, testing, deploying, and monitoring your own autonomous services. You’ll follow a fictional company throughout the book to learn how building a microservice architecture affects a single domain.

                        Discover how microservices allow you to align your system design with your organization’s goals
                        Learn options for integrating a service with the rest of your system
                        Take an incremental approach when splitting monolithic codebases
                        Deploy individual microservices through continuous integration
                        Examine the complexities of testing and monitoring distributed services
                        Manage security with user-to-service and service-to-service models
                        Understand the challenges of scaling microservice architectures
                        """;
        Book b = new Book(isbn,
            title,
            new BigDecimal("38.15"),
            about);
        Book book = bookRepository.save(b);
        assertEquals(booksInsertedInDbByMigration + 1, bookRepository.count());
        BlockingHttpClient client = httpClient.toBlocking();
        List<BookForSale> books = client.retrieve(HttpRequest.GET("/books"),
                Argument.listOf(BookForSale.class));
        assertNotNull(books);
        assertEquals(booksInsertedInDbByMigration + 1, books.size());
        assertEquals("Building Microservices", books.get(1).title());
        assertEquals("1491950358", books.get(1).isbn());
        assertEquals(new BigDecimal("46.16"), books.get(1).price());
        bookRepository.delete(book);
    }
}


```

### Usando Test Resources

```yml

datasources:
  default:
    driverClassName: org.postgresql.Driver
    dialect: POSTGRES
    schema-generate: NONE
```
