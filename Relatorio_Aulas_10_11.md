# Relatório Individual — Aulas 10 e 11

**Curso:** Ciências da Computação  
**Tema:** Testes automatizados, aplicação executável, Docker e observabilidade em Spring Boot  
**Aluno:** Willyan Sampaio

## 1. Introdução

Quando a gente começa a desenvolver uma API, geralmente a primeira preocupação é fazer as rotas funcionarem no próprio computador. Só que isso não significa que o projeto esteja realmente pronto para ser usado em outro ambiente. Se uma configuração estiver diferente ou uma alteração causar algum erro, a aplicação pode parar de funcionar. Por isso, nas aulas 10 e 11, estudamos os testes automatizados e algumas ferramentas que ajudam na execução e no acompanhamento de uma API Spring Boot.

Para relacionar esses assuntos com uma situação prática, considerei uma API simples de produtos, com cadastro, listagem, consulta, atualização e exclusão. Os exemplos a seguir mostram como eu aplicaria os conceitos nesse tipo de projeto. **A implementação e a execução desses exemplos não foram verificadas por mim neste relatório.**

## 2. Testes automatizados

Teste automatizado é um código que executa uma verificação e compara o resultado obtido com o resultado esperado. No teste manual, a pessoa abre a aplicação, faz uma ação e observa se deu certo. No automatizado, essa conferência pode ser repetida rapidamente sempre que o código muda. O teste manual ainda é útil, principalmente para avaliar a experiência de uso, mas seria trabalhoso repetir dezenas de verificações após cada alteração.

Uma maneira simples de organizar um teste é seguir **Arrange, Act e Assert**. No *Arrange*, preparo os dados e o comportamento das dependências. No *Act*, executo o método que quero testar. No *Assert*, confiro o resultado. Essa divisão deixa claro o que está sendo testado.

O **JUnit** fornece os recursos para escrever e executar testes, incluindo anotações como `@Test` e asserções como `assertEquals`. O **Mockito** é usado para simular as dependências de uma classe. Um *mock* é um objeto simulado que permite controlar respostas e verificar chamadas, sem acessar o componente verdadeiro. Por exemplo, se vou testar o Service, posso simular o Repository em vez de depender do PostgreSQL. Assim, o teste fica mais rápido e não falha por causa de um banco desligado ou de dados que outra pessoa alterou.

No Mockito, `@Mock` cria a dependência simulada; `@InjectMocks` monta a classe que está sendo testada e injeta nela os mocks disponíveis. `when(...).thenReturn(...)` define o que um método simulado deve devolver; `verify(...)` confirma se determinada interação aconteceu; e `assertThrows(...)` verifica se uma operação lançou a exceção esperada. Se uma chamada não tiver sido configurada, o Mockito normalmente devolve um valor padrão, como `null`, `false`, zero ou uma coleção vazia, conforme o tipo e o comportamento do mock. Por isso, não basta criar o mock: é preciso configurar o cenário que importa.

### 2.1 Exemplo comentado de teste de Service

```java
@ExtendWith(MockitoExtension.class)
class ProdutoServiceTest {

    @Mock
    private ProdutoRepository repository;

    @InjectMocks
    private ProdutoService service;

    @Test
    void deveCadastrarProdutoValido() {
        // Arrange: preparo o produto e simulo o retorno do banco.
        Produto produto = new Produto(null, "Mouse", "Periférico", 89.90, 10);
        Produto salvo = new Produto(1L, "Mouse", "Periférico", 89.90, 10);
        when(repository.save(produto)).thenReturn(salvo);

        // Act: chamo o método que quero testar.
        Produto resultado = service.cadastrar(produto);

        // Assert: confiro os dados retornados e a chamada ao Repository.
        assertEquals(1L, resultado.getId());
        assertEquals("Mouse", resultado.getNome());
        verify(repository).save(produto);
    }
}
```

Esse código é um **exemplo ilustrativo**: os construtores e os nomes dos métodos precisam corresponder aos usados na API real. O `assertEquals` verifica um valor retornado, enquanto o `verify` confere se o Service chamou o Repository. São verificações diferentes e as duas ajudam a analisar o comportamento.

Além do cadastro válido, eu criaria testes de sucesso para listar ou buscar um produto e testes de erro para produto inexistente e dados inválidos. Isso atende à ideia de testar tanto situações normais quanto problemas que o usuário pode provocar.

### 2.2 Teste de Service e teste de Controller

O teste de **Service** olha principalmente as regras da aplicação. Ele deve conseguir testar essas regras sem iniciar o sistema inteiro. Já o teste de **Controller** confere como a API recebe requisições e responde com códigos HTTP e JSON. Para isso, o **MockMvc** permite simular chamadas como `GET /produtos` e `POST /produtos` sem abrir um navegador ou iniciar um servidor real na porta 8080.

