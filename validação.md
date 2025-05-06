# Validação micronaut


## Tratamento de erros

Criar um NotFoundController, pagina geral de erro caso não seja encontrado o recurso.


```java
package example.micronaut;

import io.micronaut.http.HttpRequest;
import io.micronaut.http.HttpResponse;
import io.micronaut.http.HttpStatus;
import io.micronaut.http.MediaType;
import io.micronaut.http.annotation.Controller;
import io.micronaut.http.annotation.Error;
import io.micronaut.http.hateoas.JsonError;
import io.micronaut.http.hateoas.Link;
import io.micronaut.views.ViewsRenderer;

import java.util.Collections;

@Controller("/notfound")
public class NotFoundController {

    private final ViewsRenderer viewsRenderer;

    public NotFoundController(ViewsRenderer viewsRenderer) {
        this.viewsRenderer = viewsRenderer;
    }

    @Error(status = HttpStatus.NOT_FOUND, global = true)
    public HttpResponse notFound(HttpRequest request) {
        if (request.getHeaders()
                .accept()
                .stream()
                .anyMatch(mediaType -> mediaType.getName().contains(MediaType.TEXT_HTML))) {
            return HttpResponse.ok(viewsRenderer.render("notFound", Collections.emptyMap(), request))
                    .contentType(MediaType.TEXT_HTML);
        }

        JsonError error = new JsonError("Page Not Found")
                .link(Link.SELF, Link.of(request.getUri()));

        return HttpResponse.<JsonError>notFound()
                .body(error);
    }
}
```


