# Guias de usuarios Spring boot para Micronaut

## Classe de aplicação

Os aplicativos Spring Boot e Micronaut contêm uma classe de aplicativo simples que inicia o aplicativo para você.

### Classe de aplicação Spring Boot

```java
package example.micronaut;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

### Classe de aplicação Micronaut

```java
package example.micronaut;

import io.micronaut.runtime.Micronaut;

public class Application {

    public static void main(String[] args) {
        Micronaut.run(Application.class, args);
    }
}
```

Como podemos ver, a classe de aplicativo Micronaut são basicamente identicas, com a diferença de que o Spring Boot usa a anotação `@SpringBootApplication` para habilitar o recurso de inicialização automática, enquanto o Micronaut não precisa de nenhuma anotação especial, apenas chamando o método `Micronaut.run()` na classe principal.


## Definir manualmente um Bean

Tanto o Spring quanto o Micronaut são mecanismos de injeção de dependências. Neste tutorial, criaremos manualmente um bean, uma classe gerenciada pelo contexto do bean.


## Primeiro vamos comparar com a classe no spring boot

```java

package example.micronaut;

public interface Greeter {
    String greet();
}

package example.micronaut;

public class HelloGreeter implements Greeter {
    @Override
    public String greet() {
        return "Hello";
    }
}




package example.micronaut;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
class GreeterFactory {

    @Bean
    Greeter helloGreeter() {
        return new HelloGreeter();
    }
}


package example.micronaut;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

import static org.junit.jupiter.api.Assertions.*;

@SpringBootTest //1
class GreeterTest {

    @Autowired //2
    Greeter greeter;

    @Test
    void helloGreeterIsInjectedAsBeanOfTypeGreeter() {
        assertNotNull(greeter);
        assertEquals("Hello", greeter.greet());
    }

}

```

1 - A anotação `@SpringBootTest` é usada para definir a classe de teste como um teste de unidade do Spring Boot.
2 - A anotação `@Autowired` é usada para injetar o bean `Greeter` no teste.


## Agora vamos comparar com a classe no micronaut

Usamos a factory do micronaut para criar o bean

```java

package example.micronaut;

import io.micronaut.context.annotation.Factory;
import io.micronaut.context.annotation.Bean;

@Factory
class GreeterFactory {
    @Bean
    Greeter helloGreeter() {
        return new HelloGreeter();
    }
}

package example.micronaut;

import io.micronaut.test.extensions.junit5.annotation.MicronautTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

@MicronautTest
class GreeterTest {

    @Inject
    Greeter greeter;

    @Test
    void helloGreeterIsInjectedAsBeanOfTypeGreeter() {
        assertNotNull(greeter);
        assertEquals("Hello", greeter.greet());
    }

}

```

1 - A anotação `@MicronautTest` é usada para definir a classe de teste como um teste de unidade do Micronaut.
2 - A anotação `@Inject` é usada para injetar o bean `Greeter` no teste.

### Conclusão

Como podemos ver, o Micronaut e o Spring Boot são muito semelhantes, em nivel de código são quase idênticos, para trabalhar com bens no micronaut, usamos a anotação `@Factory` para definir a factory do bean, enquanto no Spring Boot usamos a anotação `@Configuration` para definir a factory.


## Marcar uma classe como um bean.- Spring Boot

Este guia compara como marcar uma classe como um bean em um aplicativo Spring Boot com @Component e em um aplicativo Micronaut com @Singleton.

Tanto o Spring quanto o Micronaut são mecanismos de injeção de dependências. Neste tutorial, criamos um bean "marcando" uma classe como bean.


### Primeiro vamos comparar com a classe no spring boot

```java

package example.micronaut;

public interface Greeter {
    String greet();
}

package example.micronaut;
import org.springframework.stereotype.Component;

@Component
public class HelloGreeter implements Greeter {
    @Override
    public String greet() {
        return "Hello";
    }
}

package example.micronaut;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

import static org.junit.jupiter.api.Assertions.*;

@SpringBootTest //1
class GreeterTest {

	@Autowired //2
	Greeter greeter;

	@Test
	void helloGreeterIsInjectedAsBeanOfTypeGreeter() {
		assertNotNull(greeter);
		assertEquals("Hello", greeter.greet());
	}

}

