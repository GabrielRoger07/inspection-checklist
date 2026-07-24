# Sistema de Checklist de Inspeção

Aplicação web para registrar e acompanhar a avaliação diária dos checklists dos funcionários, organizada por setor. Cada setor tem seus próprios itens de checklist e, a cada dia, o gerente do setor avalia — para cada funcionário — o cumprimento de cada item, classificando-o em **Não conforme**, **Parcialmente conforme** ou **Totalmente conforme**, com anotação opcional. O sistema substitui as folhas impressas mensais, preserva todo o histórico e oferece uma visão mensal consolidada e relatórios por funcionário, por item e por período. **Admins** configuram setores, itens, funcionários e gerentes; cada **gerente** acessa apenas os setores sob sua responsabilidade.

---

## 1. Visão geral

Hoje o processo é manual e em papel:

- No início de cada mês são impressas folhas, uma por setor.
- Cada folha contém o **checklist do setor** e uma **tabela** onde as **linhas** são os funcionários daquele setor e as **colunas** são os dias do mês.
- O **gerente de cada setor** avalia diariamente se cada funcionário cumpriu o checklist.
- Quando algum item **não** é cumprido, o gerente faz uma anotação na célula correspondente (funcionário × dia).
- Cada setor tem seus **próprios itens de checklist**.

O objetivo é transformar esse processo em um **sistema web**, com registro digital, histórico consultável e relatórios. Diferente do papel, cada **item** do checklist será avaliado individualmente por funcionário a cada dia (ver seção 5).

## 2. Objetivos do sistema

- Digitalizar a avaliação diária dos checklists por setor, registrando **cada item** por funcionário por dia.
- Oferecer uma tela de lançamento ágil, focada no dia, para o gerente avaliar todos os funcionários do setor de uma vez.
- Classificar cada item em três níveis: **Não conforme**, **Parcialmente conforme** e **Totalmente conforme**, com anotação livre opcional.
- Restringir cada gerente aos setores sob sua responsabilidade.
- Oferecer uma visão mensal consolidada e relatórios por funcionário, por item e por período.
- Manter histórico mês a mês, preservando o texto do item como estava na época da avaliação.

### Não objetivos (por enquanto)

- App mobile nativo (o web responsivo atende).
- Folha de pagamento / RH completo.
- Integração com ponto eletrônico.
- Tratamento de funcionário que troca de setor no meio do mês (ver seção 12).

## 3. Stack técnica

- **Frontend:** React + TypeScript + Vite.
- **Banco de dados / Backend:** Supabase (PostgreSQL, Auth, Row Level Security).
- **Autenticação:** Supabase Auth (e-mail/senha).
- **Estilização:** Tailwind CSS.
- **Componentes de UI:** shadcn/ui — componentes acessíveis (baseados em Radix UI + Tailwind) copiados para dentro do próprio projeto e livremente customizáveis, em vez de uma biblioteca instalada como dependência fechada.
- **Acesso a dados:** `@supabase/supabase-js` chamado a partir de uma **camada de serviços** (funções por entidade, ex.: `listarFuncionarios`) e consumido em **hooks customizados** com `useState`/`useEffect`. Sem biblioteca extra de estado servidor.
- **Criação de usuários:** feita pela própria aplicação. Terão dois tipos de usuários: admins e gerentes. Os usuários do tipo admin poderão ser criados apenas diretamente no Supabase, enquanto que usuários do tipo gerentes poderão ser criados por um admin logado na aplicação.
- **Roteamento:** React Router.

## 4. Papéis de usuário

| Papel        | Descrição                                                                 |
|--------------|---------------------------------------------------------------------------|
| **Admin**    | Gerencia setores, itens de checklist, funcionários e gerentes. Vê todos os setores e todos os relatórios. Faz correções retroativas sem restrição de data. |
| **Gerente**  | Avalia os funcionários **apenas dos setores atribuídos a ele** e pode **cadastrar/inativar funcionários** dos seus setores. Vê a tela de lançamento, a visão mensal e os relatórios dos seus setores. Edita avaliações do **mês corrente** e do mês anterior **até o dia 5** do mês seguinte. |

Um gerente pode ser responsável por **mais de um setor**, e um setor pode ter **mais de um gerente** (relação N:N).

## 5. Modelo de domínio

Entidades principais:

