KOUKO JOSHUA EZECHIEL
ETUDIANT EN LICENCE 3 DE SCIENCE INFORMATIQUE

Nous vous présentons le rapport que nous avons élaboré dans le cadre de notre TP sur Hadoop.


# TP Hadoop 3.3.6 — Installation, configuration et exercices MapReduce

> **Environnement :** Windows 11 + Multipass 1.17.0, machine virtuelle Ubuntu 24.04 LTS, 2 CPU, 4 Go RAM, disque 20 Go.  
> **Compte Linux :** `hadoop` — **Java :** OpenJDK 11 — **Hadoop :** 3.3.6.  
> **Mode :** pseudo-distribué (HDFS et YARN sont lancés sur une seule machine).

Ce document rassemble les étapes du TP, les commandes, leur rôle, les sorties réellement communiquées dans les échanges et les réponses aux questions identifiables. Les sorties précédées de **« sortie observée »** ont été communiquées depuis le terminal. Les résultats marqués **« résultat attendu »** sont des résultats de référence et ne doivent pas être présentés comme une capture réellement obtenue.

---

## 1. Préparation de l'environnement

### 1.1 Création de la machine virtuelle

Commande exécutée depuis Windows :

```bash
multipass launch 24.04 --name hadoop --cpus 2 --memory 4G --disk 20G
```

**Rôle des options :**
- `launch 24.04` : crée une VM Ubuntu 24.04.
- `--name hadoop` : nomme la VM `hadoop`.
- `--cpus 2` : attribue 2 processeurs virtuels.
- `--memory 4G` : attribue 4 Go de mémoire vive.
- `--disk 20G` : réserve un disque de 20 Go.

**Sortie observée :**
```text
Launched: hadoop
```

Entrée dans la VM :

```bash
multipass shell hadoop
```

Passage au compte Hadoop :

```bash
su - hadoop
```

Vérification de l'utilisateur :

```bash
whoami
```

Résultat attendu :
```text
hadoop
```

### 1.2 Mise à jour et installation des dépendances

```bash
sudo apt update && sudo apt upgrade -y
```

- `apt update` actualise la liste des paquets disponibles.
- `apt upgrade -y` installe les mises à jour disponibles.
- `sudo` exécute la commande avec les privilèges administrateur.
- `-y` valide automatiquement les confirmations.

Installation de Java, SSH et des outils :

```bash
sudo apt install -y openjdk-11-jdk ssh pdsh wget python3
```

Vérification de Java :

```bash
java -version
```

La machine a été configurée avec **OpenJDK 11.0.33**.

### 1.3 Vérification de Hadoop

```bash
hadoop version
```

Cette commande affiche la version installée. La version utilisée dans le TP est **Hadoop 3.3.6**.

Autres vérifications de l'environnement :

```bash
echo $JAVA_HOME
echo $HADOOP_HOME
```

Ces commandes affichent les chemins des variables d'environnement Java et Hadoop.

---

## 2. Configuration pseudo-distribuée

Hadoop utilise plusieurs fichiers XML pour définir le fonctionnement de ses composants.

### 2.1 `core-site.xml`

Fichier : `$HADOOP_HOME/etc/hadoop/core-site.xml`

Paramètres essentiels utilisés :

```xml
<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://localhost:9000</value>
    </property>
    <property>
        <name>hadoop.tmp.dir</name>
        <value>/home/hadoop/hadoopdata/tmp</value>
    </property>
</configuration>
```

- `fs.defaultFS` définit le système de fichiers par défaut : HDFS sur `localhost:9000`.
- `hadoop.tmp.dir` définit le répertoire temporaire utilisé par Hadoop.

### 2.2 `hdfs-site.xml`

Le paramètre essentiel du TP est le facteur de réplication :

```xml
<configuration>
    <property>
        <name>dfs.replication</name>
        <value>1</value>
    </property>
</configuration>
```

