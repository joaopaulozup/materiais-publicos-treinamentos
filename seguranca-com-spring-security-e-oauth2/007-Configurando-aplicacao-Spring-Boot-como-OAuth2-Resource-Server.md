# Configurando uma aplicação Spring Boot como OAuth2 Resource Server

Nesse conteúdo veremos como podemos criar ou configurar uma aplicação Spring Boot com OAuth2 Resource Server. Para isso precisaremos adicionar e configurar a dependência do Spring Security no nosso projeto.

## Configurando uma aplicação Spring Boot

Antes de começar, você já está rodando seu Keycloak local? 

Para que você consiga testar a configuração da aplicação ao final deste conteúdo é preciso ter um servidor Keycloak configurado e rodando localmente (assumimos a porta `18080`). Caso não tenha preparado o ambiente com Keycloak nem configurado seus Realms, Clients e Scopes, sugerimos a leitura dos seguintes materiais teoricos antes de tudo:

- [Como instalar e rodar o Keycloak em um container Docker](/seguranca-com-spring-security-e-oauth2/004-Instalando-Keycloak-via-Docker-Compose.md);
- [Como configurar o Authorization Code Flow no Keycloak](/seguranca-com-spring-security-e-oauth2/005-Configurando-Authorization-Code-Flow.md);
- [Como configurar o Resource Owner Password Credentials Flow no Keycloak](/seguranca-com-spring-security-e-oauth2/006-Configurando-Resource-Owner-Password-Credentials-Flow.md);

Você vai precisar ter o Keycloak rodando com um Realm configurado contendo um Client utilizando algum OAuth2 flow (grant type), como Authorization Code ou Resource Owner Password Credentials.

### 1. Adicione a dependência do Maven no projeto

A primeira coisa que temos que fazer é configurar nosso projeto Spring Boot com a dependência do Maven, neste caso estamos falando do **Spring Boot Starter OAuth2 Resource server**. Para isso, basta adicionar a dependência abaixo no seu `pom.xml`:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
</dependency>
```

O artefato `spring-boot-starter-oauth2-resource-server` inclui a dependência `spring-security-oauth2-resource-server`, que por sua vez contem é a biblioteca responsável por nos dar suporte ao modo Resource Server. Esta dependência também inclui as bibliotecas core do Spring Security, por esse motivo, podemos remover a dependencia `spring-boot-starter-security` do nosso `pom.xml` sem qualquer problema. 

> **Aplicação nova? Vá de Spring Initializr!** <br/>
> Nós adicionamos a dependência no `pom.xml` explicitamente pois estamos partindo de um projeto existente, porém se você está criando um novo projeto Spring Boot através do [Spring Initializr](https://start.spring.io/) e, de antemão sabe que precisará configurar a aplicação como OAuth2 Resource Server, aproveite para adicionar a dependência **OAuth2 Resource Server** no momento do setup da sua aplicação Spring Boot.

Se estiver numa IDE, lembre-se de recarregar as dependências do Maven.

### 2. Configure `application.yml` com informações do Authorization Server

O próximo passo é configurar as informações do Authorization Server (Keycloak) na nossa aplicação Spring Boot. Essa é a forma como estabelecemos uma relação de confiança entre Resource Server e Authorization Server, ou seja, precisamos desse passo para que o Resource Server consiga validar o Access Token assinado pelo Authorization Server.

Dado que temos um Keycloak local na nossa máquina rodando na porta `18080` e temos o Realm `meus-contatos` configurado, a configuração no `application.yml` seria semelhante a esta:

```yml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: http://localhost:18080/realms/meus-contatos
```

Nesta configuração também estamos indicando ao Resource Server que o Access Token segue o formato JWT em vez de um formato opaco (Opaque Tokens).

Na configuração acima, nossa aplicação (Resource Server) tentará determinar (e fazer download) das chaves públicas usando os metadados fornecidos pelo Authorization Server. Contudo, algumas vezes por medidas de segurança este endpoint de metadados não está aberto publicamente ou mesmo na rede interna da empresa, neste cenário precisamos também indicar o endpoint das chaves publicas explicitamente através da propriedade `jwk-set-uri`:

```yml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri : http://localhost:18080/realms/meus-contatos
          jwk-set-uri: http://localhost:18080/realms/meus-contatos/protocol/openid-connect/certs
