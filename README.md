# API de adocao de animais

API REST da ONG para cadastro de animais, cuidados veterinarios, candidaturas, triagem, adocao e acompanhamento. A aplicacao usa Java 17, Spring Boot, Spring Data JPA, Spring Security com JWT e H2.

## Requisitos para executar

- Java 17, para executar com Maven.
- Docker Desktop e Docker Compose, para executar em container.

## Executar com Docker

Abra o Docker Desktop. No PowerShell, navegue ate a raiz do projeto e inicie o servico:

```powershell
cd C:\Users\{SEU USUARIO}\Downloads\ONGAnimais
docker compose up --build -d
docker compose ps
```

A API fica disponivel em `http://localhost:8080`. Para acompanhar a inicializacao:

```powershell
docker compose logs -f app
```

Encerre a exibicao dos logs com `Ctrl+C`. Isso nao para o container. Para parar e remover o servico:

```powershell
docker compose down
```

O Compose usa H2 em memoria e ativa o perfil `demo`, que cria as contas de voluntario e coordenador abaixo. Ao remover o container, os dados cadastrados no H2 sao perdidos.

## Executar com Maven

Na raiz do projeto, pelo PowerShell:

```powershell
$env:SPRING_PROFILES_ACTIVE = "demo"
.\mvnw.cmd spring-boot:run
```

Para executar os testes e gerar o relatorio JaCoCo:

```powershell
.\mvnw.cmd clean verify
```

O relatorio fica em `target/site/jacoco/index.html`.

## Contas de demonstracao

Quando o perfil `demo` esta ativo, a aplicacao cria estas contas se elas ainda nao existirem:

| Perfil | E-mail | Senha |
|---|---|---|
| Voluntario | `voluntario@ong.local` | `Voluntario123!` |
| Coordenador | `coordenador@ong.local` | `Coordenador123!` |

Adotantes se cadastram pela rota `POST /api/usuarios/cadastrar`. As senhas sao armazenadas com BCrypt.

As credenciais de demonstracao podem ser substituidas por variaveis de ambiente antes de iniciar o Compose:

```powershell
$env:ONG_ADOCAO_DEMO_VOLUNTARIO_EMAIL = "voluntario@ong.local"
$env:ONG_ADOCAO_DEMO_VOLUNTARIO_SENHA = "Voluntario123!"
$env:ONG_ADOCAO_DEMO_COORDENADOR_EMAIL = "coordenador@ong.local"
$env:ONG_ADOCAO_DEMO_COORDENADOR_SENHA = "Coordenador123!"
docker compose up --build -d
```



## Autenticar uma requisicao

Faca login em `POST /api/auth/login`, enviando e-mail e senha:

```json
{
  "email": "voluntario@ong.local",
  "senha": "Voluntario123!"
}
```

A resposta inclui `token`, `tipo` (`Bearer`) e `expiraEmSegundos`. Nas chamadas protegidas, envie:

```text
Authorization: Bearer <token>
```

No Postman, selecione **Authorization**, escolha **Bearer Token** e cole o token. Sem token valido, a resposta e `401 Unauthorized`; com um perfil sem permissao, e `403 Forbidden`.

## Refinamentos e endpoints

Todas as rotas usam a URL-base `http://localhost:8080`. Para corpos JSON, selecione `Body > raw > JSON` no Postman.

### RF01 — Cadastro de adotante

**Acesso:** publico.


| `POST` | `/api/usuarios/cadastrar` | Cria um usuario com perfil `ADOTANTE`. CPF e e-mail duplicados sao rejeitados. A resposta nao contem a senha. |

Campos: `nome`, `cpf`, `email`, `telefone`, `endereco` e `senha`. A senha deve ter ao menos 8 caracteres.

```json
{
  "nome": "Maria Silva",
  "cpf": "12345678900",
  "email": "maria.teste@example.com",
  "telefone": "11999999999",
  "endereco": "Rua das Flores, 100",
  "senha": "senha123"
}
```

### RF02 — Login e JWT

**Acesso:** publico.

| `POST` | `/api/auth/login` | Valida e-mail e senha e emite um JWT com identidade e perfil. |

```json
{
  "email": "maria.teste@example.com",
  "senha": "senha123"
}
```

Para credenciais invalidas, a API responde com mensagem generica; ela nao informa se o erro foi no e-mail ou na senha.

### RF03 — Cadastro de animais

**Acesso:** voluntario.


| `POST` | `/api/animais` | Cadastra o animal com status inicial `EM_TRATAMENTO` e data de entrada igual a data do cadastro. |

Campos: `nome`, `especie`, `porte`, `sexo`, `idadeEstimada`, `temperamento` e `fotoUrl`.

### RF04 — Cuidados de saude e disponibilidade

**Acesso:** voluntario.


| `POST` | `/api/animais/{id}/cuidados` | Registra cuidado do tipo `VACINA`, `VERMIFUGACAO` ou `CASTRACAO`. |
| `PUT` | `/api/animais/{id}/disponivel` | Muda o status para `DISPONIVEL` se houver ao menos um registro de vacina e um de castracao. |

