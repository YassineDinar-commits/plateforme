# Plateforme : Uptime Kuma, ntfy et Caddy

Plateforme de supervision hébergée en conteneurs, réalisée dans le cadre du module « Linux serveur : de la VM au cloud » (CESI ASR 2026).

- **VPS public** (Scaleway, Debian 13) : services en HTTPS sur `status.ydinar.fr` et `notif.ydinar.fr`.
- **srv-lab** (VM locale, Debian 13) : reconstruction du même socle par Ansible, en HTTP, sans domaine public.

## Contenu du dépôt

| Fichier | Rôle | Cible |
|---|---|---|
| `compose.yaml` | Pile Docker : Uptime Kuma, ntfy, Caddy. Seul Caddy publie des ports (80, 443). | VPS et srv-lab |
| `Caddyfile` | Reverse proxy avec certificats Let's Encrypt automatiques. | **VPS** |
| `Caddyfile.lab` | Même reverse proxy en HTTP simple (pas de domaine public). | **srv-lab** |
| `site.yml` | Playbook Ansible : paquets, SSH durci, pare-feu nftables, Docker, pile. | **srv-lab uniquement** |
| `.gitignore` | Exclut `data/` (données des services) et `.env` (secrets). | |

> **Attention** : `site.yml` vise `localhost` et l'utilisateur `alice`. Ne jamais le lancer sur le VPS : il remplacerait son pare-feu et son Caddyfile HTTPS par ceux du lab.

## Lancer la pile (VPS)

```bash
cd /srv/plateforme
docker compose config      # valide la syntaxe
docker compose up -d
docker compose ps
```

Les données vivent dans `./data` (bind mounts), hors de Git. Les noms DNS `status` et `notif` doivent pointer vers le VPS, et les ports 80 et 443 rester ouverts (le défi Let's Encrypt passe par le port 80).

## Reconstruire srv-lab (Ansible)

Sur une Debian 13 minimale avec `sudo` et une clé SSH déposée :

```bash
sudo apt install -y git ansible
git clone git@github.com:YassineDinar-commits/plateforme.git
cd plateforme
ansible-playbook -i localhost, site.yml -K   # 1er passage : tout est appliqué
ansible-playbook -i localhost, site.yml -K   # 2e passage : changed=0 attendu
```

Point d'attention : `flush ruleset` de nftables efface les chaînes créées par Docker. Le playbook redémarre donc Docker (handler + `flush_handlers`) avant de démarrer la pile.

## Sécurité

- SSH par clé uniquement, `PermitRootLogin no`, `PasswordAuthentication no` (vérifié avec `sshd -T`).
- Pare-feu nftables en `policy drop`, seuls 22, 80 et 443 ouverts.
- CrowdSec et son bouncer nftables sur le VPS.
- Aucun secret dans le dépôt.

## Sauvegarde

- `restic` quotidien (timer systemd à 03:30) sur le VPS : `/srv/plateforme` et `/etc`.
- Kuma est arrêté pendant la copie (base MariaDB embarquée, donc cohérence).
- Copie hors du VPS vers srv-lab par `rsync` (clé dédiée, en lecture seule via `rrsync`).
- Restauration testée avec `restic restore`.
- Le mot de passe du dépôt restic est conservé hors du VPS (gestionnaire de mots de passe).

## Limites connues

- Les alertes ntfy passent par le même VPS et le même reverse proxy que les services surveillés : une panne de Caddy empêche l'alerte d'atteindre le téléphone. Piste : sonde externe ou `ntfy.sh`.
- La copie de sauvegarde vers srv-lab n'a lieu que si la VM est allumée.
- Environnement de test sous VMware : les interfaces s'appellent `ens33` et `ens34` (et non `enp0s3` et `enp0s8` comme dans le cours).