```

1 - A @SpringBootTestanotação informa ao Spring Boot para procurar uma classe de configuração principal (uma com @SpringBootApplication, por exemplo) e usá-la para iniciar um contexto de aplicativo Spring.
2 - Injete um bean do tipo Greeterusando @Autowireda definição de campo.


### Agora vamos comparar com a classe no micronaut

No framework Micronaut, podemos criar uma implementação e anotá-la com @Singleton.

```java


package example.micronaut;

import jakarta.inject.Singleton;

@Singleton
public class HelloGreeter implements Greeter {
    @Override
    public String greet() {
        return "Hello";
    }
}

package example.micronaut;

import io.micronaut.test.extensions.junit5.annotation.MicronautTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

@MicronautTest //1
class GreeterTest {

    @Inject //2
    Greeter greeter;

    @Test
    void helloGreeterIsInjectedAsBeanOfTypeGreeter() {
        assertNotNull(greeter);
        assertEquals("Hello", greeter.greet());
    }

}

```

1 - A anotação @MicronautTest informa ao Micronaut para procurar uma classe de configuração principal (uma com @MicronautApplication, por exemplo) e usá-la para iniciar um contexto de aplicativo Micronaut.
2 - Injete um bean do tipo Greeterusando @Injecta definição de campo.


### Conclusão

Como podemos ver, o Micronaut e o Spring Boot são muito semelhantes, em nivel de código são quase idênticos, para trabalhar com bens no micronaut, usamos a anotação @Singleton para definir o bean, enquanto no Spring Boot usamos a anotação @Component para definir o bean.


## Construindo URIs

Este guia compara o uso do UriComponentsBuilder do Spring com o UriBuilder do Micronaut Framework.

Ao desenvolver aplicações web, frequentemente precisamos construir URIs. Por exemplo, para criar respostas de redirecionamento. Neste artigo, comparamos duas APIs semelhantes: Spring UriComponentsBuildere Micronaut UriBuilder .

### Primeiro vamos comparar com a classe no spring boot

```java

package example.micronaut;

import org.junit.jupiter.api.Test;
import org.springframework.web.util.UriComponentsBuilder;

import static org.junit.jupiter.api.Assertions.assertEquals;

public class UriComponentsBuilderTest {

    @Test
    void youCanUseUriComponentsBuilderToBuildUris() {
        String isbn = "1680502395";
        assertEquals("/book/1680502395?lang=es", UriComponentsBuilder.fromUriString("/book")
                .path("/" + isbn)
                .queryParam("lang", "es")
                .build()
                .toUriString());
    }
}

```

### Agora vamos comparar com a classe no micronaut

Como você pode ver no trecho de código a seguir, a API é semelhante. Você deve conseguir migrar facilmente.

```java
package example.micronaut;

import io.micronaut.http.uri.UriBuilder;
import org.junit.jupiter.api.Test;

import static org.junit.jupiter.api.Assertions.assertEquals;

public class UriBuilderTest {

    @Test
    void youCanUseUriBuilderToBuildUris() {
        String isbn = "1680502395";
        assertEquals("/book/1680502395?lang=es", UriBuilder.of("/book")
                .path(isbn)
                .queryParam("lang", "es")
                .build()
                .toString());
    }
}


```

## Execute um aplicativo Spring Boot como um aplicativo Micronaut

[link](https://guides.micronaut.io/latest/micronaut-spring-boot-maven-java.html)

Este guia compara como executar um aplicativo Spring Boot como um aplicativo Micronaut.


## Micronaut Data para spring boot

[link](https://guides.micronaut.io/latest/spring-boot-micronaut-data-gradle-java.html)

## Testando serialização spring boot vs micronaut - criando rest api

Este guia compara como testar serialização em um Micronaut Framework e aplicativos Spring Boot.

Este guia é o primeiro tutorial de Construindo uma API REST - uma série de tutoriais que comparam como desenvolver uma API REST com o Micronaut Framework e o Spring Boot.


### Registro


A aplicação que construiremos é uma API REST para assinaturas de Software como Serviço (SaaS). A aplicação serializa e desserializa o seguinte registro Java de/para JSON.

```java
package example.micronaut;

record SaasSubscription(Long id, String name, Integer cents) {
}

 ```

 O Micronaut suporta serialização com o Jackson Databind ou com a serialização do Micronaut . Se você usar o Micronaut Jackson Databind, o registro acima será idêntico no Micronaut Framework e no Spring Boot.

 #### Serialização do Micronaut

Se você usar a serialização Micronaut, deverá optar pela classe a ser submetida à serialização, o que pode ser feito facilmente anotando-a com
`@Serdeable`

```java

