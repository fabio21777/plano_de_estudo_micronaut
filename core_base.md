# Guia Core base Micronaut


## [@Configuration e @ConfigurationBuilder](https://guides.micronaut.io/latest/micronaut-configuration-maven-java.html)

Aprenda a utilizar as anotações @Configuration e @ConfigurationBuilder para configurar efetivamente as propriedades declaradas.

Neste guia, criaremos um aplicativo Micronaut escrito em Java.

Neste guia, você aprenderá a usar com eficiência as anotações `@ConfigurationProperties`, `@ConfigurationBuilder`e `@EachProperty`as propriedades configuradas em uma aplicação Micronaut. Essas anotações permitem que valores declarados sejam injetados em um bean para facilitar o uso na aplicação.

## Team congigurado via @ConfigurationProperties

```yml

team:
  name: 'Steelers'
  color: 'Black'
  player-names:
    - 'Mason Rudolph'
    - 'James Connor'

```

Com o framework Micronaut, podemos usar a `@ConfigurationProperties` anotação para extrair a configuração em um bean. Cada propriedade que corresponder à configuração no bean `application.yml` chamará o setter no bean. O bean estará posteriormente disponível para injeção na aplicação!


```java

@ConfigurationProperties("team")
public class TeamConfiguration {
    private String name;
    private String color;
    private List<String> playerNames;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getColor() {
        return color;
    }

    public void setColor(String color) {
        this.color = color;
    }

    public List<String> getPlayerNames() {
        return playerNames;
    }

    public void setPlayerNames(List<String> playerNames) {
        this.playerNames = playerNames;
    }
}

```

### Testando a configuração

```java

    @Test
    void testTeamConfiguration() {
        List<String> names = Arrays.asList("Nirav Assar", "Lionel Messi");
        Map<String, Object> items = new HashMap<>();
        items.put("team.name", "evolution");
        items.put("team.color", "green");
        items.put("team.player-names", names);

        ApplicationContext ctx = ApplicationContext.run(items);
        TeamConfiguration teamConfiguration = ctx.getBean(TeamConfiguration.class);

        assertEquals("evolution", teamConfiguration.getName());
        assertEquals("green", teamConfiguration.getColor());
        assertEquals(names.size(), teamConfiguration.getPlayerNames().size());
        names.forEach(name -> assertTrue(teamConfiguration.getPlayerNames().contains(name)));

        ctx.close();
    }
```

### Team Admin Builder com @ConfigurationBuilder

O padrão Builder é uma ótima maneira de construir objetos de configuração incrementalmente.O Framework suporta o padrão Builder com @ConfigurationBuilder.

Suponhamos que queremos adicionar administradores a uma equipe. A administração da equipe é composta por um objeto de padrão construtor. Podemos adicionar um treinador, um gerente e um presidente à equipe.

```yml

team:
  name: 'Steelers'
  color: 'Black'
  player-names:
    - 'Mason Rudolph'
    - 'James Connor'
  team-admin:
    manager: 'Nirav Assar'
    coach: 'Mike Tomlin'
    president: 'Dan Rooney'
```


```java

package example.micronaut;

public class TeamAdmin { //1

    private String manager;
    private String coach;
    private String president;

    // should use the builder pattern to create the object
    private TeamAdmin() {
    }

    public String getManager() {
        return manager;
    }

    public void setManager(String manager) {
        this.manager = manager;
    }

    public String getCoach() {
        return coach;
    }

    public void setCoach(String coach) {
        this.coach = coach;
    }

    public String getPresident() {
        return president;
    }

    public void setPresident(String president) {
        this.president = president;
    }

    public static Builder builder() {//2
        return new Builder();
    }

    public static class Builder {
        private String manager;
        private String coach;
        private String president;

		//3
        public Builder withManager(String manager) {
            this.manager = manager;
            return this;
        }

        public Builder withCoach(String coach) {
            this.coach = coach;
            return this;
        }

        public Builder withPresident(String president) {
            this.president = president;
            return this;
        }

        public TeamAdmin build() { //4
            TeamAdmin teamAdmin = new TeamAdmin();
            teamAdmin.manager = this.manager;
            teamAdmin.coach = this.coach;
            teamAdmin.president = this.president;
            return teamAdmin;
        }

        public String getManager() {
            return manager;
        }

        public String getCoach() {
            return coach;
        }

        public String getPresident() {
            return president;
        }
    }
}
```

