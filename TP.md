#  <u> Mise en place d'une Intégration Continue: </u>

Le projet que nous avons choisi de forker est le projet Java SimpleAstronomyLib qui est mis en place avec Maven.

On va tout d'abord créer un pipelne sur la plateforme de Jenkins, rajouter le répositoire actuel en paramètres, activer l'option "GitHub hook trigger for GITScm polling", puis dans la partie "Pipeline", on va selectionner l'option "Pipeline from SCM", dans l'onglet SCM, selectionner Git. Puis on va set un repository avec le lien github du proet, les options de connexion, et spécifier la branche de travail (ici on va utiliser */Master même si celà est déconseillé).

Le jenkins file initialement vide, on rajoutera des éléments au fur et à mesure des étapes.

##  1- <u> Build du code: </u>

Pour ce faire on a dû mettre en place la configuration de run Maven dans notre IDE et donner comme objectif de lifecycle "Deploy". Ensuite nous avons rajouté dans pom.xml le plugin apache-maven-deploy comme suit:

<code>

    <plugin>

            <groupId>org.apache.maven.plugins</groupId>

            <artifactId>maven-deploy-plugin</artifactId>

            <version>2.8.2</version>

            <configuration>

                <skip>true</skip>

            </configuration>

    </plugin> 

</code>

Puis nous avons effectué un <code> mvn clean install </code>, buildé et run le code
qui fonctionnait sans problèmes. 

Du côté du jenkinsfile, on va rajouter des stages dont un stage "Checkout", qui va permettre au Jenkknsfile de récupérer le repositoire github en ligne. 

Puis on va rajouter un stage "Get info from pom" qui va permettre de récupérer les informations liées au projet à partir du fichier pom.xml ce qui va rendre dynamique la récupération de nom d'artéfacts et de versions. Pour ce faire on va d'abord définir les variables suivantes

<code>def groupId = ''

def artefactId = ''

def filePath = ''

def packaging = ''

def version = ''</code>

Puis lire le fichier pom.xml dans le jenkinsfile et affecter les valeurs aux variables définies dans un script comme suit :

<code>script {

pom = readMavenPom file: 'pom.xml'

groupId = pom.groupId

artifactId = pom.artifactId

packaging = pom.packaging

version = pom.version

filepath = "target/${artifactId}-${version}.jar"

isSnapshot = version.endsWith("-SNAPSHOT")

}</code>

Après ces 2 stages on peut enfin rajouter celui du build. Dans ce stage on va juste rajouter la commande shell de package maven

<code> sh 'mvn clean package' </code>

## 2- <u> Génération d'un package: </u>

Ici on va rajouter le stage de build dans le Jenkinsfile. On va utiliser la commande shell

<code> sh 'mvn clean package' </code>

Cette commande va créer un nouveau package à chaque run du pipeline et nettoyer le précédent package.

## 3- <u> Exécution des Tests Unitaires et branchement d'un serveur Sonarqube: </u>

Ici les tests unitaires étant déjà rédigés, nos avons juste dû
configurer le fichier de configuration de test sous jUnit, désigné les obets de test, le niveau de langage java et le repertoire de travail.

Dans le jenkinsfile on va rajouter un stage de test et run les commandes de tests pour jUnit comme suit:

<code>sh 'make check || true' 

junit '**/target/*.xml'</code>

la commande shell <code>make check || true</code> va retourner 1 pour toutes erreurs de tests unitaires de la commande suivante <code> junit '**/target/*.xml' </code>  .

Pour le branchement du serveur Sonarqube, on va rajouter l'extension 'Sonarqube platform' dans l'onglet Administer Jenkins -> Gestion des Plugins -> Plugins Disponibles  sur la plateforme Jenkins. Puis sur la plateforme Jenkins, on va selectionner Administrer Jenkins -> Configuration de Système et cliquer sur le bouton "Ajouter Sonarqube" sur l'onglet Sonarqube nouvellement créé et ajouter les informations de serveur demandées. 

Ensuite rajouter ce code dans le stage "Test" du Jenkinsfile

<code>def scannerHome = tool 'SonarScanner 4.0';

withSonarQubeEnv('My SonarQube Server') {

sh "${scannerHome}/bin/sonar-scanner"
</code>

Ce code va rajouter l'outil de scanner Sonar à partir du serveur Sonarqube rajouté et run les tests grâce à la commande shell de la dernière ligne.

## 4- <u> Publication du Package sur Nexus </u>

Dans cette étape on va configurer un serveur Nexus et y rajouter une étape de publiication dans le JenkinsFile. On va d'abord télécharger la dernière relase de Nexus et configurer le serveur, puis télécharger le plugin sur la plateforme jenkins. Puis on va configurer Nexus dans Administrer Jenkins -> Configuration de Système, copier toute la configuration qu'on a vue dans l'application Nexus. 

Puis on va aller dans le fichier pom.xml et rajouter le plugin de nexus, revenir dans le jenkinsfile et raouter deux stages. Un "Push Snapshot to nexus" qui va exécuter la commande shell suivante 

<code> sh "mvn deploy:deploy-file -e -DgroupId=${groupId} -Dversion=${version} -Dpackaging=${packaging} -Durl=${nexusUrl}/repository/${nexusRepoSnapshot} </code>

Cette commande ne sera exécutée que si le build est un snapshot.

Et une autre étape "Push Release to Nexus" qui elle va exécuter la commande shell suivante:

<code>nexusPublisher nexusInstanceId: 'nexus_localhost', nexusRepositoryId: "${nexusRepoRelease}", packages: [[$class: 'MavenPackage', mavenAssetList: [[classifier: '', extension: '', filePath: "${filepath}"]], mavenCoordinate: [artifactId: "${artifactId}", groupId: "${groupId}", packaging: "${packaging}", version: "${version}"]]]</code>

Cette commande ne sera exécutée que si le build est un release.

## 5- <u> Packaging sous Docker </u>



## 6- <u> Publication d'une Image sur un repo </u>



## 7- <u> Déploiement de la release </u>