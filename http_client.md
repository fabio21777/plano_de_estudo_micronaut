# http cliente

## Micronaut HTTP Client


Neste guia, criaremos um aplicativo Micronaut escrito em Java para consumir a API do GitHub com o cliente HTTP Micronaut.

### Dependências

```xml
<dependency>
    <groupId>io.micronaut</groupId>
    <artifactId>micronaut-http-client</artifactId>
    <scope>compile</scope>
</dependency>

<dependency>
    <groupId>io.micronaut</groupId>
    <artifactId>micronaut-http-client-jdk</artifactId>
    <scope>compile</scope>
</dependency>

```


### API do GitHub

Neste guia, você consumirá a API do GitHub de um aplicativo Micronaut.

Neste guia, você buscará as versões do Micronaut Core por meio do endpoint Listar versões .

Isso retorna uma lista de lançamentos, que não inclui tags regulares do Git que não foram associadas a um lançamento.

Este recurso de API pode ser consumido por clientes autenticados e anônimos. Inicialmente, você o consumirá anonimamente e, posteriormente, discutiremos a autenticação.

Crie um registro para analisar a resposta JSON em um objeto:

```java
package example.micronaut;

import io.micronaut.serde.annotation.Serdeable;

@Serdeable
public record GithubRelease(String name, String url) {
}

```

### Configuração

github.organization=micronaut-projects
github.repo=micronaut-core

```yml
micronaut:
  github:
	organization: ${GITHUB_ORGANIZATION:micronaut-projects}
	repo: ${GITHUB_REPO:micronaut-core}
```

```java
package example.micronaut;

import io.micronaut.context.annotation.ConfigurationProperties;
import io.micronaut.context.annotation.Requires;
import io.micronaut.core.annotation.Nullable;

@ConfigurationProperties(GithubConfiguration.PREFIX)
@Requires(property = GithubConfiguration.PREFIX)
public record GithubConfiguration(String organization,
                                  String repo,
                                  @Nullable String username,
                                  @Nullable String token) {
    public static final String PREFIX = "github";
}
```

micronaut.codec.json.additional-types[0]=application/vnd.github.v3+json

Adicione configuração para tratar application/vnd.github.v3+jsoncomo JSON.

```yml
micronaut:
  codec:
	json:
	  additional-types:
		- application/vnd.github.v3+json
```

#### Cliente de baixo nível

```java
package example.micronaut;

import io.micronaut.core.type.Argument;
import io.micronaut.http.HttpRequest;
import io.micronaut.http.client.HttpClient;
import io.micronaut.http.client.annotation.Client;
import io.micronaut.http.uri.UriBuilder;
import jakarta.inject.Singleton;
import org.reactivestreams.Publisher;

import java.net.URI;
import java.util.List;

import static io.micronaut.http.HttpHeaders.ACCEPT;
import static io.micronaut.http.HttpHeaders.USER_AGENT;

@Singleton//1
public class GithubLowLevelClient {

    private final HttpClient httpClient;
    private final URI uri;

    public GithubLowLevelClient(@Client(id = "github") HttpClient httpClient,//2
                                GithubConfiguration configuration) {//3
        this.httpClient = httpClient;
        uri = UriBuilder.of("/repos")
                .path(configuration.organization())
                .path(configuration.repo())
                .path("releases")
                .build();
    }

    Publisher<List<GithubRelease>> fetchReleases() {
        HttpRequest<?> req = HttpRequest.GET(uri)//4
                .header(USER_AGENT, "Micronaut HTTP Client")//5
                .header(ACCEPT, "application/vnd.github.v3+json, application/json");//6
        return httpClient.retrieve(req, Argument.listOf(GithubRelease.class));//7
    }
}
```

1. O cliente é um singleton, o que significa que ele será criado uma vez e reutilizado em todo o aplicativo.
2. O cliente HTTP é injetado no construtor. O cliente HTTP é um cliente de baixo nível que pode ser usado para fazer solicitações HTTP.
3. O URI é construído com base na configuração do GitHub.
4. O URI é usado para criar uma solicitação GET.
5. O cabeçalho USER_AGENT é adicionado à solicitação.
6. O GitHub recomenda solicitar explicitamente a versão 3 por meio do Acceptcabeçalho. Com @Header, você adiciona o Accept: application/vnd.github.v3+jsoncabeçalho HTTP a cada solicitação.
7. O cliente HTTP recupera a resposta e a analisa em uma lista de objetos GithubRelease. O tipo de retorno é Publisher<List<GithubRelease>>. Isso significa que o cliente HTTP retornará um Publisher que emitirá uma lista de objetos GithubRelease.


