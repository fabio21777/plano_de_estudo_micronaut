# Kafka and the Micronaut Framework - Event-Driven Applications


Neste guia, criaremos dois microsserviços que usarão o Kafka para se comunicar entre si de forma assíncrona e desacoplada.

```sh
mn create-app --features=kafka,reactor,graalvm,serialization-jackson example.micronaut.books --build=maven --lang=java
```

## Controller

```java

package example.micronaut;

import io.micronaut.http.annotation.Controller;
import io.micronaut.http.annotation.Get;

import java.util.List;
import java.util.Optional;

@Controller("/books")
class BookController {

    private final BookService bookService;

    BookController(BookService bookService) {
        this.bookService = bookService;
    }

    @Get
    List<Book> listAll() {
        return bookService.listAll();
    }

    @Get("/{isbn}")
    Optional<Book> findBook(String isbn) {
        return bookService.findByIsbn(isbn);
    }
}

```

## Criar serviço de analise de livro

```sh
mn create-app --features=kafka,graalvm,serialization-jackson example.micronaut.analytics --build=maven --lang=java
```

## Serviço de analise de livro

```java
package example.micronaut;

import jakarta.inject.Singleton;
import java.util.List;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.stream.Collectors;

@Singleton
public class AnalyticsService {

    private final Map<Book, Long> bookAnalytics = new ConcurrentHashMap<>();

    public void updateBookAnalytics(Book book) {
        bookAnalytics.compute(book, (k, v) -> {
            return v == null ? 1L : v + 1;
        });
    }

    public List<BookAnalytics> listAnalytics() {
        return bookAnalytics
                .entrySet()
                .stream()
                .map(e -> new BookAnalytics(e.getKey().getIsbn(), e.getValue()))
                .collect(Collectors.toList());
    }
}

```


## Teste

```java

package example.micronaut;

import static org.junit.jupiter.api.Assertions.assertEquals;

import io.micronaut.test.extensions.junit5.annotation.MicronautTest;
import org.junit.jupiter.api.Test;

import jakarta.inject.Inject;
import java.util.List;

@MicronautTest
class AnalyticsServiceTest {

    @Inject
    AnalyticsService analyticsService;

    @Test
    void testUpdateBookAnalyticsAndGetAnalytics() {
        Book b1 = new Book("1491950358", "Building Microservices");
        Book b2 = new Book("1680502395", "Release It!");

        analyticsService.updateBookAnalytics(b1);
        analyticsService.updateBookAnalytics(b1);
        analyticsService.updateBookAnalytics(b1);
        analyticsService.updateBookAnalytics(b2);

        List<BookAnalytics> analytics = analyticsService.listAnalytics();
        assertEquals(2, analytics.size());

        assertEquals(3, findBookAnalytics(b1, analytics).getCount());
        assertEquals(1, findBookAnalytics(b2, analytics).getCount());
    }

    private BookAnalytics findBookAnalytics(Book b, List<BookAnalytics> analytics) {
        return analytics
                .stream()
                .filter(bookAnalytics -> bookAnalytics.getBookIsbn().equals(b.getIsbn()))
                .findFirst()
                .orElseThrow(() -> new RuntimeException("Book not found"));
    }
}

```

## Controller de analise de livro

```java
package example.micronaut;

import io.micronaut.http.annotation.Controller;
import io.micronaut.http.annotation.Get;

import java.util.List;

@Controller("/analytics")
class AnalyticsController {

    private final AnalyticsService analyticsService;

    AnalyticsController(AnalyticsService analyticsService) {
        this.analyticsService = analyticsService;
    }

    @Get
    List<BookAnalytics> listAnalytics() {
        return analyticsService.listAnalytics();
    }
}

```

## Instalação do Kafka

```yaml
version: '2'
services:
  zookeeper:
    image: confluentinc/cp-zookeeper
    ports:
      - 2181:2181
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
  kafka:
    image: confluentinc/cp-kafka
    depends_on:
      - zookeeper
    ports:
      - 9092:9092
    environment:
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:29092,PLAINTEXT_HOST://localhost:9092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
```

## Configuração cliente Kafka no livro

```java

package example.micronaut;

import io.micronaut.configuration.kafka.annotation.KafkaClient;
import io.micronaut.configuration.kafka.annotation.Topic;
import org.reactivestreams.Publisher;
import reactor.core.publisher.Mono;

@KafkaClient
public interface AnalyticsClient {

    @Topic("analytics")
    Mono<Book> updateAnalytics(Book book);
}

```

## Teste

