# Http server

## Expondo um health endpoint para seu aplicativo Micronaut


Neste guia, criaremos um aplicativo Micronaut escrito em Java.

Você aprenderá a usar o recurso Micronaut Management para habilitar o endpoint "health" do seu aplicativo.


## O que é o Micronaut Management?

O Micronaut Management é um recurso que fornece endpoints de gerenciamento para aplicativos Micronaut. Esses endpoints podem ser usados para monitorar e gerenciar o aplicativo em tempo de execução. O endpoint "health" é um dos mais comuns e fornece informações sobre a saúde do aplicativo, como se ele está em execução e se todos os componentes estão funcionando corretamente.

```xml
<dependency>
    <groupId>io.micronaut</groupId>
    <artifactId>micronaut-management</artifactId>
    <scope>compile</scope>
</dependency>

```

## Teste do endpoint de health


```java
package example.micronaut;

import io.micronaut.http.HttpRequest;
import io.micronaut.http.HttpStatus;
import io.micronaut.http.client.HttpClient;
import io.micronaut.http.client.annotation.Client;
import io.micronaut.test.extensions.junit5.annotation.MicronautTest;
import org.junit.jupiter.api.Test;

import jakarta.inject.Inject;

import static org.junit.jupiter.api.Assertions.assertEquals;

@MicronautTest
public class HealthTest {

    @Inject
    @Client("/")
    HttpClient client;

    @Test
    public void healthEndpointExposed() {
        HttpStatus status = client.toBlocking().retrieve(HttpRequest.GET("/health"), HttpStatus.class);
        assertEquals(HttpStatus.OK, status);
    }
}
```

### path base

O caminho base para todos os endpoints é /o padrão. Se preferir que os endpoints de gerenciamento estejam disponíveis em um caminho base diferente, configure endpoints.all.path como no teste a seguir.

> O início e o fim /são necessários para endpoints.all.path, a menos que micronaut.server.context-path esteja definido, nesse caso, apenas o início /não é necessário.

```java

package example.micronaut;

import io.micronaut.http.HttpRequest;
import io.micronaut.http.HttpStatus;
import io.micronaut.http.client.HttpClient;
import io.micronaut.http.client.exceptions.HttpClientResponseException;
import io.micronaut.http.client.annotation.Client;
import io.micronaut.test.extensions.junit5.annotation.MicronautTest;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.function.Executable;
import io.micronaut.context.annotation.Property;
import jakarta.inject.Inject;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertThrows;

@Property(name = "endpoints.all.path", value = "/endpoints/")
@MicronautTest
public class HealthPathTest {

    @Inject
    @Client("/")
    HttpClient client;

    @Test
    public void healthEndpointExposedAtNonDefaultEndpointsPath() {
        HttpStatus status = client.toBlocking().retrieve(HttpRequest.GET("/endpoints/health"), HttpStatus.class);
        assertEquals(HttpStatus.OK, status);

        Executable e = () -> client.toBlocking().retrieve(HttpRequest.GET("/health"), HttpStatus.class);
        HttpClientResponseException thrown = assertThrows(HttpClientResponseException.class, e);
        assertEquals(HttpStatus.NOT_FOUND, thrown.getStatus());
    }
}

 ```

 ### Failed Health Status

O limite de espaço em disco é um dos indicadores integrados. Este teste demonstra um status de integridade com falha quando o espaço livre em disco cai abaixo do limite especificado. A endpoints.health.disk-space.threshold propriedade de configuração pode ser fornecida como uma string, como "10 MB" ou "200 KB", ou o número de bytes.

```java

package example.micronaut;

import io.micronaut.context.annotation.Property;
import io.micronaut.http.HttpRequest;
import io.micronaut.http.HttpStatus;
import io.micronaut.http.client.HttpClient;
import io.micronaut.http.client.annotation.Client;
import io.micronaut.http.client.exceptions.HttpClientResponseException;
import io.micronaut.test.extensions.junit5.annotation.MicronautTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.function.Executable;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertThrows;
import static org.junit.jupiter.api.Assertions.assertTrue;

@Property(name = "endpoints.health.disk-space.threshold", value = "999999999999999999")
@MicronautTest
public class PoorHealthTest {

    @Inject
    @Client("/")
    HttpClient client;

    @Test
    public void healthEndpointExposesOutOfDiscSpace() {

        Executable e = () -> client.toBlocking().retrieve(HttpRequest.GET("/health"));
        HttpClientResponseException thrown = assertThrows(HttpClientResponseException.class, e);

        assertEquals(HttpStatus.SERVICE_UNAVAILABLE, thrown.getStatus());
        assertTrue(thrown.getResponse().getBody(String.class).orElse("").contains("DOWN"));
    }
}

```
