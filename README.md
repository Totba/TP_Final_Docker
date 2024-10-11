## 1. Volumes Docker (1h)

### 1.1 Introduction aux volumes (15 min)
Les volumes dans Docker permettent de persister des données indépendamment du cycle de vie des conteneurs. Il existe trois principaux types de volumes :

- **Volumes nommés** : gérés par Docker, ils sont stockés dans le répertoire Docker sur l'hôte. Avantage : gestion facile par Docker, persistance des données même après la suppression du conteneur.
  
- **Bind mounts** : mappent directement un répertoire ou fichier de l'hôte dans le conteneur. Avantage : manipulation facile des fichiers hôtes depuis le conteneur. Inconvénient : dépendance directe à l'hôte, ce qui peut compromettre l'isolation.

- **Volumes tmpfs** : stockent des données en mémoire. Avantage : haute performance, car les données sont dans la RAM. Inconvénient : les données sont volatiles et disparaissent après un redémarrage.

### 1.2 Exercice : Bind mounts (25 min)

1. **Créez un fichier HTML sur votre machine hôte** :  
   - Ouvrez un terminal, allez dans un répertoire de votre choix, et créez un fichier HTML simple :
     ```bash
     echo "<h1>Hello from Docker!</h1>" > index.html
     ```

2. **Lancez un conteneur Nginx en montant ce fichier dans le conteneur** :  
   Utilisez la commande suivante pour démarrer un conteneur Nginx et monter le fichier HTML depuis l'hôte :
   ```bash
   docker run --name nginx-container -d -p 8080:80 -v $(pwd)/index.html:/usr/share/nginx/html/index.html:ro nginx
   ```
   - Ici, `$(pwd)` correspond au chemin du répertoire actuel, et l'option `-v` permet de monter le fichier hôte dans le conteneur.