```java
package example.micronaut;

import io.micronaut.configuration.kafka.annotation.KafkaListener;
import io.micronaut.configuration.kafka.annotation.Topic;
import io.micronaut.core.annotation.NonNull;
import io.micronaut.core.type.Argument;
import io.micronaut.http.HttpRequest;
import io.micronaut.http.client.HttpClient;
import io.micronaut.http.client.annotation.Client;
import io.micronaut.http.client.exceptions.HttpClientResponseException;
import io.micronaut.test.extensions.junit5.annotation.MicronautTest;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.TestInstance;

import jakarta.inject.Inject;
import java.util.Collection;
import java.util.Collections;
import java.util.Map;
import java.util.Optional;
import java.util.concurrent.ConcurrentLinkedDeque;

import static io.micronaut.configuration.kafka.annotation.OffsetReset.EARLIEST;
import static java.util.concurrent.TimeUnit.SECONDS;
import static org.awaitility.Awaitility.await;
import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertNotNull;
import static org.junit.jupiter.api.Assertions.assertThrows;
import static org.junit.jupiter.api.Assertions.assertTrue;
import static org.junit.jupiter.api.TestInstance.Lifecycle.PER_CLASS;

@MicronautTest
@TestInstance(PER_CLASS)
class BookControllerTest {

    private static final Collection<Book> received = new ConcurrentLinkedDeque<>();

    @Inject
    AnalyticsListener analyticsListener;

    @Inject
    @Client("/")
    HttpClient client;

    @Test
    void testMessageIsPublishedToKafkaWhenBookFound() {
        String isbn = "1491950358";

        Optional<Book> result = retrieveGet("/books/" + isbn);
        assertNotNull(result);
        assertTrue(result.isPresent());
        assertEquals(isbn, result.get().getIsbn());

        await().atMost(5, SECONDS).until(() -> !received.isEmpty());

        assertEquals(1, received.size());
        Book bookFromKafka = received.iterator().next();
        assertNotNull(bookFromKafka);
        assertEquals(isbn, bookFromKafka.getIsbn());
    }

    @Test
    void testMessageIsNotPublishedToKafkaWhenBookNotFound() throws Exception {
        assertThrows(HttpClientResponseException.class, () -> {
            retrieveGet("/books/INVALID");
        });

        Thread.sleep(5_000);
        assertEquals(0, received.size());
    }

    @AfterEach
    void cleanup() {
        received.clear();
    }

    @KafkaListener(offsetReset = EARLIEST)
    static class AnalyticsListener {

        @Topic("analytics")
        void updateAnalytics(Book book) {
            received.add(book);
        }
    }

    private Optional<Book> retrieveGet(String url) {
        return client.toBlocking().retrieve(HttpRequest.GET(url), Argument.of(Optional.class, Book.class));
    }
}

```

## Enviando mensagem para o Kafka usando um httpServerFilter

```java
package example.micronaut;

import io.micronaut.http.HttpRequest;
import io.micronaut.http.MutableHttpResponse;
import io.micronaut.http.annotation.Filter;
import io.micronaut.http.filter.HttpServerFilter;
import io.micronaut.http.filter.ServerFilterChain;
import reactor.core.publisher.Flux;
import org.reactivestreams.Publisher;

import java.util.Optional;

@Filter("/books/?*")
public class AnalyticsFilter implements HttpServerFilter {

    private final AnalyticsClient analyticsClient;

    public AnalyticsFilter(AnalyticsClient analyticsClient) {
        this.analyticsClient = analyticsClient;
    }

    @Override
    public Publisher<MutableHttpResponse<?>> doFilter(HttpRequest<?> request,
                                                      ServerFilterChain chain) {
        return Flux
                .from(chain.proceed(request))
                .flatMap(response -> {
                    Book book = response.getBody(Book.class).orElse(null);
                    if (book == null) {
                        return Flux.just(response);
                    }
                    return Flux.from(analyticsClient.updateAnalytics(book)).map(b -> response);
                });
    }
}

```


## criando consumidor

```java
package example.micronaut;

import io.micronaut.configuration.kafka.annotation.KafkaListener;
import io.micronaut.configuration.kafka.annotation.Topic;
import io.micronaut.context.annotation.Requires;
import io.micronaut.context.env.Environment;

@Requires(notEnv = Environment.TEST)
@KafkaListener
public class AnalyticsListener {

    private final AnalyticsService analyticsService;

    public AnalyticsListener(AnalyticsService analyticsService) {
        this.analyticsService = analyticsService;
    }

    @Topic("analytics")
    public void updateAnalytics(Book book) {
        analyticsService.updateBookAnalytics(book);
    }
}

```
