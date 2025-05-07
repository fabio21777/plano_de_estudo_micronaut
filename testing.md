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


## Testing REST API integrations using Testcontainers with WireMock or MockServer

Criaremos um aplicativo Micronaut que se comunica com uma API REST externa.

Em seguida, testaremos a integração da API REST externa usando o módulo WireMock do Testcontainers e o MockServer .

### sobre o app

Suponha que estamos criando um aplicativo para gerenciar álbuns de vídeo e usaremos uma API REST de terceiros para gerenciar os ativos de imagem e vídeo. Para este guia, usaremos uma API REST pública jsonplaceholder.typicode.com como um serviço de fotos de terceiros para armazenar fotos do álbum.

Implementaremos um endpoint de API REST para buscar um álbum para o albumId fornecido . Essa API se comunica internamente com o serviço de fotos para buscar as fotos daquele álbum.

Usaremos o WireMock , uma ferramenta para criar APIs simuladas, para simular as interações de serviços externos e testar nossos endpoints de API. O Testcontainers fornece o módulo WireMock do Testcontainers para que possamos executar o WireMock como um contêiner Docker.

```java

package example.micronaut;

import io.micronaut.serde.annotation.Serdeable;

@Serdeable
public record Photo(Long id, String title, String url, String thumbnailUrl) {
}

package example.micronaut;

import io.micronaut.serde.annotation.Serdeable;

import java.util.List;

@Serdeable
public record Album(Long albumId, List<Photo> photos) {
}

package example.micronaut;

import io.micronaut.http.annotation.Get;
import io.micronaut.http.annotation.PathVariable;
import io.micronaut.http.client.annotation.Client;

import java.util.List;

@Client(id = "photosapi")
interface PhotoServiceClient {

    @Get("/albums/{albumId}/photos")
    List<Photo> getPhotos(@PathVariable Long albumId);
}

```
Use @Clientpara usar clientes HTTP declarativos . Você pode anotar interfaces ou classes abstratas. Você pode usar o idmembro para fornecer um identificador de serviço ou especificar a URL diretamente como o valor da anotação.

```java

package example.micronaut;

import io.micronaut.http.annotation.Controller;
import io.micronaut.http.annotation.Get;
import io.micronaut.http.annotation.PathVariable;
import io.micronaut.scheduling.annotation.ExecuteOn;

import static io.micronaut.scheduling.TaskExecutors.BLOCKING;

@Controller("/api")
class AlbumController {

    private final PhotoServiceClient photoServiceClient;

    AlbumController(PhotoServiceClient photoServiceClient) {
        this.photoServiceClient = photoServiceClient;
    }

    @ExecuteOn(BLOCKING)
    @Get("/albums/{albumId}")
    public Album getAlbumById(@PathVariable Long albumId) {
        return new Album(albumId, photoServiceClient.getPhotos(albumId));
    }
}

```

