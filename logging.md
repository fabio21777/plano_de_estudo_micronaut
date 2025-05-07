# Logging


## Log incoming HTTP Request headers with a Server Filter

Aprenda a registrar cada cabeçalho de solicitação HTTP com um @ServerFilter e um método anotado com @FilterRequest.


Recomendamos que você siga as instruções nas próximas seções e crie o aplicativo passo a passo. No entanto, você pode ir direto para o exemplo completo.


### logback

```xml
<configuration>
    ...
    <logger name="example.micronaut" level="TRACE"/>
</configuration>

```


```java
package example.micronaut;

import io.micronaut.core.order.Ordered;
import io.micronaut.http.HttpRequest;
import io.micronaut.http.annotation.RequestFilter;
import io.micronaut.http.annotation.ServerFilter;
import io.micronaut.http.filter.ServerFilterPhase;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import io.micronaut.http.util.HttpHeadersUtil;
import static io.micronaut.http.annotation.Filter.MATCH_ALL_PATTERN;

@ServerFilter(MATCH_ALL_PATTERN)
class LoggingHeadersFilter implements Ordered {

    private static final Logger LOG = LoggerFactory.getLogger(LoggingHeadersFilter.class);

    @RequestFilter
    void filterRequest(HttpRequest<?> request) {
        HttpHeadersUtil.trace(LOG, request.getHeaders());
    }

    @Override
    public int getOrder() {
        return ServerFilterPhase.FIRST.order();
    }
}

```

1. `@ServerFilter`marca um bean como um filtro para o servidor HTTP. O valor da anotação `Filter.MATCH_ALL_PATTERN`significa que o filtro corresponde a todas as requisições.
2. Um método de filtro anotado com `@RequestFilter` execuções antes do processamento da solicitação. Um método de filtro deve ser declarado em um bean anotado com `@ServerFilter`ou `@ClientFilter`.
3. Os filtros podem ser ordenados implementando-os Orderedna classe filter.


### Controlador

```java

package example.micronaut;

import io.micronaut.http.annotation.Controller;
import io.micronaut.http.annotation.Get;

import java.util.Collections;
import java.util.Map;

@Controller
public class HelloController {

    @Get
    public Map<String, Object> index() {
        return Collections.singletonMap("message", "Hello World");
    }
}

```

```xml
<dependency>
    <groupId>ch.qos.logback</groupId>
    <artifactId>logback-classic</artifactId>
    <scope>test</scope>
</dependency>
```

### Escrevendo o teste

```java

package example.micronaut;

import ch.qos.logback.classic.spi.ILoggingEvent;
import ch.qos.logback.core.AppenderBase;

import java.util.ArrayList;
import java.util.List;

public class MemoryAppender extends AppenderBase<ILoggingEvent> {

    private final List<ILoggingEvent> events = new ArrayList<>();

    @Override
    protected void append(ILoggingEvent e) {
        events.add(e);
    }

    public List<ILoggingEvent> getEvents() {
        return events;
    }
}

package example.micronaut;

import ch.qos.logback.classic.Logger;
import ch.qos.logback.classic.spi.ILoggingEvent;
import io.micronaut.http.HttpHeaders;
import io.micronaut.http.HttpRequest;
import io.micronaut.http.client.BlockingHttpClient;
import io.micronaut.http.client.HttpClient;
import io.micronaut.http.client.annotation.Client;
import io.micronaut.test.extensions.junit5.annotation.MicronautTest;
import org.junit.jupiter.api.Test;
import org.slf4j.LoggerFactory;

import static org.junit.jupiter.api.Assertions.assertDoesNotThrow;
import static org.junit.jupiter.api.Assertions.assertTrue;

@MicronautTest
class HelloControllerTest {

    @Test
    void testHelloFilterLogging(@Client("/") HttpClient httpClient) {
        MemoryAppender appender = new MemoryAppender();
        Logger l = (Logger) LoggerFactory.getLogger(LoggingHeadersFilter.class);
        l.addAppender(appender);
        appender.start();
        BlockingHttpClient client = httpClient.toBlocking();
        assertDoesNotThrow(() -> client.retrieve(HttpRequest.GET("/")
                .header(HttpHeaders.AUTHORIZATION, "Bearer x")
                .header("foo", "bar")));
        assertTrue(appender.getEvents()
                .stream()
                .map(ILoggingEvent::getFormattedMessage)
                .anyMatch(it -> it.equals("foo: bar")));
        assertTrue(appender.getEvents()
                .stream()
                .map(ILoggingEvent::getFormattedMessage)
                .noneMatch(it -> it.equals("Authorization: Bearer x")));
        assertTrue(appender.getEvents()
                .stream()
                .map(ILoggingEvent::getFormattedMessage)
                .anyMatch(it -> it.equals("Authorization: *MASKED*")));

        appender.stop();
    }
}


```