1 - *TeamAdmin* é o objeto de configuração que consome as propriedades declaradas.
2 - O objeto construtor é usado para construir o objeto incrementalmente.
3 - Um exemplo de um método construtor, onde um atributo é definido e então o próprio construtor é retornado.
4 - O método final build()cria o TeamAdminobjeto.


### Utilizando o @ConfigurationBuilder

```java
package example.micronaut;

import com.fasterxml.jackson.annotation.JsonIgnoreProperties;
import io.micronaut.context.annotation.ConfigurationBuilder;
import io.micronaut.context.annotation.ConfigurationProperties;
import io.micronaut.serde.annotation.Serdeable;

import java.util.List;

@Serdeable//1
@JsonIgnoreProperties("builder")//2
//tag::teamConfigClassNoBuilder[]
@ConfigurationProperties("team")//3
public class TeamConfiguration {
    private String name;
    private String color;
    private List<String> playerNames;
//end::teamConfigClassNoBuilder[]

    public TeamConfiguration() {
    }

    @ConfigurationBuilder(prefixes = "with", configurationPrefix = "team-admin") //aqui é onde o padrão builder é aplicado //4
    protected TeamAdmin.Builder builder = TeamAdmin.builder(); //5

    public TeamAdmin.Builder getBuilder() {
        return builder;
    }

    public void setBuilder(TeamAdmin.Builder builder) {
        this.builder = builder;
    }

    //tag::gettersandsetters[]
    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getColor() {
        return color;
    }

    public void setColor(String color) {
        this.color = color;
    }

    public List<String> getPlayerNames() {
        return playerNames;
    }

    public void setPlayerNames(List<String> playerNames) {
        this.playerNames = playerNames;
    }
}
//end::gettersandsetters[]

```

1. - Declare a *@Serdeable* anotação no nível do tipo no seu código-fonte para permitir que o tipo seja serializado ou desserializado.
2. - Marcar o construtor como sendo ignorado durante a serialização.
3. - *prefixes* informa ao framework Micronaut para encontrar métodos prefixados por with; *configurationPrefix* permite que o desenvolvedor personalize o *application.yml* elemento
4. - Instancie o objeto construtor para que ele possa ser preenchido com valores de configuração.

```java

    @Test
    void testTeamConfigurationBuilder() {
        List<String> names = Arrays.asList("Nirav Assar", "Lionel Messi");
        Map<String, Object> items = new HashMap<>();
        items.put("team.name", "evolution");
        items.put("team.color", "green");
        items.put("team.team-admin.manager", "Jerry Jones");
        items.put("team.team-admin.coach", "Tommy O'Neill");
        items.put("team.team-admin.president", "Mark Scanell");
        items.put("team.player-names", names);

        ApplicationContext ctx = ApplicationContext.run(items);
        TeamConfiguration teamConfiguration = ctx.getBean(TeamConfiguration.class);
        TeamAdmin teamAdmin = teamConfiguration.builder.build();

        assertEquals("evolution", teamConfiguration.getName());
        assertEquals("green", teamConfiguration.getColor());
        assertEquals("Nirav Assar", teamConfiguration.getPlayerNames().get(0));
        assertEquals("Lionel Messi", teamConfiguration.getPlayerNames().get(1));

        // check the builder has values set
        assertEquals("Jerry Jones", teamConfiguration.builder.getManager());
        assertEquals("Tommy O'Neill", teamConfiguration.builder.getCoach());
        assertEquals("Mark Scanell", teamConfiguration.builder.getPresident());

        // check the object can be built
        assertEquals("Jerry Jones", teamAdmin.getManager());
        assertEquals("Tommy O'Neill", teamAdmin.getCoach());
        assertEquals("Mark Scanell", teamAdmin.getPresident());

        ctx.close();
    }

```

### Stadiums com @EachProperty

O padrão @EachProperty é uma maneira de criar um bean para cada propriedade em um arquivo de configuração. O padrão é útil quando você tem uma lista de propriedades que deseja injetar em um bean.

```yml
stadium:
  coors: //1
    city: 'Denver'
    size: 50000
  pnc: //2
    city: 'Pittsburgh'
    size: 35000
```


