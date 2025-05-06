# Testes


## [Using Micronaut Test REST-Assured in a Micronaut application](https://guides.micronaut.io/latest/micronaut-rest-assured-maven-java.html)


O módulo Micronaut Test REST Assured facilita a integração da biblioteca REST-Assured . Sua utilização elimina a necessidade de codificar a versão e simplifica os testes, permitindo a injeção de dados `RequestSpecification` em campos de teste ou parâmetros de método (parâmetros suportados apenas com o JUnit 5):


## Controlador

```java

package example.micronaut;

import io.micronaut.http.MediaType;
import io.micronaut.http.annotation.Controller;
import io.micronaut.http.annotation.Get;
import io.micronaut.http.annotation.Produces;

@Controller("/hello")
public class HelloController {
    @Get
    @Produces(MediaType.TEXT_PLAIN)
    public String index() {
        return "Hello World";
    }
}
```

```xml

<dependency>
    <groupId>io.micronaut.test</groupId>
    <artifactId>micronaut-test-rest-assured</artifactId>
    <scope>test</scope>
</dependency>

```

## Teste

```java
package example.micronaut;

import io.micronaut.test.extensions.junit5.annotation.MicronautTest;
import io.restassured.specification.RequestSpecification;
import org.junit.jupiter.api.Test;

import static org.hamcrest.CoreMatchers.is;

@MicronautTest
public class HelloControllerTest {

    @Test
    public void testHelloEndpoint(RequestSpecification spec) { //2
        spec//3
            .when()
                .get("/hello")
            .then()
                .statusCode(200)
                .body(is("Hello World"));
    }
}

```

1. Anote a classe com `@MicronautTest` para que o framework Micronaut inicialize o contexto da aplicação e o servidor embarcado. [Mais informações](https://micronaut-projects.github.io/micronaut-test/latest/guide/).

2. Injetar uma instância de `RequestSpecification`.

3. O Micronaut Test define a porta do servidor incorporado em `spec`, portanto não é necessário injetá-la `EmbeddedServer` e recuperá-la explicitamente.
