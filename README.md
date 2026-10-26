# 347-docker-swarm-wordpress

WordPress hautement disponible sur Docker Swarm (1 manager + 2 workers), avec nginx en reverse proxy, MariaDB, secrets Docker et Swarmpit.

**Auteur :** Albert Braimi — ETML, module 347

## Contenu

| Service | Image | Placement |
|---|---|---|
| `db` | `mariadb:12.0.2` | manager |
| `wordpress` | `wordpress:6.8.2-fpm-alpine` | 2 répliques, 1 par worker |
| `nginx` | `nginx:1.28.0-alpine3.21` | 2 répliques, port 80 |
| `swarmpit` + `swarmpit-db` | `swarmpit/swarmpit`, `treehouses/couchdb:2.3.1` | manager, port 888 |

- Réseaux overlay : `frontend` (nginx ↔ wordpress), `backend` (wordpress ↔ mariadb, interne), `swarmpit`
- Volumes : `db_data` (MariaDB), `wp_data` (code + médias WordPress, partagé en **NFS** entre les nœuds), `swarmpit_db`
- Secrets : `db_password`, `db_root_password`

> MariaDB 12.0.1 n'existe qu'en pré-version (`12.0.1-rc`) sur Docker Hub, d'où la 12.0.2.

## Prérequis

- 3 VM Linux avec Docker installé, qui se joignent en réseau. Dans ce document :

  | VM | IP |
  |---|---|
  | swarm-manager | 192.168.122.10 |
  | swarm-worker1 | 192.168.122.11 |
  | swarm-worker2 | 192.168.122.12 |

  Si l'IP du manager est différente, la modifier dans `docker-compose.yml` (volume `wp_data`, option `addr=`).
- Client NFS sur les 3 VM (`apk add nfs-utils` sur Alpine, `apt install nfs-common` sur Debian/Ubuntu).

## 1. Partage NFS (sur le manager)

Le volume `wp_data` est monté en NFS par tous les nœuds pour que les répliques partagent le même code et les mêmes médias.

```sh
apk add nfs-utils                    # Debian/Ubuntu : apt install nfs-kernel-server
mkdir -p /srv/nfs/wordpress
echo "/srv/nfs/wordpress 192.168.122.0/24(rw,sync,no_subtree_check,no_root_squash)" >> /etc/exports
rc-update add nfs && service nfs start   # Debian/Ubuntu : systemctl enable --now nfs-server
exportfs -v
```

## 2. Créer le cluster Swarm

Sur le manager :
```sh
docker swarm init --advertise-addr 192.168.122.10
```

Copier la commande `docker swarm join ...` affichée et l'exécuter sur les deux workers :
```sh
docker swarm join --token <TOKEN> 192.168.122.10:2377
```

Vérifier sur le manager (3 nœuds `Ready`) :
```sh
docker node ls
```

## 3. Lancer le projet

Copier `docker-compose.yml` et le dossier `nginx/` sur le manager, puis sur le manager :

```sh
# secrets (une seule fois)
openssl rand -base64 24 | docker secret create db_password -
openssl rand -base64 24 | docker secret create db_root_password -

# déploiement
docker stack deploy -c docker-compose.yml wordpress

# suivi : attendre que chaque service affiche toutes ses répliques (1/1, 2/2)
docker stack services wordpress
```

## 4. Accès

| | Adresse |
|---|---|
| Site WordPress | http://192.168.122.10 (ou l'IP de n'importe quel nœud) |
| Swarmpit | http://192.168.122.10:888 (créer le compte admin à la 1re visite) |

Pour utiliser le nom de domaine, ajouter dans le fichier hosts du PC client (`/etc/hosts` ou `C:\Windows\System32\drivers\etc\hosts`) :
```
192.168.122.10   www.my-wordpress.ch
```
puis ouvrir http://www.my-wordpress.ch.

## 5. Commandes utiles

```sh
docker stack ps wordpress                          # où tourne chaque conteneur
docker service logs -f wordpress_wordpress         # logs d'un service
docker service scale wordpress_nginx=3             # changer le nombre de répliques
docker service update --force wordpress_wordpress  # répartir à nouveau les répliques après le retour d'un nœud
```

**Tester la tolérance aux pannes :** éteindre un worker. `docker node ls` le montre `Down`, et le site reste accessible grâce à la réplique WordPress de l'autre worker.

## 6. Supprimer totalement la stack et ses volumes

Sur le manager :
```sh
docker stack rm wordpress
sleep 20                                    # attendre l'arrêt des conteneurs
docker container prune -f                   # supprime les conteneurs arrêtés
docker volume rm wordpress_db_data wordpress_wp_data wordpress_swarmpit_db
docker secret rm db_password db_root_password
rm -rf /srv/nfs/wordpress/* /srv/nfs/wordpress/.htaccess   # fichiers WordPress stockés sur le partage NFS
```

Sur chaque worker :
```sh
docker container prune -f                   # supprime les conteneurs arrêtés qui utilisent encore le volume
docker volume rm wordpress_wp_data
```

## Utilisation de l'IA

Conformément au chapitre 8.3 du cahier des charges, l'assistant Claude (Anthropic) a été utilisé pour m'aider à préparer la configuration et cette documentation. J'ai relu et testé l'ensemble et je suis capable d'expliquer chaque ligne.
