# DevOps - Compte Rendu - VEBER Vincent

## Vocabulaire

### Containers
Un container Docker, à l’opposé de machines virtuelles traditionnelles, ne requiert aucun système d’exploitation séparé. Il s’appuie plutôt sur les fonctionnalités du noyau du système d’exploitation sous-jacent et utilise de l’isolation de ressources (CPU, mémoire, I/O, connexions réseau etc…).

### Images
Une image Docker représente le système de fichiers, sans les processus. Elle contient tout ce que vous avez décidé d’y installer (Java, une base de donnée, un script que vous allez lancer, etc…), mais est dans un état inerte. Les images sont créées à partir de fichiers de configuration, nommés « Dockerfiles », qui décrivent exactement ce qui doit être installé sur le système. Un conteneur est l’exécution d’une image.

### Dockerfiles
Le Dockerfile est votre fichier source qui une fois compilé donne une image. Un Dockerfile peut être inclus dans d’autres Dockerfile, et être à la base de plusieurs images différentes. Vous pouvez choisir d’utiliser des images officielles, des images modifiées que vous trouvez sur Docker Hub ou de personnaliser vos propres images en écrivant un Dockerfile.

### Compose
Compose est un outil pour définir et exécuter des applications multi-conteneurs. Les conteneurs étant idéaux pour des applications basées sur des micro services, il devient évident que rapidement on doit faire fasse à l’interconnexion de plusieurs conteneurs et à la gestion de multi-conteneurs.

### Volumes
Les volumes sont le mécanisme le plus adapté pour la persistance des données générées par et utilisées par les conteneurs Docker. Les volumes Docker ont de nombreux avantages, tels qu’une grande simplicité de sauvegarde ou de migration ou encore leurs gestions facilités grâce aux outils et commandes Docker.

## TP1 - Docker

### Commandes utiles

#### Informations diverses sur Docker
$ docker info
$ docker version

#### Lancer un container :
$ docker run

#### Arrêter un container :
$ docker stop

#### Démarrer un container arrêté :
$ docker start

#### Supprimer un container (après l’avoir stopper) :
$ docker rm

#### Liste des containers actifs :
$ docker ps

#### Liste des containers actifs et inactifs :
$ docker ps -a

#### Arrêter tous les containers :
$ docker stop $(docker  ps -a -q)

#### Démarrer tous les containers :
$ docker start $(docker ps -a -q)

#### Supprimer tous les containers (après les avoir stopper) :
$ docker rm $(docker  ps -a -q)

#### Liste des images :
$ docker images

#### Supprimer une image :
$ docker rmi

#### Exécuter un Dockerfile :
$ docker-compose -d -f dockerfile.yml

#### Entrer dans un container :
$ docker exec -it [container] bash

#### Afficher les logs d’un container :
$ docker logs [container]

#### Build an image from a Dockerfile
docker build (-t <NAME>) <PATH>/<URL>

### Database

#### Basics