3. **Modifiez le fichier sur l'hôte et observez les changements** :  
   - Ouvrez le fichier `index.html` et modifiez son contenu (par exemple, remplacez "Hello" par "Bonjour").
   - Actualisez la page Web (http://localhost:8080) et observez que la modification est instantanément visible.

4. **Explorez les options de montage** :
   - L'option `:ro` dans la commande signifie "lecture seule". Pour tester cela, essayez de modifier le fichier dans le conteneur :
     ```bash
     docker exec -it nginx-container bash
     echo "New Content" > /usr/share/nginx/html/index.html
     ```
     Vous recevrez une erreur car le fichier est monté en lecture seule.

### 1.3 Exercice : Volumes tmpfs (20 min)

1. **Lancez un conteneur avec un volume tmpfs** :  
   Utilisez la commande suivante pour créer un conteneur avec un volume en RAM (tmpfs) :
   ```bash
   docker run -d --name tmpfs-container --mount type=tmpfs,destination=/app tmpfs
   ```

2. **Écrivez des données dans ce volume** :  
   Utilisez la commande `docker exec` pour accéder au conteneur et écrire des données dans le volume tmpfs :
   ```bash
   docker exec -it tmpfs-container bash
   echo "Hello from tmpfs!" > /app/data.txt
   ```

3. **Vérifiez la persistance après redémarrage** :
   - Redémarrez le conteneur :
     ```bash
     docker restart tmpfs-container
     ```
   - Entrez de nouveau dans le conteneur et vérifiez si le fichier existe toujours :
     ```bash
     docker exec -it tmpfs-container bash
     cat /app/data.txt
     ```
   - Le fichier n'existera plus, car les données tmpfs ne persistent pas après redémarrage.

4. **Comparez les performances avec un volume standard** :  
   - Pour comparer, vous pouvez créer un conteneur avec un volume standard et effectuer des opérations de lecture/écriture. Mesurez le temps pris pour lire/écrire des fichiers dans les deux cas, en utilisant des outils comme `time` ou `dd`.

### 1.4 Défi : Sauvegarde et restauration de volumes (20 min)

1. **Créez un script bash pour sauvegarder un volume** :
   - Script `backup.sh` pour sauvegarder un volume dans un fichier tar.gz :
     ```bash
     #!/bin/bash
     docker run --rm -v myvolume:/data -v $(pwd):/backup alpine tar czf /backup/volume-backup.tar.gz -C /data .
     ```
     Ce script monte le volume `myvolume` et sauvegarde son contenu dans un fichier `volume-backup.tar.gz`.

2. **Créez un second script pour restaurer un volume** :
   - Script `restore.sh` pour restaurer un volume à partir de la sauvegarde :
     ```bash
     #!/bin/bash
     docker run --rm -v myvolume:/data -v $(pwd):/backup alpine sh -c "cd /data && tar xzf /backup/volume-backup.tar.gz"
     ```

3. **Testez le processus avec des données réelles** :
   - Créez des données réelles dans un conteneur qui utilise le volume `myvolume` :
     ```bash
     docker run -d --name test-container -v myvolume:/data busybox sh -c "echo 'Test Data' > /data/file.txt && sleep 3600"
     ```
   - Exécutez le script de sauvegarde, puis supprimez et recréez le volume, et restaurez les données à l'aide du script de restauration.

### 1.5 Discussion et bonnes pratiques (10 min)
- **Quand utiliser chaque type de volume ?** :
  - Utilisez des **volumes nommés** pour la persistance à long terme.
  - Les **bind mounts** sont pratiques pour le développement local ou l'accès direct aux fichiers de l'hôte.
  - Utilisez **tmpfs** pour des données temporaires et des performances élevées.

- **Sécurité des volumes et des données** :
  - Faites attention aux permissions lors du montage des volumes, surtout avec les bind mounts.
  - Utilisez des options de montage comme `:ro` pour limiter l'accès en écriture.

- **Gestion des volumes en production** :
  - Automatisez les sauvegardes et les restaurations.
  - Limitez la taille des volumes tmpfs pour éviter de surcharger la RAM de l'hôte.

Cette méthode vous permet d'explorer les différents types de volumes et leur gestion tout en suivant les bonnes pratiques.



## 2. Secrets Docker (1h)

### 2.1 Introduction aux secrets (10 min)
Les **secrets Docker** permettent de gérer des informations sensibles (mots de passe, clés API, etc.) de manière sécurisée dans des conteneurs. Contrairement aux variables d'environnement, les secrets sont stockés de manière cryptée et ne sont accessibles aux conteneurs que lorsqu'ils sont explicitement requis. Ils sont souvent utilisés dans des environnements **Docker Swarm** pour la gestion des services en production.

**Fonctionnement :**
- Les secrets sont créés et stockés au niveau de Docker Swarm.
- Lorsqu'un conteneur a besoin d'un secret, celui-ci est monté dans le conteneur sous forme de fichier temporaire.
- Le secret est supprimé du conteneur dès qu'il est arrêté.

### 2.2 Exercice : Création et utilisation de secrets (20 min)

1. **Créez un secret contenant un mot de passe pour une base de données** :
   - Assurez-vous que votre environnement est en mode **Swarm**. Si ce n'est pas le cas, activez-le :
     ```bash
     docker swarm init
     ```
   - Créez un secret nommé `db_password` :
     ```bash
     echo "mysecretpassword" | docker secret create db_password -
     ```
   - Ce secret est désormais stocké dans le Swarm et peut être utilisé par des conteneurs.

2. **Lancez un conteneur PostgreSQL en utilisant ce secret** :
   - Créez un service Docker Swarm avec PostgreSQL, en utilisant le secret `db_password` :
     ```bash
     docker service create --name postgres-service \
       --secret db_password \
       -e POSTGRES_PASSWORD_FILE="/run/secrets/db_password" \
       postgres:latest
     ```
   - Ici, la variable d'environnement `POSTGRES_PASSWORD_FILE` pointe vers le fichier contenant le mot de passe stocké sous `/run/secrets/db_password`.

3. **Vérifiez que le secret est correctement utilisé** :
   - Vérifiez si le conteneur PostgreSQL utilise bien le mot de passe du secret en accédant à la base de données via `psql` :
     ```bash
     docker exec -it $(docker ps -q --filter name=postgres-service) psql -U postgres
     ```
   - Utilisez le mot de passe défini dans le secret pour vous connecter. Cela prouve que le mot de passe est correctement injecté depuis le secret.

### 2.3 Exercice : Secrets dans Docker Compose (20 min)

1. **Créez un fichier Docker Compose utilisant le secret précédent** :
   - Créez un fichier `docker-compose.yml` contenant les services PostgreSQL et un web service qui se connectera à PostgreSQL.
   ```yaml
   version: '3.8'

   services:
     postgres:
       image: postgres:latest
       environment:
         POSTGRES_PASSWORD_FILE: /run/secrets/db_password
       secrets:
         - db_password
       volumes:
         - pgdata:/var/lib/postgresql/data

     web:
       image: my-web-app:latest
       depends_on:
         - postgres
       environment:
         DB_PASSWORD_FILE: /run/secrets/db_password
         DB_HOST: postgres
         DB_USER: postgres
         DB_NAME: mydatabase
       secrets:
         - db_password

   secrets:
     db_password:
       external: true

   volumes:
     pgdata:
   ```

2. **Ajoutez un service web qui se connecte à la base de données en utilisant le secret** :
   - Le service `web` utilise l'image `my-web-app`, et les variables d'environnement de ce service, comme `DB_PASSWORD_FILE`, pointent vers le fichier secret stocké sous `/run/secrets/db_password`. Cela permet à votre application Web de se connecter à la base de données PostgreSQL sans que le mot de passe soit exposé dans les fichiers ou logs.

3. **Lancez le fichier Docker Compose** :
   - Exécutez la commande suivante pour déployer les services avec Docker Compose :
     ```bash
     docker stack deploy -c docker-compose.yml my_stack
     ```
   - Docker Compose utilise le secret `db_password` pour les deux services, PostgreSQL et l'application Web, de manière sécurisée.

### 2.4 Défi : Rotation des secrets (10 min)

1. **Implémentez une méthode pour mettre à jour un secret sans interruption de service** :
   - Créez un nouveau secret avec le mot de passe mis à jour :
     ```bash
     echo "newsecretpassword" | docker secret create db_password_v2 -
     ```

2. **Mettez à jour le service pour utiliser le nouveau secret** :
   - Mettez à jour le service PostgreSQL pour qu'il utilise le nouveau secret `db_password_v2` :
     ```bash
     docker service update --secret-rm db_password \
                           --secret-add source=db_password_v2,target=db_password \
                           postgres-service
     ```
   - Cette commande remplace le secret utilisé sans arrêter le conteneur PostgreSQL, garantissant ainsi la continuité du service.

3. **Supprimez l'ancien secret si nécessaire** :
   - Une fois que vous avez vérifié que le service fonctionne correctement avec le nouveau secret, vous pouvez supprimer l'ancien :
     ```bash
     docker secret rm db_password
     ```

Cette approche garantit que les secrets sensibles peuvent être mis à jour sans nécessiter de redémarrage complet des services ou d'interruption des opérations.


## 3. Réseaux Docker (1h)

### 3.1 Introduction aux réseaux Docker (10 min)

Docker offre plusieurs types de réseaux pour permettre aux conteneurs de communiquer entre eux ou avec l'extérieur. Voici les principaux types :

- **Bridge** : C'est le réseau par défaut pour les conteneurs. Les conteneurs sur un réseau bridge peuvent communiquer entre eux s'ils sont sur le même réseau, mais sont isolés du reste de l'hôte.
  
- **Host** : Le réseau "host" permet à un conteneur de partager le réseau de l'hôte directement, ce qui signifie qu'il n'a pas de couche d'abstraction réseau. Cela peut être utile pour les performances, mais il n'y a pas d'isolation entre le conteneur et l'hôte.
  
- **Overlay** : Utilisé principalement dans les environnements Docker Swarm, ce type de réseau permet à des conteneurs sur plusieurs hôtes Docker de communiquer entre eux.
  
- **Macvlan** : Ce réseau attribue une adresse MAC unique au conteneur, ce qui lui permet d'apparaître sur le réseau physique de l'hôte, comme un appareil séparé. Il est souvent utilisé pour des cas où des conteneurs doivent apparaître sur le réseau local.

### 3.2 Exercice : Réseau bridge personnalisé (20 min)

1. **Créez un réseau bridge personnalisé** :
   - Pour créer un réseau bridge personnalisé, utilisez la commande suivante :
     ```bash
     docker network create --driver bridge my_bridge_network
     ```
   - Vous pouvez vérifier que le réseau a été créé en listant tous les réseaux :
     ```bash
     docker network ls
     ```

2. **Lancez deux conteneurs sur ce réseau** :
   - Lancez deux conteneurs qui partageront le réseau bridge personnalisé :
     ```bash
     docker run -dit --name container1 --network my_bridge_network alpine sh
     docker run -dit --name container2 --network my_bridge_network alpine sh
     ```
   - Les conteneurs sont maintenant sur le réseau `my_bridge_network`.

3. **Testez la communication entre ces conteneurs** :
   - Entrez dans `container1` et testez la communication avec `container2` :
     ```bash
     docker exec -it container1 sh
     ping container2
     ```
   - Si les conteneurs peuvent se pinger, cela signifie que la communication fonctionne.

### 3.3 Exercice : Réseau overlay (20 min)

1. **Initialisez un swarm Docker** :
   - Si Docker Swarm n’est pas déjà activé, initialisez-le :
     ```bash
     docker swarm init
     ```

2. **Créez un réseau overlay** :
   - Créez un réseau overlay pour permettre la communication entre plusieurs hôtes dans un cluster Docker Swarm :
     ```bash
     docker network create --driver overlay my_overlay_network
     ```

3. **Déployez un service utilisant ce réseau** :
   - Déployez un service sur ce réseau overlay :
     ```bash
     docker service create --name my_service --network my_overlay_network -d nginx
     ```
   - Vous pouvez vérifier que le service est en cours d'exécution et qu'il utilise bien le réseau overlay en affichant les services :
     ```bash
     docker service ls
     docker network inspect my_overlay_network
     ```

   Le réseau overlay permet aux conteneurs du service de communiquer entre eux, même s'ils sont déployés sur des hôtes Docker différents dans un cluster Swarm.

### 3.4 Défi : Isolation réseau (10 min)

1. **Créez trois réseaux isolés** :
   - Créez trois réseaux Docker séparés :
     ```bash
     docker network create --driver bridge isolated_network1
     docker network create --driver bridge isolated_network2
     docker network create --driver bridge isolated_network3
     ```

2. **Lancez des conteneurs sur chaque réseau** :
   - Lancez trois conteneurs, un sur chaque réseau :
     ```bash
     docker run -dit --name container1 --network isolated_network1 alpine sh
     docker run -dit --name container2 --network isolated_network2 alpine sh
     docker run -dit --name container3 --network isolated_network3 alpine sh
     ```

3. **Démontrez l’isolation réseau** :
   - Testez l'isolement en essayant de pinger un conteneur depuis un autre :
     - Entrez dans `container1` et essayez de pinger `container2` :
       ```bash
       docker exec -it container1 sh
       ping container2
       ```
     - Vous verrez que les conteneurs sur différents réseaux ne peuvent pas communiquer entre eux, prouvant l'isolation.

   Cette configuration montre comment Docker utilise des réseaux séparés pour isoler les conteneurs et les services, garantissant ainsi la sécurité et l'isolation entre les différentes applications.


## 4. Merge de fichiers Docker Compose (1h)

### 4.1 Introduction au merge de fichiers Compose (10 min)

Le **merge de fichiers Docker Compose** permet de combiner plusieurs fichiers Compose pour créer des configurations adaptées à différents environnements (dev, test, prod, etc.). Cette fonctionnalité est utile lorsque vous souhaitez réutiliser une configuration de base tout en apportant des modifications spécifiques à un environnement. Par défaut, Docker Compose charge le fichier `docker-compose.yml` mais peut être complété par un fichier `docker-compose.override.yml` ou tout autre fichier spécifique en utilisant l'option `-f`.

**Cas d’usage** :
- Adapter les configurations selon l'environnement (par exemple, différentes bases de données ou services de monitoring).
- Ajouter ou modifier des services pour le développement ou les tests.
- Réutiliser une configuration commune pour plusieurs environnements.

### 4.2 Exercice : Merge simple (20 min)

1. **Créez un fichier `docker-compose.yml` de base** :
   - Créez un fichier `docker-compose.yml` contenant un service basique. Par exemple, un service Nginx :
     ```yaml
     version: '3.8'

     services:
       web:
         image: nginx
         ports:
           - "80:80"
     ```

2. **Créez un fichier `docker-compose.override.yml`** :
   - Le fichier `docker-compose.override.yml` peut contenir des ajustements ou des ajouts à la configuration de base. Par exemple, on peut ajouter des volumes pour le développement local :
     ```yaml
     version: '3.8'

     services:
       web:
         volumes:
           - ./src:/usr/share/nginx/html
     ```

3. **Combinez ces fichiers et observez le résultat** :
   - Lorsque vous exécutez Docker Compose, les fichiers `docker-compose.yml` et `docker-compose.override.yml` sont combinés automatiquement :
     ```bash
     docker-compose up
     ```
   - Vous pouvez voir que le service `web` est lancé avec les ports définis dans `docker-compose.yml` et le volume défini dans `docker-compose.override.yml`.

   - Pour observer les changements, vous pouvez inspecter la configuration résultante avec :
     ```bash
     docker-compose config
     ```

### 4.3 Exercice : Merge pour différents environnements (20 min)

1. **Créez des fichiers pour les environnements dev, test et prod** :
   - Créez un fichier pour chaque environnement :
     - `docker-compose.dev.yml` :
       ```yaml
       version: '3.8'

       services:
         web:
           environment:
             - DEBUG=true
           volumes:
             - ./src:/usr/share/nginx/html
       ```

     - `docker-compose.test.yml` :
       ```yaml
       version: '3.8'

       services:
         web:
           environment:
             - TEST_MODE=true
       ```

     - `docker-compose.prod.yml` :
       ```yaml
       version: '3.8'

       services:
         web:
           environment:
             - DEBUG=false
           ports:
             - "80:80"
       ```

2. **Utilisez le merge pour déployer dans chaque environnement** :
   - Pour déployer dans un environnement spécifique, vous pouvez combiner le fichier `docker-compose.yml` de base avec le fichier de l'environnement correspondant. Utilisez l’option `-f` pour spécifier plusieurs fichiers :
     ```bash
     docker-compose -f docker-compose.yml -f docker-compose.dev.yml up
     ```
   - Pour tester l’environnement de production :
     ```bash
     docker-compose -f docker-compose.yml -f docker-compose.prod.yml up
     ```
   - Cela permet de réutiliser la configuration de base tout en ajoutant des paramètres spécifiques à chaque environnement.

### 4.4 Défi : Merge complexe (10 min)

1. **Créez une structure de fichiers Compose pour plusieurs services avec des configurations spécifiques par environnement** :
   - Pour une application plus complexe avec plusieurs services (par exemple, un serveur web et une base de données), vous pouvez structurer vos fichiers Compose comme suit :

     - `docker-compose.yml` (configuration commune) :
       ```yaml
       version: '3.8'

       services:
         web:
           image: nginx
           ports:
             - "80:80"

         db:
           image: postgres
           environment:
             POSTGRES_PASSWORD: example
       ```

     - `docker-compose.dev.yml` (pour développement) :
       ```yaml
       version: '3.8'

       services:
         web:
           environment:
             - DEBUG=true
           volumes:
             - ./src:/usr/share/nginx/html

         db:
           environment:
             POSTGRES_PASSWORD: devpassword
       ```

     - `docker-compose.prod.yml` (pour production) :
       ```yaml
       version: '3.8'

       services:
         web:
           environment:
             - DEBUG=false

         db:
           environment:
             POSTGRES_PASSWORD: prodpassword
           volumes:
             - pgdata:/var/lib/postgresql/data

       volumes:
         pgdata:
       ```

2. **Gestion des services par environnement** :
   - Pour le développement, vous pouvez lancer les services avec un volume monté localement et un mot de passe de base de données adapté :
     ```bash
     docker-compose -f docker-compose.yml -f docker-compose.dev.yml up
     ```
   - En production, vous utilisez des mots de passe plus sécurisés et des volumes pour la persistance des données :
     ```bash
     docker-compose -f docker-compose.yml -f docker-compose.prod.yml up
     ```

Cette approche montre comment utiliser le merge de fichiers Docker Compose pour gérer plusieurs environnements, en réutilisant des configurations communes tout en adaptant les paramètres spécifiques pour chaque cas.


## 5. Projet final (1h)

### Objectif
Créer une application multi-conteneurs comprenant :
- **Frontend** : Serveur web Nginx
- **Backend** : Application Node.js
- **Base de données** : PostgreSQL
- **Cache** : Redis

L’application doit :
1. Utiliser des **volumes** pour la persistance des données.
2. Implémenter des **secrets** pour les mots de passe sensibles.
3. Configurer des **réseaux isolés** pour la sécurité.
4. Gérer les environnements dev et prod à l'aide du **merge de fichiers Compose**.

---

## Étapes pour réaliser le projet

### 1. Créez les Dockerfiles nécessaires

- **Frontend (Nginx)** :
  Créez un `Dockerfile` pour le service Nginx dans le répertoire `frontend` :
  ```Dockerfile
  FROM nginx:alpine
  COPY ./nginx.conf /etc/nginx/nginx.conf
  COPY ./build /usr/share/nginx/html
  ```

- **Backend (Node.js)** :
  Créez un `Dockerfile` dans le répertoire `backend` pour l’application Node.js :
  ```Dockerfile
  FROM node:14-alpine
  WORKDIR /app
  COPY package*.json ./
  RUN npm install
  COPY . .
  CMD ["npm", "start"]
  ```

- **Base de données (PostgreSQL)** :
  Pas besoin de créer un `Dockerfile` pour PostgreSQL, car nous utiliserons l’image officielle de Docker Hub.

- **Cache (Redis)** :
  Comme pour PostgreSQL, Redis utilisera l’image officielle de Docker Hub.

### 2. Configurez les volumes et les secrets

- **Volumes** :
  Les volumes vont permettre de persister les données de PostgreSQL et Redis. Les fichiers de l’application Node.js peuvent être montés pour le développement.

- **Secrets** :
  Utilisez les secrets Docker pour stocker les mots de passe des services.

1. Créez un fichier pour stocker les secrets (par exemple `db_password`) :
   ```bash
   echo "yourpassword" | docker secret create db_password -
   echo "redispassword" | docker secret create redis_password -
   ```

2. Définissez les volumes et les secrets dans le fichier `docker-compose.yml`.

### 3. Mettez en place la structure réseau

- Utilisez deux réseaux Docker pour isoler les services :
  - **Frontend** : Connecté uniquement à Nginx et au backend Node.js.
  - **Backend** : Connecté à PostgreSQL, Redis, et au service Node.js.

Dans `docker-compose.yml` :
```yaml
networks:
  frontend_net:
    driver: bridge
  backend_net:
    driver: bridge
```

### 4. Créez les fichiers Docker Compose et implémentez le merge

1. **Fichier de base `docker-compose.yml`** :
   Ce fichier contient les services communs à tous les environnements.
   ```yaml
   version: '3.8'

   services:
     frontend:
       build: ./frontend
       ports:
         - "80:80"
       networks:
         - frontend_net

     backend:
       build: ./backend
       environment:
         DATABASE_URL: postgres://postgres:password@db:5432/mydatabase
         REDIS_URL: redis://:password@redis:6379
       networks:
         - frontend_net
         - backend_net

     db:
       image: postgres:latest
       environment:
         POSTGRES_PASSWORD_FILE: /run/secrets/db_password
       volumes:
         - pgdata:/var/lib/postgresql/data
       secrets:
         - db_password
       networks:
         - backend_net

     redis:
       image: redis:alpine
       command: redis-server --requirepass $REDIS_PASSWORD
       secrets:
         - redis_password
       networks:
         - backend_net

   secrets:
     db_password:
       external: true
     redis_password:
       external: true

   volumes:
     pgdata:
   ```

2. **Fichier pour développement `docker-compose.dev.yml`** :
   Ce fichier monte des volumes pour un rechargement automatique et active le mode debug.
   ```yaml
   version: '3.8'

   services:
     backend:
       volumes:
         - ./backend:/app
       environment:
         NODE_ENV: development
   ```

3. **Fichier pour production `docker-compose.prod.yml`** :
   Ce fichier désactive le mode debug et applique des optimisations pour la production.
   ```yaml
   version: '3.8'

   services:
     backend:
       environment:
         NODE_ENV: production
     db:
       volumes:
         - pgdata:/var/lib/postgresql/data
   ```

## 5. Projet final (1h)

### Objectif
Créer une application multi-conteneurs comprenant :
- **Frontend** : Serveur web Nginx
- **Backend** : Application Node.js
- **Base de données** : PostgreSQL
- **Cache** : Redis

L’application doit :
1. Utiliser des **volumes** pour la persistance des données.
2. Implémenter des **secrets** pour les mots de passe sensibles.
3. Configurer des **réseaux isolés** pour la sécurité.
4. Gérer les environnements dev et prod à l'aide du **merge de fichiers Compose**.

---

## Étapes pour réaliser le projet

### 1. Créez les Dockerfiles nécessaires

- **Frontend (Nginx)** :
  Créez un `Dockerfile` pour le service Nginx dans le répertoire `frontend` :
  ```Dockerfile
  FROM nginx:alpine
  COPY ./nginx.conf /etc/nginx/nginx.conf
  COPY ./build /usr/share/nginx/html
  ```

- **Backend (Node.js)** :
  Créez un `Dockerfile` dans le répertoire `backend` pour l’application Node.js :
  ```Dockerfile
  FROM node:14-alpine
  WORKDIR /app
  COPY package*.json ./
  RUN npm install
  COPY . .
  CMD ["npm", "start"]
  ```

- **Base de données (PostgreSQL)** :
  Pas besoin de créer un `Dockerfile` pour PostgreSQL, car nous utiliserons l’image officielle de Docker Hub.

- **Cache (Redis)** :
  Comme pour PostgreSQL, Redis utilisera l’image officielle de Docker Hub.

### 2. Configurez les volumes et les secrets

- **Volumes** :
  Les volumes vont permettre de persister les données de PostgreSQL et Redis. Les fichiers de l’application Node.js peuvent être montés pour le développement.

- **Secrets** :
  Utilisez les secrets Docker pour stocker les mots de passe des services.

1. Créez un fichier pour stocker les secrets (par exemple `db_password`) :
   ```bash
   echo "yourpassword" | docker secret create db_password -
   echo "redispassword" | docker secret create redis_password -
   ```

2. Définissez les volumes et les secrets dans le fichier `docker-compose.yml`.

### 3. Mettez en place la structure réseau

- Utilisez deux réseaux Docker pour isoler les services :
  - **Frontend** : Connecté uniquement à Nginx et au backend Node.js.
  - **Backend** : Connecté à PostgreSQL, Redis, et au service Node.js.

Dans `docker-compose.yml` :
```yaml
networks:
  frontend_net:
    driver: bridge
  backend_net:
    driver: bridge
```

### 4. Créez les fichiers Docker Compose et implémentez le merge

1. **Fichier de base `docker-compose.yml`** :
   Ce fichier contient les services communs à tous les environnements.
   ```yaml
   version: '3.8'

   services:
     frontend:
       build: ./frontend
       ports:
         - "80:80"
       networks:
         - frontend_net

     backend:
       build: ./backend
       environment:
         DATABASE_URL: postgres://postgres:password@db:5432/mydatabase
         REDIS_URL: redis://:password@redis:6379
       networks:
         - frontend_net
         - backend_net

     db:
       image: postgres:latest
       environment:
         POSTGRES_PASSWORD_FILE: /run/secrets/db_password
       volumes:
         - pgdata:/var/lib/postgresql/data
       secrets:
         - db_password
       networks:
         - backend_net

     redis:
       image: redis:alpine
       command: redis-server --requirepass $REDIS_PASSWORD
       secrets:
         - redis_password
       networks:
         - backend_net

   secrets:
     db_password:
       external: true
     redis_password:
       external: true

   volumes:
     pgdata:
   ```

2. **Fichier pour développement `docker-compose.dev.yml`** :
   Ce fichier monte des volumes pour un rechargement automatique et active le mode debug.
   ```yaml
   version: '3.8'

   services:
     backend:
       volumes:
         - ./backend:/app
       environment:
         NODE_ENV: development
   ```

3. **Fichier pour production `docker-compose.prod.yml`** :
   Ce fichier désactive le mode debug et applique des optimisations pour la production.
   ```yaml
   version: '3.8'

   services:
     backend:
       environment:
         NODE_ENV: production
     db:
       volumes:
         - pgdata:/var/lib/postgresql/data
   ```

### 5. Testez le déploiement en dev et en prod

- Pour **développement**, utilisez :
  ```bash
  docker-compose -f docker-compose.yml -f docker-compose.dev.yml up
  ```

- Pour **production**, utilisez :
  ```bash
  docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d
  ```

Cela vous permettra de tester l'application dans les deux environnements avec des configurations adaptées et spécifiques.