```

A partir de agora a propriedade `issuer-uri` se torna opcional, contudo **entendemos como boa prática manter ambas as propriedades** pois assim o Spring Security pode utiliza-la como opção extra de segurança em alguns cenários.

> **Configurando chaves públicas locais** <br/>
> Em vez de indicar o endpoint das chaves públicas do Keycloak nós podemos armazena-la localmente em um arquivo e configurar seu caminho no disco, ou mesmo passar declará-la estaticamente via texto.
> ```
> spring:
>   security:
>     oauth2:
>       resourceserver:
>         jwt:
>           public-key-location: classpath:my-key.pub
> ```
> Apesar de ser uma abordagem incomum e fraca no ponto de vista de segurança, este método pode ser útil em algumas situações na sua empresa e contexto. Para mais informações, leia a [documentação do Spring Security](https://docs.spring.io/spring-security/reference/servlet/oauth2/resource-server/jwt.html#oauth2resourceserver-jwt-decoder-public-key-boot).

E lembre-se, para maior flexibilidade e dinamismo todas estas configurações no arquivo `application.yml` podem ser feitas diretamente em código Java.


### 3. Configure as regras de acesso (access rules)

Com as informações do Authorzation Server devidamente configuradas no `application.yml`, o próximo passo é configurar as regras de acesso a API REST da nossa aplicação (Resource Server).

A maneira mais simples e comum de declarar estas regras de acesso é via configuração Java. Para isso, basta criarmos a classe `ResourceServerConfig` que estende a classe `WebSecurityConfigurerAdapter` do Spring Security e, por fim, sobreescrever o método `configure(HttpSecurity)`, como podemos ver no código abaixo:

```java
@Configuration
public class ResourceServerConfig extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests()
                .anyRequest().authenticated()
            .and()
                .oauth2ResourceServer()  
                    .jwt(); // atencao: necessario pois sobrescrevemos a conf default do Spring Security
                ;
    }
}
```

Como estamos sobrescrevendo a configuração default do Spring Security, nós **precisamos declarar explicitamente que nosso aplicação deve se comportar como Resource Server** através da DSL `oauth2ResourceServer()`. É partir dessa DSL e do método `jwt()` que declaramos que esperamos um Access Token no formato JWT.

Como não estamos explicitamente passando a configuração que usaremos para validação do token JWT, o Spring Security continuará lendo estas informações do `application.yml`.

Além de habilitarmos o comportamento OAuth2 Resource Server na aplicação com suporte a JWT, nós também declaramos que qualquer requisição que chegar na aplicação precisa estar autenticada, ou seja, ela precisa ter um Access Token válido no cabeçalho.

#### 3.1 Configure as regras de acesso por endpoints

A configuração acima é suficiente para proteger a API REST da nossa aplicação, ou seja, ela está indicando que qualquer endpoint da nossa API REST está protegida e somente poderá ser acessada por uma request que possua um Access Token válido. Porém, é importante especificar quais os Scopes necessários para consumir cada um dos endpoints da nossa API REST.

> ⚠️ **Favoreça o uso de Scopes** <br/>
> É muito importante que você proteja os endpoitns da sua API REST via declaração de Scopes, afinal de contas o protocolo OAuth 2.0 é sobre Autorização. Simplesmente liberar o acesso a todos os endpoints da sua aplicação pelo simples fato de alguém estar autenticado (ou seja, possuir um Access Token) é muito delicado e abre brechas graves de segurança.

Para entender melhor o que estou querendo dizer, vamos a um exemplo. Dado que temos os seguintes endpoints na API REST da aplicação Meus Contatos:

```
Listagem: GET  /contatos
Detalhes: GET  /contatos/{id}
Cadastro: POST /contatos
```

Poderíamos configurar as regras de acesso para os endpoints acima onde para listar e detalhar um contato se faz necessário o Scope `contatos:read`, enquanto a cadastrar um novo contato precisa-se do Scope `contatos:write`, e qualquer outro endpoint precisa de um Access Token válido (autenticado):

```java
@Configuration
public class ResourceServerConfig extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests()
                .antMatchers(HttpMethod.GET, "/api/contatos").hasAuthority("SCOPE_contatos:read")
                .antMatchers(HttpMethod.GET, "/api/contatos/**").hasAuthority("SCOPE_contatos:read")
                .antMatchers(HttpMethod.POST, "/api/contatos").hasAuthority("SCOPE_contatos:write")
                .anyRequest()
                    .authenticated()
            .and()
                .oauth2ResourceServer()  
                    .jwt(); // atencao: necessario pois sobrescrevemos a conf default do Spring Security
                ;
    }
}
```

Repare que as _authorities_ declaradas no método `hasAuthority` são os Scopes que configuramos no Keycloack para nosso Client. Para que o Spring Security reconheça estas authorities como Scopes, se faz necessário usar o prefixo `SCOPE_`.

> ⚠️ **Spring Security 5.7 descontinuou a classe `WebSecurityConfigurerAdapter`** <br/>
> Em versões futuras o Spring Security não dará mais suporte a classe `WebSecurityConfigurerAdapter`, substituindo-a por uma abordagem mais flexível e compatível com a API do WebFlux (programação reativa) baseada sem componentes. Para maiores detalhes, pule para o tópico [Spring Security 5.7: classe `WebSecurityConfigurerAdapter` está deprecated](#4-spring-security-57-classe-websecurityconfigureradapter-está-deprecated) deste material.


#### 3.2. Habilite as regras de acesso por anotações

Por fim, vamos habilitar o controle de acesso via anotações, também conhecido como **Expression-Based Access Control**. Ele nos permite ter controle fino das regras de acesso a nível de métodos através das anotações `@PreAuthorize`, `@PreFilter`, `@PostAuthorize` and `@PostFilter`. Este tipo de controle de acesso poder ser útil para que possamos ter um controle mais fino das regras de acesso a nível de métodos de controllers, services etc.

Para isso, basta configurar nossa classe `ResourceServerConfig` com a anotação `@EnableGlobalMethodSecurity` e definir seu atributo `prePostEnabled` como `true`:

```java
@Configuration
@EnableGlobalMethodSecurity(prePostEnabled = true)
public class ResourceServerConfig extends WebSecurityConfigurerAdapter {

