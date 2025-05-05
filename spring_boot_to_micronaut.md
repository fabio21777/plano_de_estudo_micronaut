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
