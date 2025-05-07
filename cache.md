# Cache

Neste guia, você usará as anotações de cache do Micronaut para acelerar seu aplicativo.


```xml
<dependency>
    <groupId>io.micronaut.cache</groupId>
    <artifactId>micronaut-cache-caffeine</artifactId>
    <scope>compile</scope>
</dependency>

 ```

```yml
micronaut:
  caches:
    headlines:
      charset: 'UTF-8'

```

```java

package example.micronaut;

import io.micronaut.cache.annotation.CacheConfig;
import io.micronaut.cache.annotation.CacheInvalidate;
import io.micronaut.cache.annotation.CachePut;
import io.micronaut.cache.annotation.Cacheable;

import jakarta.inject.Singleton;
import java.time.Month;
import java.util.ArrayList;
import java.util.Arrays;
import java.util.Collections;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.concurrent.TimeUnit;

@Singleton
@CacheConfig("headlines")
public class NewsService {

    Map<Month, List<String>> headlines = new HashMap<Month, List<String>>() {{
        put(Month.NOVEMBER, Arrays.asList("Micronaut Graduates to Trial Level in Thoughtworks technology radar Vol.1",
                "Micronaut AOP: Awesome flexibility without the complexity"));
        put(Month.OCTOBER, Collections.singletonList("Micronaut AOP: Awesome flexibility without the complexity"));
    }};

    @Cacheable
    public List<String> headlines(Month month) {
        try {
            TimeUnit.SECONDS.sleep(3);
            return headlines.get(month);
        } catch (InterruptedException e) {
            return null;
        }
    }

    @CachePut(parameters = {"month"})
    public List<String> addHeadline(Month month, String headline) {
        if (headlines.containsKey(month)) {
            List<String> l = new ArrayList<>(headlines.get(month));
            l.add(headline);
            headlines.put(month, l);
        } else {
            headlines.put(month, Arrays.asList(headline));
        }
        return headlines.get(month);
    }

    @CacheInvalidate(parameters = {"month"})
    public void removeHeadline(Month month, String headline) {
        if (headlines.containsKey(month)) {
            List<String> l = new ArrayList<>(headlines.get(month));
            l.remove(headline);
            headlines.put(month, l);
        }
    }
}

```

1. **Use `jakarta.inject.Singleton`** para designar uma classe como singleton.

2. **Especifica o nome do cache `headlines`** para armazenar os valores da operação de cache.

3. **Indica que um método pode ser armazenado em cache.** O nome de cache `headlines` especificado em `@CacheConfig` é usado. Como o método tem apenas um parâmetro, você não precisa especificar o `month` atributo "parameters" da anotação.

4. **Emule uma operação cara** dormindo por alguns segundos.

5. **O valor de retorno é armazenado em cache** com o nome `headlines` do método fornecido `month`. A invocação do método nunca é ignorada, mesmo que o cache `headlines` do método fornecido `month` já exista.

6. **A invocação do método causa a invalidação do cache** `headlines` para o fornecido `month`.


### Testando o cache

