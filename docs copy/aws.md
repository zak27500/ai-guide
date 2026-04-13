# AWS

---

## I. Quel service de calcul choisir pour un trafic imprévisible ?

**Question** : "Nous développons une application qui doit traiter des images envoyées par les utilisateurs. Le volume est imprévisible : parfois 0 image par heure, parfois 10 000. Quelle solution de calcul choisirais-tu entre Amazon EC2 et AWS Lambda, et pourquoi ?"

**Réponse** :

Pour un volume de requêtes imprévisible, je choisirais AWS Lambda pour trois raisons.

**Scalabilité automatique** : Lambda gère nativement la montée en charge. Si 10 000 images arrivent simultanément, AWS lance 10 000 instances de la fonction en parallèle sans aucune gestion de serveur.

**Optimisation des coûts (pay-as-you-go)** : avec Lambda, on ne paye rien quand il n'y a pas de trafic. Sur EC2, on paierait pour une machine qui tourne dans le vide.

**Maintenance réduite** : c'est du NoOps — on se concentre sur le code de traitement d'image, pas sur la mise à jour de l'OS.

EC2 reste pertinent pour des charges prévisibles et constantes, des traitements longs dépassant les 15 minutes (limite Lambda), ou lorsqu'on a besoin d'un environnement d'exécution très personnalisé.

---

## II. Comment optimiser le coût de stockage de millions d'images ?

**Question** : "On veut stocker des millions d'images de manière sécurisée et durable, mais certaines ne seront consultées qu'une fois par an pour de l'archivage légal. Quel service et quelle classe de stockage utiliserais-tu pour optimiser la facture ?"

**Réponse** :

Le service de stockage d'objets phare d'AWS est Amazon S3 (Simple Storage Service). Il est virtuellement illimité et permet d'optimiser les coûts via des classes de stockage selon la fréquence d'accès.

**S3 Standard** : données consultées fréquemment, accès immédiat, durabilité 99.999999999%.

**S3 Standard-IA (Infrequent Access)** : données moins consultées mais devant rester disponibles rapidement. Moins cher que Standard, avec un coût de récupération.

**S3 Glacier Flexible Retrieval** : archivage. Stockage très peu coûteux, récupération de quelques minutes à quelques heures.

**S3 Glacier Deep Archive** : la solution la moins chère de tout AWS pour des données consultées une fois par an, récupération en 12 à 48 heures.

En complément, S3 Intelligent-Tiering déplace automatiquement les objets entre classes selon leur fréquence d'accès réelle — idéal quand le pattern de consultation est incertain.

---

## III. RDS ou DynamoDB pour un site e-commerce ?

**Question** : "Si je construis un site e-commerce classique avec des relations complexes entre les clients, les commandes et les produits, recommanderais-tu Amazon RDS ou Amazon DynamoDB ?"

**Réponse** :

Pour une application avec des relations complexes et un besoin de transactions ACID, je recommanderais Amazon RDS.

**Gestion des relations** : RDS utilise des moteurs SQL familiers (PostgreSQL, MySQL) qui permettent des jointures complexes entre plusieurs tables — clients, commandes, produits — avec des contraintes d'intégrité référentielle.

**Service géré** : AWS gère les sauvegardes automatiques, les correctifs de sécurité et la haute disponibilité via Multi-AZ, tout en laissant le contrôle sur le schéma.

**Quand choisir DynamoDB** : je réserverais DynamoDB pour des fonctionnalités nécessitant une performance constante à très grande échelle avec des accès simples par clé — gestion des sessions utilisateurs, suivi en temps réel, catalogue de produits sans jointures. DynamoDB offre une latence de l'ordre de la milliseconde quelle que soit la taille de la table.

---

## IV. Comment appliquer le principe du moindre privilège avec IAM ?

**Question** : "Comment appliquez-vous le principe du moindre privilège lorsqu'une de vos fonctions Lambda doit lire un fichier spécifique dans un bucket S3 ?"

**Réponse** :

Pour permettre à une fonction Lambda d'accéder à S3, je n'utilise jamais d'identifiants statiques (clés d'accès hardcodées). J'utilise un Rôle IAM.

**Le Rôle d'Exécution** : je crée un rôle spécifique que la fonction Lambda "endosse". Ce rôle contient une Politique IAM (JSON) qui définit précisément ce que la fonction a le droit de faire.

