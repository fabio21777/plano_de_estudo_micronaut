# [Voltar](/readme.md)

## O que é o Micronaut?

O Micronaut é um framework moderno, baseado na JVM e fullstack, projetado para criar aplicativos modulares e facilmente testáveis, com suporte para Java, Kotlin e Groovy. Ele é otimizado para ambientes de nuvem e microserviços, oferecendo recursos como injeção de dependência, programação reativa e suporte a APIs REST.

O framework Micronaut foi originalmente criado por uma equipe que também trabalhou no framework Grails. O Micronaut se inspira nas lições aprendidas ao longo dos anos construindo aplicações reais, de monólitos a microsserviços, usando Spring, Spring Boot e o framework Grails. A equipe principal continua a desenvolver e manter o projeto Micronaut, com o apoio da fundação Micronaut.

O framework Micronaut visa fornecer todas as ferramentas necessárias para construir aplicativos na JVM, como:

* Injeção de dependência
* Programação orientada a aspectos
* Padrões sensatos e configurações automáticas

Com a estrutura Micronaut, você pode criar aplicativos orientados a mensagens, aplicativos de linha de comando, servidores HTTP e muito mais. Para microsserviços, em particular, o Micronaut também fornece:

* Configuração distribuída
* Descoberta de serviço
* Roteamento HTTP
* Balanceamento de carga do lado do cliente

Ao mesmo tempo, o framework Micronaut visa evitar as desvantagens de frameworks como Spring, Spring Boot e Grails, fornecendo:

* Tempo de inicialização rápido
* Redução do consumo de memória
* Uso mínimo de reflexão
* Uso mínimo de proxies
* Nenhuma geração de bytecode em tempo de execução
* Teste de unidade fácil

Historicamente, frameworks como Spring e Grails não foram projetados para rodar em cenários como funções serverless, aplicativos Android ou microsserviços com baixo consumo de memória. Em contraste, o framework Micronaut foi projetado para ser adequado a todos esses cenários.

Este objetivo é alcançado através do uso dos processadores de anotação Java, que podem ser usados em qualquer linguagem JVM que os suporte, bem como um servidor HTTP (com vários runtimes: Netty, Jetty, Tomcat, Undertow, etc.) e um cliente HTTP (com vários runtimes: Netty, Cliente HTTP Java, etc.). Para fornecer um modelo de programação semelhante ao Spring e Grails, esses processadores de anotação pré-compilam os metadados necessários para executar DI, definir proxies AOP e configurar sua aplicação para ser executada em um ambiente com pouca memória.

Muitas APIs no framework Micronaut são fortemente inspiradas em Spring e Grails. Isso é intencional e ajuda os desenvolvedores a se familiarizarem rapidamente.


## Referências

* [Micronaut](https://docs.micronaut.io/4.8.11/guide/)