- **Setor** — unidade organizacional (ex.: Cozinha, Estoque, Atendimento).
- **Item de checklist** — pertence a um setor; é uma exigência avaliada (ex.: "Uniforme completo", "Bancada limpa").
- **Funcionário** — pertence a um setor.
- **Usuário / Perfil** — conta de acesso (admin ou gerente).
- **Vínculo Gerente–Setor** — define quais setores cada gerente avalia.
- **Avaliação de item** — o registro central: um funcionário, em um dia, para um item específico do checklist, recebe um **nível** e (opcionalmente) uma anotação.
- **Expediente do setor** — marcação por setor e dia indicando se houve expediente. Por padrão considera-se que **houve** expediente; dias marcados como **sem expediente** (feriado, folga coletiva, setor fechado) bloqueiam a avaliação daquele dia e são **excluídos** dos percentuais e relatórios.

### Regra da avaliação item a item

A unidade de registro é o cruzamento **(funcionário, data, item de checklist)**. Cada um desses registros guarda:

- **Nível** — `nao_conforme`, `parcialmente_conforme` ou `totalmente_conforme`, com **pontuação associada 0, 1 e 2** respectivamente (usada para percentuais, evolução e rankings).
- **Observação** — texto livre opcional (só por item; não há anotação geral do dia).
- **Snapshot da descrição do item** — cópia do texto do item no momento da avaliação, para o histórico não mudar se o item for editado depois.

Um item, num dia, para um funcionário, ou **não tem registro** (ainda não avaliado) ou tem um dos três níveis. Não existe um campo separado que guarde o "resultado do dia" do funcionário: o sistema grava apenas o nível de cada item. Quando for preciso mostrar um resumo do dia, ele é **calculado na hora** a partir dos itens daquele dia como um **percentual de conformidade**, usando a pontuação dos níveis: `% = (soma dos pontos) ÷ (2 × número de itens) × 100`. Isso mantém uma única fonte da verdade (os itens) e evita que um "status do dia" armazenado fique divergente dos itens após uma edição.

Exemplo: se no dia 24 o funcionário tem *Uniforme = totalmente* (2), *Bancada = parcialmente* (1) e *Higienização = não conforme* (0), a soma é 3 de um máximo de 6 → **50% de conformidade** naquele dia. Não gravamos "dia 24 = X"; o percentual é derivado desses itens.

## 6. Modelo de dados (Supabase / PostgreSQL)

Esboço de tabelas. Ajustar tipos/constraints durante a implementação.

```sql
-- Perfis de usuário (espelham auth.users via trigger ou criação manual)
create table profiles (
  id          uuid primary key references auth.users(id) on delete cascade,
  nome        text not null,
  role        text not null default 'gerente' check (role in ('admin','gerente')),
  ativo       boolean not null default true,
  created_at  timestamptz not null default now()
);

-- Setores
create table setores (
  id          uuid primary key default gen_random_uuid(),
  nome        text not null,
  ativo       boolean not null default true,
  created_at  timestamptz not null default now()
);

-- Itens de checklist (pertencem a um setor)
create table checklist_itens (
  id          uuid primary key default gen_random_uuid(),
  setor_id    uuid not null references setores(id) on delete cascade,
  descricao   text not null,
  ordem       int not null default 0,
  ativo       boolean not null default true,
  created_at  timestamptz not null default now()
);

-- Funcionários (pertencem a um setor)
create table funcionarios (
  id          uuid primary key default gen_random_uuid(),
  setor_id    uuid not null references setores(id) on delete restrict,
  nome        text not null,
  ativo       boolean not null default true,
  created_at  timestamptz not null default now()
);

-- Vínculo N:N entre gerentes e setores
create table gerente_setores (
  gerente_id  uuid not null references profiles(id) on delete cascade,
  setor_id    uuid not null references setores(id) on delete cascade,
  primary key (gerente_id, setor_id)
);

-- Avaliação item a item: um registro por (funcionário, dia, item)
create table avaliacao_itens (
  id                uuid primary key default gen_random_uuid(),
  funcionario_id    uuid not null references funcionarios(id) on delete cascade,
  data              date not null,
  checklist_item_id uuid not null references checklist_itens(id) on delete restrict,
  descricao_item    text not null,  -- snapshot do texto do item na hora da avaliação
  nivel             text not null check (nivel in ('nao_conforme','parcialmente_conforme','totalmente_conforme')),
  observacao        text,
  avaliado_por      uuid references profiles(id),
  created_at        timestamptz not null default now(),
  updated_at        timestamptz not null default now(),
  unique (funcionario_id, data, checklist_item_id)
);

-- Expediente por setor e dia (só as exceções são gravadas; ausência = houve expediente)
create table expediente_setor (
  id           uuid primary key default gen_random_uuid(),
  setor_id     uuid not null references setores(id) on delete cascade,
  data         date not null,
  houve_expediente boolean not null default true,
  marcado_por  uuid references profiles(id),
  created_at   timestamptz not null default now(),
  updated_at   timestamptz not null default now(),
  unique (setor_id, data)
);
```