`dfs.replication=1` signifie qu'une seule copie de chaque bloc est conservée. Ce choix est adapté à une VM unique ; un cluster de production utilise généralement plusieurs copies réparties sur différents DataNodes.

### 2.3 `mapred-site.xml`

Ce fichier configure l'exécution des traitements MapReduce. Dans le mode utilisé, le framework est configuré pour fonctionner avec YARN :

```xml
<configuration>
    <property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
    </property>
</configuration>
```

### 2.4 `yarn-site.xml`

Ce fichier configure les services YARN, notamment le ResourceManager et le NodeManager. Le paramètre de configuration du service auxiliaire MapReduce est :

```xml
<configuration>
    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>
</configuration>
```

### 2.5 Formatage et démarrage des services

> Le formatage du NameNode s'effectue lors de l'initialisation du système de fichiers, et non à chaque démarrage. Ne pas le répéter sur un HDFS contenant des données à conserver.

```bash
hdfs namenode -format
```

Démarrage de HDFS :

```bash
start-dfs.sh
```

Démarrage de YARN :

```bash
start-yarn.sh
```

Vérification des processus :

```bash
jps
```

**Sortie observée à un moment du TP :**
```text
1569 DataNode
1443 NameNode
1751 SecondaryNameNode
6315 ResourceManager
6494 NodeManager
6881 Jps
```

Une autre vérification, avant le démarrage de YARN, avait affiché :
```text
1569 DataNode
1443 NameNode
1751 SecondaryNameNode
6216 Jps
```

Interprétation :
- `NameNode` : gère les métadonnées du système HDFS.
- `DataNode` : stocke les blocs de données.
- `SecondaryNameNode` : réalise périodiquement des points de contrôle des métadonnées ; ce n'est pas une sauvegarde de secours du NameNode.
- `ResourceManager` : gère les ressources du cluster YARN.
- `NodeManager` : gère les ressources et conteneurs sur un nœud.
- `Jps` : outil Java qui liste les processus Java actifs.

Arrêt des services :

```bash
stop-yarn.sh
stop-dfs.sh
```

Une sortie antérieure indiquait que le NodeManager ne s'était pas arrêté proprement après 5 secondes et que Hadoop avait tenté de le terminer avec `kill -9`. Il faut alors vérifier les processus avec `jps` avant de relancer les services.

---

## 3. Exercice 1 — Premiers pas avec HDFS

HDFS (*Hadoop Distributed File System*) est le système de fichiers distribué de Hadoop. Il stocke les fichiers sous forme de blocs gérés par le NameNode et les DataNodes.

Commandes HDFS courantes :

```bash
hdfs dfs -mkdir -p /user/hadoop/tp/input
```
Crée le répertoire HDFS `input`. `-p` crée aussi les répertoires parents si nécessaire.

```bash
hdfs dfs -put poeme.txt /user/hadoop/tp/input/
```
Copie le fichier local `poeme.txt` vers HDFS.

```bash
hdfs dfs -ls /user/hadoop/tp/input
```
Affiche le contenu du répertoire HDFS.

```bash
hdfs dfs -cat /user/hadoop/tp/input/poeme.txt
```
Affiche le contenu du fichier stocké dans HDFS.

```bash
hdfs dfs -get /user/hadoop/tp/input/poeme.txt .
```
Récupère le fichier HDFS dans le répertoire local courant.

### Réponses aux questions

**1) Combien de blocs occupe `poeme.txt` ? Pourquoi ?**

Le fichier est très petit par rapport à la taille de bloc HDFS par défaut (128 Mio dans la configuration Hadoop habituelle). Il occupe donc **un seul bloc**. HDFS ne découpe pas un petit fichier en plusieurs blocs de taille fixe : le dernier bloc peut être partiellement rempli.

**2) Comment retrouver le fichier dans l'interface Web du NameNode ?**