    // ...
}
```

Dessa forma, agora podemos usar anotações para ter um controle ainda mais preciso e completo das regras de acesso a um determinado endpoint. Por exemplo, no código abaixo anotamos um método do controller com a anotação `@PreAuthorize` juntamente com a regra `hasAuthority` para verificar se o Access Token da requisição possui um determinado Scope:

```java
@DeleteMapping(value = "/contatos/{id}")
@PreAuthorize("hasAuthority('SCOPE_contatos:write')")
public ProjectDto remove(@PathVariable Long id) {
    // ...
}
```

Para mais informações sobre os tipos de expressions que podemos utilizar, leia a [documentação oficial](https://docs.spring.io/spring-security/reference/servlet/authorization/expression-based.html).

#### 3.3. Adaptando o Spring Security para APIs REST

Por padrão o Spring Security habilita diversos mecanismos de autenticação e proteção para nossa aplicação, como CSRF (Cross-Site Request Forgery), HTTP Basic Auth, Login e Logout Based Auth, uso de Session no lado servidor entre outras. Ele faz isso pois o framework foi criado e desenhado para uma realidade de aplicações Web, onde o uso de APIs REST ainda não era popular. Embora estes defaults ainda sejam interessantes para aplicações Web hoje em dia, elas **não fazem muito sentido para uma aplicação que expõe uma API REST**, como é o nosso caso.

Por esse motivo, entendemos que é uma boa prática ajustar a configuração do Spring Security da nossa aplicação desligando alguns mecanismos e habilitando outros para que a mesma fique **aderente a natureza Stateless de uma API REST**. Deste modo, podemos melhorar a configuração da nossa classe `ResourceServerConfig` como abaixo:

```java
@Configuration
@EnableGlobalMethodSecurity(prePostEnabled = true)
public class ResourceServerConfig extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        // @formatter:off
        http.cors()
            .and()
                .csrf().disable()
                .httpBasic().disable()
                .rememberMe().disable()
                .formLogin().disable()
                .logout().disable()
                .requestCache().disable()
                .headers().frameOptions().deny()
            .and()
                .sessionManagement()
                    .sessionCreationPolicy(STATELESS)
            .and()
                .authorizeRequests()
                    .antMatchers(HttpMethod.GET, "/api/contatos").hasAuthority("SCOPE_contatos:read")
                    .antMatchers(HttpMethod.GET, "/api/contatos/**").hasAuthority("SCOPE_contatos:read")
                    .antMatchers(HttpMethod.POST, "/api/contatos").hasAuthority("SCOPE_contatos:write")
                .anyRequest()
                    .authenticated()
            .and()
                .oauth2ResourceServer()
                    .jwt()
        ;
        // @formatter:on
    }
}
```

Lembre-se, estas configurações são apenas recomendações para aplicações ou microsserviços que expõe APIs REST, mas elas podem mudar de acordo com a necessidade e restrições de segurança do seu contexto. Para entender cada uma destas configurações e muitas outras que não comentamos, vale a leitura na [documentação oficial do Spring Security](https://docs.spring.io/spring-security/reference/servlet/exploits/index.html).

### 4. Inicie a aplicação e teste seus endpoints

Pronto, agora basta iniciar a aplicação e exercitar os endpoints da nossa API REST!

Aqui podemos usar qualquer ferramenta de cliente HTTP, como POSTman, Insomnia ou mesmo cURL para obter o Access Token e por fim consumir nossa API REST.

Mas primeiramente assuma que a toda a configuração no Keycloak tenha sido devidamente feita, como Realm, Users, Clients e Scopes. Portanto, existe um Client `meus-contatos-client` configurado no nosso Realm `meus-contatos` no Keycloak; existe também um usuário `rafael.ponte` com senha `123`. E, por simplificidade, este Client também está configurado para trabalhar com o fluxo **Resource Owner Password Flow** de maneira `confidential`, afinal, trata-se de um fluxo mais simples e fácil de simular para um desenvolvedor(a) backend. 

Deste modo, conseguimos obter o Access Token com apenas uma requisição HTTP para o Keycloak através do comando cURL:

```sh
curl --location --request POST 'http://localhost:18080/realms/meus-contatos/protocol/openid-connect/token' \
  --header 'Content-Type: application/x-www-form-urlencoded' \
  --data-urlencode 'grant_type=password' \
  --data-urlencode 'username=rafael.ponte' \
  --data-urlencode 'password=123' \
  --data-urlencode 'client_id=meus-contatos-client' \
  --data-urlencode 'client_secret=1tgU88Pko1bpQ8LtHz7pK7tAqnYIRNhn' \
  --data-urlencode 'scope=contatos:read contatos:write'
