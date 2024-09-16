# IN OREBIT - BACKEND

O backend do projeto inOrbit foi feito em NodeJS. Para armazenar os dados foi utilizando um  banco de dados SQL chamado Postgres.

O banco de dados Postgres fica num container Docker que deve ser iniciado antes de subir a aplicação.

A API foi criada utilizado fastify e para consumir o banco de dados foi utilizado o drizzle.

Para criar o projeto, dentro da pasta onde ele vai ficar armazenado, utilize o comando:

```npm init -y```

## Banco de Dados

O banco de dados foi configurado num container Docker. Para subir o container com o banco de dados:

```docker-compose up -d```

O container fornecido durante o evento foi alterado. Foi feita a persistência local do banco de dados e foi acrescentado o pgAdmin, uma interface web para consulta ao banco de dados.

Se houver algum conflito na hora de subir o container (por exemplo, outro container rodando que esteja usando a mesma porta do banco), será necessário primeiro encerrar este container para subir o atual.

Para listar os containers em execução:

```docker container ls```

Para interromper a execução do container que estiver usando a porta, utilize o comando a seguir informando o id do container:

```docker stop <ID_DO_CONTAINER>```

No container foi dispobilizado o pgAdmin para administrar o banco de dados

![pgAdmin](pgadmin.jpg)

## Dependências que devem ser instaladas

Uma das primeiras dependências que deve ser instalada é o TypeScript.

```npm install typescript -D```

Depois executar:

```npx tsc --init```

Este comando vai criar um arquito tsc.init na raiz do projeto.

O comando ***npx*** é utilizado para rodar scripts que ficam dentro da pasta node_modules/bin de seu projeto.

Outras dependências:

```npm install @types/node -D```

Estas são as definições de tipo usadas pelo Typescript no NodeJS.

```npm install tsx -D```

Esta biblioteca permite executar o código Typescript sem a necessidade de converter para JavaScript. Depois de instalar, é necessário criar uma entrada "script" dentro do arquivo package.json localizado na raiz do projeto.

```
  "scripts": {
    "dev": "tsx watch --env-file .env src/http/server.ts",
  },
```

Outra biblioteca utilizada no projeto foi o fastify.

```npm install fastify```

O Fastify é um framework que vai ser utilizado para criar esta API.

Outra ferramenta que vai ser instalada é o Biome, que é uma ferramenta de Lint.

```npm i -D --save-exact @biomejs/biome```

Deve ser criado um arquivo na raiz do projeto chamado biome.json para configurar a forma como o lint deve ser feito. 


Para utilizar o banco na aplicação vamos instalar primeiro o Drizzle. 
https://orm.drizzle.team/

```npm install drizzle-orm```
```npm install drizzle-kit -D```

Na raiz do projeto deve ser criado um arquivo ***drizzle-config.ts*** para configurar onde vão ficar os schemas, migrations e endereço do banco SQL.

Antes de configurar o banco, criar um arquivo .env na raiz do projetro e cria a seguinte variável:

```DATABASE_URL="postgresql://docker:docker@localhost:5432/inorbit"```

Para trabalhar com o banco Postgress precisamos instalar o driver para acessar este banco

```npm install postgres```

Outra dependência que deve ser instalada é o Zod

```npm install zod```

Esta biblioteca é usada para validação de dados.

Outra dependência instalada:

```npm install @paralleldrive-cuid2```

Este é um algoritmo para geração de IDs únicos.

```npm install dayjs```

O DayJS é uma biblioteca utilizada para trabalhar com datas.

## Criando as tabelas no banco de dados

Depois de criado o schema, para gerar o script SQL que cria a tabela é necessário executar o seguinte comando:

```npx drizzle-kit generate```

Para gravar a tabela no banco:

```npx drizzle-kit migrate```

Para acessar o banco de dados para consultar sua estrutura e dados executamos o seguinte comando

```npx drizzle-kit studio```

Este comando vai gerar um link para uma interface web para acessar o banco. O endereço que ele vai informar é este:

https://local.drizzle.studio

![Drizzle](drizzle_studio.jpg)

Foi criado um script no arquivo package.json chamado seed para fazer a carga de dados inicial no banco de dados

```
  "scripts": {
    "seed": "tsx --env-file .env src/db/seed.ts"
  },
```

 Para executar: 

```npm run seed```


## Para executar a aplicação

Para executar o backend:

```npm run dev```

Rotas da aplicação:

**POST /goals**
**POST /completions**
**GET /pending-goals**
**GET /summary**

## Estrutura da aplicação

O codigo da aplicação fica dentro da pasta src. Seu conteúdo é o seguinte:

* db
* functions
* http

A pasta db contem a estrutura das tabelas (schema) e a carga de dados inicial (seeds)

Exemplo schema:

