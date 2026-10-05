# SlvCtrl+ pour Umbrel 2.0 (umbrelOS)

Conversion non-officielle du script `install-rpi.sh` de SlvCtrl+ en app Umbrel.

```
.
├── .github/workflows/build-images.yml   # construit + pousse les images sur GHCR
├── README.md
└── slvctrlplus/
    ├── umbrel-app.yml       # manifeste de l'app (icône incluse)
    ├── icon.svg             # icône de l'app
    ├── docker-compose.yml   # 3 services : app_proxy, frontend, server
    ├── server/Dockerfile    # backend Node.js (télécharge le dist.tar.gz officiel)
    └── frontend/
        ├── Dockerfile       # frontend Vue, servi par nginx
        └── nginx.conf
```

## Mise à jour : le serveur plantait au démarrage (libasound.so.2)

Après l'install réussie, le frontend affichait "Cannot connect to server" en
boucle. Cause : `slvctrlplus_server_1` plantait immédiatement au démarrage
(rien n'écoutait sur le port 1337) avec :

```
Error: libasound.so.2: cannot open shared object file: No such file or directory
    at ... (/app/node_modules/speaker/index.js:9:42)
```

Le module natif du paquet `speaker` (sortie audio) a besoin d'ALSA au
runtime — une bibliothèque système, pas une dépendance npm. `server/Dockerfile`
installe maintenant `libasound2`, `libusb-1.0-0`, `libudev1`, `libbluetooth3`
et `libdbus-1-3` (les mêmes libs couvrent aussi `usb` et
`@stoprocent/noble`, qui auraient planté pareil dès leur premier usage).

**Pour appliquer le correctif** : remplace `server/Dockerfile` dans ton repo
par la version mise à jour, commit + push sur `main` (l'Action se relance
automatiquement), puis une fois l'Action verte, force Umbrel à retélécharger
l'image (un simple "Restart" ne suffit pas, Docker ne re-pull pas tout seul
une image `:latest` déjà présente localement) :

```bash
ssh -p 40000 antoine@192.168.1.137
cd /home/umbrel/umbrel/app-data/slvctrlplus
sudo docker compose pull server
sudo docker compose up -d --force-recreate --no-deps server
sudo docker logs --tail 50 slvctrlplus_server_1
```

Le dernier `docker logs` doit maintenant rester silencieux (pas de crash
loop) — vérifie aussi `sudo ss -tlnp | grep 1337` pour confirmer que le port
écoute.

## Ce qui a changé depuis la première version

Ton install a échoué avec :

```
Command failed with exit code 1: /opt/umbreld/source/modules/apps/legacy-compat/app-script install slvctrlplus
```

Cause la plus probable : la première version utilisait `build: ./server` /
`build: ./frontend` dans `docker-compose.yml`. **umbrelOS ne build jamais de
Dockerfile lui-même, il ne fait que `pull` des images déjà construites.**
Sans `image:`, l'étape de pull échoue et `app-script install` plante — sans
forcément détailler pourquoi dans le message que tu as collé.

La solution : un `Dockerfile` reste nécessaire (ni slvctrlplus-server ni
slvctrlplus-frontend ne publient d'image officielle), mais il est maintenant
construit **en dehors** d'Umbrel, par une GitHub Action qui pousse le
résultat sur `ghcr.io`. `docker-compose.yml` ne fait plus que `pull` ces
images, comme umbrelOS s'y attend.

J'ai aussi ajouté `icon.svg` + le champ `icon:` dans `umbrel-app.yml` (les
app stores communautaires l'affichent, contrairement au store officiel qui
héberge ses icônes ailleurs).

Si l'install replante quand même après ces changements, le message collé ici
est juste l'erreur "enveloppe" — la vraie cause est dans les logs d'umbreld.
Récupère-les en SSH pour qu'on puisse creuser :

```bash
ssh -p 40000 antoine@192.168.1.137
tail -n 200 ~/umbrel/logs/umbreld.log
# ou, si umbreld tourne lui-même en conteneur :
docker logs --tail 200 umbreld
```

## Comment publier les images (étape à faire une seule fois)

1. Crée un repo GitHub **public** (GHCR anonyme a besoin que le repo/package
   soit public, sinon umbrelOS ne pourra pas `pull` sans identifiants).
2. Mets-y tout le contenu de cette archive (`.github/`, `README.md`,
   `slvctrlplus/`) à la racine du repo.