```

Como resposta a solicitação de token, o Keycloak nos retorna um JSON com os tokens solicitados, incluindo nosso Access Token e suas demais informações, como abaixo:

```json
{
    "access_token": "eyJhbGciOiJSUzI1NiIsInR5cCIgOiAiSldUIiwia2lkIiA6ICJrN05tUzlLaXJQRVZNMk9rdTdIakVIRElyZDNoOHQySTc5eERsM3hXalJrIn0.eyJleHAiOjE2NTE2MDY5OTksImlhdCI6MTY1MTYwNjY5OSwianRpIjoiNmFjN2Y0YzQtMzA2NS00ODU5LWI4NmUtMWE5NzU0M2EyYzJhIiwiaXNzIjoiaHR0cDovL2xvY2FsaG9zdDoxODA4MC9hdXRoL3JlYWxtcy9ib290Y2FtcC1zZW5pb3JzIiwiYXVkIjoiYWNjb3VudCIsInN1YiI6IjUzYjljOWNkLTcyZjktNGNjNC05NzhhLTg5MzIwM2YxMzQ3ZiIsInR5cCI6IkJlYXJlciIsImF6cCI6Im1ldS1pbnNvbW5pYSIsInNlc3Npb25fc3RhdGUiOiIzZGVjNzVmYi0yYzg2LTRkYWItOTNkMi02ZTdkNDA0ZTA3MGEiLCJhY3IiOiIxIiwicmVhbG1fYWNjZXNzIjp7InJvbGVzIjpbIm9mZmxpbmVfYWNjZXNzIiwidW1hX2F1dGhvcml6YXRpb24iLCJkZWZhdWx0LXJvbGVzLWJvb3RjYW1wLXNlbmlvcnMiXX0sInJlc291cmNlX2FjY2VzcyI6eyJhY2NvdW50Ijp7InJvbGVzIjpbIm1hbmFnZS1hY2NvdW50IiwibWFuYWdlLWFjY291bnQtbGlua3MiLCJ2aWV3LXByb2ZpbGUiXX19LCJzY29wZSI6ImZ1bmNpb25hcmlvczpyZWFkIGZ1bmNpb25hcmlvczp3cml0ZSBlbWFpbCBwcm9maWxlIiwic2lkIjoiM2RlYzc1ZmItMmM4Ni00ZGFiLTkzZDItNmU3ZDQwNGUwNzBhIiwiZW1haWxfdmVyaWZpZWQiOnRydWUsIm5hbWUiOiJSYWZhZWwgUG9udGUiLCJwcmVmZXJyZWRfdXNlcm5hbWUiOiJyYWZhZWwucG9udGUiLCJnaXZlbl9uYW1lIjoiUmFmYWVsIiwiZmFtaWx5X25hbWUiOiJQb250ZSIsImVtYWlsIjoicmFmYWVsLnBvbnRlQHp1cC5jb20uYnIifQ.sR7E7qfXtTktIMNvNIOR_SOI2EiEmeYCAVzMkOjlPGVRZUG1GyqTFhGz5DRBHabTj0g-G5MuUrhcrtO2oHH2EM54ULv4ED5l6p_9bF2aczIB5igqskKlIwietpHLnxmIMzm32qDMBIjTnlIlwdJzTH0QzRNQ7vSHS-5pJm55nEV0RrY0QAdao3fBJ3Z8_Loh5FNJxFefZlbo0SxzLQSkZoawUO-cmLl3kQxIzBH_WiNjbxef-FX_OpIIPg-kB82Bkyhwjo1l_9eO2P2uNxepuftVhLTaAQsW3H15c9MH6jxJQGxOUX-QP9bVlpHWuStuhHzrC6pUBMfIv2Bsx3auzw",
    "expires_in": 300,
    "refresh_expires_in": 1800,
    "refresh_token": "eyJhbGciOiJIUzI1NiIsInR5cCIgOiAiSldUIiwia2lkIiA6ICJmMjMzMDI3ZC1lZmExLTQ1YzUtYWQzNy0xMzU5NmJiODQ1ZTAifQ.eyJleHAiOjE2NTE2MDg0OTksImlhdCI6MTY1MTYwNjY5OSwianRpIjoiNDk3NDgzMmItMDJmZS00MTE5LWE0YWUtMjNkN2JlYmNhOWM4IiwiaXNzIjoiaHR0cDovL2xvY2FsaG9zdDoxODA4MC9hdXRoL3JlYWxtcy9ib290Y2FtcC1zZW5pb3JzIiwiYXVkIjoiaHR0cDovL2xvY2FsaG9zdDoxODA4MC9hdXRoL3JlYWxtcy9ib290Y2FtcC1zZW5pb3JzIiwic3ViIjoiNTNiOWM5Y2QtNzJmOS00Y2M0LTk3OGEtODkzMjAzZjEzNDdmIiwidHlwIjoiUmVmcmVzaCIsImF6cCI6Im1ldS1pbnNvbW5pYSIsInNlc3Npb25fc3RhdGUiOiIzZGVjNzVmYi0yYzg2LTRkYWItOTNkMi02ZTdkNDA0ZTA3MGEiLCJzY29wZSI6ImZ1bmNpb25hcmlvczpyZWFkIGZ1bmNpb25hcmlvczp3cml0ZSBlbWFpbCBwcm9maWxlIiwic2lkIjoiM2RlYzc1ZmItMmM4Ni00ZGFiLTkzZDItNmU3ZDQwNGUwNzBhIn0.JZ2-f0sJGR_6cYmUqNiwILuvIIPOtHxYsaoR4Bu1dTE",
    "token_type": "Bearer",
    "not-before-policy": 0,
    "session_state": "3dec75fb-2c86-4dab-93d2-6e7d404e070a",
    "scope": "contatos:read contatos:write email profile"
}
```

Agora, para acessar qualquer endpoint da nossa API REST na aplicação Meus Contatos, basta submeter uma requisição com o header `Authorization` (Bearer) contendo o Access Token obtido do Keycloak na requisição anterior:

```sh
curl --request GET \
  --url 'http://localhost:8080/api/contatos/3' \
  --header 'Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCIgOiAiSldUIiwia....v2Bsx3auzw'