**Le Moindre Privilège** : au lieu de donner un accès `S3FullAccess`, ma politique limitera l'accès à une seule action (`s3:GetObject`) sur un seul bucket spécifique identifié par son ARN.

**Sécurité dynamique** : AWS génère automatiquement des jetons de sécurité temporaires pour la fonction via le service STS (Security Token Service). Il n'y a aucun mot de passe ou clé à gérer, et les credentials expirent automatiquement.

La hiérarchie IAM à retenir : **Compte AWS** → **Utilisateurs / Groupes** (personnes humaines) / **Rôles** (services AWS et applications) → **Politiques** (règles JSON `Allow`/`Deny`).

---

## V. Comment isoler une base de données sensible dans un VPC ?

**Question** : "On a une base de données RDS contenant des données clients sensibles. Doit-on la placer dans un Sous-réseau Public ou Privé, et comment l'application peut-elle y accéder ?"

**Réponse** :

Je place systématiquement la base de données RDS dans un **sous-réseau privé** — sans route vers Internet Gateway, donc inaccessible depuis Internet.

**Accès via Security Groups** : je crée une règle dans le Security Group de la base qui autorise uniquement le trafic entrant provenant du Security Group spécifique du serveur d'application. L'autorisation est basée sur l'identité de la source, pas son IP.

**Connectivité privée** : l'application communique avec la base via le réseau interne d'AWS (backbone privé). Aucune donnée ne transite par Internet public.

**Accès maintenance** : si j'ai besoin d'accéder moi-même à la base pour de la maintenance, j'utilise un Bastion Host (serveur de saut dans le sous-réseau public) ou AWS Systems Manager Session Manager (sans ouvrir de port SSH).

---

## VI. Bedrock ou SageMaker pour intégrer un LLM dans une application ?

**Question** : "Si nous voulons intégrer un modèle comme Claude 3 ou Llama 3 dans notre application sans gérer l'infrastructure ou l'entraînement, quel service AWS utiliser ?"

**Réponse** :

Pour intégrer des modèles de fondation rapidement et sans gérer d'infrastructure, je choisirais Amazon Bedrock.