### Cliente Declarativo

```java

package example.micronaut;

import io.micronaut.core.async.annotation.SingleResult;
import io.micronaut.http.annotation.Get;
import io.micronaut.http.annotation.Header;
import io.micronaut.http.client.annotation.Client;
import org.reactivestreams.Publisher;

import java.util.List;

import static io.micronaut.http.HttpHeaders.ACCEPT;
import static io.micronaut.http.HttpHeaders.USER_AGENT;

@Client(id = "github")//1
@Header(name = USER_AGENT, value = "Micronaut HTTP Client")//2
@Header(name = ACCEPT, value = "application/vnd.github.v3+json, application/json")//3
public interface GithubApiClient {

    @Get("/repos/${github.organization}/${github.repo}/releases")//2
    @SingleResult//4
    Publisher<List<GithubRelease>> fetchReleases();//5
}

```

1. O cliente é anotado com @Client, o que significa que ele é um cliente HTTP declarativo. Isso significa que você pode usar anotações para definir a solicitação HTTP.
2. O cabeçalho USER_AGENT é adicionado à solicitação.
3. O cabeçalho Accept é adicionado à solicitação.
4. Você pode usar a interpolação de parâmetros de configuração ao definir o caminho do ponto de extremidade GET.
5. Anotação para descrever que uma API emite um único resultado mesmo que o tipo de retorno seja um org.reactivestreams.Publisher.
6. Você pode retornar qualquer tipo reativo de qualquer implementação (RxJava, Reactor…​), mas é melhor usar as interfaces públicas do Reactive Streams como Publisher

### Controlador

Crie um Controlador. Ele utiliza ambos (clientes de baixo nível e declarativos). O framework Micronaut suporta implementações de Reactive Streams, como RxJava ou Project Reactor . Assim, você pode compor com eficiência múltiplas chamadas de clientes HTTP sem bloqueios (o que limitará a taxa de transferência e a escalabilidade da sua aplicação).

```java
package example.micronaut;

import io.micronaut.core.async.annotation.SingleResult;
import io.micronaut.http.annotation.Controller;
import io.micronaut.http.annotation.Get;
import org.reactivestreams.Publisher;
import java.util.List;

@Controller("/github")
public class GithubController {

    private final GithubLowLevelClient githubLowLevelClient;
    private final GithubApiClient githubApiClient;
    public GithubController(GithubLowLevelClient githubLowLevelClient,
                            GithubApiClient githubApiClient) {
        this.githubLowLevelClient = githubLowLevelClient;
        this.githubApiClient = githubApiClient;
    }

    @Get("/releases-lowlevel")
    @SingleResult
    Publisher<List<GithubRelease>> releasesWithLowLevelClient() {
        return githubLowLevelClient.fetchReleases();
    }

    @Get("/releases")
    @SingleResult
    Publisher<List<GithubRelease>> fetchReleases() {
        return githubApiClient.fetchReleases();
    }
}
```

1. A anotação `@Controller("/github")` define a classe como um controlador Micronaut com o caminho base "/github".

2. O construtor da classe `GithubController` utiliza injeção de dependência para receber instâncias de `GithubLowLevelClient` e `GithubApiClient`.

3. A anotação `@Get("/releases-lowlevel")` define um endpoint HTTP GET no caminho "/github/releases-lowlevel".

4. A anotação `@SingleResult` indica que o Publisher retornado emitirá apenas um único resultado e então completará.

5. A anotação `@Get("/releases")` define outro endpoint HTTP GET no caminho "/github/releases".

6. O método `fetchReleases()` implementa o endpoint usando o cliente `githubApiClient` para buscar dados de releases do GitHub.

Cada item corresponde a um elemento significativo na implementação do controlador que gerencia requisições relacionadas ao GitHub.

### Testes

