# scheduled

## Learn how to schedule periodic tasks inside your Micronaut microservices.

Neste guia, criaremos um aplicativo Micronaut escrito em Java.

Hoje em dia é bastante comum ter algum tipo de cron ou tarefa agendada que precisa ser executada toda meia-noite, a cada hora, algumas vezes por semana,…​

Neste guia, você aprenderá como usar os recursos do Micronaut para agendar tarefas periódicas dentro de um microsserviço Micronaut.


### Criando um job

```java
package example.micronaut;

import io.micronaut.scheduling.annotation.Scheduled;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import jakarta.inject.Singleton;
import java.text.SimpleDateFormat;
import java.util.Date;

@Singleton //1
public class HelloWorldJob {
    private static final Logger LOG = LoggerFactory.getLogger(HelloWorldJob.class);//2

    @Scheduled(fixedDelay = "10s")//3
    void executeEveryTen() {
        LOG.info("Simple Job every 10 seconds: {}", new SimpleDateFormat("dd/M/yyyy hh:mm:ss").format(new Date()));
    }

    @Scheduled(fixedDelay = "45s", initialDelay = "5s")//4
    void executeEveryFourtyFive() {
        LOG.info("Simple Job every 45 seconds: {}", new SimpleDateFormat("dd/M/yyyy hh:mm:ss").format(new Date()));
    }
}

```

1. `@Singleton` - O Micronaut irá criar uma instância única do bean.
2. `Logger` - O Micronaut irá criar um logger para o bean.
3. `@Scheduled(fixedDelay = "10s")` - O Micronaut irá executar o método a cada 10 segundos.
4. `@Scheduled(fixedDelay = "45s", initialDelay = "5s")` - O Micronaut irá executar o método a cada 45 segundos, com um atraso inicial de 5 segundos.

```log
... Simple Job every 10 seconds :15/5/2018 12:48:02
... Simple Job every 45 seconds :15/5/2018 12:48:07
... Simple Job every 10 seconds :15/5/2018 12:48:12
... Simple Job every 10 seconds :15/5/2018 12:48:22
... Simple Job every 10 seconds :15/5/2018 12:48:32
... Simple Job every 10 seconds :15/5/2018 12:48:42
... Simple Job every 45 seconds :15/5/2018 12:48:52
... Simple Job every 10 seconds :15/5/2018 12:48:52

```

### Lógica de negócios em casos de uso dedicados


Normalmente, você não deve colocar a lógica de negócios diretamente em um job. O ideal é criar um caso de uso dedicado e chamá-lo a partir do job.

```java
package example.micronaut;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import jakarta.inject.Singleton;
import java.text.SimpleDateFormat;
import java.util.Date;

@Singleton
public class EmailUseCase {
    private static final Logger LOG = LoggerFactory.getLogger(EmailUseCase.class);

    void send(String user, String message) {
        LOG.info("Sending email to {}: {} at {}", user, message, new SimpleDateFormat("dd/M/yyyy hh:mm:ss").format(new Date()));
    }
}

```

```java

package example.micronaut;

import io.micronaut.scheduling.annotation.Scheduled;

import jakarta.inject.Singleton;

@Singleton //1
public class DailyEmailJob {

    protected final EmailUseCase emailUseCase;

    public DailyEmailJob(EmailUseCase emailUseCase) {//2
        this.emailUseCase = emailUseCase;
    }

    @Scheduled(cron = "0 30 4 1/1 * ?") //3
    void execute() {
        emailUseCase.send("john.doe@micronaut.example", "Test Message"); //4
    }
}

```

1. `@Singleton` - O Micronaut irá criar uma instância única do bean.
2. `EmailUseCase` - O Micronaut irá injetar o bean `EmailUseCase` no construtor do job.
3. `@Scheduled(cron = "0 30 4 1/1 * ?")` - O Micronaut irá executar o método a cada dia, às 4:30 da manhã.
4. Executa o caso de uso `EmailUseCase` para enviar um e-mail.

