#  Mise en œuvre avec Apollo Server

GraphQL est mis en œuvre dans de nombreux langages.

Notre objectif est d'ajouter une route GraphQL à notre serveur existant.

Pour ce faire, nous utiliserons le [serveur Apollo](https://www.apollographql.com/docs/apollo-server/).

Tout d'abord, nous allons installer le serveur Apollo et les bibliothèques GraphQL :

```bash
npm install @apollo/server graphql   
```

## Schéma

Nous allons déclarer notre schéma GraphQL dans `src/model/graphql/schema.graphql.ts` 


```ts
export const GRAPHQL_SCHEMA = `#graphql

type User {
  userId: Int!             # Not null
  familyName: String
  givenName: String
  email: String!           # Not null
  files: [UserFile!]
}

type UserFile {
  fileId: Int!              
  userId: Int!
  storageKey: String!
  filename: String
  mimeType: String!     
}

type Query {
  users: [User]
  user(userId: Int!): User
}

input UserDetails {
  familyName: String
  givenName: String
}

type Mutation {
  addUser(email: String!, familyName: String, givenName: String): User
  updateUser(userId: Int!, user: UserDetails!): User
  deleteUser(userId: Int!): Boolean
}
`
```

Vous remarquerez dans le schéma ci-dessus notre entité `User` telle qu'elle apparaît dans notre schéma SQL, ainsi que l'entité `UserFile`. Chaque utilisateur a plusieurs fichiers, comme nous l'avons implémenté précédemment.

Nous avons également ajouté une requête pour récupérer une liste d'utilisateurs, ou un seul utilisateur avec son `userId`.

Enfin, nous avons ajouté deux `Mutations`, c'est-à-dire des opérations d'écriture pour créer, mettre à jour et supprimer des utilisateurs.

Nous avons donc toutes nos opérations CRUD pour l'entité `User`

## Résolveurs

Un **resolver** peut être considéré comme l'implémentation réelle de nos requêtes et mutations. C'est ici que nous faisons le lien entre le schéma et notre base de données réelle.

```ts
import { IUser, IUserCreate, IUserUpdate } from "@model/types/IUser";
import { IUserFile } from "@model/types/IUserFile";
import { ORM } from "@orm/ORM";
import { GraphQLResolveInfo } from "graphql";

const READ_COLUMNS = ['userId', 'familyName', 'givenName', 'email'];

export const GRAPHQL_RESOLVERS = {
  Query: {
    users: async (parent: any, args: any, contextValue: any, info: GraphQLResolveInfo) => {
      const users = await ORM.Index<IUser>({
        table: 'user',
        columns: READ_COLUMNS,
      })
      return users.rows;      
    }, 
    user: async (parent: any, args: any, contextValue: any, info: GraphQLResolveInfo) => {
      return ORM.Read<IUser>({
        table: 'user', 
        idKey: 'userId', 
        idValue: args.userId, 
        columns: READ_COLUMNS
      });
    },    
  },  
  User: {
    files: async (parent: IUser) => {
      const files = await ORM.Index<IUserFile>({
        table: 'user_file',
        columns: ['fileId', 'userId', 'storageKey', 'filename', 'mimeType'],
        where: {
          userId: parent.userId
        }
      })
      return files.rows;      
    }
  },
  Mutation: {
    addUser: async (parent: any, args: IUserCreate, contextValue: any, info: GraphQLResolveInfo) => {
      const result = await ORM.Create<IUserCreate>({
        table: 'user',
        body: args,
      });
      return await ORM.Read<IUser>({
        table: 'user',
        idKey: 'userId',
        idValue: result.id,
        columns: READ_COLUMNS
      })
    },
    updateUser: async (parent: any, args: { userId: number, user: IUserUpdate }, contextValue: any, info: GraphQLResolveInfo) => {
      const result = await ORM.Update<IUserUpdate>({
        table: 'user',
        idKey: 'userId', 
        idValue: args.userId, 
        body: args.user,
      });
      return await ORM.Read<IUser>({
        table: 'user',
        idKey: 'userId',
        idValue: result.id,
        columns: READ_COLUMNS
      })
    },
    deleteUser: async (parent: any, args: { userId: number }, contextValue: any, info: GraphQLResolveInfo) => {
      await ORM.Delete({
        table: 'user',
        idKey: 'userId', 
        idValue: args.userId,         
      });
      return true;
    },
  }

};
```

Commençons par les requêtes. Comme vous pouvez le voir, les `Query.users` et `Query.user` fournissent des implémentations pour les requêtes correspondantes dans notre schéma. Plus simplement, nous fournissons une implémentation qui exécute la requête de données dans notre ORM et retourne les données, tout comme le font nos contrôleurs REST !

Vous remarquerez que le paramètre `args` fournit tous les arguments entrants de la requête, y compris le `userId` utilisé pour spécifier quel `User` doit être retourné.

Le résolveur `User` fournit une implémentation supplémentaire pour tous les sous-champs qui peuvent avoir besoin d'être récupérés. Dans notre cas, la liste des fichiers de l'utilisateur. Si un `User` est demandé dans lequel le tableau `files` fait partie de la projection, alors ce résolveur est appelé avec le `User` pré-chargé dans le paramètre `parent`. Cela permet au résolveur `files` d'effectuer la requête pour obtenir tous les fichiers de cet `User`.

Enfin, il y a les trois méthodes de mutateur pour ajouter, mettre à jour et supprimer des `Utilisateurs`. 

## Intégration avec Express

Nous voulons intégrer le serveur Apollo à notre application Express existante. Nous allons utiliser un routeur Express classique pour ajouter le serveur GraphQL à la route `/graphql`.

Pour cela, nous allons créer un routeur dans `src/routes/graphql.route.ts` :

```ts
import { ApolloServer } from "@apollo/server";
import { expressMiddleware } from '@apollo/server/express4';
import { ApolloServerPluginDrainHttpServer } from '@apollo/server/plugin/drainHttpServer';
import { GRAPHQL_RESOLVERS } from "@model/graphql/resolvers.graphql";
import { GRAPHQL_SCHEMA } from "@model/graphql/schema.graphql";
import { Router } from "express";
import { Server } from "http";

export const initGraphQL = async (server: Server) => {
  const apollo = new ApolloServer({
    typeDefs: GRAPHQL_SCHEMA,
    resolvers: GRAPHQL_RESOLVERS,
    plugins: [ApolloServerPluginDrainHttpServer({ httpServer: server })],
  });

  await apollo.start();

  const router = Router({ mergeParams: true });

  router.use(
    <any>expressMiddleware(apollo, {
      /*
      context: async ({ req, res }) => {
        try {
          return await expressAuthentication(<any>req, 'jwt');        
        } catch (err: any) {
          throw new GraphQLError('User is not authenticated', {
            extensions: {
              code: 'UNAUTHENTICATED',
              http: { status: err.httpCode },
            }
          });
        }
      }
      */
    })
  )

  return router;
}

```

Notez que nous instancions le serveur Apollo, en fournissant notre schéma et nos résolveurs. Nous démarrons le serveur et insérons un middleware fourni qui traitera les requêtes Apollo pour nous ;

Dans cet exemple, nous avons commenté l'option `context`, qui nous permet d'insérer notre autorisation JWT devant notre serveur GraphQL.

Enfin, nous incluons cette route dans notre fichier `src/server_manager.ts` :


```ts
...
import { initGraphQL } from "@routes/graphql.route";
...

export const StartServer = async () => {
  ...
  // Créer l'objet Express
  const app = Express();
  const httpServer = createServer(app);

  // Configurer CORS
  app.use(Cors())

  // L'appli parse le corps du message entrant comme du json
  app.use(json());

  // Utiliser un middleware pour créer des logs
  app.use(requestLogMiddleware('req'));

  ...

  // Graphql
  const graphql = await initGraphQL(httpServer);
  app.use('/graphql', graphql);

  ...

```

## Test du serveur GraphQL

Nous pouvons maintenant tester le serveur GraphQL. Exécutez votre API en utilisant :

```bash
npm run server
```

Ensuite, dans un navigateur, ouvrez [http://localhost:5050/graphql](http://localhost:5050/graphql).

Vous verrez un éditeur joliment formaté qui affiche votre schéma et vous permet d'effectuer des requêtes.

Vous pouvez essayer certaines de ces requêtes :

**Obtenir une liste d'utilisateurs :**

```graphql
query IndexQuery {
  users {
    userId    
    email
    familyName
    givenName
  }
}
```

**Recherchez l'utilisateur avec l'ID 2 :**

```graphql
query ReadQuery {
  user(userId: 2) {    
    familyName
  }
}
```

**Ajouter un utilisateur :**

```graphql
mutation MutationAdd {
  addUser(email: "bob@builder.com") {
    email
  }
}
```


**Mettre à jour un utilisateur :**

```graphql
mutation MutationUpdate {
  updateUser(userId: 2, user: { familyName: "Bob" }) {
    email
    familyName
    givenName
  }
}
```

**Supprimer un utilisateur :**

```graphql
mutation MutationDelete {
  deleteUser(userId: 2)
}
```

**Obtenir une liste d'utilisateurs et de leurs fichiers :**

```graphql
query IndexQuery {
  users {
    userId    
    email
    familyName
    givenName
    files {
      fileId
      storageKey
    }
  }
}
```