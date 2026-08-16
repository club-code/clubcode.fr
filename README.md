# clubcode.fr

Site vitrine du Club Code de Télécom SudParis. Une page statique, sous Astro 5, FR / EN.

## Démarrage

Le projet utilise Node.js **22.22.2** (voir [`.nvmrc`](.nvmrc)) et npm 10 ou plus récent.

```bash
nvm use
npm ci
npm run dev
```

Astro démarre un serveur de développement local et affiche l'URL dans le terminal. La langue par défaut est le français ; l'anglais est accessible via `/en/`.

## Commandes

| Commande | Description |
| --- | --- |
| `npm run dev` | Démarre le serveur de développement local. |
| `npm run check` | Exécute les diagnostics Astro et TypeScript. |
| `npm run lint` | Vérifie le code avec ESLint. |
| `npm run build` | Génère le site statique de production dans `dist/`. |
| `npm run preview` | Prévisualise le build de production localement. |
| `npm run serve` | Prévisualise le build sur toutes les interfaces réseau (`0.0.0.0:8080`). |

Avant de soumettre une pull request ou de pousser :

```bash
npm run lint
npm run check
npm run build
```

## Structure du projet

- `src/i18n/` : **tous les textes FR/EN et les liens** (`ui.ts`, `fr.ts`, `en.ts`). C'est le seul endroit à modifier pour le contenu.
- `src/components/` : sections et composants de la page.
- `src/layouts/` : layout principal.
- `src/styles/` : styles globaux et variables CSS.
- `public/` : médias statiques, polices, favicons.
- `deploy/` : script de déploiement et configuration nginx.

---

## Déploiement

`npm run build` produit un dossier `dist/` entièrement statique, servi par n'importe quel serveur web (nginx, Caddy, etc.).

### 1. Déploiement manuel sur le serveur

Pour compiler et déployer manuellement sur le serveur cible avec le script fourni :

```bash
sudo ./deploy/deploy.sh
```

Le script installe les dépendances avec `npm ci`, compile le site (`npm run build`), synchronise le contenu de `dist/` vers `/var/www/clubcode.fr/`, assigne les permissions à `www-data:www-data`, et recharge `nginx`.

Vous pouvez surcharger les variables d'environnement si nécessaire :
```bash
WEBROOT=/var/www/clubcode.fr WEB_USER=www-data sudo -E ./deploy/deploy.sh
```

Consultez [`deploy/nginx.conf`](deploy/nginx.conf) pour la configuration Nginx complète avec support SSL / Let's Encrypt, compression gzip, politique de cache et en-têtes de sécurité.

---

### 2. Déploiement automatique (CI/CD GitHub Actions via Self-Hosted Runner)

Le workflow GitHub Actions ([`.github/workflows/ci.yml`](.github/workflows/ci.yml)) fonctionne en deux étapes :
1. **`validate` (sur runner GitHub `ubuntu-latest`)** : Vérifie le linting, les types TypeScript/Astro et génère l'artefact `dist/`.
2. **`deploy` (sur runner `self-hosted` sur la VM)** : Télécharge l'artefact compilé directement sur la machine hôte et met à jour `/var/www/clubcode.fr/`.

Cette architecture ne nécessite **aucun port entrant ni VPN ouvert vers l'extérieur** sur OPNsense : le runner interroge GitHub en HTTPS sortant uniquement.

#### A. Installation du runner GitHub sur la VM

1. Dans votre dépôt GitHub, allez dans **Settings → Actions → Runners → New self-hosted runner**.
2. Sélectionnez l'architecture (ex. Linux / x64) et suivez les commandes fournies sur la VM :
   ```bash
   # Créer un utilisateur ou dossier dédié (ex. /opt/actions-runner ou ~/actions-runner)
   mkdir -p ~/actions-runner && cd ~/actions-runner

   # Télécharger et extraire le runner (adapter la version selon GitHub)
   curl -o actions-runner-linux-x64.tar.gz -L https://github.com/actions/runner/releases/download/vX.Y.Z/actions-runner-linux-x64-X.Y.Z.tar.gz
   tar xzf ./actions-runner-linux-x64.tar.gz

   # Configurer le runner avec le token fourni par GitHub
   ./config.sh --url https://github.com/OWNER/REPO --token YOUR_TOKEN
   ```
3. Installez et activez le service systemd pour que le runner tourne en arrière-plan au démarrage :
   ```bash
   sudo ./svc.sh install
   sudo ./svc.sh start
   sudo ./svc.sh status
   ```

#### B. Configuration des droits sudo sur la VM

Pour permettre au runner de synchroniser les fichiers vers le dossier web et recharger nginx sans mot de passe, créez le fichier `/etc/sudoers.d/github-runner` :

```sudoers
# Remplacer 'runner-user' par le compte système exécutant le runner (ex. debian, ubuntu, etc.)
runner-user ALL=(ALL) NOPASSWD: /usr/bin/rsync, /usr/bin/chown, /bin/systemctl reload nginx, /usr/sbin/service nginx reload
```

Puis sécurisez les permissions :
```bash
sudo chmod 440 /etc/sudoers.d/github-runner
```

#### C. Création du répertoire web

Assurez-vous que le dossier de destination existe :
```bash
sudo mkdir -p /var/www/clubcode.fr
sudo chown -R www-data:www-data /var/www/clubcode.fr
```

#### D. Configuration de l'environnement GitHub

Dans le dépôt GitHub :
1. Allez dans **Settings → Environments**.
2. Créez un environnement nommé `production`.
3. (Optionnel) Configurez les règles de protection (ex: approbation manuelle avant déploiement si désiré).
