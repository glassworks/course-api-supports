# Schema

GraphQL est une norme, qui est mise en œuvre par de nombreuses bibliothèques et langages.

Lors de la construction d'une API GraphQL, nous commençons par déclarer les types d'objets pris en charge par notre API. 

Pour notre exemple d'utilisateur, nous déclarons notre type comme suit :

```graphql
type User {
  userId: Int!                # Not null
  familyName: String
  givenName: String
  email: String!              # Not null  
}
```

Une fois que nous avons déclaré un type, nous pouvons déclarer des `Queries` pour demander des instances de ce type :

```graphql
type Query {
  users: [User]
  user(userId: Int!): User
}
```

Dans l'exemple ci-dessus, la première requête obtient un tableau de `User`. La deuxième requête récupère une instance de `User`.

Ce langage de spécification présente un certain nombre d'avantages, notamment : il peut être transformé automatiquement en types de script, les validateurs peuvent être créés et exécutés automatiquement, nous pouvons construire et transmettre automatiquement la documentation, etc.

Un client peut ensuite demander des données, en spécifiant exactement ce dont il a besoin. Par exemple, ci-dessous, nous demandons la requête `user` et transmettons l'ID utilisateur `1`. Plus particulièrement, nous spécifions la **projection**, c'est-à-dire les champs de cet objet en particulier, que nous souhaitons obtenir.



```graphql
query ReadUserQuery {
  user(userId: 1) {    
    familyName
  }
}
```

Une API GraphQL répondrait avec l'objet JSON associé :

```ts
{
  "data": {
    "user": {
      "familyName": "Squarepants"
    }
  }
}
```

## Combinaison de requêtes

Imaginez que nous ayons plusieurs types (correspondant à plusieurs tables dans notre base de données).

```graphql
type User {
  userId: Int!                # Not null
  familyName: String
  givenName: String
  email: String!              # Not null  
}

type Film {
  filmId: Int! 
  title: String  
}

type Actor {
  actorId: Int! 
  familyName: String
  givenName: String
}

type Query {
  users: [User]
  films: [Film]
  actor: [Actor]
}
```

Dans une implémentation REST, nous aurions probablement plusieurs routes pour accéder à une liste d'utilisateurs, d'acteurs et de films :

- `GET /user`
- `GET /actor`
- `GET /film`

... et nous ferions trois appels HTTP pour récupérer toutes ces données.

Dans GraphQL, nous pouvons tout demander en une seule fois :

```graphql
query ReadAllQuery {
  users {
    givenName
    familyName    
  }
  films {
    title
  }
  actors {
    givenName
    familyName   
  }
}
```

Le résultat **unique** serait un objet combiné contenant toutes nos données 

```json
{
  "data": {
    "users": [
      {
        "givenName": "SpongeBob",
        "familyName": "Squarepants"
      }
    ],
    "films": [
      { 
        "title": "The Lion King"
      },
      { 
        "title": "Jurassic Park"
      }
    ],
    "actors": [
      {
        "givenName": "Bruce",
        "familyName": "Willis"
      },
      {
        "givenName": "Jodie",
        "familyName": "Foster"
      }
    ]
  }
}
```

## Relations


Les données ont généralement des relations entre les entités, qui sont facilement représentées dans GraphQL. Par exemple, un film compte de nombreux acteurs :

```graphql
type Film {
  filmId: Int! 
  title: String
  actors: [Actor!]
}

type Actor {
  actorId: Int! 
  familyName: String
  givenName: String
}

type Query {
  users: [User]
  films: [Film]
  actor: [Actor]
}
```

Pour obtenir chaque film, avec une liste imbriquée de ses acteurs sous forme de tableau, nous pourrions donc interroger le graphe comme suit :

```graphql
query ReadFilmQuery {
  films {
    title
    actors {
      familyName
    }
  }  
}
```

This would return 

```json
{
  "data": {
    "films": [
      { 
        "title": "The Lion King",
        "actors": [
          {
            "familyName": "Irons"
          },
          {
            "familyName": "Atkinson"
          },
        ]
      },
      { 
        "title": "Jurassic Park",
        "actors": [
          {
            "familyName": "Dern"
          },
          {
            "familyName": "Goldblum"
          },
        ]
      }
    ],    
  }
}
```

Comme vous pouvez le constater, GraphQL offre un moyen très naturel d'accéder aux données !

For a more in-depth analysis of the GraphQL standard, you may consult the very informative officiel website [https://graphql.org/learn/](https://graphql.org/learn/).
