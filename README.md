# 3dokr_kad_leo_kylian

Projet Docker Supinfo : on a conteneurisé l'application de vote (`voting-app/`) avec Docker Compose puis avec Docker Swarm.

## Avant de commencer

Toutes les commandes se lancent depuis le dossier `voting-app/` :

```bash
cd voting-app
```

Il faut créer un fichier `.env` dans ce dossier (il n'est pas commité, chacun crée le sien). Le plus simple est de copier le modèle `.env.example` :

```bash
cp .env.example .env
```

Puis on remplace `changez_moi` par son propre mot de passe dans `.env`.

Mettez une valeur par ligne, sans espaces ni guillemets, et évitez les caractères `$`, `"` et `#` dans le mot de passe.

## Lancer avec Docker Compose

```bash
docker compose up -d --build --wait
docker compose ps
```

Ensuite on ouvre :
- Vote : http://localhost:8080
- Résultats : http://localhost:8081

Commandes utiles :

```bash
docker compose logs -f worker   # voir les votes traités
docker compose down             # arrêter (les données sont gardées)
docker compose down -v          # arrêter et supprimer les données
```

## Lancer avec Docker Swarm

On simule un cluster de 3 nœuds (1 manager + 2 workers) sur une seule machine avec des conteneurs `docker:dind`.

**Attention :** le projet Compose utilise aussi les ports 8080 et 8081. Il faut l'arrêter avec `docker compose down` avant l'étape 1, sinon le conteneur `manager` ne pourra pas démarrer.

Il y a deux versions des commandes : une pour macOS / Linux (bash) et une pour Windows (PowerShell). Elles font exactement la même chose.

### Sous macOS / Linux (bash)

```bash
# 1. Créer le réseau et les 3 nœuds
docker network create --driver bridge swarm-lab
docker run -d --privileged --name manager --hostname manager --network swarm-lab -p 8080:8080 -p 8081:8081 docker:29-dind
docker run -d --privileged --name worker1 --hostname worker1 --network swarm-lab docker:29-dind
docker run -d --privileged --name worker2 --hostname worker2 --network swarm-lab docker:29-dind
sleep 10

# 2. Créer le cluster
docker exec manager docker swarm init --advertise-addr eth0
TOKEN=$(docker exec manager docker swarm join-token -q worker)
docker exec worker1 docker swarm join --token "$TOKEN" manager:2377
docker exec worker2 docker swarm join --token "$TOKEN" manager:2377
docker exec manager docker node ls

# 3. Registry + envoi des images
docker exec manager docker service create --name registry --publish 5000:5000 registry:2
docker compose build
for s in vote worker result; do
  docker save voting-app/$s:1.0 | docker exec -i manager docker load
  docker exec manager docker tag voting-app/$s:1.0 127.0.0.1:5000/voting-app/$s:1.0
  docker exec manager docker push 127.0.0.1:5000/voting-app/$s:1.0
done

# 4. Secret (reprend le mot de passe du .env)
grep '^POSTGRES_PASSWORD=' .env | cut -d= -f2- | tr -d '\n' | docker exec -i manager docker secret create db_password -

# 5. Déployer
docker exec -i manager sh -c 'cat > /root/docker-stack.yml' < docker-stack.yml
docker exec manager docker stack deploy -c /root/docker-stack.yml voting

# 6. Vérifier
docker exec manager docker service ls
docker exec manager docker stack ps voting
```

### Sous Windows (PowerShell)

Certaines commandes bash ne marchent pas dans PowerShell (`sleep`, `$(...)`, `grep`, `cut`, `tr` et la redirection `<`). En plus, PowerShell peut abîmer les données binaires quand on fait `docker save | docker load`. Du coup, dans cette version, on passe par des fichiers et par `docker cp` pour envoyer les images, le secret et le fichier stack dans le manager.

```powershell
# 1. Créer le réseau et les 3 nœuds
docker network create --driver bridge swarm-lab
docker run -d --privileged --name manager --hostname manager --network swarm-lab -p 8080:8080 -p 8081:8081 docker:29-dind
docker run -d --privileged --name worker1 --hostname worker1 --network swarm-lab docker:29-dind
docker run -d --privileged --name worker2 --hostname worker2 --network swarm-lab docker:29-dind
Start-Sleep -Seconds 10

# 2. Créer le cluster
docker exec manager docker swarm init --advertise-addr eth0
$TOKEN = docker exec manager docker swarm join-token -q worker
docker exec worker1 docker swarm join --token $TOKEN manager:2377
docker exec worker2 docker swarm join --token $TOKEN manager:2377
docker exec manager docker node ls

# 3. Registry + envoi des images
docker exec manager docker service create --name registry --publish 5000:5000 registry:2
docker compose build
foreach ($s in "vote", "worker", "result") {
  docker save -o "$s.tar" "voting-app/${s}:1.0"
  docker cp "$s.tar" "manager:/root/$s.tar"
  docker exec manager docker load -i "/root/$s.tar"
  docker exec manager rm "/root/$s.tar"
  Remove-Item "$s.tar"
  docker exec manager docker tag "voting-app/${s}:1.0" "127.0.0.1:5000/voting-app/${s}:1.0"
  docker exec manager docker push "127.0.0.1:5000/voting-app/${s}:1.0"
}

# 4. Secret (reprend le mot de passe du .env)
$DB_PASSWORD = ((Get-Content .env | Where-Object { $_ -like "POSTGRES_PASSWORD=*" }) -split "=", 2)[1]
[System.IO.File]::WriteAllText("$PWD\db_password.txt", $DB_PASSWORD)
docker cp db_password.txt manager:/root/db_password.txt
docker exec manager docker secret create db_password /root/db_password.txt
docker exec manager rm /root/db_password.txt
Remove-Item db_password.txt

# 5. Déployer
docker cp docker-stack.yml manager:/root/docker-stack.yml
docker exec manager docker stack deploy -c /root/docker-stack.yml voting

# 6. Vérifier
docker exec manager docker service ls
docker exec manager docker stack ps voting
```

Pour le secret, on écrit le mot de passe avec `WriteAllText` parce que cette fonction n'ajoute pas de retour à la ligne à la fin. Sinon le secret ne serait pas identique au mot de passe du `.env`. Le fichier temporaire est supprimé juste après.

Ensuite on ouvre :
- Vote : http://localhost:8080
- Résultats : http://localhost:8081

Si on modifie `docker-stack.yml`, il faut refaire l'étape 5 : le manager utilise sa propre copie du fichier (`/root/docker-stack.yml`).

Pour tout supprimer à la fin (mêmes commandes en bash et en PowerShell) :

```bash
docker rm -f -v manager worker1 worker2
docker network rm swarm-lab
```
