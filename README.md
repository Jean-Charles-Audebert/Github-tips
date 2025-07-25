# Github-tips
Configuration github

Ce repository vise à rassembler mes configurations github. 

## Gérer plusieurs profiles localement avec clés ssh

J'ai plusieurs comptes github : perso et professionnel. Voici comment gérer l'authentification pour utiliser git batch.

Je pars de [ce dépôt](https://github.com/maxwell-balla/tips-github-account/blob/master/managing-multiple-github-accounts.md)

### Arborescence des répértoires sur mon pc (windows)

Je choisis de centraliser tous mes projets dans un répértoire Workspaces :

```
📦c:
 ┣ 📂Workspaces
 ┃ ┣ 📂Perso
 ┃ ┣ 📂Pro
```

### Générer des clés SSH pour chaque profil

[la doc](https://docs.github.com/fr/authentication/connecting-to-github-with-ssh/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent)

* Créer une clé sous windows : `ssh-keygen -t ed25519 -C "your_email@example.com"`
* `> Enter file in which to save the key (/c/Users/YOU/.ssh/id_ALGORITHM):[Press enter]` > nommer le fichier par exemple `github-perso` pour s'y retrouver plus tard
* Laisser les passphrases à vide (`Enter`)
* Ajouter la clé au ssh-agent `ssh-add c:/Users/YOU/.ssh/github-perso`

### Ajouter la clé publique au compte github

* sur navigateur, se connecter au compte visé, aller dans le profile utilisateur > `Settings> ssh and gpg keys`
* dans le terminal, récupérer la clé publique générée dans l'étape précédente :
  * Se rendre dans le dossier `c:/Users/YOU/.ssh/`où on a généré la clé précedemment
  * Récupérer la valeur de la clé publique : `echo github-perso.pub`
  * Copier tout ce qui est contenu
  * Retourner sur le navigateur, cliquer sur "New SSH Key". Donner un nom (par exemple le nom du pc) et coller la clé publique, enregistrer.

**Répéter ces étapes pour chaque profile github**

### Configuration pour gérer les différents profils

La difficulté à avoir plusieurs profiles github vient du fait que l'authentification ne s'intéresse qu'à la route `github.com`, github n'est pas conçu pour gérer plusieurs profiles, donc il ne vérifie pas les noms des utilisateurs. On veut donc lui créer des alias.

* dans le dossier `c:/Users/YOU/.ssh/`, on édite ou on crée un fichier `config` (sans extension) :
  ```
  Host github-pro
  HostName github.com
  User git
  IdentityFile ~/.ssh/github_pro
  IdentitiesOnly yes

Host github-perso
  HostName github.com
  User git
  IdentityFile ~/.ssh/github_perso
  IdentitiesOnly yes
  ```
Ici, on a créé un alias pour chaque profil, qui pointe sur Host

### Tester la connexion SSH

Pour vous assurer que tout fonctionne, testez la connexion à GitHub avec les alias que vous avez créés.
Test pour le compte personnel :

`ssh -T git@github-perso`

Test pour le compte travail :

`ssh -T git@github-pro`

### Configurer l'url distante des dépôts

Aller dans le dossier de notre projet, taper `git remote set-url origin git@github-perso:<Username>/repo.git` ou github-pro selon le cas.

Cela modifie l'adresse du repo, c'est la méthode la plus sûre.

### PLUS LOIN : Configurer le changement d'authentification automatiquement

! Cela ne fonctionne pas toujours, peut être écrasé par des variables d'environnement, ignoré par les ide, etc... Pas fiable pour l'instant

* Dans le dossier `c:/Users/YOU/` créer un répertoire `.git`
* Dans ce répertoire, créer le fichier `.gitconfig-perso` :
```
[user]
    name = <votre username github perso>
    email = <email du compte perso>
[core]
    sshCommand = "ssh -i ~/.ssh/github_perso"
```

* La même chose pour le fichier pro : `.gitconfig-pro`
```
[user]
    name = <votre username github pro>
    email = <email du compte pro>
[core]
    sshCommand = "ssh -i ~/.ssh/github_pro"
```

* Revenir dans le dossier `c:/Users/YOU/` et créer un fichier `.gitconfig`

```
[user]
    name = <username du compte par défaut que l'on veut>
    email = <email du même compte>  # Valeur par défaut (compte perso)

[includeIf "gitdir:~/Workspaces/Perso/"]
    path = ~/.git/.gitconfig-perso

[includeIf "gitdir:~/Workspaces/Pro/"]
    path = ~/.git/.gitconfig-pro

[filter "lfs"]
    clean = git-lfs clean -- %f
    smudge = git-lfs smudge -- %f
    process = git-lfs filter-process
    required = true

[credential]
    helper = store
    ```

### Encore plus loin : script pour automatiser la mise à jour des remotes selon le chemin :

```bash
#!/bin/bash

set -e

# -----------------------
# CONFIG
# -----------------------
PRO_PATH="Workspaces/Pro/"
PRO_HOST="github.com-pro"
PRO_USERNAME="usernamePro"

PERSO_PATH="Workspaces/Perso/"
PERSO_HOST="github.com-perso"
PERSO_USERNAME="usernamePerso"

# Mode dry-run ? -- c'est pour tester sans rien pousser
DRY_RUN=false
if [[ "$1" == "--dry-run" ]]; then
    DRY_RUN=true
fi

# -----------------------
# Détection du chemin
# -----------------------
REPO_PATH=$(pwd)

if [[ "$REPO_PATH" == *"/$PRO_PATH"* ]]; then
    HOST="$PRO_HOST"
    USERNAME="$PRO_USERNAME"
elif [[ "$REPO_PATH" == *"/$PERSO_PATH"* ]]; then
    HOST="$PERSO_HOST"
    USERNAME="$PERSO_USERNAME"
else
    echo "❌ Dossier non reconnu : $REPO_PATH"
    echo "👉 Ajoute une nouvelle règle dans le script."
    exit 1
fi

# -----------------------
# Récupération du nom de repo
# -----------------------
GIT_TOP=$(git rev-parse --show-toplevel 2>/dev/null || true)

if [[ -z "$GIT_TOP" ]]; then
    echo "❌ Ce dossier n'est pas un dépôt Git."
    exit 1
fi

REPO_NAME=$(basename "$GIT_TOP")

# -----------------------
# Construction de l'URL SSH
# -----------------------
NEW_URL="git@${HOST}:${USERNAME}/${REPO_NAME}.git"

# -----------------------
# Vérifie si origin existe
# -----------------------
CURRENT_URL=$(git remote get-url origin 2>/dev/null || echo "absent")

echo "📦 Repo     : $REPO_NAME"
echo "📍 Chemin   : $REPO_PATH"
echo "🔐 Compte   : $USERNAME"
echo "🔗 Remote   : origin"
echo "🌐 Actuel   : $CURRENT_URL"
echo "➡️  Cible    : $NEW_URL"

# -----------------------
# Dry run
# -----------------------
if $DRY_RUN; then
    echo "✅ Mode dry-run, rien n'a été modifié."
    exit 0
fi

# -----------------------
# Ajout ou mise à jour du remote
# -----------------------
if [[ "$CURRENT_URL" == "absent" ]]; then
    git remote add origin "$NEW_URL"
    echo "✅ Remote 'origin' ajouté."
else
    git remote set-url origin "$NEW_URL"
    echo "✅ Remote 'origin' mis à jour."
fi
```

#### Utilisation :

`~/update-github-remote.sh`

ou pour tester (dry run) : 

`~/update-github-remote.sh --dry-run`

#### Bonus créer un alias


Editer ou créer le fichier .bashrc dans `c:/Users/YOU/`

`nano ~/.bashrc`

Ajouter les alias à la fin :

```
alias gitset='~/update-github-remote.sh'
alias gitcheck='~/update-github-remote.sh --dry-run'
```

Puis recharger le fichier :

`source ~/.bashrc`

Test :

```
cd ~/Workspaces/Perso/un-repo
gitcheck  # vérifie l'URL sans changer

gitset     # applique la bonne URL SSH
```







