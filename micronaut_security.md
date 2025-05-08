# Micronaut Security

## Autenticação básica do Micronaut

Neste guia, criaremos um aplicativo Micronaut escrito em Java e o protegeremos com HTTP Basic Auth.

O RFC7617 define o esquema de autenticação "Básico" do Protocolo de Transferência de Hipertexto (HTTP), que transmite credenciais como pares de ID de usuário/senha, codificados usando Base64.

## Provedor de Autenticação

Para manter este guia simples, crie um ingênuo AuthenticationProviderpara simular a autenticação do usuário.

```java
package example.micronaut;

import io.micronaut.core.annotation.NonNull;
import io.micronaut.core.annotation.Nullable;
import io.micronaut.http.HttpRequest;
import io.micronaut.security.authentication.AuthenticationFailureReason;
import io.micronaut.security.authentication.AuthenticationRequest;
import io.micronaut.security.authentication.AuthenticationResponse;
import io.micronaut.security.authentication.provider.HttpRequestAuthenticationProvider;
import jakarta.inject.Singleton;

@Singleton
class AuthenticationProviderUserPassword<B> implements HttpRequestAuthenticationProvider<B> {

    public AuthenticationResponse authenticate(
            @Nullable HttpRequest<B> httpRequest,
            @NonNull AuthenticationRequest<String, String> authenticationRequest
    ) {
        return authenticationRequest.getIdentity().equals("sherlock") && authenticationRequest.getSecret().equals("password")
                ? AuthenticationResponse.success(authenticationRequest.getIdentity())
                : AuthenticationResponse.failure(AuthenticationFailureReason.CREDENTIALS_DO_NOT_MATCH);
    }
}
```

1. O método `authenticate` é chamado quando o usuário tenta fazer login. Ele verifica se o nome de usuário e a senha estão corretos e retorna um objeto `AuthenticationResponse` apropriado.
2. Se o nome de usuário e a senha estiverem corretos, o método retorna `AuthenticationResponse.success(authenticationRequest.getIdentity())`, que indica que a autenticação foi bem-sucedida.

### Controlador

```java
package example.micronaut;

import io.micronaut.http.MediaType;
import io.micronaut.http.annotation.Controller;
import io.micronaut.http.annotation.Get;
import io.micronaut.http.annotation.Produces;
import io.micronaut.security.annotation.Secured;
import io.micronaut.security.rules.SecurityRule;

import java.security.Principal;

@Secured(SecurityRule.IS_AUTHENTICATED) //1
@Controller//2
public class HomeController {

    @Produces(MediaType.TEXT_PLAIN)
    @Get//3
    String index(Principal principal) {//4
        return principal.getName();
    }
}
```

1. O controlador é protegido pela anotação `@Secured(SecurityRule.IS_AUTHENTICATED)`, o que significa que apenas usuários autenticados podem acessá-lo.
2. O controlador é anotado com `@Controller`, o que o torna um controlador REST.
3. O método `index` é mapeado para o caminho raiz ("/") e retorna uma string simples.
4. O método `index` recebe um objeto `Principal`, que contém informações sobre o usuário autenticado. O nome do usuário é retornado como resposta.


### Substitua o Manipulador de Exceções padrão por AuthorizationExceptionHandler, a exceção gerada quando uma solicitação não é autorizada.

```java

package example.micronaut;

import io.micronaut.context.annotation.Replaces;
import io.micronaut.core.annotation.Nullable;
import io.micronaut.http.HttpRequest;
import io.micronaut.http.HttpResponse;
import io.micronaut.http.MutableHttpResponse;
import io.micronaut.http.server.exceptions.response.ErrorResponseProcessor;
import io.micronaut.security.authentication.AuthorizationException;
import io.micronaut.security.authentication.DefaultAuthorizationExceptionHandler;
import io.micronaut.security.config.RedirectConfiguration;
import io.micronaut.security.config.RedirectService;
import io.micronaut.security.errors.PriorToLoginPersistence;
import jakarta.inject.Singleton;

import static io.micronaut.http.HttpHeaders.WWW_AUTHENTICATE;
import static io.micronaut.http.HttpStatus.FORBIDDEN;
import static io.micronaut.http.HttpStatus.UNAUTHORIZED;

@Singleton //1
@Replaces(DefaultAuthorizationExceptionHandler.class)//2
public class DefaultAuthorizationExceptionHandlerReplacement extends DefaultAuthorizationExceptionHandler {

    public DefaultAuthorizationExceptionHandlerReplacement(
            ErrorResponseProcessor<?> errorResponseProcessor,
            RedirectConfiguration redirectConfiguration,
            RedirectService redirectService,
            @Nullable PriorToLoginPersistence priorToLoginPersistence
    ) {
        super(errorResponseProcessor, redirectConfiguration, redirectService, priorToLoginPersistence);
    }

    @Override
    protected MutableHttpResponse<?> httpResponseWithStatus(HttpRequest request,
                                                            AuthorizationException e) {
        if (e.isForbidden()) {
            return HttpResponse.status(FORBIDDEN);
        }
        return HttpResponse.status(UNAUTHORIZED)
                .header(WWW_AUTHENTICATE, "Basic realm=\"Micronaut Guide\"");
    }
}
```

