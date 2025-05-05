## Criando meu primeiro app Micronaut

Esse guia é baseado no tutorial oficial do Micronaut, que pode ser encontrado [aqui](https://guides.micronaut.io/latest/creating-your-first-micronaut-app-maven-java.html).


## Oqué presisamos

* Ide
* java 21


## Criando o projeto

Há duas formas de criar um projeto micronaut, através da CLI ou através do site do micronaut.

[CLI](https://micronaut-projects.github.io/micronaut-starter/latest/guide/#installation)

[Site](https://micronaut.io/launch)


## Escrevendo o codigo

Apos criar um projeto base com o micronaut, podemos indentificar a classe `Application.java` que é a classe principal do nosso projeto, onde o micronaut inicia a aplicação.

```java
package example.micronaut;

import io.micronaut.runtime.Micronaut;

public class Application {

    public static void main(String[] args) {
        Micronaut.run(Application.class, args);
    }
}
```

### Criando um controller

Apos criar o projeto, podemos criar um controller, que é a classe responsável por receber as requisições HTTP e retornar as respostas.

```java
package example.micronaut;

import io.micronaut.http.MediaType;
import io.micronaut.http.annotation.Controller;
import io.micronaut.http.annotation.Get;
import io.micronaut.http.annotation.Produces;

@Controller("/hello") //1
public class HelloController {
    @Get //2
    @Produces(MediaType.TEXT_PLAIN)//3
    public String index() {
        return "Hello World"; //4
    }
}
```

1 - A classe é definida como um controller, e o caminho da requisição é definido como `/hello`.
2 - O método `index` é definido como um endpoint HTTP do termo  GET, ou seja, ele será chamado quando uma requisição GET for feita para o caminho `/hello`.
3 - O método `index` é definido como um produtor de mídia, ou seja, ele irá retornar um tipo de mídia específico, nesse caso, `text/plain`, normalmente retornamos JSON.
4 - O método `index` retorna uma string, que será o corpo da resposta HTTP.


### Testando o controller

```java

package example.micronaut;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertNotNull;

import io.micronaut.http.HttpRequest;
import io.micronaut.http.MediaType;
import io.micronaut.http.client.HttpClient;
import io.micronaut.http.client.annotation.Client;
import io.micronaut.test.extensions.junit5.annotation.MicronautTest;
import org.junit.jupiter.api.Test;

import jakarta.inject.Inject;

@MicronautTest
public class HelloControllerTest {

    @Inject // 1
    @Client("/") // 2
    HttpClient client;

    @Test //3
    public void testHello() {
        HttpRequest<?> request = HttpRequest.GET("/hello").accept(MediaType.TEXT_PLAIN); //4
        String body = client.toBlocking().retrieve(request);

        assertNotNull(body);
        assertEquals("Hello World", body);
    }
}

```

1 - A anotação `@Inject` é usada para injetar o cliente HTTP no teste.
2 - A anotação `@Client` é usada para definir o cliente HTTP, nesse caso, estamos definindo o cliente como a raiz do projeto.
3 - A anotação `@Test` é usada para definir o método como um teste.
4 - Criar solicitações HTTP é fácil graças à API fluida do framework Micronaut.

```bash
./mvnw test
```

### Executando o projeto

Para executar o projeto, basta executar o seguinte comando:

```bash
./mvnw mn:run
```


## Gere um executável nativo com GraalVM

```bash
sdk install java 21.0.5-graal
mvnw package -Dpackaging=native-image
```

### Peronalizando o nome do executável

Para personalizar o nome do executável, basta adicionar a seguinte propriedade no arquivo `pom.xml`:

```xml

<plugin>
    <groupId>org.graalvm.buildtools</groupId>
    <artifactId>native-maven-plugin</artifactId>
    <version>0.10.3</version>
    <configuration>
        <!-- <1> -->
        <imageName>mn-graalvm-application</imageName>
        <buildArgs>
              <!-- <2> -->
          <buildArg>-Ob</buildArg>
        </buildArgs>
    </configuration>
</plugin>

```

1 - O nome do executável é definido como `mn-graalvm-application`.
2 - É possível passar argumentos de compilação extras para native-image. Por exemplo, -Obhabilita o modo de compilação rápida.
