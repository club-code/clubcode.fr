# clubcode.fr

Site vitrine du Club Code de Télécom SudParis. Une page statique, sous Astro 5, FR / EN.

## Commandes

| Commande | Description |
| --- | --- |
| `npm run dev` | Démarre le serveur de développement local. |
| `npm run check` | Exécute les diagnostics Astro et TypeScript. |
| `npm run lint` | Vérifie le code avec ESLint. |
| `npm run build` | Génère le site statique de production dans `dist/`. |
| `npm run preview` | Prévisualise le build de production localement. |
| `npm run serve` | Prévisualise le build sur toutes les interfaces réseau (`0.0.0.0:8080`). |

Avant de soumettre une pull request ou de push :

```bash
npm run lint
npm run check
npm run build
```

## Structure du projet

- `src/i18n/` : tous les textes FR/EN et les liens (`ui.ts`, `fr.ts`, `en.ts`).
- `src/components/` : sections et composants de la page.
- `src/layouts/` : layout principal.
- `src/styles/` : styles globaux et variables CSS.
- `public/` : médias statiques, polices, favicons.
- `deploy/` : script de déploiement et configuration nginx.

---

## Déploiement

### 1. Déploiement manuel sur le serveur

Pour compiler et déployer manuellement sur le serveur cible avec le script fourni :

```bash
sudo ./deploy/deploy.sh
```

Le script installe les dépendances avec `npm ci`, compile le site (`npm run build`), synchronise le contenu de `dist/` vers `/var/www/clubcode.fr/`, assigne les permissions à `www-data:www-data`, et recharge `nginx`.

Vous pouvez charger les variables d'environnement si nécessaire :
```bash
WEBROOT=/var/www/clubcode.fr WEB_USER=www-data sudo -E ./deploy/deploy.sh
```

Consultez [`deploy/nginx.conf`](deploy/nginx.conf) pour la configuration Nginx complète avec support SSL / Let's Encrypt, compression gzip, politique de cache et en-têtes de sécurité.

---

### 2. Déploiement automatique (CI/CD GitHub Actions via Self-Hosted Runner)

Le workflow GitHub Actions ([`.github/workflows/ci.yml`](.github/workflows/ci.yml)) fonctionne en deux étapes :
1. **`validate` (sur runner GitHub `ubuntu-latest`)** : Vérifie le linting, les types TypeScript/Astro et génère l'artefact `dist/`.
2. **`deploy` (sur runner `self-hosted` sur la VM)** : Télécharge l'artefact compilé directement sur la machine hôte et met à jour `/var/www/clubcode.fr/`.

#### A. Installation du runner GitHub sur la VM

1. Dans votre dépôt GitHub, allez dans **Settings → Actions → Runners → New self-hosted runner**.
2. Suivez les informations fournies par GitHub sur cette page.
3. Installez et activez le service systemd pour que le runner tourne en arrière-plan au démarrage :
   ```bash
   sudo ./svc.sh install
   sudo ./svc.sh start
   sudo ./svc.sh status
   ```

#### B. Configuration des droits sudo sur la VM

Pour permettre au runner de synchroniser les fichiers vers le dossier web sans mot de passe, créez le fichier `/etc/sudoers.d/github-runner` :

```sudoers
# Remplacer 'runner-user' par le compte exécutant le runner.
runner-user ALL=(ALL) NOPASSWD: /usr/bin/rsync, /usr/bin/chown
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