Ouvrir l'interface du NameNode dans un navigateur à l'adresse :

```text
http://localhost:9870
```

Puis choisir **Utilities → Browse the file system** et naviguer vers :

```text
/user/hadoop/tp/input/
```

Le port 9870 est le port Web habituel du NameNode dans cette configuration.

**3) Nombre de blocs pour un fichier de 1 Go et espace disque avec une réplication de 3 ?**

Avec une taille de bloc de 128 Mio et en prenant 1 Go = 1024 Mio :

- Nombre de blocs : `ceil(1024 / 128) = 8 blocs`.
- Avec un facteur de réplication de 3 : `8 × 3 = 24` copies de blocs.
- Espace de stockage total : environ **3 Go** pour un fichier de 1 Go, hors métadonnées et petits écarts de mesure.

---

## 4. Exercice 2 — WordCount avec MapReduce

### Objectif

Compter le nombre d'occurrences de chaque mot d'un texte.

Le traitement MapReduce se déroule en trois étapes :
1. **Map** : transforme chaque mot en paire `(mot, 1)`.
2. **Shuffle/Sort** : regroupe les valeurs associées à une même clé.
3. **Reduce** : additionne les valeurs pour chaque mot.

### 4.1 Test local du Mapper

Commande exécutée :

```bash
cat poeme.txt | ./mapper.py
```

**Sortie observée :**
```text
le      1
chat    1
dort    1
sur     1
le      1
tapis   1
le      1
chien   1
court   1
dans    1
le      1
jardin  1
le      1
chat    1
court   1
apres   1
le      1
chien   1
```

Le Mapper émet une paire clé-valeur pour chaque mot.

### 4.2 Simulation du Shuffle/Sort

```bash
cat poeme.txt | ./mapper.py | sort
```

**Sortie observée :**
```text
apres   1
chat    1
chat    1
chien   1
chien   1
court   1
court   1
dans    1
dort    1
jardin  1
le      1
le      1
le      1
le      1
le      1
le      1
sur     1
tapis   1
```

`sort` trie les lignes et place ensemble les clés identiques. Dans un vrai job Hadoop, le framework réalise le partitionnement, le shuffle et le tri.

### 4.3 Test local du Reducer

```bash
cat poeme.txt | ./mapper.py | sort | ./reducer.py
```

**Sortie observée :**
```text
apres   1
chat    2
chien   2
court   2
dans    1
dort    1
jardin  1
le      6
sur     1
tapis   1
```

### 4.4 Exécution avec Hadoop Streaming

Le principe de la commande Hadoop Streaming utilisée est d'envoyer les scripts Python au job et de définir les programmes Mapper et Reducer :

```bash
hadoop jar $HADOOP_HOME/share/hadoop/tools/lib/hadoop-streaming-3.3.6.jar \
-files mapper.py,reducer.py \
-mapper "python3 mapper.py" \
-reducer "python3 reducer.py" \
-input /user/hadoop/tp/input \
-output /user/hadoop/tp/output_wordcount
```

- `hadoop jar` lance un programme Java empaqueté dans un fichier JAR.
- Le JAR `hadoop-streaming-3.3.6.jar` permet d'utiliser des exécutables comme Mapper et Reducer.
- `-files` distribue les scripts aux tâches.
- `-mapper` désigne le programme Mapper.
- `-reducer` désigne le programme Reducer.
- `-input` indique le chemin d'entrée HDFS.
- `-output` indique le chemin de sortie HDFS. Le répertoire de sortie ne doit généralement pas déjà exister.

Lecture du résultat :

```bash
hdfs dfs -cat /user/hadoop/tp/output_wordcount/part-*
```

**Incident réellement rencontré :** une vérification du dossier de sortie a renvoyé `Connection refused` sur `localhost:9000`. Cela indique que le client ne parvenait pas à joindre le NameNode à cette adresse ; il fallait vérifier le démarrage de HDFS et les processus avec `jps`.

