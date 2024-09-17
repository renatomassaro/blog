---
title: "Otimizando a estrutura de tabelas no Postgres para máxima eficiência"
date: 2024-09-15T19:04:35-03:00
categories: [databases]
tags: [Postgres, SQLite, MySQL]
image: cover.webp
---

## Introdução

Quando se modela um banco de dados no Postgres, você provavelmente não presta muita atenção à ordem das colunas nas suas tabelas. Afinal, parece o tipo de coisa que não afetaria o armazenamento ou desempenho. Mas e se eu dissesse que simplesmente reordernar suas colunas poderia reduzir o tamanho das suas tabelas e índices em 20%? Isso não é um truque obscuro de banco de dados — é um resultado direto de como o Postgres alinha os dados no disco.

Neste post, vou explorar como funciona o alinhamento de colunas no Postgres, por que isso importa e como você pode otimizar suas tabelas para uma melhor eficiência. Com alguns exemplos práticos, você verá como até mesmo pequenas mudanças na ordem das colunas podem gerar melhorias mensuráveis.

## Pesando uma tupla

> Aqui, tupla é o mesmo que "linha" ou "registro", ou seja, cada entrada de uma tabela. Equivalente a "row" em inglês.

Devido ao [layout de tupla](https://www.postgresql.org/docs/current/storage-page-layout.html#STORAGE-TUPLE-LAYOUT), o menor tamanho possível para uma tupla é 24 bytes.

```sql
SELECT pg_column_size(ROW());
 pg_column_size
----------------
             24
```

Depois disso, para cada nova coluna que você adicionar na tupla, mais espaço será necessário:

```sql
-- Uma coluna de tipo inteiro: 24 + 4 = 28 bytes
SELECT pg_column_size(ROW(1::int));
 pg_column_size
----------------
             28

-- Inteiro + smallint: 24 + 4 + 2 = 30 bytes
SELECT pg_column_size(ROW(1::int, 1::smallint));
 pg_column_size
----------------
             30
```

Até aqui, tudo bem. Isso é exatamente o que você esperaria: quanto mais dados você tiver na sua tupla, mais espaço em disco ela utilizará. Além disso, nós esperamos que o uso em disco será diretamente relacionado aos tipos de dados presentes na tupla.

Em outras palavras, se você tem uma coluna `integer`, nós esperamos um tamanho de 24 + 4 = 28 bytes. Se nós temos uma coluna `integer` e uma `smallint`, nós esperamos um tamanho de 24 + 4 + 2 = 30 bytes.

Como podemos, então, explicar o seguinte resultado?

```sql
SELECT pg_column_size(ROW(1::smallint, 1::int));
 pg_column_size
----------------
             32
```

Como assim?! Nós _acabamos_ de ver que uma tupla `(integer, smallint)` ocupa 30 bytes em disco, mas uma tupla `(smallint, integer)` ocupa 32 bytes! Isso acontece com outros tipos de dados, também:

```sql
-- (bigint, boolean) = 24 + 8 + 1 = 33 bytes
SELECT pg_column_size(ROW(1::bigint, true::boolean));
 pg_column_size
----------------
             33

-- (boolean, bigint) = 24 + ? + 8 = 40 bytes!?!?
SELECT pg_column_size(ROW(true::boolean, 1::bigint));
 pg_column_size
----------------
             40
```

Tuplas com formato `(boolean, bigint)` usam 21% mais espaço em disco que tuplas `(bigint, boolean)`!

O que está acontecendo aqui?

## Alinhamento de dados

A resposta é [**alinhamento de dados** (data alignment)](https://en.wikipedia.org/wiki/Data_structure_alignment).

O Postgres vai adicionar "padding" (zeros) nos dados, de forma que eles estejam devidamente alinhados na camada física. Esse alinhamento garante um tempo de acesso mais rápido, ao ler páginas do disco.

> Na realidade, isso se trata de um [space-time tradeoff](https://en.wikipedia.org/wiki/Space%E2%80%93time_tradeoff): nós estamos adicionando dados que são desnecessários, mas que garantirão um acesso mais rápido.

### Representação visual

Vamos tentar visualizar como esses dados seriam armazenados em disco. Abaixo temos uma tupla `(integer, smallint)` devidamente alinhada:

![(integer, smallint) = 30 bytes por tupla](example_1.png)

Contraste-a com a tupla desalinhada `(smallint, integer)`:

![(smallint, integer) = 32 bytes por tupla](example_2.png)

Perceba como o Postgres teve que adicionar zeros à coluna `smallint` para garantir o alinhamento necessário de 4 bytes.

Abaixo temos outro exemplo, agora com `(bigint, smallint, boolean)` resultando numa tupla de 35 bytes.

![(bigint, smallint, boolean) = 35 bytes por tupla](example_5.png)

E a mesma tupla, numa ordem menos eficiente, resultando em 40 bytes por tupla

![(boolean, smallint, bigint) = 40 bytes por tupla](example_4.png)

### Calculando os limites de alinhamento

O que determina o alinhamento usado pelo Postgres? A partir da [documentação](https://www.postgresql.org/docs/current/catalog-pg-type.html):

{{<quote>}}
typalign é o alinhamento necessário ao armazenar um valor desse tipo. Isso se aplica ao armazenamento em disco, bem como à maioria das representações do valor dentro do PostgreSQL. Quando vários valores são armazenados consecutivamente, como na representação de uma linha completa em disco, o padding é inserido antes de um dado desse tipo para que ele comece no limite especificado. O ponto de referência para o alinhamento é o início do primeiro dado na sequência. Os valores possíveis são:

- c = alinhamento char, ou seja, nenhum alinhamento necessário.
- s = alinhamento short (2 bytes na maioria das máquinas).
- i = alinhamento int (4 bytes na maioria das máquinas).
- d = alinhamento double (8 bytes em muitas máquinas, mas não em todas).
{{< /quote >}}

Podemos confirmar isso consultando diretamente a `pg_type`:

```sql
SELECT typname, typalign, typlen
FROM pg_type
WHERE typname IN ('int4', 'int2', 'int8', 'bool', 'varchar', 'text', 'float4', 'float8', 'uuid', 'date', 'timestamp');

  typname  | typalign | typlen
-----------+----------+--------
 bool      | c        |      1
 int8      | d        |      8
 int2      | s        |      2
 int4      | i        |      4
 text      | i        |     -1
 float4    | i        |      4
 float8    | d        |      8
 varchar   | i        |     -1
 date      | i        |      4
 timestamp | d        |      8
 uuid      | c        |     16
```
Você pode ver, por exemplo, que `int8` requer um alinhamento de `d` (double, ou 8 bytes), enquanto `int4` precisa de metade do espaço.

`varchar` e `text` funcionam de maneira diferente. Embora tenham um alinhamento de `i`, seu `typlen` é negativo. Por quê? Porque eles têm tamanho variável, ou seja, usam uma estrutura `varlena`.

O fato de esses dois campos terem comprimento variável não é realmente relevante para o alinhamento, exceto pelo fato de que tal coluna variável será alinhada em limites de 4 bytes (a menos que o valor esteja TOASTed, como veremos abaixo).

Também vale a pena mencionar que o tipo `uuid` é diferente. Ele tem um `typlen` de 16 bytes, mas possui alinhamento `c` (o que significa que não precisa de alinhamento prévio). Assim, você não precisa se preocupar se tiver uma coluna `boolean` logo antes de um `uuid`, por exemplo.

No código do Postgres, você encontrará esse valor sendo usado na macro `MAXALIGN` para determinar o alinhamento necessário.

```c
define TYPEALIGN(ALIGNVAL,LEN)  \
	(((uintptr_t) (LEN) + ((ALIGNVAL) - 1)) & ~((uintptr_t) ((ALIGNVAL) - 1)))

#define MAXALIGN(LEN)			TYPEALIGN(MAXIMUM_ALIGNOF, (LEN))
```

> NOTA: a documentação afirma que **o padding é inserido antes [...] desse tipo**. Isso significa que, se tivermos uma tupla `(char, int4, char, int8)`, teremos 3 bytes de padding antes de `int4` e 7 bytes de padding antes de `int8`.

### Alinhamento se aplica aos índices também!

O que muitas vezes é esquecido é que o alinhamento de dados não afeta apenas as tuplas das suas tabelas — ele também se aplica aos índices. Isso pode ser surpreendente porque geralmente pensamos nos índices como estruturas compactas e otimizadas que existem puramente para acelerar as consultas. No entanto, o Postgres garante que os dados nos índices sigam as mesmas regras de alinhamento que as linhas da tabela, o que significa que colunas desalinhadas podem inflar o tamanho dos seus índices da mesma forma que fazem com as tabelas.

Foi assim que eu descobri a importância do alinhamento de dados no Postgres. Eu tinha um índice `(int8, int8)`, que reestruturei para `(int4, int8)` para resultar em um tamanho menor de tabela/índice. Imagine minha surpresa ao fazer o benchmark e perceber que o tamanho do índice não mudou!

Este é um ponto crítico a entender: **colunas desalinhadas afetam seus índices também**, potencialmente aumentando tanto o uso de disco quanto o consumo de memória. Manter os índices alinhados pode, portanto, ter um impacto significativo no desempenho e na eficiência dos recursos do banco de dados.

### Como isso funciona na prática

```sql
-- Vamos criar a primeira tabela, que tem a estrutura (int4, int8, int4)
CREATE TABLE table_1 (int4_1 INTEGER, int8_2 BIGINT, int4_3 INTEGER);
CREATE INDEX int_4_int_8_int_4 ON table_1 (int4_1, int8_2, int4_3);

-- A segunda tabela tem um alinhamento melhor: (int4, int4, int8)
CREATE TABLE table_2 (int4_1 INTEGER, int4_2 INTEGER, int8_3 BIGINT);
CREATE INDEX int_4_int_4_int_8 ON table_2 (int4_1, int4_2, int8_3);

-- Adicionamos 10_000_000 tuplas em cada tabela
INSERT INTO table_1 (int4_1, int8_2, int4_3) SELECT gs AS int4_1, gs AS int8_2, gs AS int4_3 FROM GENERATE_SERIES(1, 10000000) AS gs;
INSERT INTO table_2 (int4_1, int4_2, int8_3) SELECT gs AS int4_1, gs AS int4_2, gs AS int8_3 FROM GENERATE_SERIES(1, 10000000) AS gs;

-- A primeira tabela usa mais espaço em disco do que a segunda
SELECT
  pg_size_pretty(pg_relation_size('table_1')) AS table_1_size,
  pg_size_pretty(pg_relation_size('table_2')) AS table_2_size;

 table_1_size | table_2_size
--------------+--------------
 498 MB       | 422 MB

-- O primeiro índice usa mais disco (e memória) que o segundo
SELECT
  indexrelid::regclass AS index_name,
  pg_size_pretty(pg_relation_size(indexrelid::regclass)) AS index_size
FROM
  pg_stat_user_indexes;

    index_name     | index_size
-------------------+------------
 int_4_int_8_int_4 | 386 MB
 int_4_int_4_int_8 | 300 MB
```

Para uma única tabela, com apenas um dado desalinhado, a tabela devidamente alinhada usa 76 megabytes a menos (uma melhoria de 15%) e o índice devidamente alinhado usa 86 megabytes a menos (uma melhoria de 22%).

Essas tabelas experimentais têm 10 milhões de entradas, o que é um número pequeno para bancos de dados reais. Imagine o nível de melhoria que se pode encontrar nesses bancos de dados!

{{<quote>}}
NOTA: É importante mencionar que os benefícios de uma tabela devidamente alinhada não são exclusivos do armazenamento. Isso provavelmente levará a ganhos de desempenho também.

É intuitivamente fácil entender por que isso acontece. Com um melhor alinhamento de dados, suas tuplas serão menores. Com tuplas menores, você encaixará mais tuplas por página. Com páginas mais compactas:

1. Potencialmente, menos páginas precisarão ser recuperadas do disco.
2. O Postgres pode encaixar mais tuplas na memória (ou seja, mais tuplas para o mesmo número de páginas em cache).
{{</quote>}}

Não tenho tempo para medir o quanto o desempenho melhoraria, mas acredito que os ganhos de desempenho potenciais não seriam negligenciáveis.

### Quando começar a se preocupar

Amigos não deixam amigos criarem tabelas desalinhadas. Mas você não quer necessariamente se tornar a polícia do alinhamento nas revisões de PR da sua empresa!

![Você tem o direito de permanecer alinhado](data_alignment_police.webp)

Startups, especialmente as de estágio inicial, muitas vezes têm um processo de desenvolvimento frenético, e economizar 10-20% de espaço em disco geralmente não é uma prioridade. Muitas vezes, essas economias são absolutamente irrelevantes nesse estágio. Então, quando sua equipe deve se preocupar com o alinhamento das tabelas?

A resposta certa depende muito do contexto por trás do seu produto, mas eventualmente você notará custos dramáticos relacionados a infraestrutura (especificamente aos seus bancos de dados), e talvez esse seja um bom ponto de partida.

No entanto, acredito que um engenheiro de software competente deve sempre ter esse fato em mente e, sempre que possível, fazer ajustes enquanto desenvolve funcionalidades. Não há necessidade de se preocupar demais com isso, mas tente manter suas tabelas alinhadas desde o início.

Isso economizará significativamente tempo no futuro. Refatorar seu modelo de dados não é fácil. Não se trata apenas de reorganizar as colunas da tabela, mas também de reorganizar seus índices (o que também significa reordenar seu padrão de consulta na camada de aplicação).

Na realidade, para startups em estágio inicial, **minha recomendação é sempre ficar de olho em quão bem alinhados estão seus _índices_**. Você pode reordenar as colunas da tabela mais tarde, mas preste muita atenção aos seus índices. Acho que essa é uma boa recomendação porque:

1. Como mencionado acima, os índices são difíceis de reordenar. Você precisaria garantir que o padrão de acesso seja reordenado também, pois uma consulta em `(a, b)` não usará um índice em `(b, a)`.
2. É fácil mudar a ordem dos índices durante o desenvolvimento, já que eles são adicionados individualmente e não dependem de migrações anteriores.
3. Mudar a ordem das colunas durante o desenvolvimento é difícil, pois elas geralmente são construídas com base em migrações anteriores.
4. Refatorar o modelo de dados para reordenar as colunas da tabela, embora não seja trivial, não precisa de mudanças na camada de aplicação (a menos que você esteja fazendo algo incomum), então pode ser adiado para um estágio posterior.

### Regra prática

Uma regra prática genérica que tenho usado nos últimos anos é, sempre que possível, definir colunas com base em uma ordem decrescente de tamanho dos tipos de dados.

Em outras palavras: comece com seus tipos de dados maiores (`int8`, `float8`, `timestamp`) e deixe os tipos de dados menores para o final. Isso naturalmente alinhará sua tabela.

Tenha em mente que essa regra é válida somente quando "todo o resto é igual". Outros fatores, como cardinalidade ou até mesmo legibilidade, podem ter maior prioridade que alinhamento de dados, especialmente em se tratando de índices.

### Uma nota sobre valores TOASTed

Alguns tipos de dados têm comprimento variável e seus valores podem ser armazenados em outro lugar se ficarem muito grande a ponto de uma tupla não caber em uma única página. Quando isso acontece, dizemos que o valor é TOASTed. Nesse cenário, a tupla conterá um ponteiro para os valores correspondentes.

Diferentes tipos de ponteiros podem existir dependendo do tamanho dos dados. Para ponteiros de um byte, nenhum alinhamento é necessário, enquanto um alinhamento de 4 bytes é aplicado a ponteiros de 4 bytes. Da [documentação](https://www.postgresql.org/docs/current/storage-toast.html):

{{<quote>}}
Valores com cabeçalhos de um byte não estão alinhados em nenhum limite particular, enquanto valores com cabeçalhos de quatro bytes estão alinhados em pelo menos um limite de quatro bytes; essa omissão de preenchimento de alinhamento proporciona economias de espaço adicionais que são significativas em comparação com valores curtos.
{{</quote>}}

## Isso se aplica a outros bancos de dados?

Precisamos considerar o alinhamento de dados ao trabalhar com bancos de dados além do Postgres? Aqui está o que descobri:

### SQLite

Não. Aqui está uma [citação do Richard Hipp](https://sqlite.org/forum/info/5e030d06c5b32ab5y) (tradução minha; veja post para original):

{{<quote source="drh" url="https://sqlite.org/forum/info/5e030d06c5b32ab5y">}}

O SQLite não preenche ou alinha colunas dentro de uma tupla. Tudo é compactado de forma justa, usando o mínimo de espaço. Duas consequências desse design:

1. O SQLite tem que trabalhar mais (usar mais ciclos de CPU) para acessar dados dentro de uma tupla, uma vez que a tupla esteja na memória.
2. O SQLite usa menos bytes em disco, menos memória e gasta menos tempo movendo conteúdo porque há menos bytes para mover.

Acreditamos que o benefício de #2 supera a desvantagem de #1, especialmente em hardware moderno, no qual #1 opera a partir do cache L1, enquanto #2 utiliza transferências de memória principal. Mas sua experiência pode variar dependendo do que você armazena no arquivo do banco de dados.

{{</quote>}}

Para aqueles que estão familiarizados com a arquitetura do SQLite, diria que isso não é surpreendente e está alinhado com a filosofia do SQLite.

### MySQL

O MySQL não é minha especialidade, então você vai querer se aprofundar mais para corroborar minhas descobertas. Estou apenas citando a documentação e adicionando alguns pensamentos meus, que não verifiquei. Você pode usar os links abaixo como ponto de partida para sua própria pesquisa.

#### NDB

Da [documentação](https://dev.mysql.com/doc/refman/8.4/en/storage-requirements.html#data-types-storage-reqs-ndb):

{{<quote>}}

Tabelas NDB usam alinhamento de 4 bytes; todo armazenamento de dados NDB é feito em múltiplos de 4 bytes. Assim, um valor de coluna que normalmente ocuparia 15 bytes requer 16 bytes em uma tabela NDB. Por exemplo, em tabelas NDB, os tipos de coluna TINYINT, SMALLINT, MEDIUMINT e INTEGER (INT) requerem cada um 4 bytes de armazenamento por registro devido ao fator de alinhamento.

{{</quote>}}

Com base no exposto, acredito que padding seria adicionado às colunas menores que 4 bytes.

#### InnoDB

Não consegui encontrar uma menção a alinhamento e preenchimento no [capítulo 17 da documentação do MySQL](https://dev.mysql.com/doc/refman/8.4/en/innodb-storage-engine.html), exceto talvez por este trecho:

> `[innodb_compression_failure_threshold_pct]` [define] a taxa de falha de compressão para uma tabela, como uma porcentagem, a partir da qual o MySQL começa a adicionar padding dentro das páginas comprimidas para evitar falhas de compressão caras.

Não é útil. Isso sugere padding no nível da página, mas queremos saber se as colunas são individualmente preenchidas. Bem, o MySQL é Oracle-open-source, então podemos olhar o código para tentar obter algumas dicas. Aqui está um trecho interessante de [storage/innobase/row/row0mysql.cc](https://github.com/mysql/mysql-server/blob/596f0d238489a9cf9f43ce1ff905984f58d227b6/storage/innobase/row/row0mysql.cc#L339):

```cpp
/** Pad a column with spaces.
@param[in] mbminlen Minimum size of a character, in bytes
@param[out] pad Padded buffer
@param[in] len Number of bytes to pad */
void row_mysql_pad_col(ulint mbminlen, byte *pad, ulint len) {
  const byte *pad_end;

  switch (UNIV_EXPECT(mbminlen, 1)) {
    default:
      ut_error;
    case 1:
      /* space=0x20 */
      memset(pad, 0x20, len);
      break;
    case 2:
      /* space=0x0020 */
      pad_end = pad + len;
      ut_a(!(len % 2));
      while (pad < pad_end) {
        *pad++ = 0x00;
        *pad++ = 0x20;
      };
      break;
    case 4:
      /* space=0x00000020 */
      pad_end = pad + len;
      ut_a(!(len % 4));
      while (pad < pad_end) {
        *pad++ = 0x00;
        *pad++ = 0x00;
        *pad++ = 0x00;
        *pad++ = 0x20;
      }
      break;
  }
}
```

Ok, claramente algo, em algum lugar, está preenchendo colunas no mecanismo de armazenamento InnoDB. No entanto, ao pesquisar na base de código do MySQL por usuários dessa função, tenho a impressão de que ela é usada apenas em alguns casos especiais (em vez de sempre verificar o alinhamento e preencher quando necessário, como o código do Postgres faz).

De qualquer forma, recomendo ao leitor que teste o snippet na seção ["Como funciona na prática"](#what-it-looks-like-in-practice) no MySQL e analise os resultados. Algo me diz que tabelas desalinhadas não serão tão problemáticas no MySQL quanto são no Postgres.