### Observações de modelagem

- O **snapshot** (`descricao_item`) é preenchido no momento do registro copiando `checklist_itens.descricao`. Assim, relatórios históricos mostram o texto que valia na época, mesmo que o item seja editado depois.
- Manter `checklist_itens.ativo` (soft delete) para não quebrar avaliações que referenciam itens antigos; itens inativos não aparecem para novos lançamentos.
- **Pontuação dos níveis:** `nao_conforme` = 0, `parcialmente_conforme` = 1, `totalmente_conforme` = 2. O mapeamento é aplicado nas consultas (via `CASE` ou tabela de referência); o percentual de conformidade é `soma dos pontos ÷ (2 × nº de itens) × 100` (ver seção 5).
- A **visão mensal** e os relatórios são calculados **no banco**: *views* SQL para resumos fixos e funções **RPC** (parametrizadas por setor/mês/período) para relatórios com filtros. Assim o banco devolve o resultado pronto, sem trafegar linhas brutas.
- Índices sugeridos: `(funcionario_id, data)` e `(checklist_item_id, data)` para acelerar tela de lançamento e relatórios.

## 7. Segurança (Row Level Security)

Habilitar **RLS** em todas as tabelas. Regras gerais:

- **Admin** (`profiles.role = 'admin'`): acesso total a tudo, sem restrição de data.
- **Gerente**: só enxerga/edita dados dos setores em que está vinculado via `gerente_setores`.
  - Leitura de `setores`, `funcionarios`, `checklist_itens`: apenas dos seus setores.
  - Escrita em `funcionarios`: pode **criar e inativar** funcionários dos seus setores (o admin também pode, em qualquer setor).
  - Escrita em `avaliacao_itens`: apenas para funcionários dos seus setores **e** dentro da **janela de edição** — o mês corrente, ou o mês anterior enquanto o dia de hoje for **até o dia 5**. Fora dessa janela, só o admin edita.
  - Escrita em `expediente_setor`: pode marcar/desmarcar o expediente dos seus setores, dentro da mesma janela de edição das avaliações (admin sem restrição).
- Somente admin gerencia `setores`, `checklist_itens`, `profiles` e `gerente_setores`. Os cadastros de `funcionarios` e `expediente_setor` são compartilhados (admin em qualquer setor; gerente nos seus).

Sugestão: criar funções auxiliares `is_admin()` e `gerencia_setor(setor_id)` em SQL para reutilizar nas policies, e uma função `pode_editar_avaliacao(data)` que implementa a janela de edição (mês corrente + carência até o dia 5 do mês seguinte).

## 8. Funcionalidades e telas

### 8.1. Autenticação
- Login (e-mail/senha).
- Logout.
- (Opcional) recuperação de senha via Supabase.

### 8.2. Tela de lançamento (tela principal do gerente) — foco no dia
- Seletor de **setor** (limitado aos setores do usuário) e de **data**.
- Lista os **funcionários** do setor; para cada funcionário, exibe os **itens do checklist** do setor, cada um com um seletor dos três níveis (Não conforme / Parcialmente conforme / Totalmente conforme) e um campo de **observação** opcional.
- Permite lançar/atualizar o dia inteiro do setor de uma vez; salvar grava/atualiza os registros em `avaliacao_itens` (incluindo o snapshot do texto do item).
- Indicação visual de itens ainda não avaliados no dia.
- Permite marcar o dia como **sem expediente** (feriado/folga/setor fechado). Nesse caso, os itens ficam **bloqueados** para avaliação naquele dia e o dia é **excluído** dos percentuais e relatórios; se já houver avaliações lançadas, avisar antes. A marcação pode ser revertida.
- Bloqueio de edição para datas fora da janela do gerente (mês corrente + mês anterior até o dia 5); admin não é bloqueado.

### 8.3. Administração (admin)
- CRUD de **setores**.
- CRUD de **itens de checklist** por setor (com ordenação e soft delete).
- CRUD de **funcionários** (com setor). Também disponível ao **gerente**, restrito aos seus setores (ver 8.2 e seção 7).
- CRUD de **gerentes** e atribuição de setores (o admin cria os gerentes pela própria aplicação). Contas de admin são criadas diretamente no Supabase.

### 8.4. Visão mensal consolidada
- Escolhe setor + mês; mostra o mês inteiro resumido por funcionário: total de itens em cada nível e o **percentual de conformidade** (via pontuação 0/1/2, ver seção 5).
- Serve de panorama e ponto de entrada para os relatórios.

