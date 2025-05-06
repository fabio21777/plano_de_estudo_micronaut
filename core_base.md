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

# teste
