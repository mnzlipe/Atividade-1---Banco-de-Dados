# Atividade Teórica: Usuários Especialistas, IA e Distribuição Segura de Dados

**Aluno(s):** Layla Geovanna Menezes, Luiz Felipe Menezes, Rayssa Anastácio Veloso, Patrick Felipe Dellalibera
**Turma:** Banco de Dados 2026.2 - G2
**Data:** 26/11/2026
**Repositório Git:** [https://github.com/usuario/atividade-bd](https://github.com/mnzlipe/Atividade-1---Banco-de-Dados)

---

## Resumo Executivo

Ferramentas de IA generativa (geradores de *text-to-SQL*, chatbots e assistentes de análise) permitem que usuários que dominam o negócio, mas não necessariamente o modelo de dados, escrevam consultas complexas sem a mediação de programadores. Isso aumenta a autonomia analítica, mas traz riscos concretos: consultas logicamente erradas que "parecem certas", exposição de dados pessoais, consultas pesadas que degradam o SGBD, vazamento de informações por *prompts* enviados a serviços externos e uso indevido de privilégios.

A posição do grupo é que **a IA não deve ser proibida, e sim tratada como um usuário não confiável que age em nome de um usuário confiável**. Por isso, a segurança não pode depender de a IA "acertar" a consulta: ela tem de estar no próprio banco. A abordagem defendida é a **defesa em profundidade**, com uma camada de dados curada pelo DBA, que tem: (1) um esquema analítico composto apenas de *views* que já aplicam minimização de dados (LGPD); (2) *roles* por perfil que seguem o menor privilégio, somente leitura; (3) limites de tempo, conexões e memória por usuário, de preferência numa réplica de leitura; (4) auditoria por usuário com pgAudit e `pg_stat_statements`; e (5) uma política de uso de IA que proíbe enviar dados reais a ferramentas externas e exige revisão humana de consultas que fundamentam decisões. Nesse cenário o DBA deixa de ser apenas um "porteiro" que libera acessos e passa a ser o **curador e guardião da camada de dados** que a IA consome.

---

## 1. Desenvolvimento Teórico

### 1.1 O que é o DBA e quais suas funções?

Segundo Silberschatz, Korth e Sudarshan (2012), uma das principais razões para usar um SGBD é ter **controle central** sobre os dados e sobre os programas que os acessam. A pessoa que exerce esse controle central é o **administrador de banco de dados (DBA — *Database Administrator*)**. Elmasri e Navathe (2018) complementam que o DBA responde por autorizar o acesso ao banco, coordenar e monitorar seu uso e adquirir os recursos de software e hardware necessários. Também é ele quem responde por falhas de segurança e por tempos de resposta ruins.

É importante não confundir o DBA com o **administrador de dados (AD)**. O AD decide *quais* dados a organização deve guardar e quais políticas os regem, numa visão de negócio. O DBA implementa e opera essas decisões no SGBD, numa visão técnica. No cenário da atividade, a LGPD e a governança de IA exigem que os dois papéis conversem: o AD (ou o encarregado/DPO) define, por exemplo, que analistas não precisam ver CPF, e o DBA materializa essa regra em *views* e permissões.

As funções clássicas do DBA (Silberschatz; Korth; Sudarshan, 2012) e sua relação com o cenário de IA:

| Função do DBA | O que é | Relação com o uso de IA por usuários especialistas |
|---|---|---|
| **Definição do esquema** | Criar o esquema original do banco executando instruções de DDL (`CREATE TABLE`, `CREATE SCHEMA`...). | A IA gera SQL a partir do que "enxerga" do esquema. Um esquema bem projetado, com nomes claros, chaves e comentários (`COMMENT ON`), reduz alucinações (colunas e tabelas inexistentes) e erros de junção. O DBA também pode criar um **esquema separado (`analytics`)**, que é o único exposto à IA. |
| **Definição da estrutura de armazenamento e do método de acesso** | Definir estruturas físicas: índices, particionamento, *tablespaces*. | Consultas geradas por IA costumam ignorar índices, fazer *full scans* e produtos cartesianos. O DBA cria índices para os padrões de consulta observados, particiona tabelas grandes (ex.: vendas por mês) e pode direcionar a carga analítica a uma réplica. |
| **Modificação do esquema e da organização física** | Alterar o esquema para refletir novas necessidades ou melhorar o desempenho. | Quando várias consultas de IA calculam a mesma métrica de formas diferentes, o DBA pode criar uma *view* ou *materialized view* com a **definição oficial** (ex.: "receita líquida"). Assim a IA reutiliza a definição em vez de reinventá-la. |
| **Concessão de autorização para acesso aos dados** | Decidir quem acessa qual parte do banco e com quais operações (`GRANT`/`REVOKE`). | É a função mais crítica no cenário. A IA age **com as credenciais do usuário**: tudo o que o usuário pode fazer, a consulta gerada também pode. A autorização é, portanto, a última barreira real contra exposição e alteração indevida de dados. |
| **Especificação de restrições de integridade** | Definir regras que os dados devem obedecer (`PRIMARY KEY`, `FOREIGN KEY`, `CHECK`, `NOT NULL`, `UNIQUE`). | Se um usuário tiver permissão de escrita, um `UPDATE` ou `DELETE` gerado por IA sem `WHERE` correto pode corromper dados. As restrições impedem estados inválidos, mas **não impedem operações válidas e erradas**: apagar todos os clientes de uma cidade é "íntegro", mas desastroso. Por isso, para usuários especialistas, a recomendação é acesso somente leitura. |
| **Manutenção de rotina** | Backups, monitoramento de espaço e desempenho, atualizações. | Backups e *Point-in-Time Recovery* permitem recuperar o banco após um comando destrutivo. O monitoramento (`pg_stat_activity`, `pg_stat_statements`) identifica consultas abusivas vindas de IA. |

**Analogia:** o DBA funciona como o bibliotecário-chefe de um acervo com obras raras. Ele organiza as estantes (esquema e armazenamento), decide quem entra em cada sala (autorização), garante que nenhum livro seja colocado no lugar errado (integridade) e restaura as cópias de segurança. A IA é um assistente novo e muito rápido, mas que às vezes erra a estante e não sabe o que é confidencial. O bibliotecário não deve deixá-lo sozinho na sala de obras raras.

### 1.2 Perfis de usuários de banco de dados

Silberschatz, Korth e Sudarshan (2012) classificam os usuários pela forma como interagem com o sistema. A tabela a seguir resume os perfis pedidos na atividade:

| Perfil | Como interage | Vantagens | Limitações / riscos |
|---|---|---|---|
| **Usuários navegantes** (*naïve users*) | Por interfaces e aplicações prontas (formulários, telas, *apps*). Ex.: atendente que cadastra uma venda, cliente que consulta um pedido. Não escrevem SQL. | Uso simples. O acesso fica naturalmente restrito ao que a aplicação permite. | Dependem totalmente da aplicação. Não conseguem fazer análises novas. O risco está concentrado na aplicação (ex.: *SQL injection*), e não no usuário. |
| **Programadores de aplicações** | Escrevem programas (Python, Java etc.) que acessam o banco via SQL embutido, ORMs, APIs. | Produzem código revisado, testado e versionado. Podem aplicar regras de negócio e validações. | Toda nova necessidade analítica vira demanda de desenvolvimento, o que é lento. É justamente esse gargalo que a IA "contorna" no cenário. |
| **Usuários sofisticados** | Interagem sem escrever programas, formulando consultas em uma linguagem de consulta (SQL) ou com ferramentas de análise (OLAP, BI, *data mining*). Ex.: analistas. | Autonomia e rapidez na análise *ad hoc*. | Acesso direto ao SQL. Consultas mal escritas podem ser lentas ou retornar resultados errados. |
| **Usuários especializados / especialistas** | Na definição de Silberschatz, são usuários sofisticados que escrevem **aplicações especializadas** que não se encaixam no processamento tradicional (sistemas CAD, sistemas especialistas, modelagem). No sentido desta atividade, são profissionais com **domínio do negócio** (analistas financeiros, gerentes comerciais, cientistas de dados) que constroem consultas complexas e relatórios por conta própria. | Alto valor analítico. Entendem o significado do dado melhor que o programador. | Costumam ter domínio do negócio maior que o domínio técnico do SGBD (planos de execução, `NULL`, concorrência, segurança). |

**Observação terminológica:** a literatura clássica distingue "sofisticados" (fazem consultas, sem programar) de "especializados" (desenvolvem aplicações não tradicionais). No cenário proposto, a IA **borra essa fronteira**: um usuário especialista sem formação em SQL passa a produzir, via IA, consultas e até *scripts* que antes só um programador ou um usuário sofisticado escreveria.

**Por que o usuário especialista é o foco?**

1. **Combina acesso direto com conhecimento técnico parcial.** O usuário navegante é protegido pela aplicação e o programador passa por revisão de código. O usuário especialista com IA, ao contrário, executa SQL *ad hoc* direto no banco, **sem nenhuma camada intermediária de revisão**.
2. **A IA amplifica a capacidade, mas não a compreensão.** O usuário consegue gerar uma consulta com 5 *joins* e funções de janela, mas pode não ser capaz de verificar se ela está certa, se é eficiente ou se expõe dados que não deveria.
3. **Ele tende a ter privilégios amplos.** Analistas costumam receber "acesso de leitura a tudo" por conveniência. Com IA, esse acesso amplo vira uma superfície grande de exposição.
4. **Ele é o elo com o mundo externo.** É quem copia esquemas, mensagens de erro e amostras de dados para um chatbot externo.

### 1.3 Riscos do uso de IA por usuários especialistas

O OWASP Top 10 for LLM Applications (OWASP, 2025) cataloga riscos de sistemas com modelos de linguagem que se aplicam diretamente ao cenário, como *Sensitive Information Disclosure* (LLM02), *Improper Output Handling* (LLM05), *Excessive Agency* (LLM06), *Misinformation* (LLM09) e *Unbounded Consumption* (LLM10). A seguir, os riscos concretos analisados pelo grupo.

#### Risco 1 — Consultas sintaticamente corretas, mas logicamente erradas

É o risco mais perigoso, porque **não gera erro**: a consulta roda e devolve um número plausível. Benchmarks acadêmicos de *text-to-SQL*, como o BIRD (Li et al., 2023), mostram que, em bancos reais e "sujos", os modelos ainda ficam abaixo do desempenho humano. Dois exemplos típicos:

```sql
-- Pedido ao chatbot: "clientes que nunca compraram"
-- Consulta gerada (ERRADA se existir alguma venda com cliente_id NULL):
SELECT id, nome FROM clientes
WHERE id NOT IN (SELECT cliente_id FROM vendas);
-- Se a subconsulta contiver um NULL, "id NOT IN (..., NULL)" nunca é TRUE
-- e o resultado é VAZIO. O usuário conclui que "todos já compraram".

-- Forma correta:
SELECT c.id, c.nome FROM clientes c
WHERE NOT EXISTS (SELECT 1 FROM vendas v WHERE v.cliente_id = c.id);
```

```sql
-- Pedido: "receita total por região"
-- Consulta gerada (ERRADA): o JOIN com itens multiplica cada venda
-- pelo número de itens, inflando a receita.
SELECT v.regiao, SUM(v.valor_total)
FROM vendas v JOIN itens_venda i ON i.venda_id = v.id
GROUP BY v.regiao;
```

**Impacto:** afeta a **integridade da informação** (não do dado armazenado, mas do resultado usado na decisão). Relatórios errados levam a decisões erradas e, se forem divulgados, a inconsistência entre áreas ("cada um tem uma receita diferente"). Se o usuário tiver permissão de escrita, o mesmo tipo de erro num `UPDATE`/`DELETE` corrompe o dado armazenado.

#### Risco 2 — Exposição de dados pessoais nas respostas

Ao pedir "liste os maiores clientes", a IA tende a gerar `SELECT *`, trazendo nome, CPF, endereço, telefone e data de nascimento. O resultado é exportado para planilhas, enviado por e-mail ou colado em apresentações.

**Impacto:** violação dos princípios de **finalidade** e **necessidade** da LGPD (art. 6º, I e III), que exigem limitar o tratamento ao mínimo necessário. A organização (controladora) responde pelo incidente (arts. 42 e 48). Vale uma precisão terminológica: pela LGPD, **CPF e endereço são dados pessoais**, mas **não são "dados pessoais sensíveis"**. Essa categoria (art. 5º, II) abrange origem racial ou étnica, convicção religiosa, opinião política, saúde, vida sexual, dados genéticos ou biométricos, entre outros. Numa clínica, por exemplo, diagnósticos são dados sensíveis e exigem proteção ainda maior.

#### Risco 3 — Degradação de performance e disponibilidade

Consultas geradas por IA frequentemente: fazem produtos cartesianos por esquecer a condição de junção; aplicam funções sobre colunas indexadas (`WHERE date(data_venda) = ...`), impedindo o uso do índice; ordenam milhões de linhas sem `LIMIT`; ou abrem uma transação e a deixam aberta. No PostgreSQL, uma transação longa também impede o `VACUUM` de remover versões antigas de linhas, o que causa *bloat* (inchaço das tabelas).

**Impacto:** afeta a **disponibilidade** (um dos pilares da segurança da informação). Uma consulta analítica pesada no servidor de produção pode deixar lento o sistema de vendas usado pelos atendentes (usuários navegantes). Corresponde ao risco *Unbounded Consumption* (OWASP, 2025).

#### Risco 4 — Vazamento de dados por *prompts* enviados a IAs externas

Para obter ajuda, o usuário cola no chatbot o DDL das tabelas, mensagens de erro com valores reais ou até amostras de linhas ("aqui estão 50 clientes, por que minha consulta falha?"). Um caso amplamente noticiado ocorreu na Samsung em 2023: funcionários colaram código-fonte confidencial no ChatGPT, e a empresa restringiu o uso dessas ferramentas (Bloomberg, 2023).

**Impacto:** quebra de **confidencialidade** fora do perímetro da empresa. Dependendo do serviço, os dados podem ser retidos ou usados para treinamento. Se o provedor estiver fora do Brasil, pode haver **transferência internacional de dados pessoais**, que a LGPD só admite nas hipóteses do art. 33. O próprio esquema do banco já é informação sensível para um atacante, porque revela nomes de tabelas, colunas e regras de negócio.

#### Risco 5 — Escalada de privilégios e uso indevido

Quando uma consulta falha por falta de permissão, é comum a IA sugerir "execute `GRANT ALL ...`" ou "conecte-se como `postgres`". Usuários com credenciais compartilhadas ou com o papel de dono das tabelas podem seguir a sugestão. Outro vetor: funções `SECURITY DEFINER` mal escritas (sem `search_path` fixo) permitem que um usuário execute código com os privilégios do dono da função (PostgreSQL, 2024).

**Impacto:** perda do controle de acesso. Usuários passam a ler ou alterar dados fora do seu escopo, e a trilha de auditoria fica comprometida quando todos usam a mesma conta.

#### Risco 6 — Agentes com acesso direto ao banco (*Excessive Agency*)

Quando a IA não só sugere, mas **executa** as consultas (agentes conectados ao banco), ela passa a agir sozinha. Um texto malicioso armazenado no próprio banco (ex.: o campo "observação" de um cliente) pode conter instruções que manipulam o agente, o que se chama *prompt injection* indireta (OWASP, 2025, LLM01).

**Impacto:** o agente pode executar operações não pretendidas pelo usuário. O dano possível é exatamente o tamanho dos privilégios da credencial usada, o que reforça o menor privilégio.

**Síntese:** os riscos atingem as três propriedades da segurança da informação: **confidencialidade** (riscos 2, 4 e 5), **integridade** (riscos 1, 5 e 6) e **disponibilidade** (risco 3).

### 1.4 Distribuição segura de dados

Antes das estratégias, é preciso diferenciar três conceitos que costumam ser confundidos:

- **Permissão (privilégio):** o direito de executar uma operação específica sobre um objeto específico, por exemplo `SELECT` na *view* `clientes_visiveis` ou `USAGE` no esquema `analytics`. No PostgreSQL, é concedida com `GRANT` e removida com `REVOKE` (PostgreSQL, 2024a).
- **Role:** no PostgreSQL, *role* é a entidade única que representa tanto **usuários** quanto **grupos**. Uma *role* com o atributo `LOGIN` funciona como usuário; uma *role* `NOLOGIN` funciona como grupo, que agrupa privilégios. Uma *role* pode ser membro de outra e herdar seus privilégios (PostgreSQL, 2024b).
- **View:** uma **consulta armazenada** com nome, que se comporta como uma tabela virtual. Ela não guarda os dados (exceto a *materialized view*). Para segurança, o ponto central é que o usuário pode receber `SELECT` na *view* **sem ter acesso à tabela base**, porque o PostgreSQL verifica as permissões das tabelas usando o **dono da view** (PostgreSQL, 2024c).

Analogia: a **permissão** é a chave de uma sala, a **role** é o crachá que reúne várias chaves e a **view** é uma vitrine que mostra só parte do que está no depósito.

#### a) Princípio do menor privilégio (*least privilege*)

Formulado por Saltzer e Schroeder (1975): todo programa e todo usuário deve operar com o **menor conjunto de privilégios necessário** para realizar sua tarefa. O NIST trata o princípio como controle de segurança (NIST SP 800-53, controle AC-6).

**Por que é central no cenário:** a IA executa SQL com as credenciais do usuário. Logo, **o pior caso de qualquer consulta gerada é limitado pelo privilégio do usuário**. Se o analista só tem `SELECT` em *views* do esquema `analytics`, nenhum erro ou alucinação da IA consegue apagar uma tabela ou ler o CPF de alguém. Na prática:

- Revogar privilégios padrão de `PUBLIC` (no PostgreSQL, todo usuário é membro implícito de `PUBLIC`).
- Não conceder `pg_read_all_data` nem acesso ao esquema principal a usuários analíticos.
- Contas individuais, nunca compartilhadas.
- Somente leitura para usuários especialistas. Escrita apenas via aplicação.

#### b) Uso de *views* para limitar colunas e linhas

*Views* implementam dois tipos de restrição:

- **Restrição vertical (colunas):** a *view* expõe apenas as colunas necessárias, ou versões transformadas delas (mascaradas ou agregadas). Ex.: em vez de `data_nascimento`, expor `faixa_etaria`.
- **Restrição horizontal (linhas):** a cláusula `WHERE` limita as linhas. Ex.: um gerente regional só vê as vendas da sua região, filtrando por `current_user`.

Cuidados técnicos importantes:

- Usar `WITH (security_barrier = true)` em *views* que filtram linhas. Sem isso, o otimizador pode avaliar uma função do usuário **antes** do filtro da *view*, e uma função que "vaza" seus argumentos (ex.: via `RAISE NOTICE`) pode revelar linhas ocultas (PostgreSQL, 2024d).
- *Views* também simplificam o esquema para a IA. Uma *view* `vendas_mensais_uf` com a métrica pronta reduz a chance de o modelo errar *joins* e agregações (risco 1).
- **Agregação não é anonimização automática.** Grupos muito pequenos (ex.: "vendas de clientes de 90 anos em uma cidade pequena") permitem reidentificar pessoas. Uma técnica simples é suprimir grupos pequenos com `HAVING count(DISTINCT cliente_id) >= k`.
- **Hash de CPF não é anonimização.** Como o CPF tem poucas combinações possíveis (cerca de 10⁹), um *hash* sem segredo pode ser revertido por força bruta. Pela LGPD, dado anonimizado é aquele que não pode ser revertido com meios razoáveis (art. 5º, III, e art. 12). Um *hash* simples de CPF é, no máximo, pseudonimização.

**Complemento — *Row-Level Security* (RLS):** o PostgreSQL também permite políticas de linha diretamente na tabela (`CREATE POLICY`). Um detalhe relevante: **o dono da tabela ignora o RLS** (a menos que se use `FORCE ROW LEVEL SECURITY`). Como uma *view* comum acessa as tabelas com os privilégios do seu dono, combinar *view* e RLS exige criar a *view* com `security_invoker = true` (PostgreSQL 15 ou superior) (PostgreSQL, 2024e).

#### c) *Roles* customizadas por perfil

Em vez de conceder permissões a cada pessoa, cria-se uma *role* de grupo (`NOLOGIN`) por **perfil funcional** e os usuários se tornam membros dela. É o modelo **RBAC (*Role-Based Access Control*)**. Exemplo de perfis:

| Role (grupo) | Quem | Acesso |
|---|---|---|
| `r_app_vendas` | Aplicação de vendas | `SELECT/INSERT/UPDATE` nas tabelas operacionais necessárias |
| `r_analista` | Analistas especialistas | `SELECT` apenas em *views* do esquema `analytics` (sem dado identificável) |
| `r_gerente_regional` | Gerentes | `SELECT` em *views* filtradas pela região do usuário |
| `r_dpo_auditoria` | Encarregado/DPO, auditoria | Leitura de logs e de relatórios de acesso |

Vantagens: quando alguém muda de função, basta trocar a *role* de grupo. Permissões ficam documentadas e revisáveis. Novas pessoas recebem um perfil padronizado em vez de "copiar o acesso de um colega".

#### d) Controle de tempo de execução e concorrência

Parâmetros do PostgreSQL que podem ser fixados por usuário (`ALTER ROLE ... SET`) (PostgreSQL, 2024f):

- `statement_timeout`: cancela qualquer comando que ultrapasse o tempo definido.
- `idle_in_transaction_session_timeout`: encerra sessões que ficam com transação aberta e ociosa.
- `lock_timeout`: evita que uma consulta fique presa esperando bloqueios.
- `work_mem`: limita a memória por operação de ordenação/*hash*.
- `CONNECTION LIMIT` na *role*: limita conexões simultâneas.
- `default_transaction_read_only = on`: transações somente leitura por padrão.

**Ponto crítico:** `statement_timeout` e `default_transaction_read_only` podem ser **alterados pelo próprio usuário** na sessão (`SET statement_timeout = 0`). Por isso, eles são **proteção de conveniência**, não barreira de segurança. A barreira real contra escrita é **não conceder** `INSERT/UPDATE/DELETE`. Contra consultas abusivas, o DBA complementa com monitoramento ativo (`pg_stat_activity` + `pg_cancel_backend`), *pooler* de conexões e, principalmente, com uma **réplica de leitura** dedicada à análise. Assim, mesmo uma consulta ruim não afeta o servidor de produção.

#### e) Auditoria e rastreamento por usuário

Auditar é registrar **quem fez o quê, quando e de onde**. Isso permite investigar incidentes, comprovar conformidade e identificar padrões abusivos. No PostgreSQL:

- **Logs nativos:** `log_connections`, `log_disconnections`, `log_line_prefix` com `%u` (usuário) e `%a` (aplicação), e `log_min_duration_statement` para registrar consultas lentas (PostgreSQL, 2024g).
- **`pg_stat_statements`:** estatísticas agregadas por consulta normalizada e por usuário (número de execuções, tempo total e médio). Identifica quem consome mais recursos (PostgreSQL, 2024h).
- **pgAudit:** extensão que gera registros de auditoria detalhados por classe (`READ`, `WRITE`, `DDL`, `ROLE`), configuráveis por *role* (pgAudit, 2024). Como `pgaudit.log` só pode ser alterado por superusuário, o usuário auditado não consegue desligá-lo.

A auditoria só tem valor com **contas individuais**: se todos usam `analista_geral`, o log não identifica ninguém. Os logs também contêm dados (inclusive valores nas consultas) e precisam de acesso restrito e prazo de retenção definido.

#### f) Conformidade com a LGPD

A Lei nº 13.709/2018 (LGPD) estabelece obrigações que se traduzem diretamente em controles de banco de dados:

| Dispositivo da LGPD | Exigência | Controle técnico correspondente |
|---|---|---|
| Art. 6º, I, II e III (finalidade, adequação, necessidade) | Tratar só o necessário para a finalidade informada | *Views* com minimização de colunas; analistas sem acesso a dado identificável |
| Art. 6º, VII e VIII (segurança, prevenção) | Medidas para proteger os dados e prevenir danos | Menor privilégio, *roles*, criptografia em trânsito (TLS) e autenticação SCRAM |
| Art. 6º, X (responsabilização e prestação de contas) | Demonstrar as medidas adotadas | Auditoria, documentação das *roles* e revisões periódicas de acesso |
| Art. 5º, III, e art. 12 (anonimização) | Dado anonimizado não é considerado dado pessoal | Agregações com supressão de grupos pequenos; extensões como PostgreSQL Anonymizer |
| Art. 33 (transferência internacional) | Só em hipóteses legais | Proibir envio de dados pessoais a IAs externas sem contrato e base legal adequados |
| Art. 37 (registro das operações) | Controlador e operador mantêm registro do tratamento | Logs e pgAudit |
| Art. 46 (segurança) | Medidas técnicas e administrativas contra acessos não autorizados | Todo o conjunto acima |
| Art. 48 (comunicação de incidente) | Comunicar à ANPD e ao titular incidentes com risco relevante | Logs permitem saber o que vazou, quando e para quem |

### 1.5 Atuação do DBA no cenário de IA

O DBA deixa de ser apenas quem "libera acesso" e assume cinco frentes:

1. **Definindo e mantendo esquemas.** Cria e mantém o esquema `analytics`, a camada que a IA consome. Documenta tabelas e colunas com `COMMENT ON`, o que gera um dicionário de dados que pode ser fornecido à IA **sem dados reais**. Publica *views* com as métricas oficiais, para que todos calculem "receita" da mesma forma.

2. **Redesenhando políticas de acesso.** Revisa privilégios herdados ("todo analista lê tudo"), migra para *roles* por perfil, remove contas compartilhadas e faz **revisões periódicas de acesso** (ex.: a cada trimestre, confirmar com os gestores quem ainda precisa de cada perfil).

3. **Monitorando consultas geradas por IA.** Acompanha `pg_stat_activity` (consultas em andamento) e `pg_stat_statements` (histórico de consumo). Configura `log_min_duration_statement` e `auto_explain` para capturar planos de consultas lentas, cancela consultas abusivas e identifica padrões que justificam um índice ou uma *materialized view*. O DBA pode exigir que ferramentas de IA identifiquem a sessão com `application_name` (ex.: `assistente-ia`), o que facilita filtrar o que veio da IA nos logs. Isso é uma convenção, não uma garantia, porque o próprio cliente define esse valor.

4. **Orientando o uso responsável da IA.** Junto com o encarregado (DPO) e a área de segurança, define uma política simples: nunca colar dados reais ou credenciais em IA externa; fornecer à IA apenas o esquema das *views* analíticas; preferir ferramentas corporativas com contrato de não retenção, ou modelos executados internamente; validar consultas com `EXPLAIN` e testar em amostras; e exigir revisão humana (por outro analista ou pelo DBA) para consultas que embasam decisões relevantes ou relatórios externos.

5. **Mantendo backups, índices e performance.** Backups regulares com teste de restauração e arquivamento de WAL para *Point-in-Time Recovery*, que permite voltar o banco ao instante anterior a um comando destrutivo. Réplica de leitura para separar a carga analítica da transacional. Índices baseados nos padrões reais de consulta. Rotinas de `VACUUM`/`ANALYZE` para manter as estatísticas que o otimizador usa.

### 1.6 Análise crítica: qual a melhor abordagem?

O grupo considerou três abordagens possíveis:

| Abordagem | Descrição | Avaliação |
|---|---|---|
| **A — Proibir a IA** | Usuários especialistas não podem usar IA para gerar SQL. | Fácil de declarar, difícil de garantir. Os usuários tendem a usar as ferramentas mesmo assim (*shadow AI*), sem nenhum controle. Perde-se o ganho de produtividade. |
| **B — Confiar na revisão do DBA** | Toda consulta gerada por IA passa pelo DBA antes de rodar. | Recria o gargalo que a IA veio resolver e não escala. Também depende do olho humano para detectar vazamentos, o que falha. |
| **C — Defesa em profundidade com camada curada** | A segurança fica no banco: o usuário só alcança dados já minimizados, com limites de recurso e auditoria. A revisão humana é reservada ao que é crítico. | **Posição do grupo.** |

**Justificativa da posição C:**

1. **A IA é não determinística, mas o controle de acesso é determinístico.** Não é possível garantir que o modelo nunca vai gerar `SELECT *` ou uma consulta errada. É possível garantir que, mesmo se gerar, o usuário **não tem privilégio** para ver o CPF ou alterar dados. A segurança não deve depender do comportamento correto da ferramenta.
2. **Cada camada cobre a falha da anterior.** Se a IA gerar uma consulta pesada, o `statement_timeout` e a réplica isolam o impacto. Se o usuário desligar o *timeout*, o monitoramento detecta. Se algo vazar, a auditoria mostra o quê e por quem.
3. **A camada curada reduz também o erro lógico.** *Views* com métricas oficiais e nomes claros dão à IA um modelo de dados mais simples, o que melhora a qualidade do SQL gerado (risco 1), e não só a segurança.
4. **Está alinhada à LGPD.** Minimização (art. 6º, III), segurança (art. 46) e responsabilização (art. 6º, X) passam a ser demonstráveis, e não apenas declaradas.
5. **Mantém o valor da IA.** O usuário especialista continua autônomo dentro de um perímetro seguro.

**Limitações reconhecidas:** a abordagem C exige trabalho contínuo do DBA (manter *views*, revisar acessos). Ela não elimina o risco de vazamento por *prompts*, que depende de comportamento humano e de política organizacional. E também não elimina o erro lógico, apenas o reduz, o que justifica a revisão humana obrigatória para relatórios críticos.

---

## 2. Exemplos e Casos

### 2.1 Caso: sistema de vendas de uma rede varejista

**Contexto:** uma rede com lojas em vários estados usa PostgreSQL 16. Os atendentes registram vendas por uma aplicação (usuários navegantes). A diretoria quer que **analistas comerciais** (usuários especialistas) usem um assistente de IA para gerar relatórios de vendas, e que **gerentes regionais** vejam apenas os dados da sua região. Antes do projeto, todos os analistas usavam a conta compartilhada `relatorios`, com `SELECT` em todas as tabelas, inclusive `clientes` com CPF, endereço e telefone.

**Tabelas principais (simplificadas):**

```sql
CREATE ROLE app_owner NOLOGIN;            -- dono dos objetos; ninguém faz login com ela
CREATE SCHEMA core AUTHORIZATION app_owner;
CREATE SCHEMA analytics AUTHORIZATION app_owner;

SET ROLE app_owner;

CREATE TABLE core.clientes (
    id                    bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    nome                  text NOT NULL,
    cpf                   char(11) NOT NULL UNIQUE,
    email                 text,
    telefone              text,
    endereco              text,
    cidade                text NOT NULL,
    uf                    char(2) NOT NULL,
    data_nascimento       date,
    data_cadastro         date NOT NULL DEFAULT current_date,
    ativo                 boolean NOT NULL DEFAULT true
);

CREATE TABLE core.vendas (
    id            bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    cliente_id    bigint NOT NULL REFERENCES core.clientes(id),
    regiao        text NOT NULL CHECK (regiao IN ('NORTE','NORDESTE','CENTRO-OESTE','SUDESTE','SUL')),
    data_venda    timestamptz NOT NULL DEFAULT now(),
    valor_total   numeric(12,2) NOT NULL CHECK (valor_total >= 0)
);

-- Mapeia cada gerente à(s) região(ões) que pode ver
CREATE TABLE core.usuario_regiao (
    usuario  text NOT NULL,
    regiao   text NOT NULL,
    PRIMARY KEY (usuario, regiao)
);

CREATE INDEX ON core.vendas (regiao, data_venda);
CREATE INDEX ON core.vendas (cliente_id);

COMMENT ON TABLE core.vendas IS 'Uma linha por venda concluída. valor_total já inclui descontos.';

RESET ROLE;
```

Note que `cliente_id` é `NOT NULL`, uma **restrição de integridade** que, de quebra, evita o problema do `NOT IN` com `NULL` mostrado no risco 1.

### 2.2 Exemplo de *view* `clientes_visiveis`

```sql
SET ROLE app_owner;

-- Restrição vertical: sem nome completo, CPF, e-mail, telefone, endereço
-- nem data de nascimento exata. Restrição horizontal: só clientes ativos.
CREATE VIEW analytics.clientes_visiveis
WITH (security_barrier = true) AS
SELECT
    c.id                                                     AS cliente_id,
    c.cidade,
    c.uf,
    (date_part('year', age(c.data_nascimento))::int / 10) * 10 AS faixa_etaria,  -- 20, 30, 40...
    date_trunc('month', c.data_cadastro)::date               AS mes_cadastro
FROM core.clientes c
WHERE c.ativo;

COMMENT ON VIEW analytics.clientes_visiveis IS
  'Clientes ativos sem dados identificáveis diretos. Use cliente_id para junções.';
COMMENT ON COLUMN analytics.clientes_visiveis.faixa_etaria IS
  'Década de idade (ex.: 30 = 30 a 39 anos).';
```

O `cliente_id` é um identificador interno: sozinho, ele não identifica a pessoa para o analista. Isso caracteriza **pseudonimização**, e não anonimização, porque o DBA ainda consegue fazer a ligação com a tabela original. Por isso a *view* continua sujeita à LGPD.

### 2.3 *Views* de vendas: filtro por linha e agregação segura

```sql
-- Gerente regional: vê apenas linhas da(s) sua(s) região(ões).
-- Como a view não é SECURITY DEFINER, current_user retorna o usuário
-- que está consultando, e não o dono da view.
CREATE VIEW analytics.vendas_minha_regiao
WITH (security_barrier = true) AS
SELECT v.id AS venda_id, v.cliente_id, v.regiao, v.data_venda, v.valor_total
FROM core.vendas v
WHERE v.regiao IN (
    SELECT ur.regiao FROM core.usuario_regiao ur
    WHERE ur.usuario = current_user
);

-- Analistas: métrica oficial já agregada, com supressão de grupos
-- pequenos para reduzir o risco de reidentificação.
CREATE VIEW analytics.vendas_mensais_uf AS
SELECT
    date_trunc('month', v.data_venda)::date AS mes,
    c.uf,
    count(*)                                AS qtd_vendas,
    count(DISTINCT v.cliente_id)            AS qtd_clientes,
    sum(v.valor_total)                      AS receita
FROM core.vendas v
JOIN core.clientes c ON c.id = v.cliente_id
GROUP BY 1, 2
HAVING count(DISTINCT v.cliente_id) >= 5;

COMMENT ON VIEW analytics.vendas_mensais_uf IS
  'Métrica OFICIAL de receita mensal por UF. Grupos com menos de 5 clientes são omitidos.';

RESET ROLE;
```

### 2.4 Exemplo de *roles* e permissões

```sql
-- 1) Endurecimento: ninguém conecta ou cria objetos "por padrão".
REVOKE ALL ON DATABASE vendas FROM PUBLIC;
REVOKE ALL ON SCHEMA public FROM PUBLIC;   -- no PG 15+ o CREATE já vem revogado
REVOKE ALL ON SCHEMA core FROM PUBLIC;

-- 2) Roles de grupo por perfil (NOLOGIN = não são usuários)
CREATE ROLE r_analista NOLOGIN;
CREATE ROLE r_gerente_regional NOLOGIN;

GRANT CONNECT ON DATABASE vendas TO r_analista, r_gerente_regional;
GRANT USAGE ON SCHEMA analytics TO r_analista, r_gerente_regional;

-- Analista: somente as views sem dado identificável
GRANT SELECT ON analytics.clientes_visiveis, analytics.vendas_mensais_uf TO r_analista;

-- Gerente: vendas da própria região + agregados
GRANT SELECT ON analytics.vendas_minha_regiao, analytics.vendas_mensais_uf TO r_gerente_regional;

-- Observe: NENHUM grant no schema core. As views acessam core com os
-- privilégios de app_owner, então o usuário lê a view sem ler a tabela.

-- 3) Usuários individuais (nunca compartilhados), com limites de recurso
CREATE ROLE ana_analista LOGIN PASSWORD 'definir-senha-forte'
    IN ROLE r_analista CONNECTION LIMIT 3;

-- Parâmetros de ALTER ROLE ... SET valem para a role de login, não são herdados do grupo
ALTER ROLE ana_analista SET statement_timeout = '60s';
ALTER ROLE ana_analista SET idle_in_transaction_session_timeout = '5min';
ALTER ROLE ana_analista SET lock_timeout = '5s';
ALTER ROLE ana_analista SET work_mem = '32MB';
ALTER ROLE ana_analista SET default_transaction_read_only = on;
ALTER ROLE ana_analista SET pgaudit.log = 'read';  -- exige pgAudit; usuário não pode desligar

CREATE ROLE bruno_gerente LOGIN PASSWORD 'definir-senha-forte'
    IN ROLE r_gerente_regional CONNECTION LIMIT 2;
ALTER ROLE bruno_gerente SET statement_timeout = '60s';
INSERT INTO core.usuario_regiao VALUES ('bruno_gerente', 'NORTE');
```

**O que acontece quando a IA erra:**

```sql
-- Conectada como ana_analista, a IA gera:
SELECT nome, cpf, telefone FROM core.clientes ORDER BY nome;
-- ERROR:  permission denied for schema core

-- A IA "sugere" conceder privilégio:
GRANT SELECT ON core.clientes TO ana_analista;
-- ERROR:  permission denied for table clientes  (ana não é dona nem tem GRANT OPTION)

-- Uma tentativa de escrita, mesmo se alguém tivesse concedido INSERT por engano:
DELETE FROM analytics.vendas_minha_regiao;
-- ERROR:  permission denied for view vendas_minha_regiao
```

Os erros são registrados no log, e o DBA passa a saber que alguém tentou acessar dados fora do escopo.

### 2.5 Monitoramento e auditoria (exemplos)

Configuração em `postgresql.conf` (trecho):

```ini
shared_preload_libraries = 'pg_stat_statements,pgaudit,auto_explain'
log_line_prefix = '%m [%p] %u@%d app=%a '
log_connections = on
log_disconnections = on
log_min_duration_statement = 2000      # registra consultas acima de 2 s
auto_explain.log_min_duration = 5000   # registra o plano das acima de 5 s
password_encryption = scram-sha-256
```

Consultas de analistas rodando há mais de 2 minutos:

```sql
SELECT pid, usename, application_name,
       now() - query_start AS duracao,
       left(query, 120)    AS consulta
FROM pg_stat_activity
WHERE state = 'active'
  AND pg_has_role(usename, 'r_analista', 'MEMBER')
  AND now() - query_start > interval '2 minutes'
ORDER BY duracao DESC;

-- Cancela a consulta (mantém a conexão); pg_terminate_backend encerra a sessão
SELECT pg_cancel_backend(12345);
```

Quem mais consome recursos (histórico):

```sql
SELECT r.rolname                                    AS usuario,
       s.calls                                      AS execucoes,
       round((s.total_exec_time / 1000)::numeric, 1) AS segundos_total,
       round(s.mean_exec_time::numeric, 1)          AS ms_medio,
       left(s.query, 100)                           AS consulta
FROM pg_stat_statements s
JOIN pg_roles r ON r.oid = s.userid
ORDER BY s.total_exec_time DESC
LIMIT 10;
```

Se a mesma consulta pesada aparece repetidas vezes, o DBA cria um índice ou uma *materialized view* e informa os analistas, transformando o monitoramento em melhoria do esquema.

### 2.6 Política de uso de IA (exemplo prático para os analistas)

| Pode | Não pode |
|---|---|
| Enviar à IA o dicionário das *views* do esquema `analytics` (nomes, tipos e comentários) | Colar linhas reais, exportações, CPF, nomes de clientes ou credenciais |
| Pedir à IA para explicar ou otimizar uma consulta | Executar sugestões de `GRANT`, `ALTER` ou conexão com outro usuário |
| Validar a consulta com `EXPLAIN` e conferir totais com a *view* oficial | Publicar relatório externo ou para a diretoria sem revisão por um segundo analista |
| Usar a ferramenta de IA corporativa aprovada | Usar contas pessoais em chatbots públicos para assuntos da empresa |

### 2.7 Comparação com outros cenários

- **Clínica:** a mesma arquitetura vale, mas com rigor maior, porque diagnósticos e exames são **dados pessoais sensíveis** (LGPD, art. 5º, II, e art. 11). Analistas de gestão veriam apenas agregados, como "atendimentos por especialidade por mês", nunca prontuários.
- **Biblioteca:** o histórico de empréstimos revela hábitos de leitura e pode indicar convicções religiosas ou políticas. Mesmo parecendo inofensivo, deve ser exposto à análise apenas de forma agregada ("empréstimos por categoria").

---

## 3. Referências

BLOOMBERG. **Samsung bans staff's AI use after spotting ChatGPT data leak**. Bloomberg, 2 maio 2023. Disponível em: https://www.bloomberg.com/news/articles/2023-05-02/samsung-bans-chatgpt-and-other-generative-ai-use-by-staff-after-leak. Acesso em: 26 nov. 2026.

BRASIL. **Lei nº 13.709, de 14 de agosto de 2018**. Lei Geral de Proteção de Dados Pessoais (LGPD). Brasília, DF: Presidência da República, 2018. Disponível em: https://www.planalto.gov.br/ccivil_03/_ato2015-2018/2018/lei/l13709.htm. Acesso em: 26 nov. 2026.

CAFEGEEK. **Perfis de Usuários de Banco de Dados**. Material de apoio da disciplina de Banco de Dados, 2026.

CAFEGEEK. **Regras das Atividades Teóricas**. Material de apoio da disciplina de Banco de Dados, 2026.

ELMASRI, R.; NAVATHE, S. B. **Sistemas de banco de dados**. 7. ed. São Paulo: Pearson, 2018.

LI, J. et al. Can LLM already serve as a database interface? A BIg bench for large-scale database grounded text-to-SQLs. In: **Advances in Neural Information Processing Systems (NeurIPS)**, 2023. Disponível em: https://bird-bench.github.io/. Acesso em: 26 nov. 2026.

NIST — NATIONAL INSTITUTE OF STANDARDS AND TECHNOLOGY. **SP 800-53 Rev. 5: Security and Privacy Controls for Information Systems and Organizations** (controle AC-6, *Least Privilege*). Gaithersburg, 2020. Disponível em: https://csrc.nist.gov/pubs/sp/800/53/r5/upd1/final. Acesso em: 26 nov. 2026.

OWASP. **OWASP Top 10 for LLM Applications 2025**. OWASP GenAI Security Project, 2025. Disponível em: https://genai.owasp.org/llm-top-10/. Acesso em: 26 nov. 2026.

PGAUDIT. **pgAudit — PostgreSQL Audit Extension**. 2024. Disponível em: https://github.com/pgaudit/pgaudit. Acesso em: 26 nov. 2026.

POSTGRESQL GLOBAL DEVELOPMENT GROUP. **PostgreSQL Documentation** (versão atual). Disponível em: https://www.postgresql.org/docs/current/. Capítulos consultados:
- (2024a) 5.8 *Privileges*: https://www.postgresql.org/docs/current/ddl-priv.html
- (2024b) 22 *Database Roles*: https://www.postgresql.org/docs/current/user-manag.html
- (2024c) *CREATE VIEW*: https://www.postgresql.org/docs/current/sql-createview.html
- (2024d) 41.5 *Rules and Privileges* (*security_barrier*): https://www.postgresql.org/docs/current/rules-privileges.html
- (2024e) 5.9 *Row Security Policies*: https://www.postgresql.org/docs/current/ddl-rowsecurity.html
- (2024f) 20.11 *Client Connection Defaults* (`statement_timeout` etc.): https://www.postgresql.org/docs/current/runtime-config-client.html
- (2024g) 20.8 *Error Reporting and Logging*: https://www.postgresql.org/docs/current/runtime-config-logging.html
- (2024h) F.32 *pg_stat_statements*: https://www.postgresql.org/docs/current/pgstatstatements.html
- *Writing SECURITY DEFINER Functions Safely* (em *CREATE FUNCTION*): https://www.postgresql.org/docs/current/sql-createfunction.html
- *Predefined Roles*: https://www.postgresql.org/docs/current/predefined-roles.html

POSTGRESQL ANONYMIZER. **PostgreSQL Anonymizer Documentation**. Disponível em: https://postgresql-anonymizer.readthedocs.io/. Acesso em: 26 nov. 2026.

SALTZER, J. H.; SCHROEDER, M. D. The protection of information in computer systems. **Proceedings of the IEEE**, v. 63, n. 9, p. 1278–1308, 1975.

SILBERSCHATZ, A.; KORTH, H. F.; SUDARSHAN, S. **Sistema de banco de dados**. 6. ed. Rio de Janeiro: Elsevier, 2012.

---

## 4. Conclusões

1. **O problema não é a IA, é o privilégio.** A IA executa SQL com as credenciais do usuário: o dano máximo de uma consulta gerada é exatamente o tamanho do privilégio concedido. Por isso o menor privilégio é o controle de maior retorno do trabalho.
2. **A segurança precisa estar no banco, não na ferramenta.** *Views*, *roles* e permissões são determinísticas e valem para qualquer cliente (IA, planilha, BI). Instruções dadas à IA ("não mostre CPF") não são controle de segurança.
3. **Conceitos parecidos têm papéis diferentes.** A *role* organiza **quem**, a permissão define **o que pode ser feito** e a *view* define **qual recorte do dado** é visível. A auditoria não impede nada, mas permite **provar e investigar**. Parâmetros como `statement_timeout` são proteção de conveniência, porque o próprio usuário pode alterá-los.
4. **Cuidado com falsas anonimizações.** Agregados com grupos pequenos e *hash* de CPF podem permitir reidentificação. Minimizar o dado é mais seguro do que tentar escondê-lo.
5. **O erro lógico continua sendo o risco mais silencioso.** Nenhuma permissão impede que um `NOT IN` com `NULL` ou um `JOIN` duplicado produza um número errado. Métricas oficiais em *views*, validação e revisão humana para relatórios críticos são indispensáveis.
6. **O papel do DBA se amplia.** Além de administrar o SGBD, ele passa a curar a camada de dados que a IA consome, monitorar o comportamento das consultas e orientar a organização, junto com o DPO, sobre o uso responsável da IA.

Assim como o DBA audita as consultas geradas pela IA, o grupo revisou o conteúdo deste trabalho, conferindo a sintaxe dos exemplos e o comportamento descrito na documentação oficial do PostgreSQL e no texto da LGPD.

---

## Link do Repositório Git

https://github.com/usuario/atividade-bd