3. Dans `slvctrlplus/docker-compose.yml` et `slvctrlplus/umbrel-app.yml`,
   remplace `OWNER` (et `REPO` pour l'icône) par ton pseudo/org GitHub et le
   nom du repo, par exemple :

   ```yaml
   image: ghcr.io/antoine/slvctrlplus-server:latest
   ```

   ```yaml
   icon: https://raw.githubusercontent.com/antoine/mon-repo/main/slvctrlplus/icon.svg
   ```

4. Pousse sur la branche `main`. L'Action `build-images.yml` se déclenche
   automatiquement (ou lance-la à la main depuis l'onglet *Actions* →
   *Build & push SlvCtrl+ images* → *Run workflow*). Elle construit les deux
   images en `linux/amd64` + `linux/arm64` et les pousse sur
   `ghcr.io/<toi>/slvctrlplus-server` et `-frontend`.
5. Une fois l'Action verte : va dans ton profil/org GitHub → *Packages*,
   ouvre chacun des deux packages → *Package settings* → *Change visibility*
   → **Public**. C'est l'étape qu'on oublie le plus souvent, et sans elle
   umbrelOS ne pourra pas télécharger l'image.

## Comment l'installer dans Umbrel

1. Dans umbrelOS : *Paramètres → App Store → Ajouter un app store
   communautaire*, colle l'URL de ton repo GitHub (celui de l'étape
   précédente — pas besoin d'un `umbrel-app-store.yml` séparé si tu n'as que
   cette seule app dedans... en fait si, umbrelOS l'exige à la racine du
   repo : ajoute-le s'il n'existe pas déjà :

   ```yaml
   id: "mon-store"
   name: "Mon App Store"
   ```

2. L'app "SlvCtrl+" apparaît dans ce store, installe-la normalement.

## ⚠️ Point important : l'URL du backend

Le frontend ne passe PAS par la page principale de l'app pour parler au
serveur : il retient l'URL du backend dans le `localStorage` du navigateur et
appelle directement `http://<ip>:1337` en REST + WebSocket (comportement du
frontend lui-même, pas une limitation de ce packaging).

**Après l'installation**, ouvre l'app, va dans *Settings*, et renseigne :

```
http://<ip-ou-nom-de-ton-Umbrel>:1337
```

## ⚠️ Accès matériel (série / USB / Bluetooth)

Le service `server` tourne avec `privileged: true` + `network_mode: host`
pour reproduire l'accès que le script bash donnait en installant directement
sur le Pi (accès aux ports série/USB, et au Bluetooth via D-Bus/BlueZ, que
Docker ne peut pas "pontuer" proprement en réseau isolé — c'est exactement le
même schéma que l'app officielle "ee-gateway" du store Umbrel pour son
service Bluetooth). C'est documenté en commentaire dans `docker-compose.yml`,
avec une variante plus restreinte (sans réseau host) si tu n'as besoin que du
port série/USB, pas du Bluetooth.

Si ton appareil est câblé sur les pins GPIO série du Pi (UART matériel, pas
un adaptateur USB), il faut toujours activer l'UART une fois, côté host,
hors Umbrel : `sudo raspi-config` → *Interface Options* → *Serial Port*
(comme le faisait `raspi-config nonint do_serial_hw 0` dans le script
original — Umbrel ne peut pas le faire à ta place).

## Test rapide en SSH, hors Umbrel (optionnel)

Pour vérifier que les `Dockerfile` buildent bien avant de passer par GitHub
Actions :

```bash
scp -P 40000 -r slvctrlplus antoine@192.168.1.137:~/slvctrlplus
ssh -p 40000 antoine@192.168.1.137
cd ~/slvctrlplus
APP_DATA_DIR=./data docker compose -f docker-compose.yml build \
  --build-arg SLVCTRLPLUS_SERVER_REF=latest server
```

(note le `-p` pour le port SSH — pas `host:port`, comme discuté plus haut).

## Pour une vraie soumission à l'App Store officiel Umbrel

Le repo officiel `getumbrel/umbrel-apps` demande en plus des images déjà
épinglées par digest (`image:...@sha256:...`, pas juste `:latest`), et que
`privileged: true` / `network_mode: host` soient solidement justifiés en
review (voir `.agents/skills/umbrel-package-app/SKILL.md` de ce repo). Pas
nécessaire pour un usage personnel sur ton propre Umbrel.