```java

package example.micronaut;

import io.micronaut.context.annotation.EachProperty;
import io.micronaut.context.annotation.Parameter;
import io.micronaut.serde.annotation.Serdeable;

@Serdeable
@EachProperty("stadium")
public class StadiumConfiguration {
    private String name;
    private String city;
    private Integer size;

    public StadiumConfiguration(@Parameter String name) {
        this.name = name;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getCity() {
        return city;
    }

    public void setCity(String city) {
        this.city = city;
    }

    public Integer getSize() {
        return size;
    }

    public void setSize(Integer size) {
        this.size = size;
    }
}

```

## Teste

```java

package example.micronaut;

import io.micronaut.context.ApplicationContext;
import io.micronaut.inject.qualifiers.Qualifiers;
import org.junit.jupiter.api.Test;

import java.util.HashMap;
import java.util.Map;

import static org.junit.jupiter.api.Assertions.assertEquals;

public class StadiumConfigurationTest {

    @Test
    void testStadiumConfiguration() {
        Map<String, Object> items = new HashMap<>();
        items.put("stadium.fenway.city", "Boston");
        items.put("stadium.fenway.size", 60000);
        items.put("stadium.wrigley.city", "Chicago");
        items.put("stadium.wrigley.size", 45000);

        ApplicationContext ctx = ApplicationContext.run(items);


        StadiumConfiguration fenwayConfiguration = ctx.getBean(StadiumConfiguration.class, Qualifiers.byName("fenway"));
        StadiumConfiguration wrigleyConfiguration = ctx.getBean(StadiumConfiguration.class, Qualifiers.byName("wrigley"));

        assertEquals("fenway", fenwayConfiguration.getName());
        assertEquals(60000, fenwayConfiguration.getSize());
        assertEquals("wrigley", wrigleyConfiguration.getName());
        assertEquals(45000, wrigleyConfiguration.getSize());

        ctx.close();
    }
}

```

### Controlador

Os beans de configuração podem ser injetados na aplicação como quaisquer outros beans. Como demonstração, crie um controlador onde os beans sejam injetados pelo construtor. A *StadiumConfiguration* classe tem duas instâncias, portanto, para a injeção, precisamos usar a *@Named* anotação com um nome qualificador para especificar o bean.

```java

package example.micronaut;

import io.micronaut.core.annotation.Nullable;
import io.micronaut.http.annotation.Controller;
import io.micronaut.http.annotation.Get;

import jakarta.inject.Named;

@Controller("/my")
public class MyController {

    private final TeamConfiguration teamConfiguration;
    private final StadiumConfiguration stadiumConfiguration;

    public MyController(@Nullable TeamConfiguration teamConfiguration,
                        @Nullable @Named("pnc") StadiumConfiguration stadiumConfiguration) {
        this.teamConfiguration = teamConfiguration;
        this.stadiumConfiguration = stadiumConfiguration;
    }

    @Get("/team")
    public TeamConfiguration team() {
        return this.teamConfiguration;
    }

    @Get("/stadium")
    public  StadiumConfiguration stadium() {
        return this.stadiumConfiguration;
    }
}

```

```java
package example.micronaut;

import io.micronaut.http.HttpRequest;
import io.micronaut.http.client.HttpClient;
import io.micronaut.http.client.annotation.Client;
import io.micronaut.test.extensions.junit5.annotation.MicronautTest;
import org.junit.jupiter.api.Test;

import jakarta.inject.Inject;

import java.util.Arrays;
import java.util.List;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertTrue;

@MicronautTest
public class MyControllerTest {

    @Inject
    @Client("/")
    HttpClient client;

    @Test
    void testMyTeam() {
        TeamConfiguration teamConfiguration = client.toBlocking()
                .retrieve(HttpRequest.GET("/my/team"), TeamConfiguration.class);
        assertEquals("Steelers", teamConfiguration.getName());
        assertEquals("Black", teamConfiguration.getColor());
        List<String> expectedPlayers = Arrays.asList("Mason Rudolph", "James Connor");
        assertEquals(expectedPlayers.size(), teamConfiguration.getPlayerNames().size());
        expectedPlayers.forEach(name -> assertTrue(teamConfiguration.getPlayerNames().contains(name)));
    }

    @Test
    void testMyStadium() {
        StadiumConfiguration conf = client.toBlocking()
                .retrieve(HttpRequest.GET("/my/stadium"), StadiumConfiguration.class);
        assertEquals("Pittsburgh", conf.getCity());
        assertEquals(35000, conf.getSize());
    }
}
```