1. A classe é definida como um controlador com a anotação [@Controller](https://docs.micronaut.io/latest/api/io/micronaut/http/annotation/Controller.html) mapeada para o caminho `/api`.

2. Use injeção de construtor para injetar um bean do tipo `PhotoServiceClient`.

3. É essencial que quaisquer operações de E/S de bloqueio (como buscar dados do banco de dados) sejam descarregadas para um pool de threads separado que não bloqueie o loop de eventos.

4. A anotação [@Get](https://docs.micronaut.io/latest/api/io/micronaut/http/annotation/Get.html) mapeia o `getAlbumById` método para uma solicitação HTTP GET em `/albums/{albumId}`.

5. Você pode definir variáveis de caminho com um modelo de URI RFC-6570 no valor da anotação do Método HTTP. O argumento do método pode, opcionalmente, ser anotado com `@PathVariable`.


Nosso aplicativo está expondo um endpoint de API REST GET /api/albums/{albumId} que internamente faz uma chamada de API para https://jsonplaceholder.typicode.com/albums/{albumId}/photos obter fotos daquele álbum e retorna uma resposta semelhante à seguinte:

```json

{
   "albumId": 1,
   "photos": [
       {
           "id": 51,
           "title": "non sunt voluptatem placeat consequuntur rem incidunt",
           "url": "https://via.placeholder.com/600/8e973b",
           "thumbnailUrl": "https://via.placeholder.com/150/8e973b"
       },
       {
           "id": 52,
           "title": "eveniet pariatur quia nobis reiciendis laboriosam ea",
           "url": "https://via.placeholder.com/600/121fa4",
           "thumbnailUrl": "https://via.placeholder.com/150/121fa4"
       },
       ...
       ...
   ]
}

```
### Teste

É melhor simular as interações da API externa no nível do protocolo HTTP em vez de simular o método photoServiceClient.getPhotos(albumId) porque você poderá verificar quaisquer erros de marshaling/unmarshalling, simulando problemas de latência de rede, etc.

Adicione a dependência WireMock Standalone ao seu projeto:

```xml
<dependency>
    <groupId>org.wiremock</groupId>
    <artifactId>wiremock-standalone</artifactId>
    <version>3.0.4</version>
    <scope>test</scope>
</dependency>

```


```java

package example.micronaut;

import com.github.tomakehurst.wiremock.client.WireMock;
import com.github.tomakehurst.wiremock.junit5.WireMockExtension;
import io.micronaut.context.ApplicationContext;
import io.micronaut.http.MediaType;
import io.micronaut.runtime.server.EmbeddedServer;
import io.restassured.RestAssured;
import io.restassured.http.ContentType;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.RegisterExtension;
import org.testcontainers.junit.jupiter.Testcontainers;

import java.util.Collections;
import java.util.Map;

import static com.github.tomakehurst.wiremock.client.WireMock.aResponse;
import static com.github.tomakehurst.wiremock.client.WireMock.urlMatching;
import static com.github.tomakehurst.wiremock.core.WireMockConfiguration.wireMockConfig;
import static io.restassured.RestAssured.given;
import static org.hamcrest.CoreMatchers.is;
import static org.hamcrest.Matchers.hasSize;

@Testcontainers(disabledWithoutDocker = true)
class AlbumControllerTest {

    @RegisterExtension
    static WireMockExtension wireMock = WireMockExtension.newInstance()
            .options(wireMockConfig().dynamicPort())
            .build();

    private Map<String, Object> getProperties() {
        return Collections.singletonMap("micronaut.http.services.photosapi.url",
                wireMock.baseUrl());
    }

    @Test
    void shouldGetAlbumById() {
        try (EmbeddedServer server = ApplicationContext.run(EmbeddedServer.class, getProperties())) {
            RestAssured.port = server.getPort();
            Long albumId = 1L;
            wireMock.stubFor(
                    WireMock.get(urlMatching("/albums/" + albumId + "/photos"))
                            .willReturn(
                                    aResponse()
                                            .withHeader("Content-Type", MediaType.APPLICATION_JSON)
                                            .withBody(
                                                    """
                                                            [
                                                                 {
                                                                     "id": 1,
                                                                     "title": "accusamus beatae ad facilis cum similique qui sunt",
                                                                     "url": "https://via.placeholder.com/600/92c952",
                                                                     "thumbnailUrl": "https://via.placeholder.com/150/92c952"
                                                                 },
                                                                 {
                                                                     "id": 2,
                                                                     "title": "reprehenderit est deserunt velit ipsam",
                                                                     "url": "https://via.placeholder.com/600/771796",
                                                                     "thumbnailUrl": "https://via.placeholder.com/150/771796"
                                                                 }
                                                             ]
                                                            """)));

            given().contentType(ContentType.JSON)
                    .when()
                    .get("/api/albums/{albumId}", albumId)
                    .then()
                    .statusCode(200)
                    .body("albumId", is(albumId.intValue()))
                    .body("photos", hasSize(2));
        }
    }

    @Test
    void shouldReturnServerErrorWhenPhotoServiceCallFailed() {
        try (EmbeddedServer server = ApplicationContext.run(EmbeddedServer.class, getProperties())) {
            RestAssured.port = server.getPort();
            Long albumId = 2L;
            wireMock.stubFor(WireMock.get(urlMatching("/albums/" + albumId + "/photos"))
                    .willReturn(aResponse().withStatus(500)));

            given().contentType(ContentType.JSON)
                    .when()
                    .get("/api/albums/{albumId}", albumId)
                    .then()
                    .statusCode(500);
        }
    }
}

```


1. Desabilite o teste se o Docker não estiver presente.

2. Podemos criar uma instância do servidor WireMock usando **WireMockExtension**.

3. Registramos a propriedade **"micronaut.http.services.photosapi.url"** apontando para a URL do ponto de extremidade do WireMock.

4. No teste **shouldGetAlbumById()**, definimos a resposta simulada esperada para `/albums/{albumId}/photos` a chamada da API, fizemos uma solicitação ao ponto de extremidade do nosso aplicativo `/api/albums/{albumId}` e verificamos a resposta.

5. Estamos usando a biblioteca RestAssured para testar nosso endpoint de API, então capturamos a porta aleatória na qual o aplicativo iniciou e inicializamos **a porta** RestAssured.

6. Defina as expectativas para uma chamada de API.

7. No teste **shouldReturnServerErrorWhenPhotoServiceCallFailed()**, definimos a resposta simulada esperada para `/albums/{albumId}/photos` a chamada de API para retornar o código de status InternalServerError 500 e fazer uma solicitação ao ponto de extremidade do nosso aplicativo `/api/albums/{albumId}` e verificar a resposta.

### Stubbing usando arquivos de mapeamento JSON

```xml

<dependency>
    <groupId>com.github.wiremock</groupId>
    <artifactId>wiremock-testcontainers-java</artifactId>
    <version>1.0-alpha-6</version>
    <scope>test</scope>
</dependency>

```

`src/teste/recursos/wiremock/mapeamentos/obter-fotos-do-álbum.json`

```json

{
  "mappings": [
    {
      "request": {
        "method": "GET",
        "urlPattern": "/albums/([0-9]+)/photos"
      },
      "response": {
        "status": 200,
        "headers": {
          "Content-Type": "application/json"
        },
        "bodyFileName": "album-photos-resp-200.json"
      }
    },
    {
      "request": {
        "method": "GET",
        "urlPattern": "/albums/2/photos"
      },
      "response": {
        "status": 500,
        "headers": {
          "Content-Type": "application/json"
        }
      }
    },
    {
      "request": {
        "method": "GET",
        "urlPattern": "/albums/3/photos"
      },
      "response": {
        "status": 200,
        "headers": {
          "Content-Type": "application/json"
        },
        "jsonBody": []
      }
    }
  ]
}

```


```java

package example.micronaut;

import io.micronaut.context.ApplicationContext;
import io.micronaut.core.annotation.NonNull;
import io.micronaut.runtime.server.EmbeddedServer;
import io.restassured.RestAssured;
import io.restassured.http.ContentType;
import org.junit.jupiter.api.Test;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;
import org.wiremock.integrations.testcontainers.WireMockContainer;

import java.util.Collections;
import java.util.Map;

import static io.restassured.RestAssured.given;
import static org.hamcrest.CoreMatchers.is;
import static org.hamcrest.Matchers.hasSize;
import static org.hamcrest.Matchers.nullValue;

@Testcontainers(disabledWithoutDocker = true)
class AlbumControllerTestcontainersTests {

    @Container
    static WireMockContainer wiremockServer = new WireMockContainer("wiremock/wiremock:2.35.0")
            .withMapping("photos-by-album", AlbumControllerTestcontainersTests.class, "mocks-config.json")
            .withFileFromResource(
                    "album-photos-response.json",
                    AlbumControllerTestcontainersTests.class,
                    "album-photos-response.json");

    @NonNull
    public Map<String, Object> getProperties() {
        return Collections.singletonMap("micronaut.http.services.photosapi.url",
                wiremockServer.getBaseUrl());
    }

    @Test
    void shouldGetAlbumById() {
        Long albumId = 1L;
        try (EmbeddedServer server = ApplicationContext.run(EmbeddedServer.class, getProperties())) {
            RestAssured.port = server.getPort();

            given().contentType(ContentType.JSON)
                    .when()
                    .get("/api/albums/{albumId}", albumId)
                    .then()
                    .statusCode(200)
                    .body("albumId", is(albumId.intValue()))
                    .body("photos", hasSize(2));
        }
    }

    @Test
    void shouldReturnServerErrorWhenPhotoServiceCallFailed() {
        Long albumId = 2L;
        try (EmbeddedServer server = ApplicationContext.run(EmbeddedServer.class, getProperties())) {
            RestAssured.port = server.getPort();
            given().contentType(ContentType.JSON)
                    .when()
                    .get("/api/albums/{albumId}", albumId)
                    .then()
                    .statusCode(500);
        }
    }

    @Test
    void shouldReturnEmptyPhotos() {
        Long albumId = 3L;
        try (EmbeddedServer server = ApplicationContext.run(EmbeddedServer.class, getProperties())) {
            RestAssured.port = server.getPort();
            given().contentType(ContentType.JSON)
                    .when()
                    .get("/api/albums/{albumId}", albumId)
                    .then()
                    .statusCode(200)
                    .body("albumId", is(albumId.intValue()))
                    .body("photos", nullValue());
        }
    }
}

 ```

1. Desabilite o teste se o Docker não estiver presente.
2. Estamos usando as anotações da extensão JUnit 5 do Testcontainers @Container para inicializar o WireMockContainer .
3. Configuramos para carregar mapeamentos de stub do arquivo mocks-config.json

### Testando com o MockServer

```xml
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>mockserver</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.mock-server</groupId>
    <artifactId>mockserver-client-java</artifactId>
    <version>5.15.0</version>
    <scope>test</scope>
</dependency>

```

```java

package example.micronaut;

import static io.restassured.RestAssured.given;
import static org.hamcrest.CoreMatchers.is;
import static org.hamcrest.Matchers.hasSize;
import static org.mockserver.model.HttpRequest.request;
import static org.mockserver.model.HttpResponse.response;
import static org.mockserver.model.JsonBody.json;

import io.micronaut.core.annotation.NonNull;
import io.micronaut.test.extensions.junit5.annotation.MicronautTest;
import io.micronaut.test.support.TestPropertyProvider;
import io.restassured.RestAssured;
import io.restassured.http.ContentType;
import io.restassured.specification.RequestSpecification;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.TestInstance;
import org.mockserver.client.MockServerClient;
import org.mockserver.model.Header;
import org.mockserver.verify.VerificationTimes;
import org.testcontainers.containers.MockServerContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;
import org.testcontainers.utility.DockerImageName;

import java.util.Collections;
import java.util.Map;

@MicronautTest
@TestInstance(TestInstance.Lifecycle.PER_CLASS)
@Testcontainers(disabledWithoutDocker = true)
class AlbumControllerMockServerTest implements TestPropertyProvider {
    @Container
    static MockServerContainer mockServerContainer = new MockServerContainer(
            DockerImageName.parse("mockserver/mockserver:5.15.0")
    );

    static MockServerClient mockServerClient;

    @Override
    public @NonNull Map<String, String> getProperties() {
        mockServerContainer.start();
        mockServerClient =
                new MockServerClient(
                        mockServerContainer.getHost(),
                        mockServerContainer.getServerPort()
                );
        return Collections.singletonMap("micronaut.http.services.photosapi.url",
                mockServerContainer.getEndpoint());
    }

    @BeforeEach
    void setUp() {
        mockServerClient.reset();
    }

    @Test
    void shouldGetAlbumById(RequestSpecification spec) {
        Long albumId = 1L;

        mockServerClient
                .when(request().withMethod("GET").withPath("/albums/" + albumId + "/photos"))
                .respond(
                        response()
                                .withStatusCode(200)
                                .withHeaders(new Header("Content-Type", "application/json; charset=utf-8"))
                                .withBody(
                                        json(
                                                """
                                                [
                                                     {
                                                         "id": 1,
                                                         "title": "accusamus beatae ad facilis cum similique qui sunt",
                                                         "url": "https://via.placeholder.com/600/92c952",
                                                         "thumbnailUrl": "https://via.placeholder.com/150/92c952"
                                                     },
                                                     {
                                                         "id": 2,
                                                         "title": "reprehenderit est deserunt velit ipsam",
                                                         "url": "https://via.placeholder.com/600/771796",
                                                         "thumbnailUrl": "https://via.placeholder.com/150/771796"
                                                     }
                                                 ]
                                                """
                                        )
                                )
                );
        spec
                .contentType(ContentType.JSON)
                .when()
                .get("/api/albums/{albumId}", albumId)
                .then()
                .statusCode(200)
                .body("albumId", is(albumId.intValue()))
                .body("photos", hasSize(2));

        verifyMockServerRequest("GET", "/albums/" + albumId + "/photos", 1);
    }

    private void verifyMockServerRequest(String method, String path, int times) {
        mockServerClient.verify(
                request().withMethod(method).withPath(path),
                VerificationTimes.exactly(times)
        );
    }
}

```
Aprendemos como integrar APIs HTTP de terceiros em um aplicativo Micronaut e testá-lo usando o módulo WireMock ou MockServer do Testcontainer .



## Este guia mostra como substituir H2 por um banco de dados real para testes em um aplicativo Micronaut.

### Neste guia você aprenderá como:

1. Substitua um banco de dados H2 na memória usado para testes pelo mesmo tipo de banco de dados que você usa na produção.
2. Como usar o URL JDBC especial do Testcontainers para usar contêineres de banco de dados.
3. Use a extensão Testcontainers JUnit 5 para inicializar o banco de dados.
4. Teste os repositórios de dados do Micronaut .
5. Simplifique os testes com os recursos de teste do Micronaut

Vamos criar um aplicativo Micronaut de exemplo usando o Micronaut Data JPA e o PostgreSQL .

Analisaremos a prática comum de usar um banco de dados H2 em memória para testes e entenderemos as desvantagens dessa abordagem. Em seguida, aprenderemos como podemos substituí-la testando com o mesmo tipo de banco de dados (PostgreSQL, no nosso caso) que usamos em produção, utilizando o Testcontainers JDBC URL. Por fim, veremos como podemos simplificar os testes com o Micronaut Test Resources .


### Criando Entidade

```java

package example.micronaut;

import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.Id;
import jakarta.persistence.Table;

@Entity
@Table(name = "products")
public class Product {

    @Id
    private Long id;

    @Column(nullable = false, unique = true)
    private String code;

    @Column(nullable = false)
    private String name;

    public Product() {}

    public Product(Long id, String code, String name) {
        this.id = id;
        this.code = code;
        this.name = name;
    }

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public String getCode() {
        return code;
    }

    public void setCode(String code) {
        this.code = code;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
}


import io.micronaut.data.annotation.Query;
import io.micronaut.data.annotation.Repository;
import io.micronaut.data.jpa.repository.JpaRepository;

@Repository
interface ProductRepository extends JpaRepository<Product, Long> {

}

```

### Testando com H2

Uma das abordagens para testar repositórios de banco de dados é usar bancos de dados leves, como H2 ou HSQL , como bancos de dados na memória, enquanto usa um banco de dados diferente, como PostgreSQL , MySQL ou Oracle, na produção.As desvantagens de usar um banco de dados diferente para testes são:

* O banco de dados de teste pode não suportar todos os recursos do seu banco de dados de produção
* A sintaxe da consulta SQL pode não ser compatível com o banco de dados na memória e com o banco de dados de produção.
* Testar com um banco de dados diferente daquele que você usa para produção não lhe dará total confiança no seu conjunto de testes.

Mas ainda assim, bancos de dados na memória como o H2 estão sendo usados ​​predominantemente para testes devido à sua facilidade de uso.

```yml
jpa:
  default:
    properties:
      hibernate:
        hbm2ddl:
          auto: update

datasources:
  default:
    dialect: H2
    driver-class-name: org.h2.Driver
    url: "jdbc:h2:mem:devDb;LOCK_TIMEOUT=10000;DB_CLOSE_ON_EXIT=FALSE"
    username: sa
    password: ""
```

```xml
<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <scope>test</scope>
</dependency>

```

### Seed Data

```sql
insert into products(id, code, name) values(1, 'p101', 'Apple MacBook Pro');
insert into products(id, code, name) values(2, 'p102', 'Sony TV');
```

### ProductRepository Test

Vamos ver como podemos escrever testes para nosso ProductRepository usando H2 .

```java
package example.micronaut;

import io.micronaut.core.io.ResourceLoader;
import io.micronaut.test.annotation.Sql;
import io.micronaut.test.extensions.junit5.annotation.MicronautTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.sql.Connection;
import java.util.List;

import static org.junit.jupiter.api.Assertions.assertEquals;

@MicronautTest
@Sql(scripts = "classpath:sql/seed-data.sql", phase = Sql.Phase.BEFORE_EACH)
class ProductRepositoryTest {

    @Inject
    Connection connection;

    @Inject
    ResourceLoader resourceLoader;

    @Test
    void shouldGetAllProducts(ProductRepository productRepository) {
        List<Product> products = productRepository.findAll();
        assertEquals(2, products.size());
    }
}
```

Mas o desafio surge quando queremos usar recursos suportados apenas pelo nosso banco de dados de produção, mas não pelo banco de dados H2 .

Por exemplo, vamos imaginar que queremos implementar um recurso em que queremos criar um novo produto se um produto com um determinado código ainda não existir; caso contrário, não crie um novo produto.

No PostgreSQL podemos implementar isso usando a seguinte consulta:

```sql
INSERT INTO products(id, code, name) VALUES(?,?,?) ON CONFLICT DO NOTHING;
```

```bash
Caused by: org.h2.jdbc.JdbcSQLException: Syntax error in SQL statement "INSERT INTO products (id, code, name) VALUES (?, ?, ?) ON[*] CONFLICT DO NOTHING";"
```

Você pode executar o H2 com o modo de compatibilidade do PostgreSQL para oferecer suporte à sintaxe do PostgreSQL, mas ainda assim nem todos os recursos são suportados pelo H2 .

O cenário inverso também é possível, onde algumas consultas funcionam bem com H2 , mas não no PostgreSQL. Por exemplo, H2 suporta a função ROWNUM(), enquanto o PostgreSQL não. Portanto, mesmo que você escreva testes para repositórios usando o banco de dados H2 , não há garantia de que seu código funcione da mesma forma com o banco de dados de produção, e você precisará verificar após a implantação do aplicativo, o que anula todo o propósito de escrever testes automatizados.

Agora, vamos ver como é simples substituir o banco de dados H2 por um banco de dados Postgres real para testes usando Testcontainers.


```java

import io.micronaut.data.annotation.Query;
import io.micronaut.data.annotation.Repository;
import io.micronaut.data.jpa.repository.JpaRepository;

@Repository
interface ProductRepository extends JpaRepository<Product, Long> {

    default void createProductIfNotExists(Product product) {
        createProductIfNotExists(product.getId(), product.getCode(), product.getName());
    }

    @Query(
            value = "insert into products(id, code, name) values(:id, :code, :name) ON CONFLICT DO NOTHING",
            nativeQuery = true
    )
    void createProductIfNotExists(Long id, String code, String name);

}

```

### Testando com Testcontainers

### Dependências

```xml

<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>postgresql</artifactId>
    <scope>test</scope>
</dependency>

```

### Configuração

```yml
jpa:
  default:
    properties:
      hibernate:
        hbm2ddl:
          auto: none

datasources:
  default:
    db-type: postgres
    dialect: POSTGRES
    driver-class-name: org.postgresql.Driver

```

Desabilitamos a geração de esquema com jpa.default.properties.hibernate.hbm2ddl.auto=none. Usaremos um script de inicialização classpath com Testcontainers para carregar o seguinte arquivo SQL.

```sql

CREATE TABLE IF NOT EXISTS products
(
    id   int          not null,
    code varchar(255) not null,
    name varchar(255) not null,
    primary key (id),
    unique (code)
    );

```

### Teste

```java

package example.micronaut;

import io.micronaut.context.annotation.Property;
import io.micronaut.core.io.ResourceLoader;
import io.micronaut.test.annotation.Sql;
import io.micronaut.test.extensions.junit5.annotation.MicronautTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.sql.Connection;
import java.util.List;

import static org.junit.jupiter.api.Assertions.assertEquals;

@MicronautTest(startApplication = false) //1
@Property(name = "datasources.default.driver-class-name",
        value = "org.testcontainers.jdbc.ContainerDatabaseDriver") //2
@Property(name = "datasources.default.url",
        value = "jdbc:tc:postgresql:15.2-alpine:///db?TC_INITSCRIPT=sql/init-db.sql") //3
@Sql(scripts = "classpath:sql/seed-data.sql", phase = Sql.Phase.BEFORE_EACH)//4
class ProductRepositoryWithJdbcUrlTest {

    @Inject
    Connection connection;

    @Inject
    ResourceLoader resourceLoader;

    @Test
    void shouldGetAllProducts(ProductRepository productRepository) {
        List<Product> products = productRepository.findAll();
        assertEquals(2, products.size());
    }
}

```

1. **Anote a classe com `@MicronautTest`** para que o framework Micronaut inicialize o contexto da aplicação e o servidor embarcado. Por padrão, cada `@Test` método será encapsulado em uma transação que será revertida quando o teste for concluído. Esse comportamento pode ser alterado configurando `transaction`-o como `false`.

2. **Anote a classe com `@Property`** para fornecer a configuração do nome da classe do driver para o teste.

3. **Anote a classe com `@Property`** para fornecer a URL da fonte de dados que fornecemos por meio [do URL JDBC especial do Testcontainers](https://java.testcontainers.org/modules/databases/jdbc/).

4. **Semeie o banco de dados** usando a [`@Sql`](https://micronaut-projects.github.io/micronaut-test/latest/guide/#sql) anotação do micronaut-test.


Agora, se você executar o teste, poderá ver nos logs do console que nosso teste está usando um banco de dados PostgreSQL em vez do banco de dados H2 na memória. É simples assim!

Vamos entender como esse teste funciona.

Se tivermos Testcontainers e o driver JDBC apropriado no classpath, podemos simplesmente usar os URLs de conexão JDBC especiais para obter uma nova instância em contêiner do banco de dados sempre que o aplicativo for iniciado.

O URL JDBC real do PostgreSQL se parece com: jdbc:postgresql://localhost:5432/postgres

Para obter a URL especial do JDBC, insira tc: após jdbc:, conforme a seguir. (Observe que o nome do host, a porta e o nome do banco de dados serão ignorados; portanto, você pode deixá-los como estão ou defini-los com qualquer valor.)


### Inicializando o contêiner do banco de dados usando Testcontainers e JUnit

Se usar uma URL JDBC especial não atender às suas necessidades, ou se você precisar de mais controle sobre a criação do contêiner, você pode usar a extensão JUnit 5 Testcontainers da seguinte maneira:

```java
package example.micronaut;

import io.micronaut.core.annotation.NonNull;
import io.micronaut.core.io.ResourceLoader;
import io.micronaut.test.annotation.Sql;
import io.micronaut.test.extensions.junit5.annotation.MicronautTest;
import io.micronaut.test.support.TestPropertyProvider;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.TestInstance;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;
import org.testcontainers.utility.MountableFile;

import java.sql.Connection;
import java.util.List;
import java.util.Map;
import java.util.Optional;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertTrue;

@MicronautTest(startApplication = false)
@Testcontainers(disabledWithoutDocker = true)
@TestInstance(TestInstance.Lifecycle.PER_CLASS)
@Sql(scripts = "classpath:sql/seed-data.sql", phase = Sql.Phase.BEFORE_EACH)
class ProductRepositoryTest implements TestPropertyProvider {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>(
            "postgres:15.2-alpine"
    ).withCopyFileToContainer(MountableFile.forClasspathResource("sql/init-db.sql"), "/docker-entrypoint-initdb.d/init-db.sql");

    @Override
    public @NonNull Map<String, String> getProperties() {
        if (!postgres.isRunning()) {
            postgres.start();
        }
        return Map.of("datasources.default.driver-class-name", "org.postgresql.Driver",
                "datasources.default.url", postgres.getJdbcUrl(),
                "datasources.default.username", postgres.getUsername(),
                "datasources.default.password", postgres.getPassword());
    }

    @Inject
    Connection connection;

    @Inject
    ResourceLoader resourceLoader;

    @Inject
    ProductRepository productRepository;

    @Test
    void shouldGetAllProducts() {
        List<Product> products = productRepository.findAll();
        assertEquals(2, products.size());
    }

    @Test
    void shouldNotCreateAProductWithDuplicateCode() {
        Product product = new Product(3L, "p101", "Test Product");
        productRepository.createProductIfNotExists(product);
        Optional<Product> optionalProduct = productRepository.findById(product.getId());
        assertTrue(optionalProduct.isEmpty());
    }
}
```

1. **Anote a classe com `@MicronautTest`** para que o framework Micronaut inicialize o contexto da aplicação e o servidor embarcado. Por padrão, cada `@Test` método será encapsulado em uma transação que será revertida quando o teste for concluído. Esse comportamento pode ser alterado configurando `transaction`-o como `false`.

2. **Desabilite o teste** se o Docker não estiver presente.

3. **As classes que implementam `TestPropertyProvider`** devem usar esta anotação para criar uma única instância de classe para todos os testes (não necessário em testes Spock).

4. **Semeie o banco de dados** usando a [`@Sql`](https://micronaut-projects.github.io/micronaut-test/latest/guide/#sql) anotação do micronaut-test.

5. **Quando precisar definir propriedades dinâmicas**, implemente a `TestPropertyProvider` interface. Sobrescreva o método `.getProperties()` e retorne as propriedades que deseja expor à aplicação.


Usamos as anotações de extensão Testcontainers JUnit 5 @Testcontainers e @Container para iniciar o PostgreSQLContainer e registrar as propriedades da fonte de dados para o teste usando o registro de propriedade dinâmica por meio da API TestPropertyProvider .


### Testando com o Micronaut Test Resources

Quando o aplicativo é iniciado localmente — seja em teste ou executando o aplicativo  — a resolução da URL da fonte de dados é detectada e o serviço Test Resources inicia um contêiner Docker PostgreSQL local e injeta as propriedades necessárias para usá-lo como fonte de dados.

Para obter mais informações, consulte a seção JDBC da documentação de Recursos de Teste .

```java
package example.micronaut;

import io.micronaut.core.io.ResourceLoader;
import io.micronaut.test.annotation.Sql;
import io.micronaut.test.extensions.junit5.annotation.MicronautTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.sql.Connection;
import java.util.List;
import java.util.Optional;

import static org.junit.jupiter.api.Assertions.assertDoesNotThrow;
import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertTrue;

@MicronautTest(startApplication = false)
@Sql(scripts = {"classpath:sql/init-db.sql", "classpath:sql/seed-data.sql"},
        phase = Sql.Phase.BEFORE_EACH)
class ProductRepositoryTest {

    @Inject
    Connection connection;

    @Inject
    ResourceLoader resourceLoader;

    @Inject
    ProductRepository productRepository;

    @Test
    void shouldGetAllProducts() {
        List<Product> products = productRepository.findAll();
        assertEquals(2, products.size());
    }

    @Test
    void shouldNotCreateAProductWithDuplicateCode() {
        Product product = new Product(3L, "p101", "Test Product");
        assertDoesNotThrow(() -> productRepository.createProductIfNotExists(product));
        Optional<Product> optionalProduct = productRepository.findById(product.getId());
        assertTrue(optionalProduct.isEmpty());
    }

}

```

#### Dados JDBC do Micronaut

```java

package example.micronaut;

import io.micronaut.data.annotation.Query;
import io.micronaut.data.jdbc.annotation.JdbcRepository;
import io.micronaut.data.model.query.builder.sql.Dialect;
import io.micronaut.data.repository.CrudRepository;

@JdbcRepository(dialect = Dialect.POSTGRES)
interface ProductRepository extends CrudRepository<Product, Long> {

    default void createProductIfNotExists(Product product) {
        createProductIfNotExists(product.getId(), product.getCode(), product.getName());
    }

    @Query(
            value = "insert into products(id, code, name) values(:id, :code, :name) ON CONFLICT DO NOTHING",
            nativeQuery = true
    )
    void createProductIfNotExists(Long id, String code, String name);
}

```

### Conclusão

Analisamos como testar repositórios JPA do Micronaut Data usando o banco de dados H2 na memória e falamos sobre as desvantagens de usar diferentes bancos de dados (na memória) para testes enquanto usamos um tipo diferente de banco de dados na produção.

Em seguida, aprendemos como é fácil substituir o banco de dados H2 por um banco de dados real para testes usando a URL JDBC especial do Testcontainer. Também vimos como usar as anotações da extensão JUnit 5 do Testcontainer para iniciar o banco de dados para testes, o que proporciona mais controle sobre o ciclo de vida do contêiner do banco de dados.

Aprendemos que o Micronaut Test Resources simplifica os testes com contêineres descartáveis ​​por meio de sua integração com o Testcontainers.
