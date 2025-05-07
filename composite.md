# Micronaut Patterns - Composite



## Composite

Um padrão comum no desenvolvimento de aplicações Micronaut é criar uma interface funcional ordenada. Muitas vezes, você deseja avaliar cada implementação em ordem. Combinando a `@Primary` anotação e a injeção de uma coleção de beans de um tipo específico, você alcança esse padrão facilmente em uma aplicação Micronaut.

Imagine que você queira criar uma API para resolver uma cor:

```java
package example.micronaut;

import io.micronaut.core.annotation.NonNull;
import io.micronaut.core.order.Ordered;
import io.micronaut.http.HttpRequest;

import java.util.Optional;

@FunctionalInterface//1
public interface ColorFetcher extends Ordered {//2

    @NonNull
    Optional<String> favouriteColor(@NonNull HttpRequest<?> request);
}

```

1. Uma interface com uma declaração de método abstrato é conhecida como interface funcional. O compilador verifica se todas as interfaces anotadas com `@FunctionInterface` realmente contêm um e apenas um método abstrato.


```java

package example.micronaut;

import io.micronaut.core.annotation.NonNull;
import io.micronaut.http.HttpRequest;
import jakarta.inject.Singleton;

import java.util.Optional;

@Singleton
public class HttpHeaderColorFetcher implements ColorFetcher {
    @Override
    @NonNull
    public Optional<String> favouriteColor(@NonNull HttpRequest<?> request) {
        return request.getHeaders().get("color", String.class);
    }

    @Override
    public int getOrder() {
        return 10;
    }
}

package example.micronaut;

import io.micronaut.http.HttpRequest;
import jakarta.inject.Singleton;

import java.util.Locale;
import java.util.Optional;
import java.util.stream.Stream;

@Singleton
public class PathColorFetcher implements ColorFetcher {

    private static final String[] COLORS = {
            "Red",
            "Blue",
            "Green",
            "Orange",
            "White",
            "Black",
            "Yellow",
            "Purple",
            "Silver",
            "Brown",
            "Gray",
            "Pink",
            "Olive",
            "Maroon",
            "Violet",
            "Charcoal",
            "Magenta",
            "Bronze",
            "Cream",
            "Gold",
            "Tan",
            "Teal",
            "Mustard",
            "Navy Blue",
            "Coral",
            "Burgundy",
            "Lavender",
            "Mauve",
            "Peach",
            "Rust",
            "Indigo",
            "Ruby",
            "Clay",
            "Cyan",
            "Azure",
            "Beige",
            "Turquoise",
            "Amber",
            "Mint"
    };

    @Override
    public Optional<String> favouriteColor(HttpRequest<?> request) {
        return Stream.of(COLORS)
                .filter(c -> request.getPath().contains(c.toLowerCase(Locale.ROOT)))
                .map(String::toLowerCase)
                .findFirst();
    }

    @Override
    public int getOrder() {
        return 20;
    }
}


```

```log

Message: Multiple possible bean candidates found:
[example.micronaut.PathColorFetcher,
example.micronaut.HttpHeaderColorFetcher]
Path Taken: new ColorController(ColorFetcher colorFetcher)
--> new ColorController([ColorFetcher colorFetcher])
io.micronaut.context.exceptions.DependencyInjectionException:
Failed to inject value for parameter [colorFetcher]
of class: example.micronaut.ColorController

```

```java
package example.micronaut;

import io.micronaut.core.annotation.NonNull;
import io.micronaut.http.HttpRequest;
import io.micronaut.http.MediaType;
import io.micronaut.http.annotation.Controller;
import io.micronaut.http.annotation.Get;
import io.micronaut.http.annotation.Produces;

import java.util.Optional;

@Controller("/color")
public class ColorController {

    private final ColorFetcher colorFetcher;

    public ColorController(ColorFetcher colorFetcher) {
        this.colorFetcher = colorFetcher;
    }

    @Produces(MediaType.TEXT_PLAIN)
    @Get("/mint")
    Optional<String> mint(@NonNull HttpRequest<?> request) {
        return colorFetcher.favouriteColor(request);
    }

    @Produces(MediaType.TEXT_PLAIN)
    @Get
    Optional<String> index(@NonNull HttpRequest<?> request) {
        return colorFetcher.favouriteColor(request);
    }
}

```

### @Primary

Para resolver o problema de múltiplas implementações, você pode usar a anotação `@Primary` em uma das implementações. Isso fará com que o Micronaut use essa implementação como padrão quando houver várias opções disponíveis.

```java

package example.micronaut;

import io.micronaut.context.annotation.Primary;
import io.micronaut.http.HttpRequest;
import jakarta.inject.Singleton;

import java.util.List;
import java.util.Optional;

@Primary
@Singleton
public class CompositeColorFetcher implements ColorFetcher {

    private final List<ColorFetcher> colorFetcherList;

    public CompositeColorFetcher(List<ColorFetcher> colorFetcherList) {
        this.colorFetcherList = colorFetcherList;
    }

    @Override
    public Optional<String> favouriteColor(HttpRequest<?> request) {
        return colorFetcherList.stream()
                .map(colorFetcher -> colorFetcher.favouriteColor(request))
                .filter(Optional::isPresent)
                .map(Optional::get)
                .findFirst();
    }
}
```

### Teste

```java

package example.micronaut;

import io.micronaut.context.BeanContext;
import io.micronaut.test.extensions.junit5.annotation.MicronautTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import static org.junit.jupiter.api.Assertions.assertTrue;

@MicronautTest(startApplication = false)
class CompositeColorFetcherTest {

    @Inject
    BeanContext beanContext;

    @Test
    void compositeColorFetcherIsThePrimaryBeanOfTypeColorFetcher() {
        assertTrue(beanContext.getBean(ColorFetcher.class) instanceof CompositeColorFetcher);
    }
}

```

### Teste do controlador

```java
package example.micronaut;

import io.micronaut.http.HttpRequest;
import io.micronaut.http.client.BlockingHttpClient;
import io.micronaut.http.client.HttpClient;
import io.micronaut.http.client.annotation.Client;
import io.micronaut.http.client.exceptions.HttpClientResponseException;
import io.micronaut.test.extensions.junit5.annotation.MicronautTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertThrows;

@MicronautTest
class ColorControllerTest {

    @Inject
    @Client("/")
    HttpClient httpClient;

    @Test
    void testCompositePattern() {
        BlockingHttpClient client = httpClient.toBlocking();
        assertEquals("yellow", client.retrieve(HttpRequest.GET("/color").header("color", "yellow")));
        assertThrows(HttpClientResponseException.class, () -> client.retrieve(HttpRequest.GET("/color")));

        assertEquals("yellow", client.retrieve(HttpRequest.GET("/color/mint").header("color", "yellow")));
        assertEquals("mint", client.retrieve(HttpRequest.GET("/color/mint")));
    }
}

```