package example.micronaut;

import io.micronaut.serde.annotation.Serdeable;

@Serdeable
record SaasSubscription(Long id, String name, Integer cents) {
}

```

#### Json esperado

```json
{
  "id": 1,
  "name": "Micronaut",
  "cents": 1000
}
```

#### Teste

Neste tutorial, usamos AssertJ nos testes. Além disso, usamos Jayway JsonPath - uma DSL Java para leitura de documentos JSON

#### Teste de unidade do Spring Boot

Como a criação de um contexto de aplicação no Spring Boot é lenta , os testes de integração no Spring Boot utilizam o fatiamento de teste. Ao usar o fatiamento de teste, o Spring cria um contexto de aplicação reduzido para uma fatia específica.

```java

package example.micronaut;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.json.JsonTest;
import org.springframework.boot.test.json.JacksonTester;

import java.io.IOException;

import static org.assertj.core.api.Assertions.assertThat;

@JsonTest //1
class SaasSubscriptionJsonTest {

    @Autowired//2
    private JacksonTester<SaasSubscription> json; //3

    @Test
    void saasSubscriptionSerializationTest() throws IOException {
        SaasSubscription subscription = new SaasSubscription(99L, "Professional", 4900);
        assertThat(json.write(subscription)).isStrictlyEqualToJson("expected.json");
        assertThat(json.write(subscription)).hasJsonPathNumberValue("@.id");
        assertThat(json.write(subscription)).extractingJsonPathNumberValue("@.id")
                .isEqualTo(99);
        assertThat(json.write(subscription)).hasJsonPathStringValue("@.name");
        assertThat(json.write(subscription)).extractingJsonPathStringValue("@.name")
                .isEqualTo("Professional");
        assertThat(json.write(subscription)).hasJsonPathNumberValue("@.cents");
        assertThat(json.write(subscription)).extractingJsonPathNumberValue("@.cents")
                .isEqualTo(4900);
    }

    @Test
    void saasSubscriptionDeserializationTest() throws IOException {
        String expected = """
           {
               "id":100,
               "name": "Advanced",
               "cents":2900
           }
           """;
        assertThat(json.parse(expected))
                .isEqualTo(new SaasSubscription(100L, "Advanced", 2900));
        assertThat(json.parseObject(expected).id()).isEqualTo(100);
        assertThat(json.parseObject(expected).name()).isEqualTo("Advanced");
        assertThat(json.parseObject(expected).cents()).isEqualTo(2900);
    }
}

```

1 - Para testar se a serialização e desserialização do Object JSON estão funcionando conforme o esperado, você pode usar a `@JsonTest` anotação . `@JsonTest` que configura automaticamente o Jackson ObjectMapper, quaisquer `@JsonComponent` beans e quaisquer módulos Jackson.
2 - O `@Autowired` é usado para injetar o JacksonTester no teste.
3 - O `JacksonTester` é um recurso do Spring Boot que fornece uma maneira fácil de testar a serialização e desserialização de objetos JSON. Ele fornece métodos para escrever e ler objetos JSON, além de verificar se o JSON gerado corresponde ao esperado.

#### Teste de unidade do Micronaut

```java
package example.micronaut;

import com.jayway.jsonpath.DocumentContext;
import com.jayway.jsonpath.JsonPath;
import io.micronaut.core.io.ResourceLoader;
import io.micronaut.json.JsonMapper;
import io.micronaut.test.extensions.junit5.annotation.MicronautTest;
import org.junit.jupiter.api.Test;

import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStream;
import java.io.InputStreamReader;
import java.io.Reader;
import java.nio.charset.StandardCharsets;
import java.util.Optional;
import java.util.stream.Collectors;

import static org.assertj.core.api.Assertions.assertThat;

@MicronautTest(startApplication = false) //1
class SaasSubscriptionJsonTest {

    @Test
    void saasSubscriptionSerializationTest(JsonMapper json, ResourceLoader resourceLoader) throws IOException { //2
        SaasSubscription subscription = new SaasSubscription(99L, "Professional", 4900);
        String expected = getResourceAsString(resourceLoader, "expected.json");
        String result = json.writeValueAsString(subscription);
        assertThat(result).isEqualToIgnoringWhitespace(expected);
        DocumentContext documentContext = JsonPath.parse(result);
        Number id = documentContext.read("$.id");
        assertThat(id)
                .isNotNull()
                .isEqualTo(99);

        String name = documentContext.read("$.name");
        assertThat(name)
                .isNotNull()
                .isEqualTo("Professional");

        Number cents = documentContext.read("$.cents");
        assertThat(cents)
                .isNotNull()
                .isEqualTo(4900);
    }

