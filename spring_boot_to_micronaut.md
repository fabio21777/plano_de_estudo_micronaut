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


##
