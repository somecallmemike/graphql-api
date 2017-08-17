# [MO] Kick start a JS GraphQL 3 layers API with Apollo-server, Dataloader and Knex in 1/2 hour (~30 min)

## Owner: [Thomas Pucci](https://github.com/tpucci)

## Prerequisites (~5min)

- Have **Yarn** installed (~1min)
- Have **Docker** and **Docker-compose** installed (~3min)
- Have **Postman** installed (~1min)

## Context

During this standard, we will create a Heroes graphQL api.
We will have a Hero model with superheroes real and hero names. We will add one example of association.

Our API will be lightly protected and use batching to minimise DB round-trips.

## Steps

### Intialise a new project

- Create and go to a new directory for the project: `mkdir graphql_formation && cd graphql_formation`
- Init a git repository: `git init`
- Create two services with Docker-compose, one postgres database and one node server:
  - For this step, notice that our final folder architecture looks like this:
  ```
  📂 graphql_formation
  ├ 📂 api
  │ └ 🗋 Dockerfile
  ├ 📂 db
  │ └ 🗋 Dockerfile
  ├ 🗋 config.env
  └ 🗋 docker-copose.yml
  ```
  - > Make sure your local 3000 port is available as we will use this port to reach our api
  - In a new `api/Dockerfile` file, write all the commands to assemble the api image:
  ```Dockerfile
  FROM node:8.1.0-alpine

  WORKDIR /usr/src/api

  EXPOSE 3000
  CMD ["yarn", "run", "serve"]
  ```
  - In a new `db/Dockerfile` file, write all the commands to assemble the db image:
  ```Dockerfile
  FROM postgres:9.6.3
  ```
  - In a new `docker-compose.yml` file, declare the two services:
  ```yml
  version: '3'
  services:
    api:
      build:
        context: ./api
      image: heroes-api
      env_file: config.env
      volumes:
        - ./api:/usr/src/api
      ports:
        - 3000:3000
      links:
        - db:db
    db:
      build:
        context: ./db
      env_file: config.env
      image: heroes-db
  ```
  - In a new `config.env` file, declare your environnement variable for these Docker containers:
  ```
  POSTGRES_USER=heroesuser
  POSTGRES_PASSWORD=heroespassword
  POSTGRES_DB=heroesdb
  PGDATA=/data
  ```
- Build these services with the command: `docker-compose build`

> **CHECK 1**: You terminal should prompt successively these lines confirming Docker images have been built:
>
> `Successfully tagged heroes-db:latest`
>
> `Successfully tagged heroes-api:latest`

### Install nodemon and run our project

- Add this to the project .gitignore: `echo "api/node_modules" > .gitignore`
- In the `api` folder, interactively create a `package.json` file: `cd api && yarn init`
- Add `nodemon`, `babel-cli`, `babel-plugin-transform-class-properties`, `babel-preset-flow` and `babel-preset-es2015` to our dev dependencies: `yarn add nodemon babel-cli babel-plugin-transform-class-properties babel-preset-es2015 babel-preset-flow -D`
- In our `package.json`, write the command to launch the server:
```json
"scripts": {
  "serve": "nodemon index.js --exec babel-node --presets=es2015,flow --plugins transform-class-properties"
}
```
- Create a new empty file `index.js`
- Go back to the root of the project: `cd ..`
- Run the project: `docker-compose up` 

> **CHECK 1**: You terminal should prompt the logs of the two containers together with two different colors
>
> **CHECK 2**: From another terminal, you can access the api and see the following folder structure: `docker-compose exec api /bin/sh` then inside the container: `ls -lath`;
> ```
> total 64
> drwxrwxr-x    3 node     node        4.0K Aug  9 09:00 .
> -rw-rw-r--    1 node     node         186 Aug  9 09:00 package.json
> -rw-rw-r--    1 node     node       42.1K Aug  9 09:00 yarn.lock
> -rw-rw-r--    1 node     node           0 Aug  7 20:05 index.js
> drwxrwxr-x  138 node     node        4.0K Aug  7 20:04 node_modules
> -rw-rw-r--    1 node     node          86 Aug  7 20:03 Dockerfile
> drwxr-xr-x    3 root     root        4.0K Aug  3 11:50 ..
> ```
> Exit with: `CTRL-D`
>
> **CHECK 3**: You can access the db and prompt the PostgreSQL version: `docker-compose exec db psql -U heroesuser -d heroesdb` then inside the container: `select version()`;
> ```bash
> PostgreSQL 9.6.3 on x86_64-pc-linux-gnu, compiled by gcc (Debian 4.9.2-10) 4.9.2, 64-bit
> ```
> Exit with: `CTRL-D`