1. A classe é definida como um controlador com a anotação [@Controller](https://docs.micronaut.io/latest/api/io/micronaut/http/annotation/Controller.html) mapeada para o caminho `/notfound`.

2. [Injete um bean ViewRenderer](https://micronaut-projects.github.io/micronaut-views/latest/api/) disponível para renderizar uma visualização HTML.

3. O erro declara qual `HttpStatus` código de erro tratar (neste caso, 404). Declaramos o método como um manipulador de erros global devido a `global = true`.

4. Se o `Accept` cabeçalho HTTP da solicitação contiver `text/html`, responderemos com uma visualização HTML.

5. Por padrão, respondemos JSON.


### Validações locais

A validação do Micronaut é baseada na estrutura padrão JSR 380 , também conhecida como Validação de Beans 2.0. A Validação do Micronaut possui suporte integrado para validação de beans anotados com jakarta.validationanotações.

Para usar o Micronaut Validation, você precisa das seguintes dependências:

```xml
<!-- Add the following to your annotationProcessorPaths element -->
<path>
    <groupId>io.micronaut.validation</groupId>
    <artifactId>micronaut-validation-processor</artifactId>
</path>
<dependency>
    <groupId>io.micronaut.validation</groupId>
    <artifactId>micronaut-validation</artifactId>
    <scope>compile</scope>
</dependency>

```


Java pojo com validação:

```java

package example.micronaut;

import io.micronaut.serde.annotation.Serdeable;

import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Positive;

@Serdeable
public class CommandBookSave {

    @NotBlank
    private String title;

    @Positive
    private int pages;

    public String getTitle() {
        return title;
    }

    public void setTitle(String title) {
        this.title = title;
    }

    public int getPages() {
        return pages;
    }

    public void setPages(int pages) {
        this.pages = pages;
    }
}

```

1. Declare a anotação `@Serdeable` no nível do tipo em seu código-fonte para permitir que o tipo seja serializado ou desserializado.

2. `title` é obrigatório e não pode estar em branco.

3. `pages` deve ser maior que 0.


```java

...
class BookController {
...
..
    private final MessageSource messageSource; //1

    public BookController(MessageSource messageSource) {
        this.messageSource = messageSource;
    }
...
.
    @View("bookscreate")
    @Error(exception = ConstraintViolationException.class)//2
    public Map<String, Object> onSavedFailed(HttpRequest request, ConstraintViolationException ex) {//3
        final Map<String, Object> model = createModelWithBlankValues();
        model.put("errors", messageSource.violationsMessages(ex.getConstraintViolations()));
        Optional<CommandBookSave> cmd = request.getBody(CommandBookSave.class);
        cmd.ifPresent(bookSave -> populateModel(model, bookSave));
        return model;
    }

    private void populateModel(Map<String, Object> model, CommandBookSave bookSave) {
        model.put("title", bookSave.getTitle());
        model.put("pages", bookSave.getPages());
    }


    private Map<String, Object> createModelWithBlankValues() {
        final Map<String, Object> model = new HashMap<>();
        model.put("title", "");
        model.put("pages", "");
        return model;
    }

..
...
}

package example.micronaut;

import jakarta.inject.Singleton;
import jakarta.validation.ConstraintViolation;
import jakarta.validation.Path;
import java.util.List;
import java.util.Set;
import java.util.stream.Collectors;

@Singleton
public class MessageSource {

    public List<String> violationsMessages(Set<ConstraintViolation<?>> violations) {
        return violations.stream()
                .map(MessageSource::violationMessage)
                .collect(Collectors.toList());
    }

    private static String violationMessage(ConstraintViolation violation) {
        StringBuilder sb = new StringBuilder();
        Path.Node lastNode = lastNode(violation.getPropertyPath());
        if (lastNode != null) {
            sb.append(lastNode.getName());
            sb.append(" ");
        }
        sb.append(violation.getMessage());
        return sb.toString();
    }

    private static Path.Node lastNode(Path path) {
        Path.Node lastNode = null;
        for (final Path.Node node : path) {
            lastNode = node;
        }
        return lastNode;
    }
}

```


1. Injete o `MessageSource` para obter mensagens de erro traduzidas.
2. Declare o manipulador de erro para `ConstraintViolationException` no controlador.
3. O manipulador de erro é chamado quando a validação falha. Ele retorna um mapa com os erros de validação.


### Manipulação de exceções


Outro mecanismo para lidar com exceção global é usar um ExceptionHandler.

Modifique o controlador e adicione um método para lançar uma exceção:

```java

@Controller("/books")
public class BookController {
...
..
.
    @Produces(MediaType.TEXT_PLAIN)
    @Get("/stock/{isbn}")
    public Integer stock(String isbn) {
        throw new OutOfStockException();
    }
}

package example.micronaut;

public class OutOfStockException extends RuntimeException {
}


package example.micronaut;

import io.micronaut.context.annotation.Requires;
import io.micronaut.http.HttpRequest;
import io.micronaut.http.HttpResponse;
import io.micronaut.http.annotation.Produces;
import io.micronaut.http.server.exceptions.ExceptionHandler;
import jakarta.inject.Singleton;

@Produces
@Singleton //1
@Requires(classes = {OutOfStockException.class, ExceptionHandler.class})//2
public class OutOfStockExceptionHandler implements ExceptionHandler<OutOfStockException, HttpResponse> {//3

    @Override
    public HttpResponse handle(HttpRequest request, OutOfStockException exception) {
        return HttpResponse.ok(0);//4
    }
}


```

1. Use `jakarta.inject.Singleton` para designar uma classe como singleton.

2. Este bean carrega se `OutOfStockException`, `ExceptionHandler` estiverem disponíveis.

3. Especifique o `Throwable` identificador.

4. Retornar 200 OK com corpo 0; sem estoque.


## Anotação de restrição personalizada para validação

Como criar uma anotação de restrição personalizada para validação em seu aplicativo Micronaut


```java

package example.micronaut;

import io.micronaut.core.annotation.Nullable;
import jakarta.annotation.Nonnull;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;

import java.util.ArrayList;
import java.util.Arrays;
import java.util.Comparator;
import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

/**
 * Every country code in the world.
 * @see <a href="https://www.itu.int/dms_pub/itu-t/opb/sp/T-SP-E.164D-11-2011-PDF-E.pdf">LIST OF ITU-T RECOMMENDATION E.164 ASSIGNED COUNTRY CODES</a>
 */
public enum CountryCode {
	...
    BRAZIL("55", "Brazil (Federative Republic of)"),
    PORTUGAL("351", "Portugal"),
    ...

    private final String code;
    private final String countryName;

    /**
     * Constructor for countries whose name does not match the enum.
     * @param code country code
     * @param countryName full country name
     */
    CountryCode(String code, String countryName) {
        this.code = code;
        this.countryName = countryName;
    }

    public String getCode() {
        return this.code;
    }

    /**
     * Country name.
     * @return country name
     */
    public String getCountryName() {
        return this.countryName;
    }

    private static final String PLUS_SIGN = "+";

    private static final Map<String, List<CountryCode>> COUNTRYCODESBYCODE =
            Arrays.stream(CountryCode.values())
                    .collect(Collectors.groupingBy(CountryCode::getCode));

    private static final List<String> CODES = COUNTRYCODESBYCODE.keySet().stream()
            .sorted(Comparator.comparing(String::length).reversed()).toList();

    /**
     *
     * @param code Country code
     * @return a List of {@link CountryCode} for a found code or an empty list
     */
    @NotNull
    @Nonnull
    public static List<CountryCode> countryCodesByCode(@Nonnull @NotBlank String code) {
        if (COUNTRYCODESBYCODE.containsKey(code)) {
            return COUNTRYCODESBYCODE.get(code);
        }
        return new ArrayList<>();
    }

    /**
     *
     * @return Country codes ordered from codes of longer length to less length.
     */
    public static List<String> getCodes() {
        return CODES;
    }

    /**
     *
     * @param number Phone number
     * @return the Country code found in the phone number or {@code null} if not found.
     */
    @Nullable
    public static String parseCountryCode(@Nonnull @NotBlank String number) {
        String phone = number.startsWith(PLUS_SIGN) ? number.substring(1) : number;
        for (String code : getCodes()) {
            if (phone.startsWith(code)) {
                return code;
            }
        }
        return null;
    }

    @Override
    public String toString() {
        return this.code;
    }
}
```

### Anotação personalizada

```java

package example.micronaut;

import jakarta.validation.Constraint;
import jakarta.validation.Payload;
import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Repeatable;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * The annotated element must be a E.164 phone number.
 *
 * @see <a href="https://www.itu.int/rec/T-REC-E.164/en">ITU E.164 recommendation</a>
 * @see <a href="https://www.twilio.com/docs/glossary/what-e164">E.614</a>
 */
@Target({ElementType.METHOD, ElementType.FIELD, ElementType.ANNOTATION_TYPE, ElementType.PARAMETER, ElementType.TYPE_USE})
@Retention(RetentionPolicy.RUNTIME)
@Repeatable(E164.List.class)
@Documented
@Constraint(validatedBy = {})
public @interface E164 {

    String MESSAGE = "example.micronaut.E164.message";

    /**
     * @return message The error message
     */
    String message() default "{" + MESSAGE + "}";

    /**
     * @return Groups to control the order in which constraints are evaluated,
     * or to perform validation of the partial state of a JavaBean.
     */
    Class<?>[] groups() default {};

    /**
     * @return Payloads used by validation clients to associate some metadata information with a given constraint declaration
     */
    Class<? extends Payload>[] payload() default {};

    /**
     * List annotation.
     */
    @Target({ElementType.METHOD, ElementType.FIELD, ElementType.ANNOTATION_TYPE, ElementType.PARAMETER, ElementType.TYPE_USE})
    @Retention(RetentionPolicy.RUNTIME)
    @Documented
    @interface List {

        /**
         * @return An array of E164.
         */
        E164[] value();
    }
}


package example.micronaut;

import io.micronaut.context.annotation.Factory;
import io.micronaut.validation.validator.constraints.ConstraintValidator;
import jakarta.inject.Singleton;

@Factory
class CustomValidationFactory {

    /**
     * @return A {@link ConstraintValidator} implementation of a {@link E164} constraint for type {@link String}.
     */
    @Singleton
    ConstraintValidator<E164, String> e164Validator() {
        return (value, annotationMetadata, context) -> E164Utils.isValid(value);
    }
}

package example.micronaut;

import io.micronaut.context.StaticMessageSource;
import jakarta.inject.Singleton;

/**
 * Adds validation messages.
 */
@Singleton
public class CustomValidationMessages extends StaticMessageSource {

    public static final String E164_MESSAGE = "must be a phone in E.164 format";
    /**
     * The message suffix to use.
     */
    private static final String MESSAGE_SUFFIX = ".message";

    /**
     * Default constructor to initialize messages.
     * via {@link #addMessage(String, String)}
     */
    public CustomValidationMessages() {
        addMessage(E164.class.getName() + MESSAGE_SUFFIX, E164_MESSAGE);
    }
}

package example.micronaut;

import io.micronaut.core.annotation.Introspected;
import io.micronaut.core.annotation.NonNull;

import jakarta.validation.constraints.NotBlank;

@Introspected
public class Contact {

    @E164
    @NotBlank
    @NonNull
    private final String phone;

    public Contact(@NonNull String phone) {
        this.phone = phone;
    }


    @NonNull
    public String getPhone() {
        return phone;
    }
}


```
