# CLAUDE.md

API de agendamento online para profissionais autônomos (barbeiros, manicures, personal trainers, consultórios). Clientes veem horários livres e reservam; profissionais configuram serviços, expediente e folgas. O problema central do projeto é **nunca permitir duas reservas no mesmo horário**, mesmo com requisições simultâneas.

Projeto de portfólio: o código precisa ser legível, testado e explicável numa entrevista. Prefira a solução simples e correta à solução esperta.

## Stack

- Java 21, Spring Boot 3.x, Maven (use sempre o wrapper `./mvnw`)
- PostgreSQL 16 + Flyway
- Spring Data JPA (Hibernate), Spring Security com JWT, Bean Validation
- springdoc-openapi (Swagger UI)
- JUnit 5, Mockito, AssertJ, Testcontainers
- Docker Compose para o ambiente local (Postgres + Mailpit para e-mail)
- GitHub Actions para CI

Não adicione dependências novas sem perguntar antes. Tudo precisa rodar de graça e localmente.

## Comandos

```bash
docker compose up -d          # Postgres e Mailpit
./mvnw spring-boot:run        # sobe a API em http://localhost:8080
./mvnw test                   # testes unitários
./mvnw verify                 # unitários + integração (Testcontainers) + formatação
./mvnw spotless:apply         # formata o código
```

Swagger: http://localhost:8080/swagger-ui.html. Mailpit (e-mails enviados): http://localhost:8025.

Antes de dizer que uma tarefa terminou, rode `./mvnw verify` e confirme que passou.

## Arquitetura

Arquitetura hexagonal, organizada por módulo de negócio:

```
src/main/java/dev/joaofs/booking/
  scheduling/            # reservas e disponibilidade (núcleo)
    domain/              # entidades e regras, sem Spring nem JPA
    application/         # casos de uso (um por classe) e portas
    adapter/in/web/      # controllers e DTOs
    adapter/out/persistence/  # entidades JPA, repositórios, mappers
  catalog/               # serviços oferecidos e expediente do profissional
  identity/              # usuários, autenticação e perfis
  notification/          # e-mails de confirmação e lembrete
  shared/                # erros, configuração, utilitários de tempo
```

Regras:
- `domain` não importa nada de Spring, JPA ou de outro módulo.
- Controllers só convertem DTO, chamam o caso de uso e devolvem resposta. Nenhuma regra de negócio em controller.
- Entidades JPA nunca saem da camada de persistência e nunca são devolvidas pela API.
- Um módulo conversa com outro pela porta de aplicação dele, nunca pelo repositório.

## Regras de domínio

**Sem conflito de horário.** Duas reservas ativas do mesmo profissional não podem se sobrepor. A garantia final é do banco, não do Java:

```sql
CREATE EXTENSION IF NOT EXISTS btree_gist;
ALTER TABLE booking ADD CONSTRAINT booking_no_overlap
  EXCLUDE USING gist (professional_id WITH =, tstzrange(starts_at, ends_at, '[)') WITH &&)
  WHERE (status IN ('PENDING', 'CONFIRMED'));
```

A checagem em Java existe só para dar uma mensagem de erro melhor. Quando o banco rejeitar a reserva por violar essa restrição, devolva `409 Conflict`. Sempre que mexer nessa lógica, explique a decisão no PR.

**Tempo.**
- Grave tudo em UTC (`timestamptz`), use `Instant` no domínio.
- Cada profissional tem um fuso (`ZoneId`). Expediente e folgas são em horário local dele; converta na borda.
- Intervalos são semiabertos: `[início, fim)`. Uma reserva das 10h às 11h não conflita com outra das 11h às 12h.
- Nunca use `LocalDateTime.now()` ou `Instant.now()` direto: injete um `Clock`, para os testes controlarem o tempo.

**Status da reserva:** `PENDING` → `CONFIRMED` → `COMPLETED`, ou `CANCELLED` / `NO_SHOW`. Transições inválidas lançam exceção de domínio.

**Cancelamento:** o cliente só cancela até X horas antes (configurável por profissional). Reservas canceladas liberam o horário.

## Convenções de código

- Nomes de classes, métodos e variáveis em inglês. Mensagens de erro para o usuário em português.
- DTOs como `record`. Validação com Bean Validation nos DTOs de entrada.
- Injeção por construtor, sem `@Autowired` em campo. Sem Lombok em classes de domínio.
- Erros da API no formato `ProblemDetail` (RFC 9457) via `@RestControllerAdvice`.
- Listagens sempre paginadas.
- Use `@Transactional` nos casos de uso, não nos controllers nem nos repositórios.
- Evite consultas N+1: confira as queries geradas quando carregar associações.

## Banco e migrações

- Toda mudança de esquema é uma nova migração Flyway em `src/main/resources/db/migration`, no padrão `V{n}__descricao.sql`.
- **Nunca edite uma migração que já existe.** Para corrigir, crie outra.
- `spring.jpa.hibernate.ddl-auto=validate`. O Hibernate nunca cria tabela.

## Segurança

- JWT com perfis `CLIENT`, `PROFESSIONAL` e `ADMIN`. Autorização com `@PreAuthorize` nos casos de uso ou na configuração de segurança.
- Um profissional só acessa a própria agenda; um cliente só vê e cancela as próprias reservas. Teste os dois casos de acesso negado.
- Senhas com BCrypt. Segredos (chave do JWT, senha do banco) só por variável de ambiente; nunca no código nem no `application.yml` versionado.

## Testes

- Regras de domínio: testes unitários puros, sem Spring.
- Casos de uso e endpoints: testes de integração com Testcontainers (Postgres real, nunca H2).
- **Teste de concorrência obrigatório:** várias threads tentando reservar o mesmo horário; exatamente uma vence e as outras recebem conflito.
- Nomes de teste descrevem o comportamento: `shouldRejectBookingWhenSlotOverlapsExistingOne`.
- Ao corrigir um bug, escreva primeiro o teste que falha.

## Git

- Uma branch por issue (`feat/12-cancel-booking`), commits pequenos no padrão Conventional Commits (`feat:`, `fix:`, `test:`, `refactor:`, `docs:`).
- Não faça commit nem push sem eu pedir.

## Como trabalhar comigo

- Para qualquer funcionalidade nova, mostre o plano antes de escrever código.
- Faça uma tarefa por vez e pare para eu revisar.
- Se algo nesta página estiver em conflito com o que eu pedir, pergunte.
- Estou usando este projeto para aprender e para explicar em entrevistas: quando tomar uma decisão não óbvia (concorrência, transação, índice, modelagem), explique o porquê em poucas linhas.
- O README é em inglês; comentários no código, só quando explicam o porquê.