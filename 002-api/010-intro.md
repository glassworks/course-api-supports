# Introduction

L'objectif de ce cours est d'être capable de construire une API REST en utilisant NodeJS, Typescript et Express.

Avant de commencer cette section, vous devez avoir configuré votre conteneur de développement.

Si vous n'avez pas suivi le chapitre précédent sur les bases de javascript et de typescript, vous devriez au moins avoir un `package.json` à la racine de votre projet : 

```json
{
  "name": "api",
  "version": "1.0.0",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "author": "",
  "license": "ISC",
  "description": "",
  "devDependencies": {
    "nodemon": "^3.1.9",
    "ts-node": "^10.9.2",
    "typescript": "^5.7.3"
  }
}
```

Et un `tsconfig.json` à la racine de votre projet :

```json
{
  "compilerOptions": {    
    "target": "es2021",                                  /* Set the JavaScript language version for emitted JavaScript and include compatible library declarations. */    
    /* Modules */
    "module": "commonjs",                                /* Specify what module code is generated. */    
    "outDir": "./build",                                  /* Specify an output folder for all emitted files. */    
    "esModuleInterop": true,                             /* Emit additional JavaScript to ease support for importing CommonJS modules. This enables 'allowSyntheticDefaultImports' for type compatibility. */    
    "forceConsistentCasingInFileNames": true,            /* Ensure that casing is correct in imports. */

    /* Type Checking */
    "strict": true,                                      /* Enable all strict type-checking options. */    
    /* Completeness */
    "skipLibCheck": true                                 /* Skip type checking all .d.ts files. */
  }
}
```

Ensuite, exécutez :

```bash
npm install
```