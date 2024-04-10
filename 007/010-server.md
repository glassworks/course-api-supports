# Serveur

Nous allons nous connecter à un serveur que j'ai mis à votre disposition pour héberger votre API.

```
unixshell.hetic.glassworks.tech
```

## Clés SSH

L'intervenant va vous envoyer par voie privé une paire de clés SSH qui vous donneront accès à votre compte.

Dans vscode, créez un dossier "keys", et collez les 2 fichiers dedans.

Modifiez les droits sur la clé privée en émettant, dans le terminal VSCode :

```sh
chmod 600 keys/sshkey.pri
```

Maintenant vous pouvez vous connecter à mon serveur avec :


```sh
ssh -i keys/sshkey.pri [VOTRE_IDENTIFIANT]@unixshell.hetic.glassworks.tech
```

Remplacez VOTRE_IDENTIFIANT par votre adresse email hetic, ayant remplacé le "@" par un ".".

## Cloner votre projet

Configurez votre git comme sur votre machine locale :

```sh
git config --global credential.helper "store --file $HOME/.git-credentials"
```

On peut maintenant récupérer notre projet du serveur Git !

```sh
git clone LE_CHEMIN_DE_VOTRE_PROJET
```

Utiliser votre nom d'utilisateur, et le "access token" que vous avez crée dans l'étape précédente.

Naviguez dans le dossier de votre projet :

```sh
ls
cd [le nom de votre projet]
ls
```

Vous devez voir tous vos fichiers !