## [Micronaut Dependency Injection Types](https://guides.micronaut.io/latest/micronaut-dependency-injection-types.html)

Injeção de parâmetros de construtor, campo e método.

O que é uma dependência? A Classe A tem uma dependência da Classe B quando interage com ela de alguma forma. Por exemplo, a Classe A executa um método em uma instância da Classe B. Normalmente, não se instanciam dependências de classe ao usar o mecanismo de injeção de dependências do Micronaut Framework. Em vez disso, confia-se ao framework para fornecer instâncias de dependência.

### jakarta.inject vs javax.inject

As primeiras versões do Micronaut Framework utilizavam *javax.inject* anotações. Devido a restrições de marca registrada impostas ao *javax.*namespace*, o framework Micronaut migrou das anotações *javax.injectpara* para as  *jakarta.inject* anotações. Desde a versão 2.4 , o framework Micronaut suporta *jakarta.inject* anotações. Além disso, o Micronaut Framework migrou para jakarta.injecto conjunto de anotações incluído por padrão, e recomendamos fortemente que você o utilize *jakarta.inject* daqui para frente.

### Injeção de dependência

O Micronaut Framework oferece três tipos de injeção de dependência


### Injeção de construtor

Na injeção baseada em construtor, o Micronaut Framework fornece as dependências necessárias para a classe como argumentos para o construtor.

```java

package example.micronaut.constructor;

import example.micronaut.MessageService;
import io.micronaut.http.MediaType;
import io.micronaut.http.annotation.Controller;
import io.micronaut.http.annotation.Get;
import io.micronaut.http.annotation.Produces;

@Controller("/constructor")
class MessageController {
    private final MessageService messageService;

    MessageController(MessageService messageService) {
        this.messageService = messageService;
    }

    @Get
    @Produces(MediaType.TEXT_PLAIN)
    String index() {
        return messageService.compose();
    }
}

```

### Injeção de campo

Com a injeção de campo, o Micronaut preenche os pontos finais de injeção para campos anotados com jakarta.inject.Inject.

```java

package example.micronaut.field;

import example.micronaut.MessageService;
import io.micronaut.http.MediaType;
import io.micronaut.http.annotation.Controller;
import io.micronaut.http.annotation.Get;
import io.micronaut.http.annotation.Produces;
import jakarta.inject.Inject;

@Controller("/field")
class MessageController {
    @Inject // aqui ocorrerá a injeção
    MessageService messageService;

    @Get
    @Produces(MediaType.TEXT_PLAIN)
    String index() {
        return messageService.compose();
    }
}
```

A injeção de campo dificulta a compreensão dos requisitos de uma classe, facilitando a obtenção de uma resposta NullPointerExceptionao testar uma classe usando injeção de campo. Recomendamos o uso de injeção de construtor .

> O exemplo de código anterior usa um campo com o modificador de acesso default. Existem quatro tipos de modificadores de acesso disponíveis em Java: *private*, *protected*, *public* e o default. A injeção de campo funciona com todos eles. No entanto, a injeção de campo em um campo com *private* modificador de acesso requer reflexão. Portanto, recomendamos que você não use private.
> Use injeção de campo em classes de teste anotadas com MicronautTest. O Micronaut não instancia a classe de teste. Portanto, você não pode usar injeção de construtor nessas classes.
>


### Injeção de método

Para injeção de parâmetros de método, você define um método com um ou mais parâmetros e anota o método com a jakarta.inject.Inject anotação. O Micronaut Framework fornece os parâmetros necessários para o método.

```java

package example.micronaut.methodparameter;

import example.micronaut.MessageService;
import io.micronaut.http.MediaType;
import io.micronaut.http.annotation.Controller;
import io.micronaut.http.annotation.Get;
import io.micronaut.http.annotation.Produces;
import jakarta.inject.Inject;

@Controller("/setter")
class MessageController {
    private MessageService messageService;

    @Get
    @Produces(MediaType.TEXT_PLAIN)
    String index() {
        return messageService.compose();
    }

    @Inject
    void populateMessageService(MessageService messageService) {
        this.messageService = messageService;
    }
}

```