---

## 5. Exercice 3 — Chiffre d'affaires par ville

### Objectif

À partir d'un fichier de ventes, calculer pour chaque ville :
- le chiffre d'affaires total ;
- le nombre de ventes.

Le Mapper extrait la ville et le montant de chaque ligne. Le Reducer additionne les montants et compte les lignes par ville.

### Test local

Commande exécutée :

```bash
cat ventes.csv | python3 mapper_ventes.py | sort | python3 reducer_ventes.py
```

**Sortie observée :**
```text
Abidjan         910000  3
Bouake          445000  2
Korhogo         40000   1
San-Pedro       260000  2
Yamoussoukro    540000  2
```

### Interprétation

| Ville | Chiffre d'affaires total (FCFA) | Nombre de ventes |
|---|---:|---:|
| Abidjan | 910 000 | 3 |
| Bouaké | 445 000 | 2 |
| Korhogo | 40 000 | 1 |
| San-Pédro | 260 000 | 2 |
| Yamoussoukro | 540 000 | 2 |
| **Total** | **2 195 000** | **10** |

Contrôle : les nombres de ventes totalisent bien 10 lignes de ventes.

---

## 6. Exercice 4 — Traitement des données météorologiques

### Objectif

Déterminer la température maximale pour chaque année à partir du fichier `meteo.txt`.

Le Mapper associe chaque année à une température. Le Reducer conserve la température maximale pour chaque année.

### Présence du fichier dans HDFS

Commande exécutée :

```bash
hdfs dfs -ls /user/hadoop/tp/input_meteo
```

**Sortie observée :**
```text
Found 1 items
-rw-r--r--   1 hadoop supergroup         84 2026-11-21 20:21 /user/hadoop/tp/input_meteo/meteo.txt
```

### Exécution du job

Le job Hadoop Streaming utilise les scripts Mapper et Reducer météorologiques, le répertoire d'entrée HDFS `/user/hadoop/tp/input_meteo` et un répertoire de sortie distinct.

Lecture du résultat :

```bash
hdfs dfs -cat /user/hadoop/tp/output_meteo/part-*
```

**Résultat de référence associé aux données de l'exercice :**
```text
2024    35.0
2025    36.1
```

Le job communiqué comme terminé avec succès portait l'identifiant `job_1795295683414_0001`. Son suivi indiquait 100 % pour Map et Reduce, avec 6 enregistrements en entrée et 2 en sortie.

Interprétation : la température maximale est de 35,0 pour 2024 et de 36,1 pour 2025, dans l'unité indiquée par le jeu de données.

---

## 7. Exercice 5 — Moyenne des notes par matière

### Objectif

Calculer la moyenne des notes pour chaque matière.

Le Mapper extrait la matière et la note. Le Reducer calcule la somme des notes et la divise par le nombre de notes de la matière.

Le fichier d'entrée a été envoyé dans HDFS sous le répertoire :

```text
/user/hadoop/tp/input_notes
```

Vérification du fichier :

```bash
hdfs dfs -ls /user/hadoop/tp/input_notes
```

Lecture du résultat :

```bash
hdfs dfs -cat /user/hadoop/tp/output_notes/part-*
```

**Sortie observée :**
```text
Informatique    14.00
Maths           14.33
```

Interprétation : la moyenne des notes en Informatique est de 14,00 et celle en Maths est de 14,33.

---

## 8. Exercice 6 — Création d'un index inversé

### Objectif

Créer un index qui associe chaque mot aux fichiers dans lesquels il apparaît.

Exemple conceptuel :
- le Mapper lit les documents et émet `(mot, nom_du_fichier)`.
- le Shuffle/Sort regroupe les noms de fichiers par mot.
- le Reducer élimine les doublons éventuels et rassemble les noms de fichiers.

Lecture du résultat HDFS :