Corpo para registrar cuidado:

```json
{
  "tipo": "VACINA",
  "dataRealizacao": "2026-10-25",
  "descricao": "Vacinacao basica"
}
```

Registre separadamente a castracao, alterando `tipo` para `CASTRACAO`. Sem os dois cuidados, a tentativa de disponibilizar o animal e rejeitada.

### RF05 — Busca publica e paginada

**Acesso:** publico.


| `GET` | `/api/animais/disponiveis` | Lista somente animais `DISPONIVEL`, com filtros e paginacao. |

Filtros opcionais: `especie`, `porte`, `sexo`, `idadeMinima` e `idadeMaxima`. Os limites de idade sao inclusivos, em anos estimados. Parametros de pagina: `page` (comeca em zero) e `size`.

```text
GET /api/animais/disponiveis?especie=Cachorro&porte=Grande&idadeMinima=1&idadeMaxima=8&page=0&size=5
```

Idades negativas e idade minima maior que a maxima sao rejeitadas.

### RF06 — Questionario do adotante

**Acesso:** adotante. A identidade vem do token; nao e aceito um ID de usuario enviado pelo cliente.


| `POST` | `/api/questionarios` | Cria ou atualiza o questionario do adotante autenticado. |
| `GET` | `/api/questionarios/meu-perfil` | Retorna apenas o questionario do adotante autenticado. |

Campos: `tipoMoradia`, `temQuintal`, `janelasComTela`, `outrosAnimais` e `horasForaDeCasa`.

```json
{
  "tipoMoradia": "CASA",
  "temQuintal": true,
  "janelasComTela": true,
  "outrosAnimais": "Nenhum",
  "horasForaDeCasa": 8
}
```

### RF07 — Candidaturas e pre-triagem

**Acesso:** adotante.


| `POST` | `/api/candidaturas` | Cria uma candidatura para o `animalId` enviado. |
| `GET` | `/api/candidaturas/minhas` | Lista somente candidaturas do adotante autenticado. |

O questionario deve estar preenchido. O limite e de duas candidaturas `EM_ANALISE` simultaneamente. A candidatura pode ser feita para animal `DISPONIVEL` ou `EM_PROCESSO_DE_ADOCAO`. A primeira candidatura aceita muda o animal para `EM_PROCESSO_DE_ADOCAO`.

Regras automaticas: cao/cachorro de porte grande exige quintal; gato exige janelas com tela. Quando a condicao nao e atendida, a candidatura e criada como `RECUSADA`, com o motivo registrado.

```json
{
  "animalId": 1
}
```

### RF08 — Triagem e recusa

**Acesso:** voluntario.

| `POST` | `/api/candidaturas/{id}/triagem` | Registra a data e hora da visita e a recomendacao. |

```json
{
  "agendaVisita": "2026-11-01T14:00:00",
  "recomendacao": "APROVAR",
  "motivoRecusa": null
}
```

`recomendacao` aceita `APROVAR` ou `RECUSAR`. Recusar exige `motivoRecusa`; a candidatura passa a `RECUSADA`. Se nao restarem candidaturas em analise para o animal, ele volta para `DISPONIVEL`. Uma recomendacao `APROVAR` mantem a candidatura em analise ate a decisao do coordenador.

### RF09 — Aprovacao da adocao

**Acesso:** coordenador.


| `POST` | `/api/candidaturas/{id}/aprovar` | Aprova candidatura com triagem favoravel. |

A operacao e transacional: aprova a candidatura, muda o animal para `ADOTADO`, recusa as demais candidaturas em analise para o mesmo animal e cria acompanhamentos para 30, 90 e 180 dias apos a aprovacao. Responde `204 No Content` quando conclui.

### RF10 — Acompanhamento apos adocao

**Acesso:** voluntario para registrar visitas; voluntario ou coordenador para consultar acompanhamentos.


| `GET` | `/api/acompanhamentos/animal/{animalId}` | Lista acompanhamentos e seus IDs para o animal. |
| `POST` | `/api/acompanhamentos/{id}/visita` | Registra resultado e identifica o voluntario que realizou a visita. |

```json
{
  "resultado": "Animal esta adaptado ao novo lar."
}
```

Depois de registrar a visita, o acompanhamento passa a `REALIZADO`.

### RF11 — Devolucao

**Acesso:** voluntario ou coordenador.


| `POST` | `/api/animais/{id}/devolucao` | Registra a devolucao do animal adotado pelo usuario indicado. |

```json
{
  "adotanteId": 1
}
```

A API exige que exista uma adocao aprovada daquele animal para o adotante. Na mesma transacao, o animal volta para `EM_TRATAMENTO`, os acompanhamentos pendentes ou atrasados sao cancelados e o adotante fica bloqueado para novas candidaturas por um ano.

### RF12 — Consulta e cancelamento das proprias candidaturas

**Acesso:** adotante.