```java

package example.micronaut;

import io.micronaut.test.extensions.junit5.annotation.MicronautTest;
import org.junit.jupiter.api.MethodOrderer;
import org.junit.jupiter.api.Order;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.TestMethodOrder;
import org.junit.jupiter.api.Timeout;

import jakarta.inject.Inject;
import java.time.Month;
import java.util.List;

import static org.junit.jupiter.api.Assertions.assertDoesNotThrow;
import static org.junit.jupiter.api.Assertions.assertEquals;

@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
@MicronautTest(startApplication = false)
class NewsServiceTest {

    @Inject
    NewsService newsService;

    @Timeout(4)
    @Test
    @Order(1)
    public void firstInvocationOfNovemberDoesNotHitCache() {
        List<String> headlines = newsService.headlines(Month.NOVEMBER);
        assertEquals(2, headlines.size());
    }

    @Timeout(1)
    @Test
    @Order(2)
    public void secondInvocationOfNovemberHitsCache() {
        List<String> headlines = newsService.headlines(Month.NOVEMBER);
        assertEquals(2, headlines.size());
    }

    @Timeout(4)
    @Test
    @Order(3)
    public void firstInvocationOfOctoberDoesNotHitCache() {
        List<String> headlines = newsService.headlines(Month.OCTOBER);
        assertEquals(1, headlines.size());
    }

    @Timeout(1)
    @Test
    @Order(4)
    public void secondInvocationOfOctoberHitsCache() {
        List<String> headlines = newsService.headlines(Month.OCTOBER);
        assertEquals(1, headlines.size());
    }

    @Timeout(1)
    @Test
    @Order(5)
    public void addingAHeadlineToNovemberUpdatesCache() {
        List<String> headlines = newsService.addHeadline(Month.NOVEMBER, "Micronaut 1.3 Milestone 1 Released");
        assertEquals(3, headlines.size());
    }

    @Timeout(1)
    @Test
    @Order(6)
    public void novemberCacheWasUpdatedByCachePutAndThusTheValueIsRetrievedFromTheCache() {
        List<String> headlines = newsService.headlines(Month.NOVEMBER);
        assertEquals(3, headlines.size());
    }

    @Timeout(1)
    @Test
    @Order(7)
    public void invalidateNovemberCacheWithCacheInvalidate() {
        assertDoesNotThrow(() -> {
            newsService.removeHeadline(Month.NOVEMBER, "Micronaut 1.3 Milestone 1 Released");
        });
    }

    @Timeout(1)
    @Test
    @Order(8)
    public void octoberCacheIsStillValid() {
        List<String> headlines = newsService.headlines(Month.OCTOBER);
        assertEquals(1, headlines.size());
    }

    @Timeout(4)
    @Test
    @Order(9)
    public void novemberCacheWasInvalidated() {
        List<String> headlines = newsService.headlines(Month.NOVEMBER);
        assertEquals(2, headlines.size());
    }
}

```

Quando a chamada ocorre pela primeira vez, o cache não existe e o método é executado. Na segunda chamada, o cache já existe e o método não é executado. O tempo de execução do teste é reduzido de 4 segundos para 1 segundo, pois o método `headlines()` não executa o `TimeUnit.SECONDS.sleep(3)`. O teste falha se o tempo de execução for maior que 1 segundo.

### Controlando o tempo de execução do teste

```java

package example.micronaut;

import io.micronaut.serde.annotation.Serdeable;

import java.time.Month;
import java.util.List;

@Serdeable
public class News {
    private Month month;

    private List<String> headlines;

    public News() {

    }

    public News(Month month, List<String> headlines) {
        this.month = month;
        this.headlines = headlines;
    }

    public Month getMonth() {
        return month;
    }

    public void setMonth(Month month) {
        this.month = month;
    }

    public List<String> getHeadlines() {
        return headlines;
    }

    public void setHeadlines(List<String> headlines) {
        this.headlines = headlines;
    }
}

package example.micronaut;

import io.micronaut.http.annotation.Controller;
import io.micronaut.http.annotation.Get;

import java.time.Month;

@Controller
public class NewsController {

    private final NewsService newsService;

    public NewsController(NewsService newsService) {
        this.newsService = newsService;
    }

    @Get("/{month}")
    public News index(Month month) {
        return new News(month, newsService.headlines(month));
    }
}

```
### Teste do controlador

```java

package example.micronaut;

import io.micronaut.http.HttpRequest;
import io.micronaut.http.client.HttpClient;
import io.micronaut.http.client.annotation.Client;
import io.micronaut.http.uri.UriBuilder;
import io.micronaut.runtime.server.EmbeddedServer;
import io.micronaut.test.extensions.junit5.annotation.MicronautTest;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.Timeout;

import jakarta.inject.Inject;
import java.time.Month;
import java.util.Arrays;

import static org.junit.jupiter.api.Assertions.assertEquals;

@MicronautTest
public class NewsControllerTest {

    @Inject
    EmbeddedServer server;

    @Inject
    @Client("/")
    HttpClient client;

    @Timeout(5)
    @Test
    void fetchingOctoberHeadlinesUsesCache() {
        HttpRequest request = HttpRequest.GET(UriBuilder.of("/").path(Month.OCTOBER.name()).build());
        News news = client.toBlocking().retrieve(request, News.class);
        String expected = "Micronaut AOP: Awesome flexibility without the complexity";
        assertEquals(Arrays.asList(expected), news.getHeadlines());

        news = client.toBlocking().retrieve(request, News.class);
        assertEquals(Arrays.asList(expected), news.getHeadlines());
    }
}
```