### Agendamento manual

Considere o seguinte cenário. Você quer enviar um e-mail a cada usuário duas horas após o cadastro no seu aplicativo e perguntar sobre suas experiências durante essa primeira interação.

Neste guia, agendaremos um trabalho para ser acionado após um minuto.


```java

package example.micronaut;

import io.micronaut.context.event.ApplicationEventListener;
import io.micronaut.runtime.Micronaut;
import io.micronaut.runtime.server.event.ServerStartupEvent;

import jakarta.inject.Singleton;

@Singleton//1
public class Application implements ApplicationEventListener<ServerStartupEvent> { //2

    private final RegisterUseCase registerUseCase;

    public Application(RegisterUseCase registerUseCase) {//3
        this.registerUseCase = registerUseCase;
    }

    public static void main(String[] args) {
        Micronaut.run(Application.class);
    }

    @Override
    public void onApplicationEvent(ServerStartupEvent event) {//4
        try {
            registerUseCase.register("harry@micronaut.example");
            Thread.sleep(20000);//4
            registerUseCase.register("ron@micronaut.example");//5
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}

```

1. `@Singleton` - O Micronaut irá criar uma instância única do bean.
2. `ApplicationEventListener<ServerStartupEvent>` - O Micronaut irá criar um listener para o evento de inicialização do servidor.
3. `RegisterUseCase` - O Micronaut irá injetar o bean `RegisterUseCase` no construtor do job.
4. `onApplicationEvent(ServerStartupEvent event)` - O Micronaut irá executar o método quando o evento de inicialização do servidor for acionado.
5. Executa o caso de uso `RegisterUseCase` para registrar um usuário.
6. `Thread.sleep(20000)` - O Micronaut irá aguardar 20 segundos antes de registrar o próximo usuário.


```java

package example.micronaut;

import io.micronaut.scheduling.TaskExecutors;
import io.micronaut.scheduling.TaskScheduler;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import jakarta.inject.Named;
import jakarta.inject.Singleton;
import java.text.SimpleDateFormat;
import java.time.Duration;
import java.util.Date;

@Singleton //1
public class RegisterUseCase {

    private static final Logger LOG = LoggerFactory.getLogger(RegisterUseCase.class);

    protected final TaskScheduler taskScheduler;
    protected final EmailUseCase emailUseCase;

    public RegisterUseCase(EmailUseCase emailUseCase, //2
                           @Named(TaskExecutors.SCHEDULED) TaskScheduler taskScheduler) {//2
        this.emailUseCase = emailUseCase;
        this.taskScheduler = taskScheduler;
    }

    public void register(String email) {
        LOG.info("saving {} at {}", email, new SimpleDateFormat("dd/M/yyyy hh:mm:ss").format(new Date()));
        scheduleFollowupEmail(email, "Welcome to the Micronaut framework");
    }

    private void scheduleFollowupEmail(String email, String message) {
        EmailTask task = new EmailTask(emailUseCase, email, message);//3
        taskScheduler.schedule(Duration.ofMinutes(1), task);//4
    }
}


```

1. `@Singleton` - O Micronaut irá criar uma instância única do bean.
2. `TaskScheduler` - O Micronaut irá injetar o bean `TaskScheduler` no construtor do job.
3. Criar uma `Runnable` tarefa.
4. `taskScheduler.schedule(Duration.ofMinutes(1), task)` - O Micronaut irá agendar a tarefa para ser executada em 1 minuto.

```log
INFO  example.micronaut.RegisterUseCase - saving harry@micronaut.example at 15/5/2018 06:25:14
INFO  example.micronaut.RegisterUseCase - saving ron@micronaut.example at 15/5/2018 06:25:34
INFO  example.micronaut.EmailUseCase - Sending email to harry@micronaut.example : Welcome to Micronaut at 15/5/2018 06:26:14
INFO  example.micronaut.EmailUseCase - Sending email to ron@micronaut.example : Welcome to the Micronaut framework at 15/5/2018 06:26:34
```