### Create a koa server

- Install koa and koa-router in our api: `cd api & yarn add koa koa-router`
- In the `index.js` file, create our server:
```js
import Koa from 'koa';
import koaRouter from 'koa-router';

const app = new Koa();
const router = new koaRouter();

router.get('/', ctx => {
  ctx.body = 'Hello World!';
});

app.use(router.routes())
app.use(router.allowedMethods());
app.listen(3000);

console.log('Server is up and running');
```

> **CHECK 1**: In your terminal which run docker-compose, you should see `Server is up and running`
>
> **CHECK 2**: Hitting `localhost:3000` should return `Hello World!`: `curl localhost:3000`

### Create a presentation layer with graphQL

> This layer will let our api know how to present data: what data one user can query ? How should he query this data (fields, root queries, sub queries...) ?

- Install Koa body parser and Koa graphiQL: `yarn add koa-bodyparser koa-graphiql`
- In the `index.js` file, let our api knows it should use Koa body parser and koa-graphiql:
```js
import koaBody from 'koa-bodyparser';
import graphiql from 'koa-graphiql';

...

router.get('/graphiql', graphiql(async (ctx) => ({
  url: '/api',
})));

...

app.use(koaBody());
```
- Install graphQL, graphQL Server Koa and graphQL tools: `yarn add graphql graphql-server-koa graphql-tools`
- In a new folder `presentation` add a new `schema.js` file describing a simple graphQL schema:
```js
import { makeExecutableSchema } from 'graphql-tools';

const typeDefs = [`
  type Hero {
    id: Int!
    firstName: String
    lastName: String
  }

  type Query {
    heroes: [Hero]
  }

  schema {
    query: Query
  }
`];

const resolvers = {
  Query: {
    heroes: () => ([
      {
        id: 1,
        firstName: 'Clark',
        lastName: 'Kent',
      },
      {
        id: 2,
        firstName: 'Bruce',
        lastName: 'Wayne',
      }
    ]),
  }
}

const schema = makeExecutableSchema({ typeDefs, resolvers });

export default schema;
```
- In the `index.js` file, add our `api` endpoint:
```js
import { graphqlKoa } from 'graphql-server-koa';
import schema from './presentation/schema';

...

router.post(
  '/api',
  graphqlKoa(async ctx => {
    return {
      schema: schema,
      context: {},
      debug: true,
    };
  })
);
```

> **CHECK 1**: In **Postman**, making a *POST* request to `localhost:3000/api` which content-type is *JSON(application/json)* with the following raw body:
> ```json
> {
>   "query": "{heroes { firstName lastName }}"
> }
> ```
> ...should return our two heroes, Clark and Bruce:
> ![](assets/presentation_layer.png)
>
> **CHECK 2**: Hitting `localhost:3000/graphiql` should return graphiql interface and show the Docs
>
> **CHECK 3**: Using graphiql interface with the following query:
> ```
> {
>   heroes {
>     firstName
>     lastName
>   }
> }
> ```
> ...should return our two heroes, Clark and Bruce:

### Create a business layer