```
import { pgTable, text, integer, timestamp } from 'drizzle-orm/pg-core'
import { createId } from '@paralleldrive/cuid2'

export const goals = pgTable('goals', {
  id: text('id')
    .primaryKey()
    .$defaultFn(() => createId()),
  title: text('title').notNull(),
  desiredWeeklyFrequency: integer('desired_weekly_frequency').notNull(),
  createdAt: timestamp('created_at', { withTimezone: true })
    .notNull()
    .defaultNow(),
})
```

Exemplo seeds:

```
import { client, db } from '.'
import { goalCompletions, goals } from './schema'
import dayjs from 'dayjs'

async function seed() {
  await db.delete(goalCompletions)
  await db.delete(goals)

  const result = await db
    .insert(goals)
    .values([
      { title: 'Acordar cedo', desiredWeeklyFrequency: 5 },
      { title: 'Me exercitar', desiredWeeklyFrequency: 3 },
      { title: 'Meditar', desiredWeeklyFrequency: 1 },
    ])
    .returning()

  const startOfWeek = dayjs().startOf('week')

  await db.insert(goalCompletions).values([
    { goalId: result[0].id, createdAt: startOfWeek.toDate() },
    { goalId: result[1].id, createdAt: startOfWeek.add(1, 'day').toDate() },
  ])
}

seed().finally(() => {
  client.end()
})
```

A pasta functions possui as operações de inserção ou consulta que serão feitas no banco de dados. Todas as operações no banco de dados SQL serão feitas utilizando o ORM Drizzle.

https://orm.drizzle.team/docs/overview

Exemplo consulta:

```
import dayjs from 'dayjs'
import { db } from '../db'
import { goalCompletions, goals } from '../db/schema'
import { and, count, eq, gte, lte, sql } from 'drizzle-orm'

export async function getWeekPendingGoals() {
  const firstDayOfWeek = dayjs().startOf('week').toDate()
  const lastDayOfWeek = dayjs().endOf('week').toDate()

  (... outras consultas...)
  
  const pendingGoals = await db
    .with(goalsCreatedUpToWeek, goalCompletionCounts)
    .select({
      id: goalsCreatedUpToWeek.id,
      title: goalsCreatedUpToWeek.title,
      desiredWeeklyFrequency: goalsCreatedUpToWeek.desiredWeeklyFrequency,
      completionCount: sql /*sql*/`
        COALESCE(${goalCompletionCounts.completionCount}, 0)
      `.mapWith(Number),
    })
    .from(goalsCreatedUpToWeek)
    .leftJoin(
      goalCompletionCounts,
      eq(goalCompletionCounts.goalId, goalsCreatedUpToWeek.id)
    )

  return { pendingGoals }
}
```

Exemplo de inserção de dados no banco de dados

```
import { db } from '../db'
import { goals } from '../db/schema'

interface CreateGoalRequest {
  title: string
  desiredWeeklyFrequency: number
}

export async function createGoal({
  title,
  desiredWeeklyFrequency,
}: CreateGoalRequest) {
  const result = await db
    .insert(goals)
    .values({
      title,
      desiredWeeklyFrequency,
    })
    .returning()

  const goal = result[0]

  return {
    goal,
  }
}
```

Tanto para inserir ou consultas é utilizado o await.

Na pasta http ficam as rotas e o arquivo que faz subir a aplicação.

O arquivo com a lógica para subir a aplicação tem a seguinte estrutura (exemplo):

```
import fastify from 'fastify'
import {
  serializerCompiler,
  validatorCompiler,
  type ZodTypeProvider,
} from 'fastify-type-provider-zod'
import { createGoalRoute } from './routes/create-goal'
import fastifyCors from '@fastify/cors'

const app = fastify().withTypeProvider<ZodTypeProvider>()

app.register(fastifyCors, {
  origin: '*',
})

app.setValidatorCompiler(validatorCompiler)
app.setSerializerCompiler(serializerCompiler)

app.register(createGoalRoute)

app
  .listen({
    port: 3333,
  })
  .then(() => {
    console.log('HTTP server running!')
  })
```

Cada rota da aplicação foi ativada utilizando app.register(funcao_com_a_logica_da_rota)

A estrutura de um arquivo de rota é a seguinte:

```
import { z } from 'zod'
import type { FastifyPluginAsyncZod } from 'fastify-type-provider-zod'
import { createGoal } from '../../functions/create-goal'

export const createGoalRoute: FastifyPluginAsyncZod = async app => {
  app.post(
    '/goals',
    {
      schema: {
        body: z.object({
          title: z.string(),
          desiredWeeklyFrequency: z.number().int().min(1).max(7),
        }),
      },
    },
    async request => {
      const { title, desiredWeeklyFrequency } = request.body

      await createGoal({
        title,
        desiredWeeklyFrequency,
      })
    }
  )
}
```

Neste exemplo a rota recebe dados através do método **POST**. O primeiro parâmetro vai ser o endereço, que vai ser **"/goals"**. O segundo parâmetro é o schema dos dados de entrada. O terceiro parâmetro é a função para inserir dados no banco de dados.
