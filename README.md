# TP DEVOPS DE PIERRE KHETTAL

## TP1

### Base Postgres

* Créationd'un docker
  
* Effectuer le build :  `sudo docker build -t pierre/tp1/api .`

* Lancer le docker : `sudo docker run -p 5000:5432 --network=app-network -e POSTGRES_DB="db" -e POSTGRES_USER="usr" -e POSTGRES_PASSWORD="pwd" -v /tp1/data:/var/lib/postgresql/data --name tp1 tp1/postgres`

* -e pour ne pas laisser les identifiants BD dans le Dockerfile

* On attache un volume au postgres grace a la commande `-v /tp1/data:/var/lib/postgresql/data` c'est pour assurer la persistence des données. Quand le docker est supprimé le volume contiendra les données de notre BD et sera chargé au lancement du docker.


* Docker file
```Dockerfile
# Image postgres
 FROM postgres:11.6-alpine

#  Copy des scripts sql
COPY CreateScheme.sql /docker-entrypoint-initdb.d
COPY InsertData.sql /docker-entrypoint-initdb.d
```


### Backend

#### Simple API
* Création d'un main.java
  
* Lancer le docker sudo docker run --network=app-network --name tp1-api pierre/tp1/api

```Dockerfile
# Build
# On donne un alias a notre JDK pour pouvoir le reutiliser dans la 2eme partie
# Le JDK permet de compiler le java
FROM maven:3.6.3-jdk-11 AS myapp-build
ENV MYAPP_HOME /opt/myapp
WORKDIR $MYAPP_HOME
# Copie du fichier de configuration
COPY pom.xml .
# Copie des fichiers sources à compiler
COPY src ./src
# Lancement du build avec Maven
RUN mvn package -DskipTests

# Run
# Importation d'une image plus simple pour que ça soit plus leger
FROM openjdk:11-jre
ENV MYAPP_HOME /opt/myapp
WORKDIR $MYAPP_HOME
# Creation du .jar grace à la premiere partie du Dockerfile
COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar
# Lancement du .jar
ENTRYPOINT java -jar myapp.jar
```
* Le multistage build nous permet d'utiliser des images differents avec l'instruction FROM. Ici on en utilise deux le JDK et le JRE.

* Build du docker :  `sudo docker build -t simple-api .`

* Lancer le docker : `sudo docker run -p 8080:8080 simple-api`

* Redirection du port du docker grace à `-p 8080:8080`

#### Back-api

* Recuperation de l'api sur git
* On lance le docker `sudo docker run -p 8080:8080 --network=app-network --name back-api back-api`
* L'api est fonctionnel, on recupere bien les données de notre BD avec la requete `http://localhost:8080/departments/IRC/students`

### Serveur apache

Mis en place du docker
```Dockerfile
FROM httpd:2.4
COPY index.html /usr/local/apache2/htdocs/

COPY httpd.conf /usr/local/apache2/conf/
```

#### Reverse proxy

* Modification de la configuration du serveur mis en place du reverse proxy

* Le reverse proxy permet de proteger en entrée notre application. Les clients ne peuvent plus interroger notre api directement.


### Docker compose

* Docker-compose est un manager de docker, il permet de gerer les dockers comme un ensemble de services inter-connectés.

```Dockerfile
version: '3.3'
services:
 back-api:
  build: /tp1/back-api
  networks: 
   - app-network
  depends_on:
   - postgres
 postgres:
  build: /tp1/postgres
  networks:
   - app-network
 httpd:
  build: /tp1/http
  ports: 
   - 80:80
  networks: 
   - app-network
  depends_on:
   - back-api
networks: 
 app-network:
 ```

 * Lancer le docker-compose : `docker-compose up`

##### Publish

* Pour se connecter à docker hub : `docker login`
* On tag l'image de la bd :`docker tag tp1_postgres pierrektl/tp1_postgres:1.0`
* Push de l'image sur docker hub : `docker push pierrektl/tp1_postgres`
* L'image a été publié sur docker hub, cela rend mon image accessible depuis d'autres postes


## TP2

### Continious integration

* `mvn clean verify` permet de build et de tester notre application. Cette commande necessite d'etre effectuer dans le dossier ou le pom.xml se trouve.
* Le fichier pom.xml definit les dependances et les tests.
  
* Les testcontainers sont des librairies java. Elles permettent de créer des dockers pour la réalisation des tests

```yml
name: CI devops 2022 CPE # Nom de la continious integration
on:
  push:
    branches: # List des branches de la CI
      - main
      - develop
    pull_request:
jobs:
  test-backend:
    runs-on: ubuntu-18.04 # On renseigne l'os
    steps:
      - uses: actions/checkout@v2.3.3 # On charge le repo
      - name: Set up JDK 11
        uses: actions/setup-java@v2 # On prépare le jdk pour pouvoir compiler avec maven
        with:
          java-version: '11'
          distribution: 'adopt' 
      - name: Build with Maven
        run: mvn clean verify
        working-directory: ./back-api/simple-api/ # On se place dans le repo de l'appli
```
 * On ajoute des variables sécurisées dans github actions pour ne pas faire trainer les login/mot de passe en clair dans des fichiers 

* Needs permet d'executer les taches l'une apres l'autre. Sans le need les tâches de build et de déploiement seront réalisées en parallèles

* Push une image Docker permet de la rendre accessible par d'autres machines.

### Quality gate
```yml
   - name: Build and test with Maven
     working-directory: ./back-api/simple-api/
     run: mvn -B verify sonar:sonar -Dsonar.projectKey=PierreKcpe_DevOps2 -Dsonar.organization=pierrekcpe -Dsonar.host.url=https://sonarcloud.io -Dsonar.login=${{ secrets.SONAR_TOKEN }} --file ./pom.xml  
```   

## TP3

### Commandes 

* Pour ping les hotes

sudo ansible all -i ansible/inventories/setup.yml -m ping

* setup.yml
```yml
all:
 vars:
  ansible_user: centos
  ansible_ssh_private_key_file: ./id_rsa
 children:
  prod:
   hosts: pierre.khettal.takima.cloud
```
sudo ansible-playbook -i ansible/inventories/setup.yml ansible/playbook.yml
