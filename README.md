# 347-docker-swarm-wordpress

WordPress hautement disponible sur Docker Swarm.

## Contenu

| Service | Image | Placement |
|---|---|---|
| `db` | `mariadb:12.0.2` | manager |
| `wordpress` | `wordpress:6.8.2-fpm-alpine` | 2 répliques, 1 par worker |
| `nginx` | `nginx:1.28.0-alpine3.21` | 2 répliques, port 80 |
| `swarmpit` + `swarmpit-db` | `swarmpit/swarmpit`, `treehouses/couchdb:2.3.1` | manager, port 888 |

- Réseaux overlay : `frontend`, `backend`, `swarmpit`
- Volumes : `db_data`, `wp_data`, `swarmpit_db`
- Secrets : `db_password`, `db_root_password`

> MariaDB 12.0.1 n'existe qu'en release candidate (`12.0.1-rc`) sur Docker Hub, d'où la 12.0.2.

## Prérequis

- 3 VM Linux Apline-virt avec Docker installé, qui se joignent en réseau. Dans ce document :

  | VM | IP |
  |---|---|
  | swarm-manager | 192.168.122.10 |
  | swarm-worker1 | 192.168.122.11 |
  | swarm-worker2 | 192.168.122.12 |

  Si l'IP du manager est différente, la modifier dans `docker-compose.yml`.

## 1. Partage NFS

Le volume `wp_data` est monté en NFS par tous les nœuds pour que les répliques partagent le même code et les mêmes médias.

```sh
apk add nfs-utils
mkdir -p /srv/nfs/wordpress
echo "/srv/nfs/wordpress 192.168.122.0/24(rw,sync,no_subtree_check,no_root_squash)" >> /etc/exports
rc-update add nfs && rc-service nfs start
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

Vérifier sur le manager que les trois noeuds soit en `Ready` :
```sh
docker node ls
```

## 3. Lancer le projet

Copier `docker-compose.yml` et le dossier `nginx/` sur le manager, puis sur le manager :

```sh
openssl rand -base64 24 | docker secret create db_password -
openssl rand -base64 24 | docker secret create db_root_password -

docker stack deploy -c docker-compose.yml wordpress

docker stack services wordpress
```

## 4. Accès

| | Adresse |
|---|---|
| Site WordPress | http://192.168.122.10/, http://192.168.122.11/, http://192.168.122.12/ |
| Swarmpit | http://192.168.122.10:888 |

Pour utiliser le nom de domaine, ajouter dans le fichier hosts du PC client (`/etc/hosts` ou `C:\Windows\System32\drivers\etc\hosts`) :
```
192.168.122.10   www.my-wordpress.ch
```
puis ouvrir http://www.my-wordpress.ch.

## 6. Supprimer totalement la stack et ses volumes

Sur le manager :
```sh
docker stack rm wordpress
sleep 20
docker container prune -f
docker volume rm wordpress_db_data wordpress_wp_data wordpress_swarmpit_db
docker secret rm db_password db_root_password
rm -rf /srv/nfs/wordpress/* /srv/nfs/wordpress/.htaccess
```

Sur chaque worker :
```sh
docker container prune -f
docker volume rm wordpress_wp_data
```

## Utilisation de l'IA

Conformément au chapitre 8.3 du cahier des charges, l'assistant Claude a été utilisé :

- **Documentation :** une première version de ce README a été générée avec Claude, puis je l'ai retravaillée et simplifiée moi-même.
- **Configuration :** Claude Code a mit en place l'environnement sur ma machine, créé et installé les 3 VMs alpine, j'ai ensuite fait le reste - à partir de l'installation docker - et j'ai utilisé uniquement Claude.ai comme documentation interractive.
