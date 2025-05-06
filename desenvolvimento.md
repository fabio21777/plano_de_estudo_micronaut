# Desenvolvimento

## Gestão

Inspirada no Spring Boot e no Grails, a dependência de gerenciamento do Micronaut adiciona suporte ao monitoramento do seu aplicativo por meio de endpoints: URIs especiais que retornam detalhes sobre a saúde e o estado do seu aplicativo.

Para usar os recursos de gerenciamento descritos nesta seção, adicione a dependência no seu classpath.

```xml
<dependency>
    <groupId>io.micronaut</groupId>
    <artifactId>micronaut-management</artifactId>
    <scope>compile</scope>
</dependency>

```

```yml

endpoints:
  info:
    enabled: true
    sensitive: false


```

### Plug-in de ID de confirmação do Maven Git

O plug-in de ID de confirmação do Maven Git é um plug-in do Maven que fornece uma maneira fácil de adicionar informações de controle de versão ao seu projeto. Ele pode ser usado para gerar automaticamente o número da versão do seu projeto com base no número de confirmação do Git.

```xml
<properties>
   ...
   <gitCommitIdPlugin.version>4.0.5</gitCommitIdPlugin.version>
</properties>

...

<plugin>
    <groupId>pl.project13.maven</groupId>
    <artifactId>git-commit-id-plugin</artifactId>
    <version>${gitCommitIdPlugin.version}</version>
    <executions>
        <execution>
            <id>get-the-git-infos</id>
            <goals>
                <goal>revision</goal>
            </goals>
            <phase>compile</phase>
        </execution>
    </executions>
    <configuration>
        <generateGitPropertiesFile>true</generateGitPropertiesFile>
    </configuration>
</plugin>
```

#### Teste

```java

package example.micronaut;

import io.micronaut.http.HttpRequest;
import io.micronaut.http.HttpResponse;
import io.micronaut.http.client.HttpClient;
import io.micronaut.http.client.annotation.Client;
import io.micronaut.test.extensions.junit5.annotation.MicronautTest;
import org.junit.jupiter.api.Test;

import jakarta.inject.Inject;
import java.util.Map;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertNotNull;

@MicronautTest
public class InfoTest {

    @Inject
    @Client("/")
    HttpClient client;

    @Test
    public void testGitComitInfoAppearsInJson() {
        HttpRequest request = HttpRequest.GET("/info");

        HttpResponse<Map> rsp = client.toBlocking().exchange(request, Map.class);

        assertEquals(200, rsp.status().getCode());

        Map json = rsp.body();

        assertNotNull(json.get("git"));
        assertNotNull(((Map) json.get("git")).get("commit"));
        assertNotNull(((Map) ((Map) json.get("git")).get("commit")).get("message"));
        assertNotNull(((Map) ((Map) json.get("git")).get("commit")).get("time"));
        assertNotNull(((Map) ((Map) json.get("git")).get("commit")).get("id"));
        assertNotNull(((Map) ((Map) json.get("git")).get("commit")).get("user"));
        assertNotNull(((Map) json.get("git")).get("branch"));
    }
}
```

## [Using IntelliJ IDEA to develop Micronaut applications](https://guides.micronaut.io/latest/micronaut-intellij-idea-ide-setup-maven-java.html)


