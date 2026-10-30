# Déploiement de la Stack Traefik v3 Durcie (via GitHub Desktop)

Cette architecture met en place **Traefik v3** avec **isolation du Socket Docker** (`socket-proxy`), chiffrement **TLS 1.3 / HTTP3**, en-têtes de sécurité **HSTS** et noms obfusqués.

---

## 1. Pourquoi les noms ont été modifiés pour la sécurité ?

1. **Obfuscation du sous-domaine d'administration :**
   - Les robots scanners cherchent systématiquement `traefik.domaine.fr` ou `proxy.domaine.fr` dans les registres publics *Certificate Transparency* (Let's Encrypt).
   - Nous utilisons un sous-domaine neutre comme `edge-ctrl.alsek.fr` (modifiable dans `.env`).
2. **Noms de réseaux et de conteneurs neutres :**
   - `traefik-net` devient **`edge-public-net`** (réseau externe pour vos sites web).
   - `socket-guard-net` est un réseau interne hermétique entre Traefik et le proxy Docker.
   - Les conteneurs sont nommés `core-edge-router` et `core-socket-guard`.
3. **Protection contre la faille Docker Socket (`core-socket-guard`) :**
   - Monter `/var/run/docker.sock` directement dans Traefik donne un accès `root` total sur votre VPS si Traefik est compromis.
   - Ici, Traefik passe par `tecnativa/docker-socket-proxy` en **lecture seule stricte** (`POST=0`) : même en cas d'attaque, il est impossible de créer ou supprimer un conteneur.

---

## 2. Étape 1 : Pousser les fichiers avec GitHub Desktop (Sur votre PC)

1. Ouvrez **GitHub Desktop** et créez un nouveau dépôt privé nommé `vps-infra-edge`.
2. Copiez les fichiers de ce dossier dans votre dépôt local :
   - `docker-compose.yml`
   - `traefik.yml`
   - `dynamic/middlewares.yml`
   - `.env.example`
   - `.gitignore` *(Crucial : empêche d'envoyer vos mots de passe et certificats sur GitHub)*
3. Faites un **Commit to main** puis cliquez sur **Push origin** (en prenant soin de laisser le dépôt en **Private** sur GitHub).

---

## 3. Étape 2 : Préparer le VPS (Une seule fois en SSH)

Connectez-vous à votre VPS allemand :
```bash
ssh debian@IP_DE_VOTRE_VPS
```

### A. Créer l'entrée DNS chez OVH
Dans votre zone DNS `alsek.fr` chez OVH, créez une entrée de type **A** :
- Sous-domaine : `edge-ctrl.alsek.fr` -> Cible : `IP_DE_VOTRE_VPS`

### B. Créer le dossier et cloner votre dépôt GitHub
```bash
sudo mkdir -p /opt/docker
sudo chown -R $USER:$USER /opt/docker
cd /opt/docker

# Clonez votre dépôt (utilisez une Deploy Key SSH GitHub ou un Token PAT)
git clone https://github.com/VOTRE_COMPTE/vps-infra-edge.git traefik
cd traefik
```

### C. Créer le réseau externe sécurisé et le fichier SSL
```bash
# 1. Création du réseau public pour vos futurs conteneurs web
docker network create edge-public-net

# 2. Création du fichier acme.json avec les permissions strictes (600 obligatoire)
touch acme.json
chmod 600 acme.json
```

### D. Générer votre fichier `.env` secret sur le serveur
```bash
# Installez l'utilitaire apache2-utils pour chiffrer votre mot de passe
sudo apt update && sudo apt install -y apache2-utils

# Générez votre mot de passe (remplacez MonSuperMotDePasse par le vôtre)
htpasswd -nb admin 'MonSuperMotDePasse'
```
Copiez la ligne retournée (ex. `admin:$apr1$xyz...`), puis créez le fichier `.env` :
```bash
cp .env.example .env
nano .env
```
Remplissez `.env` (dans le fichier `.env`, doublez chaque signe `$` par `$$` pour que Docker Compose ne les interprète pas comme des variables vides) :
```env
TRAEFIK_DASHBOARD_DOMAIN=edge-ctrl.alsek.fr
TRAEFIK_BASIC_AUTH=admin:$$apr1$$xyz...
```

---

## 4. Étape 3 : Lancer Traefik v3

```bash
cd /opt/docker/traefik
docker compose up -d
```

Vérifiez que tout tourne proprement :
```bash
docker compose ps
docker logs core-edge-router
```
Rendez-vous sur `https://edge-ctrl.alsek.fr` pour vérifier que le cadenas Let's Encrypt et votre mot de passe fonctionnent.

---

## 5. Votre routine quotidienne avec GitHub Desktop

À chaque fois que vous modifiez une règle dans `dynamic/middlewares.yml` ou `docker-compose.yml` sur votre PC :
1. **Commit & Push** dans **GitHub Desktop**.
2. Sur le VPS (ou via Claude Code / Antigravity) :
   ```bash
   cd /opt/docker/traefik && git pull && docker compose up -d
   ```

---

## 6. Comment brancher vos futurs sites (ex: Alsek) sur ce Traefik ?

Dans le `docker-compose.yml` de vos sites (`www.alsek.fr`, `vitrine-caladoise.fr`), il suffira d'utiliser le réseau `edge-public-net` :

```yaml
services:
  alsek-web:
    build: .
    container_name: app-alsek-prod
    restart: unless-stopped
    networks:
      - edge-public-net
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.alsek.rule=Host(`alsek.fr`) || Host(`www.alsek.fr`)"
      - "traefik.http.routers.alsek.entrypoints=websecure"
      - "traefik.http.routers.alsek.tls.certresolver=letsencrypt"
      - "traefik.http.routers.alsek.middlewares=sec-headers@file"
      - "traefik.http.services.alsek.loadbalancer.server.port=3000"

networks:
  edge-public-net:
    external: true
```