```

Show de bola, como resposta temos um Status HTTP `200 (OK)` com o seguinte payload:

```json
{
    "id": 3,
    "nome": "Jordi Silva",
    "empresa": "Zup",
    "criadoPor": "rafael.ponte",
    "telefones": [
        {
            "id": 7,
            "tipo": "celular",
            "numero": "+5511977777777"
        },
    ]
}
```

Não foi tão dificil, não é mesmo?

Nós usamos o cURL, mas sinta-se a vontade para usar a ferramenta de cliente HTTP da sua preferência.

## Extraindo os atributos do Access Token (JWT)

Podemos extrair diretamente os atributos do Access Token na nossa aplicação Resource Server para utilizar em lógicas de negócio, dados de auditoria ou qualquer regra de segurança nas nossas rotas ou métodos.

Normalmente o Resource Server não se preocupa sobre quem é o usuário requisitando acesso a um recurso, em vez disso ele geralmente se importa mais com as permissões concedidas a este usuário. Apesar de não o principal interesse do Resource Server, não é incomum precisarmos acessar alguns dados básicos do usuário/cliente que está acessando nossa aplicação, como seu nome, email, username etc, por esse motivo é importante entendermos como podemos extrair essas informações do Access Token da requisição.

### 1. Injetando o objeto `Authentication` no controller

No nosso Resource Server, para acessar o Access Token no controller nós podemos injetar a instância de `Authentication` do Spring Security como parâmetro de um dos seus métodos:

```java
@GetMapping("/user/info")
public Map<String, Object> getUserInfo(Authentication authentication) {
    // ...
}
```

Isto somente é possível porque **depois que o Token é autorizado** uma instância de `Authentication` é definida no contexto do Spring Security (`SecurityContext`). E por estarmos trabalhando com a estratégia de autenticação OAuth2 com JWT, esta instância de `Authentication` acaba definindo o _Principal_ com o Access Token da requisição. Tanto é que podemos acessar o token deste modo:

```java
@GetMapping("/user/info")
public Map<String, Object> getUserInfo(Authentication authentication) {

    Jwt jwt = (Jwt) authentication.getPrincipal();

    return Collections
            .singletonMap("token", jwt);
}
```

Perceba que tivemos que fazer o casting explicito do _Principal_ para o tipo `Jwt`. Estamos fazendo isto pois nosso Resource Server está **configurada para trabalhar com token em formato JWT**, e como resultado temos o _Principal_ como um objeto `Jwt` do Spring Security. Se estivessemos trabalhando com token no formato Opaque nós continuaríamos acessando o `Authentication` e seu token do mesmo jeito, porém ele seria de outro tipo.

> **Qual a diferença entre Authentication e Principal?** <br/>
>  No Spring Security, o objeto [Authentication](https://docs.spring.io/spring-security/reference/servlet/authentication/architecture.html#servlet-authentication-authentication) representa a request de autenticação do usuário dentro do framework. Ele é composto de metadados da estratégia de autenticação, um objeto que representa o usuário (_Principal_), suas credenciais, coleção de permissões e um `boolean` indicando se o usuário está de fato autenticado ou não.
>
> Enquanto o _Principal_ identifica o usuário logado. Essa identificação pode ser desde uma `String` com o username ou email, até um objeto customizado do framework ou aplicação como `UserDetails`, mas seu tipo depende estritamente da estratégia de autenticação utilizada.

### 2. Injetando o Principal no controller via anotação `@AuthenticationPrincipal`

Outra forma mais prática de accessar o _Principal_ diretamente é via uso da anotação `@AuthenticationPrincipal` como parâmetro de um método do controller, algo como a seguir:

```java
@GetMapping("/user/info")
public Map<String, Object> getUserInfo(@AuthenticationPrincipal Jwt principalUser) {
    return Collections
            .singletonMap("jwt", principalUser);
}
```

A anotação se encarrega de invocar o método `Authentication.getPrincipal()` e preencher o atributo `principalUser` anotado do método do controller. Deste modo eliminamos a necessidade do casting que havíamos feito anteriormente.

Se acessarmos o endpoing `/user/info/` teremos como resposta o payload abaixo:

```json
{
	"tokenValue": "eyJhbGciOiJSUzI1NiIsInR5cCIgOiAiSldUIiwia2lkIiA6ICJrN05tUzlLaXJQRVZNMk9rdTdIakVIRElyZDNoOHQySTc5eERsM3hXalJrIn0.eyJleHAiOjE2NTE2MDczNjQsImlhdCI6MTY1MTYwNzA2NCwianRpIjoiNmNhNjU3MmMtOTg5NS00MDkzLWI3NjYtNDY1OTQ2OWQ2YzE1IiwiaXNzIjoiaHR0cDovL2xvY2FsaG9zdDoxODA4MC9hdXRoL3JlYWxtcy9ib290Y2FtcC1zZW5pb3JzIiwiYXVkIjoiYWNjb3VudCIsInN1YiI6IjUzYjljOWNkLTcyZjktNGNjNC05NzhhLTg5MzIwM2YxMzQ3ZiIsInR5cCI6IkJlYXJlciIsImF6cCI6Im1ldS1pbnNvbW5pYSIsInNlc3Npb25fc3RhdGUiOiJmZWU4YjBkYS1kZTcxLTQ2NGYtODNlZi0yYWM1MDM0MWM5ZGYiLCJhY3IiOiIxIiwicmVhbG1fYWNjZXNzIjp7InJvbGVzIjpbIm9mZmxpbmVfYWNjZXNzIiwidW1hX2F1dGhvcml6YXRpb24iLCJkZWZhdWx0LXJvbGVzLWJvb3RjYW1wLXNlbmlvcnMiXX0sInJlc291cmNlX2FjY2VzcyI6eyJhY2NvdW50Ijp7InJvbGVzIjpbIm1hbmFnZS1hY2NvdW50IiwibWFuYWdlLWFjY291bnQtbGlua3MiLCJ2aWV3LXByb2ZpbGUiXX19LCJzY29wZSI6ImZ1bmNpb25hcmlvczpyZWFkIGZ1bmNpb25hcmlvczp3cml0ZSBlbWFpbCBwcm9maWxlIiwic2lkIjoiZmVlOGIwZGEtZGU3MS00NjRmLTgzZWYtMmFjNTAzNDFjOWRmIiwiZW1haWxfdmVyaWZpZWQiOnRydWUsIm5hbWUiOiJSYWZhZWwgUG9udGUiLCJwcmVmZXJyZWRfdXNlcm5hbWUiOiJyYWZhZWwucG9udGUiLCJnaXZlbl9uYW1lIjoiUmFmYWVsIiwiZmFtaWx5X25hbWUiOiJQb250ZSIsImVtYWlsIjoicmFmYWVsLnBvbnRlQHp1cC5jb20uYnIifQ.icIxc_dNVO0Hj1fJJb0O7H9E3xneEReAYvaOSiVsNhPA5xfJ_OJrq8ZqbL736hJxllo4S4yOgkw_vD5cSQpUblh0PaYD3qOh_fFiTLLeRAyb0Xzhm3zKj3A8vAuLm0xOFr428M_QpjBjkVbGynPPIEzuKmBRFXM6ogTmqZqrbB8Zdhnpxc8GVS6c9Eh4f_p_rv9ehohYSfXHbkrBG4OgOwRJMsGsum0KMozgTZmGqfhVM1NiLYMThEhCXuc2Ip-4thqPsID4ZCTFxZDi4anHJ67OxEJTQegBhLepuOc0RmP83YywHuw6wSTMnk-V3SMqDc-dwEdR62ZQObrcTy9Msg",
	"issuedAt": "2022-05-03T19:44:24Z",
	"expiresAt": "2022-05-03T19:49:24Z",
	"headers": {
		"kid": "k7NmS9KirPEVM2Oku7HjEHDIrd3h8t2I79xDl3xWjRk",
		"typ": "JWT",
		"alg": "RS256"
	},
	"claims": {
		"sub": "53b9c9cd-72f9-4cc4-978a-893203f1347f",
		"resource_access": {
			"account": {
				"roles": [
					"manage-account",
					"manage-account-links",
					"view-profile"
				]
			}
		},
		"email_verified": true,
		"iss": "http://localhost:18080/auth/realms/meus-contatos",
		"typ": "Bearer",
		"preferred_username": "rafael.ponte",
		"given_name": "Rafael",
		"sid": "fee8b0da-de71-464f-83ef-2ac50341c9df",
		"aud": [
			"account"
		],
		"acr": "1",
		"realm_access": {
			"roles": [
				"offline_access",
				"uma_authorization",
				"default-roles-meus-contatos"
			]
		},
		"azp": "meus-contatos-client",
		"scope": "contatos:read contatos:write email profile",
		"name": "Rafael Ponte",
		"exp": "2022-05-03T19:49:24Z",
		"session_state": "fee8b0da-de71-464f-83ef-2ac50341c9df",
		"iat": "2022-05-03T19:44:24Z",
		"family_name": "Ponte",
		"jti": "6ca6572c-9895-4093-b766-4659469d6c15",
		"email": "rafael.ponte@zup.com.br"
	},
	"id": "6ca6572c-9895-4093-b766-4659469d6c15",
	"notBefore": null,
	"issuer": "http://localhost:18080/auth/realms/meus-contatos",
	"subject": "53b9c9cd-72f9-4cc4-978a-893203f1347f",
	"audience": [
		"account"
	]
}
```

Agora, com a instância de `Jwt` em mãos, podemos acessar qualquer claim deste token. Por exemplo, vamos acessar a claim `preferred_username` que indica o username do nosso usuário:

```java
@GetMapping("/user/info/username")
public Map<String, Object> getUserName(@AuthenticationPrincipal Jwt principalUser) {

    String username = principalUser.getClaim("preferred_username");

    return Collections
            .singletonMap("username", principalUser);
}
```

Se acessarmos o endpoint `/user/info/username` teremos o seguinte payload como resposta:

```json
{
    "username": "rafael.ponte"
}
```

Com o token JWT você pode acessar qualquer uma de suas claims para implementar lógicas de negócio e segurança na aplicação.

### 3. Acessando atributos do Principal via SpEL

Se desejar, você também pode injetar claims diretamente nos parâmetros do método através de SpEL expressions:

```java
@GetMapping("/user/info/username")
public Map<String, Object> getUserName(@AuthenticationPrincipal(expression = "claims") Map<String, Object> claims) {

    return Collections
        .singletonMap("username", claims.get("preferred_username"));
}
```

Essa solução pode ser útil quando você não desejar lidar com o tipo `Jwt` explicitamente no seu controller.

Para mais detalhes sobre uso da anotação `@AuthenticationPrincipal` ou como tirar melhor proveito dela no seu código, você pode consultar a [documentação oficial do Spring Security](https://docs.spring.io/spring-security/reference/servlet/integrations/mvc.html#mvc-authentication-principal).


### 4. Spring Security 5.7: classe `WebSecurityConfigurerAdapter` está deprecated

A classe `WebSecurityConfigurerAdapter` foi descontinuada a partir do **Spring Security 5.7**, que foi adotada no **Spring Boot 2.7.x**. Isto significa que em versões futuras ela será removida do framework. Isso aconteceu pois a idéia é encorajar o desenvolvedor(a) a configurar os detalhes de segurança baseado em componentes, ou seja, sem a necessidade estender a classe `WebSecurityConfigurerAdapter`.

Embora ainda seja possível utilizar e estender a classe `WebSecurityConfigurerAdapter` como estamos acostumados há quase uma decada, é importante que você fique atento com esta mudaça ao atualizar as versões do Spring Security ou mesmo Spring Boot na sua aplicação.

Sem mais delongas, o que antes era configurado assim:

```java
@Configuration
public class SecurityConfiguration extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests((authz) -> authz
                .anyRequest().authenticated()
            )
            .httpBasic(withDefaults());
    }
}
```

Passa a ser recomendado assim:

```java
@Configuration
public class SecurityConfiguration {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests((authz) -> authz
                .anyRequest().authenticated()
            )
            .httpBasic(withDefaults());
        return http.build();
    }
}
```

Repare que a classe `SecurityConfiguration` não estende mais a classe `WebSecurityConfigurerAdapter`. E em vez de sobrescrever o método `configure`, nós declaramos um método de fábrica (factory method) com `@Bean` para customizar o `SecurityFilterChain` como nossas regras de segurança.

Para mais detalhes, recomendamos a leitura deste artigo publicado pela equipe do Spring: [Spring Security without the WebSecurityConfigurerAdapter](https://spring.io/blog/2022/02/21/spring-security-without-the-websecurityconfigureradapter).

## Links e referências

- [OAuth 2.0 Resource Server](https://docs.spring.io/spring-security/reference/servlet/oauth2/resource-server/index.html)
- [OAuth 2.0 Resource Server JWT](https://docs.spring.io/spring-security/reference/servlet/oauth2/resource-server/jwt.html)
- [Baeldung: OAuth 2.0 Resource Server With Spring Security 5](https://www.baeldung.com/spring-security-oauth-resource-server)
- [Expression-Based Access Control](https://docs.spring.io/spring-security/reference/servlet/authorization/expression-based.html)
- [Protection Against Exploits](https://docs.spring.io/spring-security/reference/servlet/exploits/index.html)
- [Baeldung: Introduction to Spring Method Security](https://www.baeldung.com/spring-security-method-security)
- [Spring Security: @AuthenticationPrincipal](https://docs.spring.io/spring-security/reference/servlet/integrations/mvc.html#mvc-authentication-principal)
- [Spring Security without the WebSecurityConfigurerAdapter](https://spring.io/blog/2022/02/21/spring-security-without-the-websecurityconfigureradapter)
- [Spring Security: Authorize HttpServletRequests with AuthorizationFilter](https://docs.spring.io/spring-security/reference/servlet/authorization/authorize-http-requests.html)