| `GET` | `/api/candidaturas/minhas` | Lista candidaturas do usuario autenticado. |
| `DELETE` | `/api/candidaturas/{id}/cancelar` | Cancela uma candidatura propria que ainda esteja `EM_ANALISE`. |

Se todas as candidaturas do animal forem `RECUSADA` ou `CANCELADA`, o status do animal volta para `DISPONIVEL`. O cancelamento responde `204 No Content`.

### RF13 — Relatorio de adocoes por mes

**Acesso:** coordenador.


| `GET` | `/api/relatorios/adocoes-por-mes` | Retorna a quantidade de adocoes aprovadas agrupadas por ano e mes. |

### RF14 — Tempo medio de adocao por especie e porte

**Acesso:** coordenador.


| `GET` | `/api/relatorios/tempo-medio-adocao` | Retorna, por especie e porte, o numero de adocoes e a media de dias entre a entrada do animal e a aprovacao. |

Animais cadastrados antes da inclusao de `dataEntrada` e sem essa informacao nao entram no calculo, pois a data original nao pode ser inferida. Novos cadastros recebem essa data automaticamente.

### RF15 — Acompanhamentos atrasados

**Acesso:** coordenador.


| `GET` | `/api/relatorios/acompanhamentos-atrasados` | Lista acompanhamentos cuja data prevista ja passou e que ainda nao foram realizados. |

Ao consultar o relatorio, registros vencidos que ainda estejam `PENDENTE` sao atualizados para `ATRASADO`.

## Status usados pela API

- Animais: `EM_TRATAMENTO`, `DISPONIVEL`, `EM_PROCESSO_DE_ADOCAO`, `ADOTADO`.
- Candidaturas: `EM_ANALISE`, `APROVADA`, `RECUSADA`, `CANCELADA`.
- Acompanhamentos: `PENDENTE`, `REALIZADO`, `ATRASADO`, `CANCELADO`.
- Cuidados: `VACINA`, `VERMIFUGACAO`, `CASTRACAO`.

## Validacao automatizada

Execute `.\mvnw.cmd clean verify` para compilar, rodar toda a suite e gerar o relatorio JaCoCo. A ultima execucao validada terminou com 24 testes, sem falhas ou erros. O relatorio HTML fica em `target/site/jacoco/index.html`.

### Como os testes estao organizados

**Testes unitarios de servico** ficam em `src/test/java/com/ong/adocao/service`. Eles instanciam os servicos com dependencias simuladas pelo Mockito, sem iniciar o Spring nem acessar o banco. Verificam regras de negocio em cenarios controlados:

- `CandidaturaServiceTest`: pre-triagem, limite de candidaturas, cancelamento e recusa.
- `AdocaoServiceTest`: aprovacao, encerramento das demais candidaturas e criacao dos acompanhamentos.
- `DevolucaoServiceTest`: retorno do animal, cancelamento de acompanhamentos e bloqueio temporario do adotante.
- `RelatorioServiceTest`: calculo do tempo medio e marcacao de acompanhamentos atrasados.

Esses testes sao rapidos e isolam a logica, mas nao confirmam que o Spring conectou os componentes, que as consultas JPA funcionam no banco ou que as rotas HTTP estao corretas.

**Testes de integracao HTTP** ficam em `src/test/java/ifsp/apis/ONGAnimais`. Eles iniciam o contexto real da aplicacao com `@SpringBootTest`, usam MockMvc para enviar requisicoes HTTP sem abrir uma porta de rede e usam o banco H2 configurado para os testes. Assim, exercitam em conjunto controllers, validacao, seguranca/JWT, servicos e persistencia.

- `ApiEndpointsIntegrationTest`: percorre os fluxos de cadastro/login/questionario; cadastro e disponibilizacao de animais; cuidados de saude; busca com filtros; candidatura, triagem, aprovacao, acompanhamento, devolucao e relatorios. Tambem verifica erros de validacao, autenticacao e autorizacao, recusa na triagem e cancelamento de candidatura.
- `OngAnimaisApplicationTests`: verifica inicializacao do contexto e das contas de demonstracao, busca publica, filtro etario, login/JWT e permissoes por perfil.

Considerando os mapeamentos declarados nos controllers, os testes HTTP exercitam as 19 rotas da API, incluindo casos de sucesso e, onde aplicavel, respostas de erro ou acesso negado. Isso nao significa que todas as combinacoes de entrada ou todos os ramos internos do codigo estejam cobertos.

**Teste de seguranca**: `JwtUtilTest` confere a geracao de token compativel com o decoder do Spring Security e a rejeicao de token com assinatura alterada.

### O que o JaCoCo mede

JaCoCo registra quais instrucoes, linhas e desvios foram executados durante a suite. Ele ajuda a encontrar codigo ainda sem exercicio, mas percentual de cobertura nao prova que um comportamento esta correto. A cobertura de rotas e a cobertura de regras sao verificadas pelos testes e asserts; o relatorio e um indicador complementar, nao uma garantia de cobertura total.