Um exemplo seria enviar um cadastro sem nome e verificar se a resposta é `400 Bad Request`. Para a busca de um produto inexistente, o retorno esperado poderia ser `404 Not Found`. Mesmo que uma ferramenta mostre **100% de cobertura**, isso não garante a ausência de erros, porque o teste pode passar por todas as linhas sem verificar os comportamentos corretos ou todas as situações possíveis.

## 3. Build e execução do JAR

O *build* é o processo de preparar o código para execução ou distribuição. Em um projeto Maven, o comando `mvn clean package` apaga os arquivos antigos de compilação, compila o projeto, executa os testes configurados e gera o pacote final, desde que as etapas terminem sem erro.

O Maven coloca os resultados no diretório `target`, incluindo arquivos compilados, resultados de testes e, em um projeto Spring Boot configurado para isso, um **JAR executável**. Esse arquivo reúne a aplicação e suas dependências necessárias para iniciar o programa. Com o Java adequado instalado, ela pode ser executada pelo terminal, sem depender do IntelliJ ou de outra IDE:

```bash
mvn clean package
java -jar target/api-produtos-1.0.0.jar
```

O nome do arquivo deve ser ajustado ao que o projeto realmente gerar. Isso facilita entregar a aplicação para outra pessoa, porque ela não precisa abrir todo o código em uma IDE só para iniciar o sistema.

## 4. Configuração externa e profiles

Uma aplicação não deve guardar senhas e outras informações sensíveis diretamente no código-fonte. Se esses dados forem enviados ao GitHub, outras pessoas podem acabar tendo acesso a eles. Por isso, informações como usuário, senha e endereço do banco podem ser configuradas por **variáveis de ambiente**.

No Spring Boot, uma configuração possível seria:

```properties
spring.datasource.url=${DB_URL}
spring.datasource.username=${DB_USER}
spring.datasource.password=${DB_PASSWORD}
```

Assim, o programa lê os valores do ambiente onde está sendo executado. Os **profiles** do Spring ajudam a separar configurações de desenvolvimento, testes e produção. Por exemplo, posso usar `application-dev.properties` para o desenvolvimento local e `application-prod.properties` para um servidor. A ideia é manter o mesmo código e mudar apenas o necessário para cada ambiente, sem colocar credenciais reais no repositório.

## 5. Docker e Docker Compose

O Docker permite empacotar uma aplicação com o ambiente necessário para executá-la. **Imagem** e **container** não são a mesma coisa: a imagem é o pacote preparado, e o container é uma instância dessa imagem em execução. Quando paro um container, ele deixa de executar, mas continua existindo. Quando removo o container, sua instância é apagada; a imagem e os volumes gerenciados separadamente podem continuar disponíveis.

No **Dockerfile**, cada instrução tem uma finalidade. `FROM` escolhe a imagem base, `WORKDIR` define a pasta de trabalho, `COPY` copia arquivos para a imagem e `ENTRYPOINT` determina o comando executado ao iniciar o container.

### 5.1 Exemplo comentado de Dockerfile

```dockerfile
# Utiliza uma imagem com Java 21 para executar a aplicação.
FROM eclipse-temurin:21-jre

# Define a pasta de trabalho dentro do container.
WORKDIR /app

# Copia o JAR que foi gerado pelo Maven.
COPY target/api-produtos-1.0.0.jar app.jar

# Documenta a porta utilizada pela API.
EXPOSE 8080

# Inicia o programa quando o container for executado.
ENTRYPOINT ["java", "-jar", "app.jar"]
```

Esse exemplo pressupõe que o JAR já foi gerado e que o caminho informado está correto. O `EXPOSE` não publica a porta automaticamente: isso deve ser configurado ao iniciar o container ou no Compose.

### 5.2 API e PostgreSQL com Docker Compose

O **Docker Compose** facilita subir vários serviços que precisam funcionar juntos. Uma API pode ter um container próprio e o PostgreSQL outro. Nesse caso, os serviços se comunicam pela rede interna do Compose.

```yaml
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_DB: produtos
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - dados_postgres:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $$POSTGRES_USER -d $$POSTGRES_DB"]
      interval: 5s
      timeout: 5s
      retries: 10

  api:
    build: .
    ports:
      - "8080:8080"
    environment:
      DB_URL: jdbc:postgresql://db:5432/produtos
      DB_USER: ${DB_USER}
      DB_PASSWORD: ${DB_PASSWORD}
    depends_on:
      db:
        condition: service_healthy

volumes:
  dados_postgres:
```