### Benefícios da injeção de construtor

#### Contrato claro

A injeção de construtor expressa claramente os requisitos da classe e não requer nenhuma anotação adicional.

#### Imutabilidade

A injeção de construtor permite definir dependências `final`, criando objetos imutáveis.

#### Identificando odores de código

A injeção de construtor ajuda você a identificar facilmente se seu bean depende de muitos outros objetos.

#### Teste

A injeção de construtor simplifica a escrita de testes unitários e de integração. O construtor nos obriga a fornecer objetos válidos para todas as dependências. Assim, diminui a chance de ocorrência de uma NullPointerException durante os testes.


## [Tipos de escopo Micronaut](https://guides.micronaut.io/latest/micronaut-scope-types-maven-java.html)


O Micronaut apresenta um mecanismo de escopo de bean extensível baseado no JSR-330 .


## Cenario

```java
package example.micronaut.singleton;

import io.micronaut.http.annotation.Controller;
import io.micronaut.http.annotation.Get;

import java.util.Arrays;
import java.util.List;

@Controller("/singleton")

public class RobotController {

    private final RobotFather father; //será injetado como singleton
    private final RobotMother mother; //será injetado como singleton

    public RobotController(RobotFather father,
                           RobotMother mother) {
        this.father = father;
        this.mother = mother;
    }

    @Get
    List<String> children() {
        return Arrays.asList(
                father.child().getSerialNumber(),
                mother.child().getSerialNumber()
        );
    }
}

package example.micronaut.singleton;

import io.micronaut.core.annotation.NonNull;
import jakarta.inject.Singleton;

@Singleton
public class RobotFather {
    private final Robot robot;

    public RobotFather(Robot robot) {
        this.robot = robot;
    }

    @NonNull
    public Robot child() {
        return this.robot;
    }
}

package example.micronaut.singleton;

import io.micronaut.core.annotation.NonNull;
import jakarta.inject.Singleton;

@Singleton
public class RobotMother {
    private final Robot robot;

    public RobotMother(Robot robot) {
        this.robot = robot;
    }

    @NonNull
    public Robot child() {
        return this.robot;
    }
}

```

### Singleton


O escopo singleton indica que existirá apenas uma instância do bean .

Para definir um singleton, anote uma classe com *jakarta.inject.Singleton* no nível de classe.

A classe a seguir cria um identificador único no construtor. Esse identificador nos permite identificar quantas *Robotin* stâncias são usadas.

```java

package example.micronaut.singleton;

import io.micronaut.core.annotation.NonNull;
import jakarta.inject.Singleton;

import java.util.UUID;

@Singleton
public class Robot {
    @NonNull
    private final String serialNumber;

    public Robot() {
        serialNumber = UUID.randomUUID().toString();
    }

    @NonNull
    public String getSerialNumber() {
        return serialNumber;
    }
}

```

### Testando o singleton

```java

import io.micronaut.core.type.Argument;
import io.micronaut.http.HttpRequest;
import io.micronaut.http.client.BlockingHttpClient;
import io.micronaut.http.client.HttpClient;
import io.micronaut.http.client.annotation.Client;
import io.micronaut.test.extensions.junit5.annotation.MicronautTest;
import jakarta.inject.Inject;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.ValueSource;

import java.util.HashSet;
import java.util.List;
import java.util.Set;

import static org.junit.jupiter.api.Assertions.assertEquals;

@MicronautTest
class SingletonScopeTest {
    @Inject
    @Client("/")
    HttpClient httpClient;

    @ParameterizedTest
    @ValueSource(strings = {"/singleton"})

    void onlyOneInstanceOfTheBeanExistsForSingletonBeans(String path) {
        BlockingHttpClient client = httpClient.toBlocking();
        Set<String> responses = new HashSet<>(executeRequest(client, path));
        assertEquals(1, responses.size());
        responses.addAll(executeRequest(client, path));
        assertEquals(1, responses.size());
    }

    List<String> executeRequest(BlockingHttpClient client, String path) {
        return client.retrieve(HttpRequest.GET(path),
          Argument.listOf(String.class));
    }
}

```

### Prototype

O escopo do protótipo indica que uma nova instância do bean é criada cada vez que ele é injetado

Vamos usar @Prototypeem vez de @Singleton.