```bash
hdfs dfs -cat /user/hadoop/tp/output_index/part-*
```

**Sortie observée :**
```text
distribue       fichier2.txt
est             fichier1.txt,fichier2.txt,fichier3.txt
hadoop          fichier1.txt,fichier2.txt
informatique    fichier1.txt,fichier2.txt
langage         fichier1.txt,fichier2.txt,fichier3.txt
```

Interprétation : le mot `hadoop` apparaît dans `fichier1.txt` et `fichier2.txt`. Le mot `langage` apparaît dans les trois fichiers. L'index inversé permet donc de retrouver rapidement les documents contenant un terme donné.

---

## 9. Commandes de diagnostic et d'administration

| Commande | Rôle |
|---|---|
| `jps` | Affiche les processus Java actifs. |
| `start-dfs.sh` | Démarre les services HDFS. |
| `start-yarn.sh` | Démarre les services YARN. |
| `stop-yarn.sh` | Arrête YARN. |
| `stop-dfs.sh` | Arrête HDFS. |
| `hdfs dfs -ls CHEMIN` | Liste le contenu d'un chemin HDFS. |
| `hdfs dfs -mkdir -p CHEMIN` | Crée un répertoire HDFS. |
| `hdfs dfs -put FICHIER CHEMIN` | Envoie un fichier local vers HDFS. |
| `hdfs dfs -cat FICHIER` | Affiche un fichier HDFS. |
| `hdfs dfs -rm -r CHEMIN` | Supprime un fichier ou répertoire HDFS. À utiliser avec prudence. |
| `hdfs dfs -get SOURCE DESTINATION` | Télécharge un fichier HDFS vers le système local. |

### Erreur `Connection refused`

Exemple observé lors de l'accès à HDFS :

```text
java.net.ConnectException: Connection refused
```

Signification : le client n'a pas réussi à établir une connexion au service NameNode configuré sur `localhost:9000`.

Vérifications à effectuer :

```bash
jps
```

Puis, si les services ne sont pas actifs :

```bash
start-dfs.sh
```

Et vérifier à nouveau :

```bash
jps
```


---

## 10. Questions de synthèse 

### Quelle est la différence entre HDFS et MapReduce ?

- **HDFS** stocke les données de manière distribuée.
- **MapReduce** traite les données en parallèle à travers les phases Map, Shuffle/Sort et Reduce.

### Quel est le rôle du NameNode ?

Il gère les métadonnées : arborescence, noms des fichiers, emplacements et informations sur les blocs. Les données elles-mêmes sont stockées par les DataNodes.

### Quel est le rôle du DataNode ?

Il stocke et sert les blocs de données HDFS.

### Quel est le rôle de YARN ?

YARN gère les ressources et l'exécution des applications dans le cluster.

### Pourquoi le répertoire de sortie d'un job MapReduce doit-il être nouveau ?

Hadoop refuse normalement d'écrire dans un répertoire de sortie qui existe déjà. Il faut choisir un nouveau chemin ou supprimer explicitement l'ancien résultat si l'on n'en a plus besoin.

### Pourquoi utiliser Python avec Hadoop Streaming ?

Hadoop Streaming permet d'utiliser des programmes qui lisent sur l'entrée standard et écrivent sur la sortie standard. Il est donc possible d'écrire Mapper et Reducer en Python sans les programmer en Java.

---

## 11. Bilan du TP

Ce TP a permis de mettre en place un environnement Hadoop pseudo-distribué et de manipuler HDFS, YARN et MapReduce. Les exercices ont couvert le comptage de mots, l'agrégation de ventes par ville, l'analyse de températures, le calcul de moyennes et la construction d'un index inversé.

La démarche de travail adoptée était progressive : une commande était exécutée, sa sortie était examinée, puis l'étape suivante était engagée. Cette méthode facilite le diagnostic des erreurs et permet de comprendre le rôle de chaque commande.

---