A senha e o usuário seriam definidos fora do arquivo, em variáveis de ambiente ou em um arquivo `.env` que **não deve ser publicado no GitHub**. Dentro do container da API, `localhost` representa o próprio container, e não o banco. Por isso, o endereço usa `db`, que é o nome do serviço PostgreSQL no Compose.

O volume `dados_postgres` guarda os dados do banco fora da camada gravável do container. Sem essa persistência, remover o container pode fazer os dados armazenados apenas nele desaparecerem. O uso de um volume ajuda a conservar as informações mesmo quando o container é recriado. O healthcheck e o `depends_on` melhoram a inicialização, mas a API ainda deve tratar possíveis quedas do banco.

## 6. Logs e Spring Boot Actuator

Os **logs** são registros de acontecimentos dentro da aplicação. Eles ajudam a entender quando o sistema iniciou, quando ocorreu um erro e onde procurar a causa de um problema. O ideal é registrar informações úteis, sem exagerar na quantidade e sem expor senhas, tokens, dados bancários ou outras informações pessoais desnecessárias.

O **Spring Boot Actuator** disponibiliza endpoints usados para acompanhar o funcionamento da aplicação. O mais conhecido é o `/actuator/health`. Quando a resposta apresenta `"status": "UP"`, significa que as verificações de saúde configuradas estão indicando funcionamento naquele momento. Isso não garante que todas as regras de negócio estejam corretas. Uma API pode estar de pé e ainda apresentar um erro no cadastro de produtos, por exemplo.

Os endpoints de monitoramento também exigem cuidado: não é indicado expor informações sensíveis ou recursos administrativos para qualquer pessoa na internet. O acesso e a quantidade de informações mostradas devem ser configurados de acordo com o ambiente.

## 7. Solução do estudo de caso

No estudo de caso, uma API de pedidos funciona no computador de quem desenvolveu, mas apresenta problemas em outra máquina. Além disso, ela depende da IDE, guarda credenciais no código, não possui testes e não oferece uma forma automática de conferir sua saúde.

Eu resolveria o problema em etapas. Primeiro, organizaria as regras de cadastro e busca no Service e criaria testes unitários com JUnit e Mockito, simulando o Repository. Depois, faria testes de Controller com MockMvc para verificar os retornos HTTP. Isso ajudaria a encontrar erros antes da entrega, desde que os testes realmente cobrissem situações importantes.

Em seguida, usaria `mvn clean package` para gerar o JAR e executaria esse arquivo pelo terminal, comprovando que a aplicação não depende da IDE. Também moveria as credenciais para variáveis de ambiente e utilizaria profiles para separar as configurações locais das configurações de produção. O cuidado principal seria não publicar senhas e não assumir que todos os ambientes têm as mesmas configurações.

Para facilitar a execução em outro computador, criaria um Dockerfile para a API e um Compose contendo a aplicação e o PostgreSQL. O banco teria um volume para persistir os dados. Antes de considerar tudo pronto, seria importante verificar se o container inicia corretamente, se a API consegue acessar o banco e se os dados continuam disponíveis após recriar os containers.

Por último, adicionaria logs que mostrassem erros de forma útil e o Actuator para acompanhar o estado da aplicação. Também evitaria registrar informações confidenciais e restringiria os endpoints de monitoramento conforme o ambiente.

Essa sequência ajuda a transformar um projeto que só funciona localmente em uma aplicação com processo de execução mais previsível, testes repetíveis e melhores condições de acompanhamento.

## 8. Conclusão

O principal aprendizado das aulas 10 e 11 é que fazer uma API funcionar no próprio computador é só uma parte do desenvolvimento. Também precisamos conseguir verificar as funcionalidades, gerar um arquivo executável, configurar o ambiente corretamente e entender o que acontece quando a aplicação apresenta algum problema.

Os testes automatizados ajudam a diminuir erros e facilitam alterações no projeto. O JAR permite executar a aplicação sem abrir uma IDE. O Docker e o Compose tornam mais simples preparar o ambiente, enquanto os logs e o Actuator ajudam a acompanhar o funcionamento. Na prática, essas ferramentas se complementam e fazem diferença quando o sistema deixa de ser apenas um exercício local.

## 9. Registro da aplicação prática e evidências

Os códigos apresentados aqui são **exemplos comentados para estudo**. Não foram feitos, nesta entrega, testes executados de uma API própria, geração verificada de JAR, inicialização de containers ou consulta real ao Actuator. Portanto, não há resultados ou capturas de tela que possam ser apresentados como evidência de execução. Para atender integralmente à atividade, ainda seriam necessários a implementação individual da API, os testes de sucesso e erro, a execução dos comandos e o registro verdadeiro dos resultados.