### 8.5. Relatórios
- Desempenho por **funcionário** em um período (pontuação/percentual de conformidade e não conformidades).
- **Itens de checklist mais reincidentes** (por setor / período).
- **Evolução mensal por setor** (conformes vs. parciais vs. não conformes ao longo dos meses).
- **Ranking de funcionários** por conformidade (por setor / período).

Cada relatório é **exibido na tela** e oferece um botão para **baixá-lo em PDF** (gerado a partir da versão apresentada).

## 9. Fluxos principais

**Configuração inicial (admin):**
1. Cria os setores.
2. Cadastra os itens de checklist de cada setor.
3. Cadastra os funcionários e associa a um setor.
4. Cria as contas de gerentes pela aplicação e vincula aos setores.

**Uso diário (gerente):**
1. Faz login e seleciona o setor + a data.
2. Vê os funcionários do setor com os itens do checklist.
3. Para cada funcionário, define o nível de cada item e, se necessário, escreve uma observação.
4. Consulta a visão mensal e os relatórios quando necessário.

## 10. Estrutura de pastas sugerida (frontend)

```
src/
  lib/
    supabaseClient.ts      # inicialização do cliente Supabase
  types/
    database.ts            # tipos gerados do schema (supabase gen types)
  hooks/                   # hooks de dados com useState/useEffect (sem react-query)
  components/
    ui/                    # componentes do shadcn/ui (copiados para o projeto)
                           # + componentes reutilizáveis próprios (ex.: seletor de nível)
  features/
    auth/
    lancamento/            # tela de lançamento (foco no dia): api.ts + hooks + componentes
    mensal/                # visão mensal consolidada
    admin/                 # setores, itens, funcionários, gerentes
    relatorios/
                           # cada feature tem seu api.ts (funções que usam o supabase-js)
  routes/                  # definição de rotas e guards por papel
  App.tsx
  main.tsx
```

## 11. Decisões já tomadas

**Stack**
- **Frontend:** React + TypeScript + Vite; **roteamento** com React Router; **estilo** com Tailwind CSS; **componentes de UI** com shadcn/ui (Radix UI + Tailwind).
- **Backend/dados:** Supabase (PostgreSQL + Auth + RLS); acesso via supabase-js + camada de serviços + hooks (`useState`/`useEffect`), sem lib de estado servidor.
- **Autenticação:** apenas e-mail/senha.
- **Organização do código:** por **feature** (`features/lancamento`, `features/mensal`, `features/admin`, `features/relatorios`) + pastas comuns.

**Usuários e acesso**
- **Papéis:** apenas **admin** e **gerente**.
- **Criação de usuários:** admins criados diretamente no Supabase; gerentes criados por um admin **pela própria aplicação**.
- **Gerente ↔ setor:** relação **N:N**; gerente só acessa seus setores.
- **Cadastro de funcionários:** admin (qualquer setor) **e** gerente (seus setores) podem criar/inativar.
- **Segurança:** **RLS ativado em todas as tabelas** (regras no banco), com funções SQL auxiliares.
- **Edição retroativa:** gerente edita o **mês corrente** e o mês anterior **até o dia 5** do mês seguinte; depois disso, só o admin corrige.

**Modelo e avaliação**
- **Granularidade:** avaliação **item a item por dia** (unidade = funcionário × data × item), não um status único por dia.
- **Níveis e pontuação:** Não conforme = 0, Parcialmente conforme = 1, Totalmente conforme = 2.
- **Observação:** livre e opcional, **só por item** (sem anotação geral do dia).
- **Histórico do item:** guardar **snapshot** do texto do item na avaliação.
- **Item ↔ setor:** cada item pertence a um único setor (duplicar quando o mesmo item vale para vários setores).
- **Soft delete:** setores, itens e funcionários são inativados, nunca apagados.
- **Dias avaliados:** todos os dias do mês (incluindo fins de semana).
- **Cálculo de resumos/relatórios:** no **banco**, via *views* SQL + funções **RPC**.
- **Expediente do setor:** gerente (nos seus setores, mesma janela de edição) e admin podem marcar um dia como **sem expediente**; isso bloqueia a avaliação e exclui o dia dos cálculos. Padrão: com expediente.

**Interface**
- **Tela principal:** **foco no dia** (setor + data → funcionários → itens).
- **Visão mensal consolidada:** tela separada do lançamento, com percentual de conformidade.
- **Resumo do dia:** exibido como **percentual de conformidade** (fórmula na seção 5).
- **Relatórios:** por funcionário, itens reincidentes, evolução mensal por setor e ranking de funcionários.
- **Exportação:** relatórios exibidos na tela e baixáveis em **PDF**.

## 12. Decisões em aberto / a confirmar

- **Funcionário que troca de setor no meio do mês:** adiado; tratar como evolução futura (por ora, um funcionário pertence a um único setor).