> This layer will contain all business logic: access controll, scoping / whitelisting, batching and caching and computed properties. More explanations can be found [here, in the bam-api repo](https://github.com/bamlab/bam-api). In this MO, we will only cover access control logic and batching and caching.

- In a new `business` folder add a new `hero.js` file describing our class for this business object:
```js
const mockedHeroes = [
  {
    id: 1,
    firstName: 'Clark',
    lastName: 'Kent',
  },
  {
    id: 2,
    firstName: 'Bruce',
    lastName: 'Wayne',
  }
];

class Hero {
  id: number;
  firstName: string;
  lastName: string;

  constructor(data) {
    this.id = data.id;
    this.firstName = data.firstName;
    this.lastName = data.lastName;
  }

  static async load(ctx, args) {
    const data = mockedHeroes[args.id];
    if (!data) return null;

    return new Hero(data);
  }

  static async loadAll(ctx, args) {
    const data = mockedHeroes;

    return data.map(row => new Hero(row));
  }

}

export default Hero;
```
- In our previous `presentation/schema.js` file, modify our mocked resolvers to use our business layer:
```diff
+import Hero from '../business/hero';

...

  type Query {
    heroes: [Hero]
+    hero(id: Int!): Hero
  }

...

const resolvers = {
  Query: {
-    heroes: () => ([
-      {
-        firstName: 'Clark',
-        lastName: 'Kent',
-      },
-      {
-        firstName: 'Bruce',
-        lastName: 'Wayne',
-      }
-    ]),
+    heroes: async (_, args, ctx) => Hero.loadAll(ctx, args),
+    hero: async (_, args, ctx) => Hero.load(ctx, args),
  }
}
```

> **CHECK 1**: Using graphiql interface with the following query:
> ```
> {
>   heroes {
>     id
>     firstName
>     lastName
>   }
> }
> ```
> ...should return our two heroes, Clark and Bruce.
>
> **CHECK 2**: Using graphiql interface with the following query:
> ```
> {
>   hero(id:0) {
>     id
>     firstName
>     lastName
>   }
> }
> ```
> ...should return Clark Kent with its `id: 1`.
>
> **CHECK 3**: Using graphiql interface with the following query:
> ```
> {
>   hero(id:1) {
>     id
>     firstName
>     lastName
>   }
> }
> ```
> ...should return Bruce Wayne with its `id: 2`.

### Seed our database

- Install `knex` and `pg`: `yarn add knex pg`
- At the root of our project, add a `knexfile.js` file:
```js
module.exports = {
  development: {
    client: 'pg',
    connection: {
      host: 'localhost',
      user: 'heroesuser',
      password: 'heroespassword',
      database: 'heroesdb',
    },
    migrations: {
      directory: './api/db/migrations',
    },
    seeds: {
      directory: './api/db/seeds',
    },
  },
};
```
- Create a migration file: `yarn knex migrate:make add_heroes_table` and complete the new created file with this:
```js
exports.up = function(knex, Promise) {
  return knex.schema.createTableIfNotExists('Heroes', function(table) {
    table.increments('id');
    table.string('firstName');
    table.string('lastName');
    table.string('heroName');
  });
};

exports.down = function(knex, Promise) {
  return knex.schema.dropTableIfExists('Heroes');
};
```
- Create a seed file: `yarn knex seed:make heroes` and complete the new created file with this:
```js
exports.seed = function(knex, Promise) {
  return knex('Heroes').del()
    .then(function () {
      return knex('Heroes').insert([
        {id: 1, firstName: 'Clark', lastName: 'Kent'},
        {id: 2, firstName: 'Bruce', lastName: 'Wayne'},
        {id: 3, firstName: 'Peter', lastName: 'Parker'},
        {id: 4, firstName: 'Susan', lastName: 'Storm-Richards'},
      ]);
    });
};
```
- Run the migration and the seed: `yarn knex migrate:latest && yarn knex seed:run`

> **CHECK 1**: You can access the db and prompt content of the `Heroes` table: `docker-compose exec db psql -U heroesuser -d heroesdb` then inside the container: `select * from "Heroes";`;
> ```bash
>  id | firstName |    lastName    |    heroName     
> ----+-----------+----------------+-----------------
>   1 | Clark     | Kent           | Superman        
>   2 | Bruce     | Wayne          | Batman          
>   3 | Peter     | Parker         | Spiderman       
>   4 | Susan     | Storm-Richards | Invisible Woman 
> (4 rows)
> ```
> Exit with: `CTRL-D`

### Create a db layer with knex

> This layer let our api query the data using knex query builder.

- Install `knex` and `pg`: `yarn add knex pg`
- In a new `db` folder add a new `index.js` file:
```js
import knex from 'knex';

export default knex({
  client: 'pg',
  connection: {
    host: process.env.DB_HOST,
    user: process.env.POSTGRES_USER,
    password: process.env.POSTGRES_PASSWORD,
    database: process.env.POSTGRES_DB,
  },
  debug: true,
});
```
- In a new `db/queryBuilders` subfolder, create a new `hero.js` file and add these few methods to query our data:
```js
// @flow
import db from '..';

class Hero {
  static async getById(id: number) {
    return db
    .first()
    .table('Heroes')
    .where('id', id);
  }

  static async getByIds(ids: Array<number>): Promise<Array<CostDBType>> {
    return db
    .select()
    .table('Heroes')
    .whereIn('id', ids);
  }

  static async getAll(): Promise<Array<CostDBType>> {
    return db
    .select()
    .table('Heroes');
  }
}

export default Hero;
```
- Modify the `hero.js` file in our business layer this way:
```diff
-const heroes = [
-  {
-    id: 0,
-    firstName: 'Clark',
-    lastName: 'Kent',
-  },
-  {
-    id: 1,
-    firstName: 'Bruce',
-    lastName: 'Wayne',
-  }
-];
+import HeroDB from '../db/queryBuilders/hero';
 
 class Hero {

   ...
 
   static async load(ctx, args) {
-    const data = heroes[args.id];
+    const data = await HeroDB.getById(args.id);
     if (!data) return null;
 
     return new Hero(data);
   }
 
   static async loadAll({ authToken, dataLoaders }) {
-    const data = heroes;
+    const data = await HeroDB.getAll();
 
     return data.map(row => new Hero(row));
   }
```

> **CHECK 1**: Using graphiql interface with the following query:
> ```
> {
>   hero(id:1) {
>     id
>     firstName
>     lastName
>   }
> }
> ```
> ...should return Clark Kent with its `id: 1`.
>
> **CHECK 2**: Using graphiql interface with the following query:
> ```
> {
>   heroes {
>     id
>     firstName
>     lastName
>   }
> }
> ```
> ...should return all 4 heroes of our database.

### Add association to our api

> Association are made both in our db and in our api, in our presentation layer.

- Create a new migration: `yarn knex migrate:make add_heroes_heroname_and_enemy`
- Complete the newly created migration file with this:
```js
exports.up = function(knex, Promise) {
  return knex.schema.table('Heroes', function(table) {
    table.string('heroName');
    table.integer('enemyId').references('id').inTable('Heroes');
  });
};

exports.down = function(knex, Promise) {
  return knex.schema.table('Heroes', function(table) {
    table.dropColumn('enemyId');
    table.dropColumn('heroName');
  });
};
```
- Modify our `heroes.js` seeds:
```js
exports.seed = function(knex, Promise) {
  return knex('Heroes').del()
    .then(function () {
      return knex('Heroes').insert([
        {id: 1, firstName: 'Clark', lastName: 'Kent', heroName: 'Superman', enemyId: 2},
        {id: 2, firstName: 'Bruce', lastName: 'Wayne', heroName: 'Batman', enemyId: 1},
        {id: 3, firstName: 'Peter', lastName: 'Parker', heroName: 'Spiderman'},
        {id: 4, firstName: 'Susan', lastName: 'Storm-Richards', heroName: 'Invisible Woman'},
      ]);
    });
};
```
- Run these migrations: `yarn knex migrate:latest && yarn knex seed:run`
- In our business layer, modify `hero.js` this way:
```diff
class Hero {
  id: number;
  firstName: string;
  lastName: string;
+  heroName: string;
+  enemyId: number;

  constructor(data) {
    this.id = data.id;
    this.firstName = data.firstName;
    this.lastName = data.lastName;
+    this.heroName = data.heroName;
+    this.enemyId = data.enemyId;
  }
...
}
```
- In our api, in our presentation layer, modify our `schema.js`:
```diff
const typeDefs = [`
  type Hero {
    id: Int
    firstName: String
    lastName: String
+    heroName: String
+    enemy: Hero
  }
...
`];

const resolvers = {
  Query: {
...
  },
+  Hero: {
+    enemy: async (hero, args, ctx) => Hero.load(ctx, {id: hero.enemyId}),
+  },
}
```

> **CHECK 1**: Using graphiql interface with the following query:
> ```
> {
>   hero(id:1) {
>     id
>     firstName
>     lastName
>     heroName
>     enemy {
>       heroName
>     }
>   }
> }
> ```
> ...should return Clark Kent with its heroName and its enemy: Batman.

### Push your api to the next level: use caching with Dataloader

> Trying to query heroes and their enemies'heroName will show up a N+1 problem. Indeed, our api make 5 round-trips to our database! Try yourself:
> ```json
> {
> 	"query": "{heroes { id firstName lastName heroName enemy { heroName } }}"
> }
> ```
> We can reduce these calls adding caching to our business layer

- Install `Dataloader`: `yarn add dataloader`
- Add a `getLoaders` method to our `hero.js` file in our business layer:
```js
import DataLoader from 'dataloader';

...

  static getLoaders() {
    const getById = new DataLoader(ids => HeroDB.getByIds(ids));
    const primeLoaders = (heroes) => {
      heroes.forEach(hero =>
        getById.clear(hero.id).prime(hero.id, hero))
      ;
    };
    return { getById, primeLoaders };
  }
```
- In our `index.js` file, add a new dataloader to our context for each query on `/api` route:
```diff
+import Hero from './business/hero';

...

router.post(
  '/api',
  graphqlKoa(async ctx => {
    return {
      schema: schema,
+      context: {
+        dataLoaders: {
+          hero: Hero.getLoaders(),
+        }
+      },
      debug: true,
    };
  })
);
```
- Back in our `hero.js` business layer file, modify `load` and `loadAll` methods to use our dataloader:
```diff
  static async load(ctx, args) {
+    const data = await ctx.dataLoaders.hero.getById.load(args.id);
    if (!data) return null;

    return new Hero(data);
  }

  static async loadAll(ctx, args) {
    const data = await HeroDB.getAll();
+    ctx.dataLoaders.hero.primeLoaders(data);

    return data.map(row => new Hero(row));
  }
```
- Protect `loader.load()` function call if no argument is supplied:
```diff
  static async load(ctx, args) {
+    if (!args.id) return null;
    const data = await ctx.dataLoaders.hero.getById.load(args.id);
    if (!data) return null;

    return new Hero(data);
  }
```

> **CHECK 1**: Using graphiql interface with the following query:
> ```
> {
>   heroes {
>     id
>     firstName
>     lastName
>     heroName
>     enemy {
>       heroName
>     }
>   }
> }
> ```
> ...should return all heroes and their enemies and your terminal should prompt only one request to the DB.
>
> **CHECK 2**: Using graphiql interface with the following query:
> ```
> {
>   h1: hero(id:1) {
>     id
>     firstName
>     lastName
>     heroName
>     enemy {
>       heroName
>     }
>   }
>   h2: hero(id:2) {
>     id
>     firstName
>     lastName
>     heroName
>     enemy {
>       heroName
>     }
>   }
> }
> ```
> ...should return Clark Kent and Bruce Wayne; and only one *SELECT* call should have beeen made to our DB.

### Add access control to our api

> This is a very simple example, for a more advanced solution, prefer using [Koa Jwt](https://github.com/koajs/jwt).

- In a new `utils.js` file, add these two methods to parse Authorization header and verify token:
```js
export const parseAuthorizationHeader = (req) => {
  const header = req.headers.authorization;

  if (typeof header === 'undefined' || header === 'null') {
    return null;
  }

  const [, scheme, token] = (/(\w+) ([\w.-]+)/g).exec(header);

  return token;
};

// Not production-ready: this is a simple example for the tutorial
export const verifyToken = token => new Promise((resolve, reject) => {
  if (token !== 'authorized') {
    const error = new Error('UNAUTHORIZED');
    error.code = 401;
    reject(error);
  }
  resolve();
});
```
- In our `index.js` file, parse authorization header and pass it to our context:
```diff
+import { parseAuthorizationHeader } from './utils';

...

router.post(
  '/api',
  graphqlKoa(async ctx => {
    return {
      schema: schema,
      context: {
+        authToken: parseAuthorizationHeader(ctx.req),
        dataLoaders: {
          hero: Hero.getLoaders(),
        }
      },
      debug: true,
    };
  })
);
```
- In our business layer, modify `hero.js`:
```diff
+import { verifyToken } from '../utils';

...

  static async load(ctx, args) {
+    await verifyToken(ctx.authToken);

...

  static async loadAll(ctx, args) {
+    await verifyToken(ctx.authToken);
```

> **CHECK 1**: In **Postman**, making a *POST* request to `localhost:3000/api` which content-type is *JSON(application/json)* with the following raw body:
> ```json
> {
> 	"query": "{hero(id:1) { id firstName lastName heroName }}"
> }
> ```
> ...should return `UNAUTHORIZED`.
>
> **CHECK 2**: In **Postman**, making a *POST* request to `localhost:3000/api` which content-type is *JSON(application/json)* with the following raw body:
> ```json
> {
> 	"query": "{heroes { id firstName lastName heroName }}"
> }
> ```
> ...should return `UNAUTHORIZED`.
>
> **CHECK 3**: In **Postman**, making a *POST* request to `localhost:3000/api` which content-type is *JSON(application/json)* and *Authorization Header* is `Bearer authorized` with the following raw body:
> ```json
> {
> 	"query": "{hero(id:1) { firstName lastName }}"
> }
> ```
> ...should return Clark Kent.
>
> ![](assets/authorization.png)

### Troubleshooting: Accessing data by id in the correct order

> You should notice that in **Postman** making a *POST* request to `localhost:3000/api` which content-type is *JSON(application/json)* and *Authorization Header* is `Bearer authorized` with the following raw body:
> ```json
> {
> 	"query": "{h1: hero(id:1) { id firstName lastName heroName enemy { heroName } } h2: hero(id:2) { id firstName lastName heroName enemy { heroName } }}"
> }
> ```
> ...returns the same than the following request (ids switched):
> ```json
> {
> 	"query": "{h1: hero(id:2) { id firstName lastName heroName enemy { heroName } } h2: hero(id:1) { id firstName lastName heroName enemy { heroName } }}"
> }
> ```
>
> This is due to our DB query: `select * from "Heroes" where "id" in (1, 2)` return the same result than: `select * from "Heroes" where "id" in (2, 1)`.

- In `utils.js`, add the following method:
```js
export const orderByArgIdsOrder = ids => ("array_position(string_to_array(?, ',')::integer[], id)", ids.join(','));
```
- In our db layer, modify `hero.js` like this:
```diff
+import { orderByArgIdsOrder } from '../../utils';

class Hero {
...

  static async getByIds(ids: Array<number>): Promise<Array<CostDBType>> {
    return db
    .select()
    .table('Heroes')
    .whereIn('id', ids)
+    .orderByRaw(orderByArgIdsOrder(ids));
  }
```

> **CHECK 1**: In **Postman**, making a *POST* request to `localhost:3000/api` which content-type is *JSON(application/json)* and *Authorization Header* is `Bearer authorized` with the following raw body:
> ```json
> {
> 	"query": "{h1: hero(id:2) { heroName } h2: hero(id:1) { heroName }}""
> }
> ```
> ...should return Batman (as `h1`) then Superman (as `h2`).

## Next steps

- Add [Koa Jwt](https://github.com/koajs/jwt)
- Add graphiQL with authorization header (get inspired by [BAM api](https://github.com/bamlab/bam-api)