1. A anotação `@Singleton` indica que esta classe é um bean gerenciado pelo Micronaut.
2. A anotação `@Replaces` indica que esta classe substitui a implementação padrão do `DefaultAuthorizationExceptionHandler`.

### Testes

```java
package example.micronaut;

import io.micronaut.http.HttpRequest;
import io.micronaut.http.HttpResponse;
import io.micronaut.http.client.HttpClient;
import io.micronaut.http.client.annotation.Client;
import io.micronaut.http.client.exceptions.HttpClientResponseException;
import io.micronaut.test.extensions.junit5.annotation.MicronautTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.function.Executable;

import static io.micronaut.http.HttpStatus.OK;
import static io.micronaut.http.HttpStatus.UNAUTHORIZED;
import static io.micronaut.http.MediaType.TEXT_PLAIN;
import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertThrows;
import static org.junit.jupiter.api.Assertions.assertTrue;

@MicronautTest
public class BasicAuthTest {

    @Inject
    @Client("/")
    HttpClient client;

    @Test
    void verifyHttpBasicAuthWorks() {
        //when: 'Accessing a secured URL without authenticating'
        Executable e = () -> client.toBlocking().exchange(HttpRequest.GET("/").accept(TEXT_PLAIN));

        // then: 'returns unauthorized'
        HttpClientResponseException thrown = assertThrows(HttpClientResponseException.class, e);
        assertEquals(UNAUTHORIZED, thrown.getStatus());

        assertTrue(thrown.getResponse().getHeaders().contains("WWW-Authenticate"));
        assertEquals("Basic realm=\"Micronaut Guide\"", thrown.getResponse().getHeaders().get("WWW-Authenticate"));

        //when: 'A secured URL is accessed with Basic Auth'
        HttpResponse<String> rsp = client.toBlocking().exchange(HttpRequest.GET("/")
                        .accept(TEXT_PLAIN)
                        .basicAuth("sherlock", "password"),
                String.class);
        //then: 'the endpoint can be accessed'
        assertEquals(OK, rsp.getStatus());
        assertEquals("sherlock", rsp.getBody().get());
    }
}

```

###  Use o cliente HTTP Micronaut e a autenticação básica

Se quiser acessar um ponto de extremidade seguro, você também pode usar um cliente HTTP Micronaut e fornecer a autenticação básica como o valor do cabeçalho de autorização.

Primeiro crie um `@Client` com um método homeque aceite um Authorizationcabeçalho HTTP.

```java
package example.micronaut;

import io.micronaut.http.annotation.Consumes;
import io.micronaut.http.annotation.Get;
import io.micronaut.http.annotation.Header;
import io.micronaut.http.client.annotation.Client;

import static io.micronaut.http.MediaType.TEXT_PLAIN;

@Client("/")
public interface AppClient {

    @Consumes(TEXT_PLAIN)
    @Get
    String home(@Header String authorization);
}
```

1. O método consome texto simples, então a estrutura do Micronaut inclui o cabeçalho HTTP Accept: text/plain.
2. O primeiro caractere do nome do parâmetro é escrito em maiúscula e esse valor ( Authorization) é usado como nome do cabeçalho HTTP. Para alterar o nome do parâmetro, especifique o @Headervalor da anotação.


```java
package example.micronaut;

import io.micronaut.test.extensions.junit5.annotation.MicronautTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.util.Base64;

import static org.junit.jupiter.api.Assertions.assertEquals;

@MicronautTest
public class BasicAuthClientTest {

    @Inject
    AppClient appClient;

    @Test
    void verifyBasicAuthWorks() {
        String creds = basicAuth("sherlock", "password");
        String rsp = appClient.home(creds);
        assertEquals("sherlock", rsp);
    }
    private static String basicAuth(String username, String password) {
        return "Basic " + Base64.getEncoder().encodeToString((username + ":" + password).getBytes());
    }
}
```

###