```java
package example.micronaut.prototype;

import io.micronaut.context.annotation.Prototype;
import io.micronaut.core.annotation.NonNull;

import java.util.UUID;

@Prototype
public class Robot {
    @NonNull
    private final String serialNumber;

    public Robot() {
        serialNumber = UUID.randomUUID().toString();
    }

    @NonNull
    public String getSerialNumber() {
        return serialNumber;
    }
}
```


Use io.micronaut.context.annotation.Prototypepara designar o escopo do bean como Protótipo - um escopo não singleton que cria um novo bean para cada ponto de injeção.


```java
import io.micronaut.core.type.Argument;
import io.micronaut.http.HttpRequest;
import io.micronaut.http.client.BlockingHttpClient;
import io.micronaut.http.client.HttpClient;
import io.micronaut.http.client.annotation.Client;
import io.micronaut.test.extensions.junit5.annotation.MicronautTest;
import jakarta.inject.Inject;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.ValueSource;

import java.util.HashSet;
import java.util.List;
import java.util.Set;

import static org.junit.jupiter.api.Assertions.assertEquals;

@MicronautTest
class PrototypeScopeTest {
    @Inject
    @Client("/")
    HttpClient httpClient;

    @ParameterizedTest
    @ValueSource(strings = {"/prototype"})

    void prototypeScopeIndicatesThatANewInstanceOfTheBeanIsCreatedEachTimeItIsInjected(String path) {
      BlockingHttpClient client = httpClient.toBlocking();
      Set<String> responses = new HashSet<>(executeRequest(client, path));
      assertEquals(2, responses.size());
      responses.addAll(executeRequest(client, path));
      assertEquals(2, responses.size());
    }

    private List<String> executeRequest(BlockingHttpClient client, String path) {
      return client.retrieve(HttpRequest.GET(path), Argument.listOf(String.class));
    }
}

```

### RequestScoped


RequestScope scope é um escopo personalizado que indica que uma nova instância do bean é criada e associada a cada solicitação HTTP

```java
package example.micronaut.request;

import io.micronaut.core.annotation.NonNull;
import io.micronaut.http.HttpRequest;
import io.micronaut.runtime.http.scope.RequestAware;
import io.micronaut.runtime.http.scope.RequestScope;
import java.util.Objects;

@RequestScope
public class Robot implements RequestAware {
    @NonNull
    private String serialNumber;

    @NonNull
    public String getSerialNumber() {
        return serialNumber;
    }

    @Override
    public void setRequest(HttpRequest<?> request) {
        this.serialNumber = Objects.requireNonNull(request.getHeaders().get("UUID"));
    }
}
```

1. Use io.micronaut.runtime.http.scope.RequestScopepara designar o escopo do bean como Solicitação - uma nova instância do bean é criada e associada a cada solicitação HTTP.
2. RequestAwareA API permite que @RequestScopeos beans acessem a requisição atual.

### Teste

```java
import io.micronaut.core.type.Argument;
import io.micronaut.http.HttpRequest;
import io.micronaut.http.client.BlockingHttpClient;
import io.micronaut.http.client.HttpClient;
import io.micronaut.http.client.annotation.Client;
import io.micronaut.test.extensions.junit5.annotation.MicronautTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.util.HashSet;
import java.util.List;
import java.util.Set;
import java.util.UUID;

import static org.junit.jupiter.api.Assertions.assertEquals;

@MicronautTest
class RequestScopeTest {
    @Inject
    @Client("/")
    HttpClient httpClient;

    @Test
    void requestScopeScopeIsACustomScopeThatIndicatesANewInstanceOfTheBeanIsCreatedAndAssociatedWithEachHTTPRequest() {
        String path = "/request";
        BlockingHttpClient client = httpClient.toBlocking();
        Set<String> responses = new HashSet<>(executeRequest(client, path));
        assertEquals(1, responses.size());
        responses.addAll(executeRequest(client, path));
        assertEquals(2, responses.size());
    }

    private List<String> executeRequest(BlockingHttpClient client,
                                        String path) {
        return client.retrieve(createRequest(path), Argument.listOf(String.class));
    }
    private HttpRequest<?> createRequest(String path) {
        return HttpRequest.GET(path).header("UUID", UUID.randomUUID().toString());
    }
}
```

## RefreshScope