**Serverless API-first** : Bedrock est entièrement géré. On appelle le modèle via une API unifiée (similaire à l'API OpenAI), sans serveur à configurer ni GPU à louer. C'est parfait pour des architectures d'agents construits avec LangGraph.

**Multi-modèles** : Bedrock donne accès à plusieurs fournisseurs — Anthropic (Claude), Meta (Llama), Mistral, Cohere, Amazon Titan — depuis une seule API, ce qui évite le vendor lock-in.

**Souveraineté des données** : les données envoyées à Bedrock ne sont jamais utilisées pour entraîner les modèles de base — garantie contractuelle importante pour les entreprises soumises au RGPD.

**Quand choisir SageMaker** : uniquement si on a besoin d'entraîner un modèle propriétaire à partir de zéro, de faire du Fine-Tuning avec contrôle total sur l'infrastructure GPU, ou de déployer ses propres endpoints d'inférence personnalisés.

---

## VII. Comment garantir la disponibilité si un Data Center tombe en panne ?

**Question** : "Comment s'assure-t-on que notre application reste disponible même si un centre de données entier d'AWS tombe en panne ?"

**Réponse** :

Pour garantir la disponibilité face à une panne physique, j'utilise une architecture Multi-AZ.

**Définition d'une AZ** : une Zone de Disponibilité est composée d'un ou plusieurs centres de données physiquement isolés les uns des autres au sein d'une même Région, avec leur propre alimentation électrique, réseau et système de refroidissement.

**RDS Multi-AZ** : AWS crée une instance RDS secondaire dans une autre AZ et réplique les données de façon synchrone. Si l'AZ principale tombe, le basculement (failover) est automatique et transparent — typiquement en moins de 60 secondes.

**Application Load Balancer** : pour le calcul (EC2 ou ECS), je distribue les instances sur plusieurs AZ derrière un ALB. Si une AZ devient indisponible, l'ALB redirige automatiquement tout le trafic vers les AZ saines.

À ne pas confondre : **Multi-AZ** = haute disponibilité dans une région. **Multi-Region** = disaster recovery à l'échelle géographique (latence plus basse pour les utilisateurs mondiaux, tolérance aux catastrophes régionales).

---

## VIII. Comment déclencher automatiquement une Lambda quand un fichier arrive sur S3 ?

**Question** : "Chaque fois qu'un utilisateur télécharge un fichier PDF sur S3, nous devons automatiquement extraire le texte et l'enregistrer dans une base de données. Comment metttre cela en place de la manière la plus efficace ?"

**Réponse** :

Je configurerais une **S3 Event Notification** sur le bucket pour déclencher une architecture Event-Driven entièrement serverless.

**Le déclencheur** : dès qu'un fichier est uploadé (événement `s3:ObjectCreated`), S3 envoie automatiquement un message contenant les métadonnées du fichier (nom, taille, emplacement).

**L'exécution** : ce message déclenche immédiatement une fonction Lambda. Pour absorber des pics de charge, on peut intercaler une file SQS entre S3 et Lambda — S3 publie dans SQS, Lambda consomme depuis SQS avec un batch contrôlé.

**Le traitement** : la Lambda récupère le fichier depuis S3, utilise Amazon Textract pour l'extraction de texte, et écrit le résultat dans DynamoDB.

**L'avantage** : architecture 100% serverless — on ne paye rien si aucun fichier n'est déposé, et le système peut traiter des milliers de fichiers en parallèle sans intervention manuelle.

---

## IX. Peut-on interroger un fichier S3 sans le télécharger entièrement ?

**Question** : "Si nous avons un fichier CSV de 10 Go stocké sur S3 et que nous ne voulons extraire que les lignes concernant l'année 2024, est-il obligatoire de télécharger tout le fichier dans notre code ?"

**Réponse** :

Non, grâce à **S3 Select** (et son équivalent pour Glacier, Glacier Select).

**Principe** : S3 Select permet d'exécuter des requêtes SQL simples directement sur S3, côté serveur. AWS filtre les données avant de les envoyer — seules les lignes correspondant au filtre sont transférées.

**Avantage concret** : pour un CSV de 10 Go dont seulement 200 Mo concernent 2024, on ne transfère que les 200 Mo utiles au lieu des 10 Go. Gain direct sur la bande passante, la mémoire Lambda consommée et donc le coût.

**Formats supportés** : CSV, JSON, Parquet (colonnes sélectives).

Pour des analyses encore plus complexes, Amazon Athena permet de lancer des requêtes SQL complètes sur des fichiers S3 sans aucun serveur — architecture Data Lake serverless.

---

## X. Comment monitorer et être alerté en cas d'erreur sur AWS ?

**Question** : "Comment faites-vous pour savoir qu'une erreur s'est produite, et où allez-vous pour lire les messages d'erreur générés par votre code Python dans le Cloud ?"

**Réponse** :

Le service central de surveillance sur AWS est **Amazon CloudWatch**, qui repose sur trois piliers.

**CloudWatch Logs** : centralise tous les journaux d'erreurs et les `print()` des fonctions Lambda ou des serveurs EC2. C'est là qu'on va pour débugger un crash — chaque invocation Lambda crée automatiquement un log stream.

**CloudWatch Metrics** : collecte des données chiffrées — taux d'utilisation CPU, nombre de requêtes par seconde, durée d'exécution Lambda, taux d'erreur.

**CloudWatch Alarms** : surveille une métrique et déclenche une action quand un seuil est dépassé (ex : taux d'erreur Lambda > 5%).

**Le circuit d'alerte** : l'alarme CloudWatch publie dans un Topic **Amazon SNS** (Simple Notification Service), qui distribue ensuite la notification à tous ses abonnés — email, SMS, Slack via Lambda, ou déclenchement d'une action automatique (ex : Auto Scaling).

---

## XI. Comment éviter de perdre des données lors d'un pic de charge ?

**Question** : "Imaginez que votre application reçoive des milliers de commandes par seconde, mais que votre base de données soit trop lente pour les enregistrer toutes instantanément. Quel service utiliseriez-vous pour ne perdre aucune donnée ?"

**Réponse** :

Pour absorber un pic de charge sans perdre de données, j'utiliserais **Amazon SQS** pour découpler les composants.

**Mise en file** : au lieu d'écrire directement dans la base, l'application place chaque message (commande) dans une file SQS. L'opération d'enqueue est quasi instantanée et n'est jamais bloquée par la vitesse de la base.

**Traitement asynchrone** : une Lambda (ou un serveur EC2/ECS) consomme les messages de la file à son rythme, avec contrôle du batch et de la concurrence.

**Résilience** : si la base est temporairement indisponible, les messages restent en sécurité dans SQS jusqu'à 14 jours. Une Dead Letter Queue (DLQ) capture les messages qui échouent plusieurs fois pour analyse ultérieure.

**SQS vs SNS** : SQS = file point-à-point (un seul consommateur traite chaque message). SNS = pub/sub (un message est distribué à tous les abonnés simultanément). En production on combine souvent les deux : SNS fan-out vers plusieurs files SQS.

---

## XII. Comment déployer la même infrastructure en Test et en Production ?

**Question** : "Si je vous demande de recréer exactement la même infrastructure (VPC, Lambda, S3, RDS) pour un environnement de Test et un environnement de Production, allez-vous tout refaire à la main dans la console AWS ?"

**Réponse** :

Non. Je n'utilise jamais la console pour provisionner des environnements. J'utilise l'**Infrastructure as Code** avec AWS CloudFormation ou Terraform.

**Automatisation et répétabilité** : on écrit un fichier de configuration (YAML pour CloudFormation, HCL pour Terraform) qui décrit toutes les ressources. Toute l'infrastructure se recrée en quelques minutes sans erreur humaine.

**Versionnage** : le code d'infrastructure est stocké sur Git. On voit l'historique des modifications, on peut rollback en cas de problème, et plusieurs personnes peuvent contribuer via des PR.

**Parité des environnements** : cela garantit que Test est le miroir exact de Production — ce qui évite les bugs spécifiques à un environnement et facilite la détection des régressions.

**CloudFormation vs Terraform** : CloudFormation est natif AWS (gratuit, bien intégré, gestion d'état automatique). Terraform est agnostique au cloud (multi-providers), plus flexible, mais nécessite de gérer le state file (souvent stocké dans S3 + DynamoDB pour le lock).

---

## XIII. Comment automatiser le déploiement d'une Lambda depuis GitHub ?

**Question** : "Une fois que ton code Python est poussé sur GitHub, comment fais-tu pour qu'il soit déployé automatiquement sur AWS sans cliquer manuellement dans la console ?"

**Réponse** :

Je mettrais en place un pipeline CI/CD avec la suite AWS CodeSuite ou GitHub Actions + AWS.

**CodePipeline** : c'est le chef d'orchestre. Dès qu'un développeur push sur la branche main, le pipeline se déclenche automatiquement via un webhook.

**CodeBuild** : le code est récupéré, les dépendances sont installées, les tests unitaires s'exécutent dans un environnement isolé. Si un test échoue, le déploiement s'arrête — la production est protégée.

**CodeDeploy / Lambda** : une fois les tests validés, le nouveau code est déployé. Pour Lambda, on peut utiliser des stratégies de déploiement progressif : Canary (10% du trafic sur la nouvelle version, puis 100% après validation) ou Linear (incrément progressif toutes les N minutes).

**Avec GitHub Actions** : alternative légère et flexible, on utilise `aws-actions/configure-aws-credentials` pour authentifier le pipeline via un Rôle IAM OIDC (sans stocker de clés statiques dans GitHub Secrets).

---

## Schémas récapitulatifs — Tout AWS en un coup d'oeil

---

### 1. Hiérarchie des services AWS — Compute, Storage, Database

**Ce que montre ce schéma** : la carte mentale des trois familles de services AWS les plus importantes. **Compute** : EC2 (serveurs persistants) vs Lambda (fonctions éphémères event-driven) vs ECS/EKS (conteneurs). **Storage** : S3 (objets, illimité) vs EBS (disque attaché à EC2) vs EFS (système de fichiers partagé). **Database** : RDS (SQL relationnel géré) vs DynamoDB (NoSQL clé-valeur, ms de latence) vs ElastiCache (cache Redis/Memcached en mémoire). La règle : choisir le service le plus spécialisé pour chaque besoin plutôt qu'un service généraliste surchargé.

```
                        COMPTE AWS
                            |
              +-------------+-------------+
              |             |             |
           COMPUTE       STORAGE      DATABASE
           -------       -------      --------
           EC2            S3           RDS
           (VM persistante)(objets)   (PostgreSQL/MySQL)
                |           |             |
           Lambda          EBS           DynamoDB
           (event-driven) (disque EC2)  (NoSQL, <10ms)
                |           |             |
           ECS / EKS       EFS          ElastiCache
           (conteneurs)  (NFS partagé) (Redis/Memcached)

           NETWORKING           MESSAGING         MONITORING
           ----------           ---------         ----------
           VPC                  SQS               CloudWatch
           (réseau isolé)       (file d'attente)  (logs+metrics)
                |                    |                  |
           ALB/NLB                  SNS               X-Ray
           (load balancing)         (pub/sub)         (tracing)
                |                    |
           Route 53             EventBridge
           (DNS)                (bus d'événements)
```

---

### 2. VPC — Isolation réseau et flux de trafic

**Ce que montre ce schéma** : comment structurer un réseau AWS sécurisé. Le VPC est un réseau privé virtuel dans une Région. Il contient deux types de sous-réseaux : **publics** (accessibles depuis Internet via Internet Gateway) pour les composants exposés (ALB, Bastion), et **privés** (sans route Internet) pour les ressources sensibles (RDS, Lambda, EC2 applicatif). Les Security Groups jouent le rôle de firewall stateful au niveau instance — la règle clé : le Security Group de RDS n'accepte que le trafic venant du Security Group de l'application, pas depuis Internet.

```
INTERNET
    |
    v
[Internet Gateway]
    |
    v
+-- VPC (10.0.0.0/16) -----------------------------------------+
|                                                                |
|  Sous-réseau PUBLIC (10.0.1.0/24)   AZ-1                      |
|  +------------------------------------------+                 |
|  | Application Load Balancer (ALB)          |                 |
|  | Bastion Host (accès SSH maintenance)     |                 |
|  +------------------------------------------+                 |
|              |                                                 |
|              | (trafic filtré par Security Group)              |
|              v                                                 |
|  Sous-réseau PRIVE (10.0.2.0/24)   AZ-1                       |
|  +------------------------------------------+                 |
|  | EC2 / Lambda / ECS (applicatif)          |                 |
|  | SG App: accepte 443 depuis ALB SG        |                 |
|  +------------------------------------------+                 |
|              |                                                 |
|  Sous-réseau PRIVE (10.0.3.0/24)   AZ-1                       |
|  +------------------------------------------+                 |
|  | RDS PostgreSQL                           |                 |
|  | SG DB: accepte 5432 depuis SG App ONLY   |                 |
|  +------------------------------------------+                 |
|                                                                |
|  [Meme structure miroir dans AZ-2 pour Multi-AZ]              |
+----------------------------------------------------------------+
```

---

### 3. Pipeline Event-Driven Serverless — S3 déclenche Lambda

**Ce que montre ce schéma** : le pattern le plus courant en architecture serverless AWS. Un événement sur S3 (upload de fichier) déclenche automatiquement une chaîne de traitement sans aucun serveur à gérer. L'ajout d'une file SQS entre S3 et Lambda est la bonne pratique de production : elle absorbe les pics (si 10 000 fichiers arrivent en même temps, SQS les met en file et Lambda les traite à son rythme), et la Dead Letter Queue (DLQ) capte les messages en échec pour investigation.

```
Utilisateur
    |
    | upload fichier
    v
[Amazon S3]
    |
    | s3:ObjectCreated Event
    v
[Amazon SQS]  <-- tampon qui absorbe les pics de charge
    |               Messages conservés jusqu'a 14 jours
    | poll (batch)
    v
[AWS Lambda]  (code Python / LangChain)
    |
    +----------> [Amazon Textract]  (extraction texte PDF)
    |
    +----------> [Amazon Bedrock]   (analyse IA, résumé)
    |
    +----------> [Amazon DynamoDB]  (stockage résultat)
    |
    +----------> [CloudWatch Logs]  (logs de l'exécution)
                      |
                      | si erreur > seuil
                      v
               [CloudWatch Alarm]
                      |
                      v
                [Amazon SNS Topic]
                      |
               +------+------+
               |             |
           Email          Slack
           (support)      (webhook Lambda)

Cas d'erreur : Lambda envoie le message en DLQ (Dead Letter Queue)
pour analyse manuelle plutôt que de le perdre.
```

---

### 4. IAM — Rôles, Politiques et flux d'authentification

**Ce que montre ce schéma** : comment IAM contrôle qui peut faire quoi sur AWS. La distinction centrale : les **Utilisateurs/Groupes** sont pour les humains (identifiants permanents), les **Rôles** sont pour les services AWS et applications (credentials temporaires générés par STS). Une Politique est un document JSON qui liste des `Allow` ou `Deny` sur des actions (`s3:GetObject`) et des ressources (ARN du bucket). Le principe du moindre privilège : commencer par tout refuser, n'ouvrir que ce qui est strictement nécessaire.

```
IDENTITES IAM
=============

Compte Root                     (ne jamais utiliser au quotidien)
    |
    +-- Utilisateur IAM          (personne humaine, clés permanentes)
    |       |
    |       +-- Groupe IAM       (ensemble d'utilisateurs, politiques communes)
    |
    +-- Role IAM                 (identité endossée par un service ou une app)
            |
            +-- EC2 Instance Profile  --> EC2 accede a S3/DynamoDB
            +-- Lambda Execution Role --> Lambda accede a S3/RDS
            +-- ECS Task Role         --> conteneur accede a Secrets Manager


POLITIQUE IAM (JSON)
====================

{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["s3:GetObject"],          <-- action precise
    "Resource": "arn:aws:s3:::mon-bucket/images/*"  <-- ressource precise
  }]
}

Evaluation des permissions :
  Request --> IAM evalue toutes les politiques attachees
      |
      +-- DENY explicite ?  --> REFUSE (prioritaire sur tout)
      +-- ALLOW explicite ? --> AUTORISE
      +-- Aucun des deux   --> REFUSE (deny par defaut)


FLUX STS (credentials temporaires)
====================================

Lambda demarre
    |
    v
AWS STS genere : AccessKeyId + SecretAccessKey + SessionToken
    |             (valables 15min a 12h, jamais stockes dans le code)
    v
Lambda appelle S3 avec ces credentials temporaires
```

---

### 5. Haute Disponibilité Multi-AZ

**Ce que montre ce schéma** : comment une application résiste à la panne d'un Data Center entier. L'ALB distribue le trafic sur plusieurs AZ — si AZ-1 tombe, AZ-2 et AZ-3 absorbent 100% du trafic automatiquement, sans intervention manuelle. Pour RDS, la réplication synchrone vers l'instance standby garantit 0 perte de données lors du failover. À retenir pour l'entretien : Multi-AZ = haute disponibilité (même région), Multi-Region = disaster recovery (régions géographiques différentes, latence réduite pour utilisateurs mondiaux).

```
                    INTERNET
                        |
                        v
             [Route 53 DNS]
                        |
                        v
         [Application Load Balancer]
          (distribue sur toutes les AZ)
                        |
          +-------------+-------------+
          |             |             |
          v             v             v
        AZ-1           AZ-2          AZ-3
   +----------+   +----------+  +----------+
   | EC2 / ECS|   | EC2 / ECS|  | EC2 / ECS|
   +----------+   +----------+  +----------+
          |             |             |
          +------+------+             |
                 |                    |
                 v                    |
         +---------------+           |
         | RDS Primary   | <---------+
         | (AZ-1)        |   réplication synchrone
         +---------------+           |
                 |                   v
                 |          +---------------+
                 |          | RDS Standby   |
                 |          | (AZ-2)        |
                 |          +---------------+
                 |
         Si AZ-1 tombe :
         1. ALB detecte les instances unhealthy (health checks)
         2. Tout le trafic est redirige vers AZ-2 et AZ-3
         3. RDS bascule automatiquement sur le Standby (< 60s)
         4. Aucune intervention manuelle requise
```

---

### 6. Pipeline CI/CD AWS — Du code au déploiement

**Ce que montre ce schéma** : comment le code passe automatiquement de GitHub à la production sur AWS. Chaque push déclenche la chaîne complète : tests → build → déploiement. Le déploiement Canary est le pattern le plus prudent pour une Lambda critique : 10% du trafic sur la nouvelle version pendant N minutes, CloudWatch surveille les erreurs, et si tout va bien, bascule automatique à 100%. En cas d'erreur détectée, rollback automatique vers la version précédente.

```
Développeur
    |
    | git push origin main
    v
[GitHub / CodeCommit]
    |
    | webhook / event
    v
[AWS CodePipeline]  -- chef d'orchestre du pipeline
    |
    v
[AWS CodeBuild]  -- environnement isolé
    | - pip install -r requirements.txt
    | - pytest tests/                    <-- si echec : pipeline STOP
    | - docker build / sam build
    v
[Artifact Store S3]  -- package deployable stocke
    |
    v
[AWS CodeDeploy / Lambda Deploy]
    |
    +-- Strategie ALL_AT_ONCE   : deploiement immediat (dev/test)
    |
    +-- Strategie CANARY        : (production recommandee)
    |       |
    |       | 10% trafic --> nouvelle version
    |       | CloudWatch surveille taux d'erreur
    |       |     |
    |       |     +-- Erreur detectee --> ROLLBACK automatique
    |       |     |
    |       |     +-- OK apres 10min  --> 100% trafic bascule
    |
    +-- Strategie LINEAR        : incrementiel (5% toutes les 5min)

Monitoring post-deploy : CloudWatch Alarm
    --> si erreur rate > 1% dans les 30min post-deploy
    --> rollback automatique + notification SNS
```

---

### 7. Architecture complète — Projet Analyseur de Feedback Client

**Ce que montre ce schéma** : l'intégration de tous les services AWS vus dans ce fichier dans une architecture production réelle. Chaque service joue un rôle précis : API Gateway expose l'entrée, SQS absorbe les pics, Lambda exécute la logique métier, Bedrock fournit l'intelligence IA, DynamoDB stocke les résultats structurés, CloudWatch + SNS gèrent les alertes. Terraform et CodePipeline automatisent le déploiement de toute cette infrastructure. C'est le schéma à citer en entretien pour montrer qu'on sait assembler AWS de bout en bout.

```
Client (app mobile / web)
         |
         | POST /feedback
         v
[API Gateway]                   <-- point d'entrée HTTPS unique
         |
         v
[Amazon SQS]                    <-- découplage, absorbe les pics (soldes)
         |
         | trigger (batch de 10 messages)
         v
[AWS Lambda]  (Python + LangChain / LangGraph)
         |
         +--------> [Amazon Bedrock (Claude 3)]
         |           analyse sentiment + résumé de l'avis
         |                    |
         |                    v
         +--------> [Amazon DynamoDB]
         |           stockage : {feedback_id, sentiment, résumé, timestamp}
         |
         +--------> [CloudWatch Logs]
                     logs d'exécution + métriques custom
                              |
                              | si sentiment = "DANGER" ou erreur Lambda
                              v
                    [CloudWatch Alarm]
                              |
                              v
                    [Amazon SNS Topic]
                              |
                    +---------+---------+
                    |                   |
                 Email              Slack webhook
                (support)          (équipe produit)


INFRASTRUCTURE & DÉPLOIEMENT
==============================

Code (GitHub)
    |
    v
[CodePipeline --> CodeBuild --> CodeDeploy]
    - Tests unitaires automatiques
    - Déploiement Canary sur Lambda
    - Rollback automatique si erreur

[Terraform]
    - Provisionne VPC, Lambda, SQS, DynamoDB, IAM Roles
    - State stocké dans S3 + DynamoDB (lock)
    - Même code = environnements Test / Prod identiques


SYNTHESE DES SERVICES
======================

Service          Role dans ce projet
-----------      -------------------
API Gateway      Entrée HTTPS, throttling, auth JWT
SQS              Tampon, découplage, résilience aux pics
Lambda           Logique métier Python, NoOps
Bedrock          IA générative sans gestion GPU
DynamoDB         Stockage NoSQL rapide, scalable
CloudWatch       Logs, métriques, alertes
SNS              Distribution notifications multi-canal
Terraform        Infrastructure as Code
CodePipeline     CI/CD automatisé
IAM              Sécurité, moindre privilège, rôles
VPC              Isolation réseau, sous-réseaux privés
```
