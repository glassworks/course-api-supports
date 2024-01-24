# Introduction

Qu'est-ce qu'une API ?

En anglais, API signifie **Application Programming Interface**, ce qui signifie que c'est l'*interface* qu'un développeur crée pour nous permettre d'accéder à ses systèmes.

Imaginez un logiciel comme une boîte noire, qui possède en interne toutes ses fonctions, ses procédures, ses classes, sa logique commerciale, son accès aux bases de données, etc. Le développeur de cette boîte noire ne veut pas nécessairement que nous sachions comment elle fonctionne.

Le développeur de cette boîte noire ne souhaite pas nécessairement que nous sachions comment elle fonctionne. Mais il se peut que nous ayons besoin de récupérer certaines informations de la boîte noire, de lui envoyer des données ou de lui demander d'effectuer une tâche.

Une API fournit un ensemble de fonctions ou de points de terminaison **visés par le public** qui nous permettent d'interagir avec la boîte noire.

Les logiciels système modernes ont des API. Par exemple, si vous voulez demander à Windows d'ouvrir une nouvelle fenêtre, il existe une fonction spéciale que vous pouvez appeler dans l'API de Windows pour le faire !

Dans le domaine de l'internet, les API concernent principalement l'échange de données. En général, une plateforme en ligne publie un **endpoint** (ou URL) accessible au public, que l'on peut invoquer à l'aide de requêtes HTTP. L'invocation d'un tel point final peut nous permettre de récupérer des données, ou d'injecter des données, ou de déclencher une action.