O escopo atualizável é um escopo personalizado que permite que o estado de um bean seja atualizado.


```java

package example.micronaut.refreshable;

import io.micronaut.core.annotation.NonNull;
import io.micronaut.runtime.context.scope.Refreshable;
import java.util.UUID;

@Refreshable
public class Robot {
    @NonNull
    private final String serialNumber;

    public Robot() {
        serialNumber = UUID.randomUUID().toString();
    }

    @NonNull
    public String getSerialNumber() {
        return serialNumber;
    }
}

import io.micronaut.context.annotation.Property;
import io.micronaut.core.type.Argument;
import io.micronaut.core.util.StringUtils;
import io.micronaut.http.HttpRequest;
import io.micronaut.http.client.BlockingHttpClient;
import io.micronaut.http.client.HttpClient;
import io.micronaut.http.client.annotation.Client;
import io.micronaut.test.extensions.junit5.annotation.MicronautTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.util.Collections;
import java.util.HashSet;
import java.util.List;
import java.util.Set;

import static org.junit.jupiter.api.Assertions.assertEquals;

@Property(name = "endpoints.refresh.enabled", value = StringUtils.TRUE)
@Property(name = "endpoints.refresh.sensitive", value = StringUtils.FALSE)
@MicronautTest
class RefreshableScopeTest {
    @Inject
    @Client("/")
    HttpClient httpClient;

    @Test
    void refreshableScopeIsACustomScopeThatAllowsABeansStateToBeRefreshedViaTheRefreshEndpoint() {

        String path = "/refreshable";
        BlockingHttpClient client = httpClient.toBlocking();
        Set<String> responses = new HashSet<>(executeRequest(client, path));
        assertEquals(1, responses.size());
        responses.addAll(executeRequest(client, path));
        assertEquals(1, responses.size());
        refresh(client);
        responses.addAll(executeRequest(client, path));
        assertEquals(2, responses.size());
    }

    private void refresh(BlockingHttpClient client) {
        client.exchange(HttpRequest.POST("/refresh",
                        Collections.singletonMap("force", true)));
    }

    private List<String> executeRequest(BlockingHttpClient client, String path) {
        return client.retrieve(HttpRequest.GET(path),
                               Argument.listOf(String.class));
    }
}

```

### Context


O escopo do contexto indica que o bean será criado ao mesmo tempo que o ApplicationContext (inicialização rápida)

O exemplo a seguir usa @Contextem combinação com @ConfigurationProperties.

```java

package example.micronaut.context;

import io.micronaut.context.annotation.ConfigurationProperties;
import io.micronaut.context.annotation.Context;
import jakarta.validation.constraints.Pattern;

@Context//1
@ConfigurationProperties("micronaut")//2
public class MicronautConfiguration {

    @Pattern(regexp = "groovy|java|kotlin") //3
    private String language;

    public String getLanguage() {
        return language;
    }

    public void setLanguage(String language) {
        this.language = language;
    }
}

```


1. Use @Contextpara designar o escopo do bean como Contexto - o bean será criado ao mesmo tempo que o ApplicationContext (inicialização rápida).
2. Use @ConfigurationPropertiespara designar o bean como um bean de configuração.
3. Use @Patternpara validar o valor do bean.


```java

package example.micronaut;

import io.micronaut.context.ApplicationContext;
import io.micronaut.context.exceptions.BeanInstantiationException;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.function.Executable;

import java.util.Collections;

import static org.junit.jupiter.api.Assertions.*;

class ContextTest {

    @Test
    void lifeCycleOfClassesAnnotatedWithAtContextIsBoundToThatOfTheBeanContext() {
        Executable e = () -> ApplicationContext.run(Collections.singletonMap("micronaut.language", "scala"));
        BeanInstantiationException thrown = assertThrows(BeanInstantiationException.class, e);
        assertTrue(thrown.getMessage().contains("language - must match \"groovy|java|kotlin\""));
    }
}

```

### Outros escopos

O Micronaut Framework vem com outros escopos integrados:

#### @Infraestrutura

O *@Infraestrutura* escopo de infraestrutura representa um bean que não pode ser substituído ou anulado @Replaces porque é crítico para o funcionamento do sistema.

#### @ThreadLocal

O escopo @ThreadLocal é um escopo personalizado que associa um bean por thread por meio de um ThreadLocal