On lance d'abord notre image docker (ici tp1_database-app) avec `docker run -p 8888:5000 --name tp1_database-app -d vebervi/tp1_database-app`.
<br>
Ensuite, on lance l'image Adminer pour avoir une interface directement liée à la base de données de tp1_database-app avec `docker run --link tp1_database-app -p 8080:8080 -d adminer`.
<br>
A la place, on peut aussi définir un network avec `--network tp1` dans nos commandes (il faut d'abord le créer avant avec `docker network create tp1`) : on obtient alors :
```
docker run -p 8888:5000 --name tp1_database-app -d -e POSTGRES_DB="db" -e POSTGRES_USER="usr" -e POSTGRES_PASSWORD="pwd" -v /home/osboxes/Documents/DevOps/TP1/database-app_data:/var/lib/postgresql/data --network tp1 vebervi/tp1_database-app
docker run -p 8080:8080 -d --network tp1 adminer
```
<br>
On pourra donc se connecter à cette interface à l'adresse `http://localhost:8080/` en entrant les informations suivantes :

- Système : PostgreSQL
- Serveur : tp1_database-app
- Utilisateur : usr (valeur définie dans Dockerfile de database-app)
- Mot de passe : pwd (valeur définie dans Dockerfile de database-app)
- Base de données : db (valeur définie dans Dockerfile de database-app)

_Also, does it seem right to have passwords written in plain text in a file? You may rather define those environment parameters when running the image using the flag “-e”. Why should we run the container with a flag -e to give the environment variables ?_
Il est vrai qu'avoir les informations de connexion en clair dans un fichier texte n'est pas terrible. On peut donc supprimer les trois lignes de définition de ces variables d'environnement du fichier Dockerfile et lancer la commande `docker run -p 8888:5000 --name tp1_database-app -d -e POSTGRES_DB="db" -e POSTGRES_USER="usr" -e POSTGRES_PASSWORD="pwd" vebervi/tp1_database-app` à la place.
<br>
Grâce au flag `-e`, on peut définir les informations de connexion plus simplement qu'en passant par un Dockerfile et rebuilder à chaque fois notre image docker.

#### Init database

Après avoir rajouter les deux fichiers, il ne faut pas oublier de rajouter les deux lignes suivantes dans le Dockerfile, autrement nos fichiers ne seront pas transmis au container : 
```
COPY ./docker-entrypoint-initdb.d/01-CreateScheme.sql /docker-entrypoint-initdb.d/
COPY ./docker-entrypoint-initdb.d/02-InsertData.sql /docker-entrypoint-initdb.d/
```

#### Persist data

_Why do we need a volume to be attached to our postgres container ?_
Pour le pas perdre nos données si le container crash (persistance des données).
<br>
Pour que les données soient persistantes, on doit rajouter le paramètre `-v` qui permet de relier un dossier du container avec un dossier de la machine hôte (binding) : ainsi, les données sont sauvegardées en continu sur la machine hôte.
<br>
Au final, notre commande de lancement de notre image docker devient : `docker run -p 8888:5000 --name tp1_database-app -d -e POSTGRES_DB="db" -e POSTGRES_USER="usr" -e POSTGRES_PASSWORD="pwd" -v /home/osboxes/Documents/DevOps/TP1/database-app_data:/var/lib/postgresql/data vebervi/tp1_database-app`.

### Backend API

#### Multistage Build

_Why do we need a multistage build ?_
Le but du multistage build est de garder nos docker légers en ayant une partie qui build et une autre qui exécute (supprime le jdk dès qu'on en a plus besoin).
<br>
J'ai eu une erreur bizarre où une classe était "duplicate" (peut-être à cause de la succession de build qui ne fonctionnaient pas) : la solution était de complètement supprimer l'image docker avec `docker image rm vebervi/tp1_backendapi-app`.
<br>
Une fois que le build veut bien marcher (`docker build -t vebervi/tp1_backendapi-app .`), on lance notre docker avec `docker run -p 8989:8080 --name tp1_backendapi-app -d --network tp1 vebervi/tp1_backendapi-app`.
<br>
Quand on arrive sur localhost:8989, on a bien notre petit message du backend.
<br>
Par ailleurs, pour s'assurer que notre application s'est bien lancée sans erreur, on peut regarder les logs d'un container avec `docker logs tp1_backendapi-app` ou `docker logs [--follow] tp1_backendapi-app`.

#### Backend API

On récupère le code de GitHub et l'unique chose à modifier est l'accès à la base de données : dans `application.yml`, on spécifie que la base de données est sur notre container à un certain port avec `url: "jdbc:postgresql://tp1_database-app:5432/db"`.
<br>
On peut ensuite correctement accéder à notre API (par exemple http://localhost:8080/departments/IRC/students ou http://localhost:8989/students/5 ou encore http://localhost:8989/departments/IRC/count).

### HTTP Server

#### Basics

Pas grand chose de particulier, notre Dockerfile est la suivante :
```
FROM httpd
COPY ./public-html/ /usr/local/apache2/htdocs/
```
On build, on exécute `docker run -d --name tp1_httpserver-app -p 9090:80 --network tp1 vebervi/tp1_httpserver-app` et c'est bo !

#### Configuration

On exécute la commande `docker exec tp1_httpserver-app cat /usr/local/apache2/conf/httpd.conf` pour que notre container nous renvoie le résultat de la commande, c'est à dire le contenu du fichier de configuration du serveur.
<br>
Ainsi, on pourra le modifier et rajouter à notre Dockerfile la ligne `COPY ./httpd.conf /usr/local/apache2/conf/`.

#### Reverse Proxy

Après avoir modifié le httpd.conf avec les bons serveurs de proxy (http://tp1_backendapi-app:8080/), notre proxy fonctionne correctement : en se connectant à http://localhost:9090 (le httpserver-app), on atterit bien sur notre backend (http://localhost:9090/departments/IRC/students fonctionne bien).
<br>
_Why do we need a reverse proxy ?_
Loadbalancing/ne pas attaquer directement le server / (ne pas etre obligé de faire du ssl entre chaque micro-service...)

### Link application

#### Docker-compose

Au moment de construire notre docker-compose.yml, il faut faire attention au nom des services : il faut bien mettre ceux que l'on utilisait dans les commandes.
<br>
Ainsi, notre docker-compose.yml est le suivant :
```yml
version: '3.7'
services:
  tp1_backendapi-app:
    build: ./backendapi-app/simple-api/
    networks:
      - tp1
    depends_on: [tp1_database-app]

  tp1_database-app:
    build: ./database-app/
    networks:
      - tp1
    volumes :
      - /home/osboxes/Documents/DevOps/TP1/database-app_data:/var/lib/postgresql/data vebervi/tp1_database-app
    environment:
      - POSTGRES_DB=db
      - POSTGRES_USER=usr
      - POSTGRES_PASSWORD=pwd

  tp1_httpserver-app:
    build: ./httpserver-app/
    ports:
      - '9090:80'
    networks:
      - tp1
    depends_on: [tp1_backendapi-app]

networks:
  tp1:
    name: tp1
    driver: bridge
```

On build alors notre docker-compose avec `docker-compose build` et on le run avec `docker-compose up -d [--remove-orphans]` (remove orphans permet de supprimer les anciens containers lancés quand le run ne fonctionnait pas encore tout à fait).
<br>
On comprent alors que docker-compose est important car il permet de builder et runner différents containers dans un ordre particulier avec certains paramètres à l'aide d'une seule commande. Cela permet donc d'avoir un contrôle total de notre chaîne de déploiement et de gagner pas mal de temps grâce à ces deux commandes.

#### Publish

Tout comme Git, cela permet d'avoir accès à ses containers depuis plusieurs endroits et de gérer la collaboration et les builds plus facilement.

## TP 2 - CI/CD (Continuous Integration / Continuous Delivery|Deployment)

### Technologies

- CI : `Travis CI (yaml)` | `Jenkins (Groovy)` | `Gitlab CI` | `Bitbucket Pipeline`

Logiciel libre d'intégration continue. Il fournit un service en ligne utilisé pour compiler, tester et déployer le code source des logiciels développés, notamment en lien avec le service d'hébergement du code source GitHub.

### Sample application

#### Let's run

Effectivement, c'est tout simple de lancer l'application puisqu'il suffit d'exécuter les commandes `docker-compose build` et `docker-compose up -d` puis de se rendre sur `localhost` (pas de port à préciser puisque c'est le 80 (celui de base de http)).

### Setup Travis CI

#### Build and test your application

La commande `mvn clean verify` permet de "mettre au propre" l'application (téléchargement des librairies et détruire les dossiers créés précedemment (des anciens build)) ainsi que lancer des tests unitaires et de build (côté java) et `npm run test` pour lancer les tests du frontend.
<br>
_Unit tests ? Component test ?_
Unit tests: En programmation informatique, le test unitaire ou test de composants est une procédure permettant de vérifier le bon fonctionnement d'une partie précise d'un logiciel ou d'une portion d'un programme.
<br>
Component test: composant servant à faire le test
Les tests de composants ont pour but de tester les différents composants du logiciel séparément afin de s'assurer que chaque élément fonctionne comme spécifié. Ces tests sont aussi appelés test unitaires et sont généralement écrits et exécutés par le développeur qui a écrit le code du composant
<br>
_What are testcontainers?_
Les testcontainers viennent d'une librairie Java qui permet de tester (tests unitaires (JUnit), des bases de données...) l'application dans des containers à part.
<br>
_What does the default java and node.js travis images do?_
Les images (languages) par défaut de travis (java et node_js) permet de lancer nos build dans des "espaces connus" par Travis CI - c'est donc important de bien avoir les projets liés au bons languages.
<br>
Le .yml fourni n'est pas tout à fait complet puisqu'il manque la partie script. Les commandes à exécuter se trouvent dans les README des répertoires en question (backend et frontend). Hormis cela, il n'y a pas grand chose à dire hormis que c'est bien `.travis.yml` (avec un point au début) et que `git: depth: 5` est le nombre de commit que Travis CI va considérer (il peut y en avoir jusqu'à 50 ou bien être mis à 0 (false)).
<br>
Il faut aussi faire attention à l'indentation car un espace en trop et c'est le drame... (pour vérifier notre fichier, on peut aller sur https://config.travis-ci.com/explore).
```yml
git:
  depth: 5

jobs:
  include:
    - stage: "Build and Test Java"
      language: java
      jdk: oraclejdk11
      before_script:
        - cd sample-application-backend
      script:
        - mvn clean install
        - mvn test
    
    - stage: "Build and Test Nodejs"
      language: node.js
      node_js: "12.20"
      before_script:
        - cd sample-application-frontend
      script:
        - npm install
        - npm run build

cache:
  directories:
    - "$HOME/.m2/repository"
    - "$HOME/.npm"
```

#### First steps into the CI world

_Why do we need this branch (develop)?_
https://www.atlassian.com/fr/git/tutorials/comparing-workflows/gitflow-workflow
<br>
Nous avons besoin de la branche `develop` car elle permet d'avoir une branche master "propre" et qui devrait être fonctionnelle tout le temps ainsi que cette seconde branche qui est destinée aux développements et autres tests.
<br>
Cette branche permet aussi de mettre un webhook afin de vérifier le code pushé avant de l'envoyer sur master qui doit etre un code fonctionnel (qu'on peut envoyer en prod).
<br>
_Add your docker hub credentials to the environment variables in Travis CI (and let them be secured). Secured variables, why ?_
Ces variables sont sécurisées pour éviter de transmettre des informations sensibles (compte, mot de passe, etc.) qui pourraient se trouver en clair dans `.travis.yml`.
<br>
Pour cela, on va sur notre repo sur Travis CI, dans Settings, et on peut les ajouter. J'ai donc choisi de les nommer DOCKERHUB_USER et DOCKERHUB_PWD (bien entendu, on peut choisir ce que l'on veut, du moment que cela correspond à ce que l'on va mettre dans notre .travis.yml (avec $DOCKERHUB_USER)).
<br>
Pour switcher de branche avec Git Bash, on fait `git pull` (si la branche a été créée depuis GitHub) puis `git checkout develop` pour aller dessus.
<br>
Après que l'on a bien créer nos repo pour les deux images docker, on commit notre nouveau script :
```yml
git:
  depth: 5

stages:
  - "Build and Test"
  - "Package"

jobs:
  include:
    - stage: "Build and Test"
      language: java
      jdk: oraclejdk11
      before_script:
        - cd sample-application-backend
      script:
        - echo "Maven build"
        - mvn clean install
        - echo "Run test coverage and Quality Gate"
        - mvn test
    - stage: "Build and Test"
      language: node.js
      node_js: "12.20"
      before_script:
        - cd sample-application-frontend
      script:
        - echo "NPM install and build"
        - npm install
        - npm run build
	- npm run lint
    - stage: "Package"
      before_script:
        - cd sample-application-backend
      script:
        - echo "Docker build ..."
        - docker build -t $DOCKERHUB_USER/backend .
        - echo "Docker login ..."
        - docker login -u $DOCKERHUB_USER -p $DOCKERHUB_PWD
        - echo "Docker push ..."
        - docker push $DOCKERHUB_USER/backend
    - stage: "Package"
      before_script:
        - cd sample-application-frontend
      script:
        - echo "Docker build ..."
        - docker build -t $DOCKERHUB_USER/frontend .
        - echo "Docker login ..."
        - docker login -u $DOCKERHUB_USER -p $DOCKERHUB_PWD
        - echo "Docker push ..."
        - docker push $DOCKERHUB_USER/frontend

cache:
  directories:
    - "$HOME/.m2/repository"
    - "$HOME/.npm"

services:
  - docker
```

Enfin, pour tester "à la main", on peut récupérer nos deux images et les lancer avec :
```
docker run -d --network app-network --name database -e POSTGRES_PASSWORD="takimapass" -e POSTGRES_USER="takima" -e POSTGRES_DB="SchoolOrganisation" postgres:12.0-alpine
docker run -d --network app-network --name frontend -p 80:80 vebervi/frontend
docker run -d --network app-network --name backend -e SPRING_DATASOURCE_URL="jdbc:postgresql://database:5432/SchoolOrganisation" vebervi/backend
```
En gros, ce que l'on fait, c'est de mettre sous forme de commandes le `docker-compose.yml` (puisque l'on a nos deux images). Ou alors, on peut carrément se recréer un `docker-compose.yml` dans un dossier différent :
```yml
version: '3.3'
services:
  backend:
    container_name: backend
    image: vebervi/backend
    environment:
      - SPRING_DATASOURCE_URL=jdbc:postgresql://database:5432/SchoolOrganisation
    networks:
      - app-network
    depends_on:
      - database

  database:
    container_name: database
    image: postgres:12.0-alpine
    networks:
      - app-network
    environment:
      - POSTGRES_PASSWORD=takimapass
      - POSTGRES_USER=takima
      - POSTGRES_DB=SchoolOrganisation

  frontend:
    container_name: frontend
    image: vebervi/frontend
    ports:
      - "80:80"
    networks:
      - app-network

volumes:
  my_db_volume:
    driver: local

networks:
  app-network:
```
Il ne reste plus qu'à le lancer (même pas à le builder puisque nous utilisons des images) avec `docker-compose up -d`. Et c'est beau !
<br>
_For what purpose (push on docker hub)?_
Pour y avoir accès depuis l'extérieur / versionner les images pour faciliter un roleback.

### Setup Quality Gate

Depuis SonarCloud, on nous propose différents moyens de lancer nos tests, notamment avec Travis CI. En cliquant dessus, SonarCloud nous donne une clé (que l'on rajoute dans Travis CI en variable d'environnement) et une ligne à rajouter dans le .travis.yml (`mvn clean org.jacoco:jacoco-maven-plugin:prepare-agent install sonar:sonar -Dsonar.projectKey=vebervi-cpe_sample-application-students` mais comme c'est pas terrible niveau lisibilité, on la split en trois : `mvn clean verify` `mvn org.jacoco:jacoco-maven-plugin:prepare-agent` `mvn sonar:sonar -Dsonar.projectKey=vebervi-cpe_sample-application-students`) ainsi que l'addon SonarCloud (les dernières lignes du fichier).
<br>
Ainsi, on obtient ce fichier :
```yml
git:
  depth: 5

stages:
  - "Build and Test"
  - "Package"

jobs:
  include:
    - stage: "Build and Test"
      language: java
      jdk: oraclejdk11
      before_script:
        - cd sample-application-backend
      script:
        - echo "Maven build & test"
        - mvn clean verify 
        - echo "Run test coverage"
        - mvn org.jacoco:jacoco-maven-plugin:prepare-agent
        - echo "Run Quality Gate"
        - mvn sonar:sonar -Dsonar.projectKey=vebervi-cpe_sample-application-students
    - stage: "Build and Test"
      language: node.js
      node_js: "12.20"
      before_script:
        - cd sample-application-frontend
      script:
        - echo "NPM install and build"
        - npm install
        - npm run build
        - npm run lint
    - stage: "Package"
      before_script:
        - cd sample-application-backend
      script:
        - echo "Docker build ..."
        - docker build -t $DOCKERHUB_USER/backend .
        - echo "Docker login ..."
        - docker login -u $DOCKERHUB_USER -p $DOCKERHUB_PWD
        - echo "Docker push ..."
        - docker push $DOCKERHUB_USER/backend
    - stage: "Package"
      before_script:
        - cd sample-application-frontend
      script:
        - echo "Docker build ..."
        - docker build -t $DOCKERHUB_USER/frontend .
        - echo "Docker login ..."
        - docker login -u $DOCKERHUB_USER -p $DOCKERHUB_PWD
        - echo "Docker push ..."
        - docker push $DOCKERHUB_USER/frontend

cache:
  directories:
    - "$HOME/.m2/repository"
    - "$HOME/.npm"

services:
  - docker

addons:
  sonarcloud:
    organization: "vebervi-cpe"
    token: "$SONARCLOUD_TOKEN"
```

## TP3 - Ansible

### Commandes
#### Récupérer des infos
ansible all -i inventories/setup.yml -m setup -a "filter=ansible_distribution*"
#### Initialiser un role
ansible-galaxy init roles/docker
Un role est une scene reutilisable dans un playbook
#### Lancer un playbook
ansible-playbook -i inventories/setup.yml playbook.yml

### Intro

Ansible : plate-forme logicielle libre pour la configuration et la gestion des ordinateurs via ssh (en mode push) (il faut que la machine qui l'utilise ait python).

#### Setup

Cette dépendance de l'application se trouve bien dans le fichier `pom.xml` du backend.
<br>
Cette configuration de Spring se trouve bien dans `src/main/resources/application.yml`.
<br>
Ce endpoint `/api/actuator` signifie au service que son API fonctionne.

#### Facts

Pour obtenir de nombreuses informations, on peut lancer la commande `ansible all -i inventories/setup.yml -m setup`.
Entre autre, on peut noter ces informations :
```
[...]
"ansible_architecture":"x86_64"
"ansible_bios_date":"08/24/2006"
"size": "8.00 GB" (taille de la partition)
"ansible_distribution": "CentOS"
"ansible_python_version": "2.7.5"
"Intel(R) Xeon(R) CPU E5-2676 v3 @ 2.40GHz" (processeur)
[...]
```

#### Advanced playbook

`$basearch` est une espèce de variable d'environnement qui retourne notre architecture (x86_64). Elle nous permet donc d'installer la version idoine de Docker. Par ailleurs, on peut lancer la commande `ansible-playbook -v -i inventories/setup.yml playbook.yml` avec `-v` pour avoir un peu plus d'informations sur l'exécution des commandes (verbose).

#### Using roles

On exécute la commande `ansible-galaxy init roles/docker` depuis notre dossier ansible pour créer notre rôle.
<br>
On rajoute cela dans notre playbook.yml pour utiliser notre rôle (en dessous de hosts) :
```
roles:
  - docker
```

### Deploy your app

Ainsi, on va avoir cinq rôles différent : 

- docker : installe docker et s'assure qu'il soit bien installé
- network : créé le réseau docker sur lequel les containers seront connectés
- database : lance un docker qui contiendra notre base de données
- app : lance un docker qui contiendra notre image vebervi/backend
- proxy : lance un docker qui contiendra notre image vebervi/frontend

On obtient ainsi les fichiers suivants :

```yml
# playbook.yml

- hosts: all
  gather_facts: false
  become: yes
  roles:
    - docker
    - network
    - database
    - app
    - proxy
```

```yml
# docker/tasks/main.yml

# Install Docker
- name: Install yum-utils
  yum:
    name: yum-utils
    state: latest

- name: Install device-mapper-persistent-data
  yum:
    name: device-mapper-persistent-data
    state: latest

- name: Install lvm2
  yum:
    name: lvm2
    state: latest

- name: Add Docker stable repository
  yum_repository:
    name: docker-ce
    description: Docker CE Stable - $basearch
    baseurl: https://download.docker.com/linux/centos/7/$basearch/stable
    state: present
    enabled: yes
    gpgcheck: yes
    gpgkey: https://download.docker.com/linux/centos/gpg

- name: Install Docker
  yum:
    name: docker-ce
    state: present

# Install tools needed by Ansible to run Docker
- name: Install epel-release
  yum:
    name: epel-release
    state: latest

- name: Install python-pip
  yum:
    name: python-pip
    state: latest

- name: Install python setup tools
  yum: name=python-setuptools
  tags: docker

- name: Install Pypi
  easy_install: name=pip
  tags: docker

- name: Install docker-py
  pip: name=docker-py
  tags: docker

- name: Make sure Docker is running
  service: name=docker state=started
  tags: docker
```

```yml
# network/tasks/main.yml

- name: Create network
  docker_network:
    name: app-network

# Sa tâche est de lancer cette commande :
# docker network create app-network
```

```yml
# database/tasks/main.yml

- name: Launch database
  docker_container:
    name: database
    state: started
    restart_policy: always
    image: postgres:12.0-alpine
    pull: true
    detach: yes
    networks:
      - name: app-network
    env:
      POSTGRES_PASSWORD: takimapass
      POSTGRES_USER: takima
      POSTGRES_DB: SchoolOrganisation

# Sa tâche est de lancer cette commande :
# docker run -d --network app-network --name database -e POSTGRES_PASSWORD="takimapass" -e POSTGRES_USER="takima" -e POSTGRES_DB="SchoolOrganisation" postgres:12.0-alpine
```

```yml
# app/tasks/main.yml

- name: Launch app
  docker_container:
    name: backend
    state: started
    restart_policy: always
    image: vebervi/backend
    pull: true
    detach: yes
    networks:
      - name: app-network
    env:
      SPRING_DATASOURCE_URL: jdbc:postgresql://database:5432/SchoolOrganisation

# Sa tâche est de lancer cette commande :
# docker run -d --network app-network --name backend -e SPRING_DATASOURCE_URL="jdbc:postgresql://database:5432/SchoolOrganisation" vebervi/backend
```

```yml
# proxy/tasks/main.yml

- name: Launch proxy
  docker_container:
    name: frontend
    state: started
    restart_policy: always
    image: vebervi/frontend
    pull: true
    detach: yes
    ports:
      - "80:80"
    networks:
      - name: app-network

# Sa tâche est de lancer cette commande :
# docker run -d --network app-network --name frontend -p 80:80 vebervi/frontend
```

(de toutes façons, on pourra retrouver tous mes fichiers sur mon GitHub : https://github.com/vebervi-cpe/sample-application-students)

### Continuous Deployment

Comme on a besoin du chemin de notre clé SSH dans l'inventaire de notre ansible (mais comme on ne devrait pas commit de clé pour raison de sécurité), j'ai pensé à la mettre dans une variable d'environnement de Travis CI puis, avant de lancer le déploiement, on créer notre fichier de clé avec la variable dedans, on déploie, puis on supprime ce fichier de clé.
<br>
Ainsi, notre clé n'est pas stocké sur le Git ni sur le serveur mais uniquement stockée au niveau de Travis CI.
<br>
Par conséquent, on peut mettre en commentaire la ligne `ansible_ssh_private_key_file:` de notre ansible/inventories/setup.yml (car on va utiliser la position "par défaut" de notre clé, c'est à dire `~/.ssh/id_rsa`) et rajouter un stage dans .travis.yml :

```yml
    - stage: "Deployment"
      before_script:
        - cd ansible
        - sudo apt-add-repository --yes --update ppa:ansible/ansible
        - sudo apt-get install ansible
      script:
        - echo "Create temp file id_rsa"
        - echo $SSH_KEY >> ~/.ssh/id_rsa
        - echo "Change rights to id_rsa"
        - chmod 600 ~/.ssh/id_rsa
        - echo "Ansible playbook ..."
        - ansible-playbook -v -i inventories/setup.yml playbook.yml
        - echo "Remove temp file id_rsa"
        - rm ~/.ssh/id_rsa
```

On y est presque ! Sauf que qu'au moment de lancer la commande `ansible-playbook ...`, on nous demande la passphrase de notre clé... Donc on s'arrête là, si près du but...