    @Test
    void saasSubscriptionDeserializationTest(JsonMapper json) throws IOException {
        String expected = """
           {
               "id":100,
               "name": "Advanced",
               "cents":2900
           }
           """;
        assertThat(json.readValue(expected, SaasSubscription.class))
                .isEqualTo(new SaasSubscription(100L, "Advanced", 2900));
        assertThat(json.readValue(expected, SaasSubscription.class).id()).isEqualTo(100);
        assertThat(json.readValue(expected, SaasSubscription.class).name()).isEqualTo("Advanced");
        assertThat(json.readValue(expected, SaasSubscription.class).cents()).isEqualTo(2900);
    }

    private static String getResourceAsString(ResourceLoader resourceLoader, String resourceName) {
        return resourceLoader.getResourceAsStream(resourceName)
                .flatMap(stream -> {
                    try (InputStream is = stream;
                         Reader reader = new InputStreamReader(is, StandardCharsets.UTF_8);
                         BufferedReader bufferedReader = new BufferedReader(reader)) {
                        return Optional.of(bufferedReader.lines().collect(Collectors.joining("\n")));
                    } catch (IOException e) {
                        e.printStackTrace();
                    }
                    return Optional.empty();
                })
                .orElse("");
    }
}

```

1 - Anote a classe com `@MicronautTest` para que o framework Micronaut inicialize o contexto da aplicação e o servidor embarcado. Por padrão, cada @Testmétodo será encapsulado em uma transação que será revertida quando o teste for concluído. Esse comportamento pode ser alterado configurando transaction-o como false.

2 - Por padrão, com o `JUnit 5`, os parâmetros do método de teste serão resolvidos para beans, se possível.

#### Conclusão

Como mostrado neste tutorial, testar serialização no Micronaut Framework e no Spring Boot é semelhante. A principal diferença é que o slicing de teste não é necessário no Micronaut Framework.



## Implementando GET  spring boot vs micronaut


Este guia compara como testar a implementação de um GET em um Micronaut Framework e em aplicativos Spring Boot.

Este guia é o segundo tutorial de Construindo uma API REST - uma série de tutoriais que comparam como desenvolver uma API REST com o Micronaut Framework e o Spring Boot.

![alt text](image.png)

> Outra diferença importante é a visibilidade dos métodos do controlador. **O Micronaut Framework não utiliza reflexão (o que resulta em melhor desempenho e melhor integração com tecnologias como GraalVM)** . Portanto, ele exige que os métodos do controlador sejam públicos, protegidos ou privados (sem modificadores). Ao longo destes tutoriais, os métodos dos controladores Micronaut utilizam privados.

### Controlador no Spring Boot

```java

package example.micronaut;

import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController //1
@RequestMapping("/subscriptions") //2
class SaasSubscriptionController {

    @GetMapping("/{id}") //3
    private ResponseEntity<SaasSubscription> findById(@PathVariable Long id) { //4
        if (id.equals(99L)) {
            SaasSubscription subscription = new SaasSubscription(99L, "Advanced", 2900);
            return ResponseEntity.ok(subscription);
        }
        return ResponseEntity.notFound().build(); //5
    }
}

```

1 - A anotação `@RestController` é usada para definir a classe como um controlador REST, o que significa que ela pode manipular requisições HTTP e retornar respostas HTTP.
2 - A anotação `@RequestMapping` é usada para definir o caminho base da requisição, que neste caso é `/subscriptions`.
3 - A anotação `@GetMapping` é usada para definir o método como um endpoint HTTP do tipo GET, ou seja, ele será chamado quando uma requisição GET for feita para o caminho `/subscriptions/{id}`.
4 - O método `findById` recebe o id como parâmetro e retorna um objeto `ResponseEntity<SaasSubscription>`, que é a resposta HTTP.
5 - Se o id for igual a 99, o método retorna um objeto `ResponseEntity` com o status HTTP 200 (OK) e o objeto `SaasSubscription` como corpo da resposta. Caso contrário, ele retorna um objeto `ResponseEntity` com o status HTTP 404 (NOT FOUND).


### Controlador no Micronaut

```java
package example.micronaut;

