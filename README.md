# Dockeriser et déployer une app Node \(2/2\)

Lorsque l'on a besoin de déployer souvent sur le serveur de production, mieux vaut être organisé. Un bon outil pour cela est [Stagecoach](https://github.com/punkave/stagecoach), développé par Punk'Ave, le créateur d'[Apostrophe](/blog/presentation-du-cms-apostrophe-1-3).

## Présentation

Stagecoach permet de déployer des applications Node sur plusieurs serveurs \(staging, production..\) et de conserver des versions précédentes des déploiements pour revenir en arrière si nécessaire. Il s'occupe de faire la connexion avec le serveur, exécuter des scripts de pré et post-installation \(pour construire les ressources telles que les images, les fichiers CSS..\) et déposer notre application sur le serveur distant.

Par défaut, il utilise `forever`, un package NPM qui surveille l'état d'un script, pour redémarrer l'appli, mais comme nous utilisons Docker avec un restart automatique de Nginx, nous allons modifier Stagecoach.

## Configuration

L'installation du projet se déroule comme ceci :

```bash
[on your development machine]
cd /User/MYUSERNAME
mkdir -p src
cd src
git clone https://github.com/punkave/stagecoach
cd stagecoach
subl ~/.profile
[add /User/MYUSERNAME/src/stagecoach/bin to your PATH]
```

Cela permettra d'utiliser la commande `sc-deploy`. Dans cet article, nous allons configurer un déploiement pour un environnement de production \(`sc-deploy production`\), mais Stagecoach peut se configurer pour toutes sortes d'environnement.

Dans l'appli à déployer, nous allons créer un dossier `deployment` issu de Stagecoach :

```bash
cp -r src/stagecoach/example/deployment src/YOURAPPHERE/deployment
```

Ce dossier contient des exemples de configuration \(`settings.example` et `settings.production.example`\). Nous allons les copier et les nommer `settings` et `settings.production`. Là, c'est à vous de jouer pour mettre les bons paramètres \(IP du serveur, user, port, dossier de destination) mais les examples sont assez parlants et la doc aide bien.

Dans ce dossier `deployment`, 3 scripts sont à modifier pour notre app dockerisée : `stop`, `start` et `before-connecting`.

Dans `stop`, nous allons remplacer le contenu existant par :

```bash
#!/bin/bash

docker stop falkodev-site-db falkodev-site falkodev-web
docker rm falkodev-web
echo "Prune docker images and volumes"
docker image prune -a --force
docker system prune --force
docker volume prune --force
docker network prune --force
echo "Site stopped"
```

A remarquer : nous supprimons au passage l'utilisation de `forever`. A chaque déploiement, Stagecoach joue d'abord le script `stop` pour arrêter l'instance en cours. Ici, nous arrêtons les images Docker de notre app et, pour sauvegarder de l'espace, nous les supprimons, car cela peut prendre beaucoup de place après seulement quelques déploiements.

Dans `start`, voici le contenu à mettre :

```bash
#!/bin/bash

docker network create siteperso
docker-compose up -d
docker run --rm --link falkodev-site-db:mongo --net siteperso -v $(pwd)/backup/db/latest/site/:/dump mongo bash -c 'mongorestore -d site /dump --host mongo:27017 --drop'
echo "Site started"
```

Ici, on recrée le réseau de conteneur que nous avions vu dans [le billet précédent](/dockeriser-et-deployer-une-app-node-1-2) et nous relançons un build en arrière-plan de l'image Docker contenant l'application.

Enfin, une restauration d'une sauvegarde de la base de données permet de rafraîchir l'appli avec les données les plus récentes \(plus d'informations sur cette sauvegarde de BD dans la suite de l'article\).

Le contenu de `before-connecting` ne fait que démarrer un script :

```bash
#!/bin/bash

sh ./scripts/build-dist.sh
echo "Pre-deploy scripts ok"
```

## Préparation du déploiement

Que fait ce script `build-dist` ? C'est celui qui prépare le déploiement en créant un dossier `dist`, en [babelifiant](https://babeljs.io/) son contenu et en créant la fameuse sauvegarde de base de données.

Voici son contenu, adapté à mon application mais les commentaires sont assez explicatifs je pense :

```bash
#!/bin/bash

# save db
mongodump --db site --out ~/site_perso/backup/db/latest/

# save uploads and sync them to remote server
chmod -R 755 public/uploads
rsync -a --exclude='**/.DS_Store' public/uploads backup
rsync -caPzy --no-relative --stats --human-readable -e "ssh -p 22 -l user" --exclude='**/.DS_Store' ./backup/uploads/ user@remote_server:/opt/stagecoach/apps/site/uploads/

# clean data folder and regenerate public folder
rm -rf data && rm -rf public
npm run webpack

# recreate dist folder
rm -rf dist
mkdir dist

# generate sitemap
NODE_ENV=production node app apostrophe-site-map:map > public/sitemap.xml && tail -n +2 public/sitemap.xml > public/sitemap.xml.tmp && mv public/sitemap.xml.tmp public/sitemap.xml

# copy custom code, transpile it to ES5 and prettify it
rsync -a --exclude='**/.DS_Store' config dist
rsync -a --exclude='**/.DS_Store' --exclude='lib/modules/webpack-custom/' lib dist
rsync -a app.js dist
npx babel dist -d dist
npx prettier --write 'dist/**/*.js'

# copy other folders
rsync -a --exclude='**/.DS_Store' favicons/output/ public/favicons
rsync -a --exclude='**/.DS_Store' public dist
rsync -a --exclude='**/.DS_Store' nginx dist
rsync -a --exclude='**/.DS_Store' letsencrypt dist
rsync -a --exclude='**/.DS_Store' backup/uploads public
rsync -a robots.txt dist/public
```

## Déploiement

Il est temps de déployer l'application sur le serveur de production en exécutant la commande `sc-deploy production` dans le dossier de notre appli, sur la machine de développement. Il faut bien entendu avoir déjà un serveur avec un accès ssh paramétré et un utilisateur configuré.

Les scripts préparatoires se déroulent, puis Stagecoach envoie le contenu de `dist` sur le serveur et arrête l'instance précédente :

![](/assets/stop.png)

Puis, démarre une nouvelle instance :

![](/assets/start.png)

Par défaut, Stagecoach conserve les 5 derniers déploiements pour faire des rollbacks en cas de besoin. Voici sur mon serveur à quoi ressemble le répertoire `/opt/stagecoach/apps/site/deployments` par exemple :

![](/assets/deployments.png)

On constate que Stagecoach crée un nouveau dossier à chaque déploiement avec la date et l'heure courantes, puis supprime les dossiers caducs. Il fait un lien symbolique entre le dossier le plus récent et un répertoire nommé `current` qui contient la version de l'application à exécuter.

Voilà, j'espère que cet article en 2 parties vous aura aidé à appréhender la mise en pratique de Docker, du poste de développement au serveur de production.

