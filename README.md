# Dispositif de veille technique, artefacts

![N8N](https://img.shields.io/badge/n8n-1.72.1-EA4B71?logo=n8n&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-compose-2496ED?logo=docker&logoColor=white)
![Tavily](https://img.shields.io/badge/Tavily-recherche%20web-6E56CF)
![Groq](https://img.shields.io/badge/Groq-Llama%203.3%2070B-F55036)
![Conventional Commits](https://img.shields.io/badge/Conventional%20Commits-1.0.0-FE5196?logo=conventionalcommits&logoColor=white)
![License](https://img.shields.io/badge/license-MIT-green)

Ce dépôt contient les artefacts techniques du dispositif de veille du projet de tableau de bord intelligent pour bibliothèques publiques mené dans le cadre de mon alternance dans l'entreprise AFI, et de la certification RNCP 37827. Il porte la configuration exécutable du dispositif, docker-compose, variables d'environnement d'exemple, exports des workflows N8N, prompts Groq versionnés, journal de rotation des angles de veille.

## Contenu du dépôt

```
.
├── docker-compose.yml         Configuration Docker de N8N
├── .env.example               Modèle du fichier d'environnement
├── .gitignore                 Exclusions Git
├── README.md                  Le présent fichier
├── workflows/                 Exports JSON des workflows N8N
├── prompts/                   Prompts Groq versionnés
└── journal-angles.md          Journal de rotation des angles de veille
```

Les dossiers `workflows/` et `prompts/`, ainsi que le fichier `journal-angles.md`, sont créés au fil de l'eau, ils apparaissent à mesure que le dispositif se remplit.

## Prérequis

- Docker Engine avec le plugin compose, ou Docker Desktop. Version minimale conseillée, Docker 24.
- Un vault Obsidian existant, avec un dossier de destination des synthèses prêt à recevoir des écritures.
- Un compte Tavily, formule gratuite suffisante, 1000 requêtes par mois.
- Un compte Groq dédié au projet AFI, séparé de tout autre compte personnel.

## Installation

### 1. Cloner le dépôt

```bash
git clone git@github.com:votre-username/veille-dispositif-afi.git
cd veille-dispositif-afi
```

### 2. Générer les secrets

Deux valeurs sont à générer, elles sont utilisées dans le fichier `.env`.

```bash
# Mot de passe de l'authentification basic N8N
openssl rand -base64 24

# Cle de chiffrement des credentials N8N, a conserver precieusement
openssl rand -hex 32
```

La clé de chiffrement N8N est particulièrement sensible. Si elle est perdue, les credentials Tavily et Groq stockés dans le volume Docker deviennent illisibles et doivent être re-saisis intégralement. Elle est à sauvegarder dans un gestionnaire de secrets ou dans un endroit sûr, jamais dans le dépôt.

### 3. Créer le fichier d'environnement

```bash
cp .env.example .env
```

Éditer `.env` et renseigner les quatre variables.

- `N8N_BASIC_AUTH_USER`, identifiant de connexion à l'interface N8N.
- `N8N_BASIC_AUTH_PASSWORD`, mot de passe généré à l'étape précédente.
- `N8N_ENCRYPTION_KEY`, clé de chiffrement générée à l'étape précédente.
- `OBSIDIAN_VAULT_PATH`, chemin absolu vers la racine du vault Obsidian sur la machine hôte.

### 4. Vérifier le dossier de destination dans le vault

Le workflow de veille écrit ses synthèses dans un sous-dossier hebdomadaire du vault. Le dossier parent doit exister avant le premier démarrage.

```bash
mkdir -p "$OBSIDIAN_VAULT_PATH/Nom_du_dossier"
```

### 5. Démarrer le conteneur

```bash
docker compose up -d
docker compose logs -f n8n
```

Attendre le message indiquant que N8N est disponible sur le port 5678, puis interrompre le suivi des logs par Ctrl+C, le conteneur continue de tourner en arrière-plan.

### 6. Configurer N8N via l'interface web

Ouvrir un navigateur sur `http://localhost:5678`. À la première connexion, N8N demande de créer un compte propriétaire, dissocié de l'authentification basic. Une fois connecté, aller dans `Credentials` et créer deux entrées.

- Credential Tavily, avec la clé API récupérée sur `app.tavily.com`.
- Credential Groq, avec la clé API récupérée sur `console.groq.com`, compte AFI dédié.

### 7. Importer les workflows

Les exports JSON des workflows sont dans le dossier `workflows/`. Depuis l'interface N8N, menu `Workflows`, bouton `Import from File`, sélectionner le workflow à importer, puis attacher les credentials Tavily et Groq aux nœuds correspondants.

## Utilisation courante

Le workflow principal est déclenché manuellement au moment du créneau hebdomadaire de veille. Depuis l'interface N8N, ouvrir le workflow, cliquer sur `Execute Workflow`. La synthèse est écrite dans le vault, elle est immédiatement disponible dans Obsidian.

Le prompt système Groq utilisé par le workflow est versionné dans `prompts/scoring-groq-v1.md`. Toute modification du prompt donne lieu à un nouveau fichier `prompts/scoring-groq-vX.md` et à une mise à jour du workflow pour pointer vers la nouvelle version.

## Sauvegarde et restauration

Les données persistantes de N8N (workflows, credentials chiffrés, historique d'exécution) vivent dans le volume Docker nommé `veille-n8n-data`. Ce volume n'est pas dans le dépôt Git, il faut le sauvegarder séparément.

Sauvegarde ponctuelle du volume.

```bash
docker run --rm \
  -v veille-n8n-data:/data \
  -v "$PWD:/backup" \
  alpine tar czf /backup/n8n-data-$(date +%Y%m%d).tar.gz -C /data .
```

Restauration sur une nouvelle machine, après avoir cloné le dépôt et créé `.env`.

```bash
docker volume create veille-n8n-data
docker run --rm \
  -v veille-n8n-data:/data \
  -v "$PWD:/backup" \
  alpine tar xzf /backup/n8n-data-AAAAMMJJ.tar.gz -C /data
docker compose up -d
```

## Arrêt et redémarrage

```bash
# Arret propre du conteneur, le volume est preserve
docker compose down

# Redemarrage
docker compose up -d

# Mise a jour de la version de N8N, apres avoir modifie l'image dans docker-compose.yml
docker compose pull
docker compose up -d
```

## Convention de contribution

Ce dépôt suit une stratégie mono branche, main est la seule branche. Chaque modification est un commit direct sur main, avec un message respectant la convention Conventional Commits, `type(scope): description en français`.

Types utilisés, feat, fix, chore, docs, refactor.

Scopes propres à ce dépôt, docker, env, workflow, prompt, angles, docs.

Exemples.

- `chore(docker): bump n8n vers 1.75.0`
- `feat(angles): rotation trimestrielle Q4, remplacer A2 par angle monitoring modele`
- `feat(prompt): version v1.1 du prompt de scoring Groq`
- `fix(workflow): corriger le filtre de deduplication Levenshtein`

Les changements structurants (bump de version d'image, refonte des angles, changement de version des prompts) sont accompagnés d'un tag annoté.

```bash
git tag -a v1.0.0 -m "Version initiale du dispositif"
git push origin v1.0.0
```

## Licence

Ce dépôt est distribué sous licence MIT. Le texte complet de la licence est dans le fichier `LICENSE` à la racine du dépôt.