import io.micronaut.http.HttpResponse;
import io.micronaut.http.annotation.Controller;
import io.micronaut.http.annotation.Get;
import io.micronaut.http.annotation.PathVariable;

@Controller("/subscriptions") //1
class SaasSubscriptionController {

    @Get("/{id}")//2
    HttpResponse<SaasSubscription> findById(@PathVariable Long id) { //3
        if (id.equals(99L)) {
            SaasSubscription subscription = new SaasSubscription(99L, "Advanced", 2900);
            return HttpResponse.ok(subscription);
        }
        return HttpResponse.notFound(); //4
    }
}

//versão simplificada com retorno nulo
@Controller("/subscriptions")
class SaasSubscriptionController {

    @Get("/{id}")
    SaasSubscription findById(@PathVariable Long id) {
        if (id.equals(99L)) {
            return new SaasSubscription(99L, "Advanced", 2900);
        }
        return null;
    }
}

```

1 - A anotação `@Controller` é usada para definir a classe como um controlador REST, o que significa que ela pode manipular requisições HTTP e retornar respostas HTTP.
2 - A anotação `@Get` é usada para definir o método como um endpoint HTTP do tipo GET, ou seja, ele será chamado quando uma requisição GET for feita para o caminho `/subscriptions/{id}`.
3 - O método `findById` recebe o id como parâmetro e retorna um objeto `HttpResponse<SaasSubscription>`, que é a resposta HTTP.
4 - Se o id for igual a 99, o método retorna um objeto `HttpResponse` com o status HTTP 200 (OK) e o objeto `SaasSubscription` como corpo da resposta. Caso contrário, ele retorna um objeto `HttpResponse` com o status HTTP 404 (NOT FOUND).

> O código de status HTTP padrão em um método do controlador Micronaut é 200. No entanto, quando o método de um controlador Micronaut retorna nulo, o aplicativo responde com um código de status 404.


### Testes

```java

package example.micronaut;

import com.jayway.jsonpath.DocumentContext;
import com.jayway.jsonpath.JsonPath;
import io.micronaut.http.HttpResponse;
import io.micronaut.http.HttpStatus;
import io.micronaut.http.client.BlockingHttpClient;
import io.micronaut.http.client.HttpClient;
import io.micronaut.http.client.annotation.Client;
import io.micronaut.http.client.exceptions.HttpClientResponseException;
import io.micronaut.test.extensions.junit5.annotation.MicronautTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.AssertionsForClassTypes.catchThrowableOfType;

@MicronautTest //1
class SaasSubscriptionControllerGetTest {

    @Inject //2
    @Client("/") //3
    HttpClient httpClient;

    @Test
    void shouldReturnASaasSubscriptionWhenDataIsSaved() {
        BlockingHttpClient client = httpClient.toBlocking();
        HttpResponse<String> response = client.exchange("/subscriptions/99", String.class);
        assertThat(response.status().getCode()).isEqualTo(HttpStatus.OK.getCode());

        DocumentContext documentContext = JsonPath.parse(response.body());
        Number id = documentContext.read("$.id");
        assertThat(id).isNotNull();
        assertThat(id).isEqualTo(99);

        String name = documentContext.read("$.name");
        assertThat(name).isNotNull();
        assertThat(name).isEqualTo("Advanced");

        Integer cents = documentContext.read("$.cents");
        assertThat(cents).isEqualTo(2900);
    }

    @Test
    void shouldNotReturnASaasSubscriptionWithAnUnknownId() {
        BlockingHttpClient client = httpClient.toBlocking();
        HttpClientResponseException thrown = catchThrowableOfType(() -> //4
                client.exchange("/subscriptions/1000", String.class), HttpClientResponseException.class);
        assertThat(thrown.getStatus().getCode()).isEqualTo(HttpStatus.NOT_FOUND.getCode());
        assertThat(thrown.getResponse().getBody()).isEmpty();
    }
}

```

1 - Anote a classe com `@MicronautTest` para que o framework Micronaut inicialize o contexto da aplicação e o servidor embarcado. Mais informações .
2 - Injete o `HttpClient` bean e aponte-o para o servidor incorporado.
3 - Quando o cliente HTTP recebe uma resposta com um código de status HTTP >= 400, ele lança um `HttpClientResponseException`. Você pode obter o status e o corpo da resposta a partir da exceção.


### Conclusão

A definição de rotas é extremamente semelhante em ambos os frameworks. No entanto, o Micronaut Framework oferece validação de rotas em tempo de compilação e uma abordagem sem reflexão para isso.