```java
package example.micronaut;

import io.micronaut.context.ApplicationContext;
import io.micronaut.context.annotation.Requires;
import io.micronaut.core.io.ResourceLoader;
import io.micronaut.core.type.Argument;
import io.micronaut.http.HttpRequest;
import io.micronaut.http.HttpResponse;
import io.micronaut.http.HttpStatus;
import io.micronaut.http.annotation.Controller;
import io.micronaut.http.annotation.Get;
import io.micronaut.http.annotation.Produces;
import io.micronaut.http.client.BlockingHttpClient;
import io.micronaut.http.client.HttpClient;
import io.micronaut.runtime.server.EmbeddedServer;
import org.junit.jupiter.api.Test;

import java.io.IOException;
import java.util.Collections;
import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.regex.Pattern;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertNotNull;
import static org.junit.jupiter.api.Assertions.assertTrue;
import static java.nio.charset.StandardCharsets.UTF_8;

class GithubControllerTest {
    private static Pattern MICRONAUT_RELEASE =
            Pattern.compile("Micronaut (Core |Framework )?v?\\d+.\\d+.\\d+( (RC|M)\\d)?");

    @Test
    void verifyGithubReleasesCanBeFetchedWithLowLevelHttpClient() {
        EmbeddedServer github = ApplicationContext.run(EmbeddedServer.class,
                Map.of("micronaut.codec.json.additional-types", "application/vnd.github.v3+json",
                        "spec.name", "GithubControllerTest"));
        EmbeddedServer embeddedServer = ApplicationContext.run(EmbeddedServer.class,
                Collections.singletonMap("micronaut.http.services.github.url",
                        "http://localhost:" + github.getPort()));
        HttpClient httpClient = embeddedServer.getApplicationContext()
                .createBean(HttpClient.class, embeddedServer.getURL());
        BlockingHttpClient client = httpClient.toBlocking();
        assertReleases(client, "/github/releases");
        assertReleases(client, "/github/releases-lowlevel");
        httpClient.close();
        embeddedServer.close();
        github.close();
    }

    private static void assertReleases(BlockingHttpClient client, String path) {
        HttpRequest<Object> request = HttpRequest.GET(path);

        HttpResponse<List<GithubRelease>> rsp = client.exchange(request,
                Argument.listOf(GithubRelease.class));

        assertEquals(HttpStatus.OK, rsp.getStatus());
        assertReleases(rsp.body());
    }

    private static void assertReleases(List<GithubRelease> releases) {
        assertNotNull(releases);
        assertTrue(releases.stream()
                .map(GithubRelease::name)
                .allMatch(name -> MICRONAUT_RELEASE.matcher(name)
                        .find()));
    }

    @Requires(property = "spec.name", value = "GithubControllerTest")
    @Controller
    static class GithubReleases {
        private final ResourceLoader resourceLoader;
        GithubReleases(ResourceLoader resourceLoader) {
            this.resourceLoader = resourceLoader;
        }

        @Produces("application/vnd.github.v3+json")
        @Get("/repos/micronaut-projects/micronaut-core/releases")
        Optional<String> coreReleases() {
            return resourceLoader.getResourceAsStream("releases.json")
                    .flatMap(inputStream -> {
                        try {
                            return Optional.of(new String(inputStream.readAllBytes(), UTF_8));
                        } catch (IOException e) {
                            return Optional.empty();
                        }
                    });
        }
    }
}
 ```

### Filtro do cliente HTTP

Frequentemente, você precisa incluir os mesmos cabeçalhos HTTP ou parâmetros de URL em um conjunto de solicitações para uma API de terceiros ou ao chamar outro microsserviço. Para simplificar isso, o framework Micronaut inclui a capacidade de definir HttpClientFilterclasses que são aplicadas a todos os clientes HTTP correspondentes.

Como exemplo prático, vamos fornecer a Autenticação do GitHub por meio de um HttpClientFilter.

Criar um filtro:

```java
package example.micronaut;

import io.micronaut.context.annotation.Requires;
import io.micronaut.http.MutableHttpRequest;
import io.micronaut.http.annotation.ClientFilter;
import io.micronaut.http.annotation.RequestFilter;

@ClientFilter("/repos/**")
@Requires(property = GithubConfiguration.PREFIX + ".username")
@Requires(property = GithubConfiguration.PREFIX + ".token")
public class GithubFilter {

    private final GithubConfiguration configuration;

    public GithubFilter(GithubConfiguration configuration) {
        this.configuration = configuration;
    }

    @RequestFilter
    public void doFilter(MutableHttpRequest<?> request) {
        request.basicAuth(configuration.username(), configuration.token());
    }
}
```

## Download a big file with StreamingHttpClient


### Cliente HTTP do Micronaut Reactor

Para usar uma implementação do Micronaut HTTP Client com base no Project Reactor , adicione a seguinte dependência:

```xml
<dependency>
	<groupId>io.micronaut.reactor</groupId>
	<artifactId>micronaut-http-client-reactor</artifactId>
	<scope>compile</scope>
</dependency>
```

### Controlador

```java
package example.micronaut;

import io.micronaut.context.exceptions.ConfigurationException;
import io.micronaut.core.io.buffer.ByteBuffer;
import io.micronaut.core.io.buffer.ReferenceCounted;
import io.micronaut.http.HttpRequest;
import io.micronaut.http.annotation.Controller;
import io.micronaut.http.annotation.Get;
import io.micronaut.reactor.http.client.ReactorStreamingHttpClient;
import jakarta.annotation.PreDestroy;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import reactor.core.publisher.Flux;

import java.net.MalformedURLException;
import java.net.URI;
import java.net.URL;

@Controller
class HomeController implements AutoCloseable {
    private static final Logger LOG = LoggerFactory.getLogger(HomeController.class);
    private static final URI DEFAULT_URI = URI.create("https://guides.micronaut.io/micronaut5K.png");

    private final ReactorStreamingHttpClient reactorStreamingHttpClient;

    HomeController() {
        String urlStr = "https://guides.micronaut.io/";
        URL url;
        try {
            url = new URL(urlStr);
        } catch (MalformedURLException e) {
            throw new ConfigurationException("malformed URL" + urlStr);
        }
        this.reactorStreamingHttpClient = ReactorStreamingHttpClient.create(url);
    }

    @Get
    Flux<ByteBuffer<?>> download() {
        HttpRequest<?> request = HttpRequest.GET(DEFAULT_URI);
        return reactorStreamingHttpClient.dataStream(request).doOnNext(bb -> {
            if (bb instanceof ReferenceCounted rc) {
                rc.retain();
            }
        });
    }

    @PreDestroy
    @Override
    public void close() {
        if (reactorStreamingHttpClient != null) {
            reactorStreamingHttpClient.close();
        }
    }
}
```

1. A classe é definida como um controlador com a anotação [@Controller](https://docs.micronaut.io/latest/api/io/micronaut/http/annotation/Controller.html) mapeada para o caminho `/`.

2. [ReactorStreamingHttpClient](https://micronaut-projects.github.io/micronaut-reactor/latest/guide/) é uma variação da interface `StreamingHttpClient` do Projeto Reactor, que estende o HttpClient para oferecer suporte ao streaming de respostas.

3. A anotação [@Get](https://docs.micronaut.io/latest/api/io/micronaut/http/annotation/Get.html) mapeia o método para uma solicitação HTTP GET.

4. O método `dataStream` emite instâncias de `ByteBuffer`. Essas instâncias são coletadas como lixo após a emissão de cada bloco. Para propagar os blocos para o fluxo de resposta, eles precisam ser retidos chamando o método `retain()`. O framework chamará `release()` nesses blocos coletados como lixo assim que cada um for gravado na resposta.

5. Para invocar um método quando o bean é destruído, use a anotação `jakarta.annotation.PreDestroy`.

### Teste

crie um teste que verifique se o hash de download do controlador corresponde ao mesmo hash de arquivo, que também está armazenado em src/test/resources.

```java
package example.micronaut;

import io.micronaut.core.io.ResourceLoader;
import io.micronaut.http.HttpRequest;
import io.micronaut.http.HttpResponse;
import io.micronaut.http.client.BlockingHttpClient;
import io.micronaut.http.client.HttpClient;
import io.micronaut.http.client.annotation.Client;
import io.micronaut.test.extensions.junit5.annotation.MicronautTest;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.condition.DisabledInNativeImage;

import java.io.IOException;
import java.net.URISyntaxException;
import java.net.URL;
import java.nio.file.Files;
import java.nio.file.Path;
import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;
import java.util.HexFormat;
import java.util.Optional;

import static org.junit.jupiter.api.Assertions.assertDoesNotThrow;
import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertTrue;

@MicronautTest
class HomeControllerTest {

    private final HexFormat hexFormat = HexFormat.of();

    @DisabledInNativeImage
    @Test
    void downloadFile(@Client("/") HttpClient httpClient,
                      ResourceLoader resourceLoader)
            throws URISyntaxException, IOException, NoSuchAlgorithmException {
        Optional<URL> resource = resourceLoader.getResource("micronaut5K.png");
        assertTrue(resource.isPresent());
        byte[] expectedByteArray = Files.readAllBytes(Path.of(resource.get().toURI()));

        MessageDigest digest = MessageDigest.getInstance("SHA-256");
        byte[] expectedEncodedHash = digest.digest(expectedByteArray);
        String expected = hexFormat.formatHex(expectedEncodedHash);

        BlockingHttpClient client = httpClient.toBlocking();
        HttpResponse<byte[]> resp = assertDoesNotThrow(() -> client.exchange(HttpRequest.GET("/"), byte[].class));
        byte[] responseBytes = resp.body();

        byte[] responseEncodedHash = digest.digest(responseBytes);
        String response = hexFormat.formatHex(responseEncodedHash);
        assertEquals(expected, response);
    }
}
```
