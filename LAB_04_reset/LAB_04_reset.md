# Lab `git reset`

---

## 1. Fondements Architecturaux de `git reset` en DevSecOps

Dans l'écosystème Git, `git reset` est le moteur principal de manipulation des trois arbres fondamentaux : **`HEAD`**, l'**Index (Staging Area)** et le **Working Directory**. 

Contrairement à `git revert` (qui applique un patch inverse en créant un nouveau commit), `git reset` **déplace physiquement le pointeur de la branche courante** vers un commit antérieur du DAG (*Directed Acyclic Graph*).

```text
       Modèle d'Exécution Séquentielle de "git reset" :

                    +-----------------------------+
                    |  git reset --soft <target>  |
                    +-----------------------------+
                                   |
                       [1. Déplacement de HEAD]
                                   v
                    +-----------------------------+
                    |  git reset --mixed <target> |  (Comportement par défaut)
                    +-----------------------------+
                                   |
                       [2. Écrasement de l'Index]
                                   v
                    +-----------------------------+
                    |  git reset --hard <target>  |
                    +-----------------------------+
                                   |
                 [3. Écrasement du Working Directory]
```

### 1.1. Les Trois Paliers d'Arrêt
1. **`--soft` (Étape 1)** : Déplace le pointeur `HEAD` et le nom de la branche. L'Index et le répertoire de travail restent inchangés. Les deltas entre l'ancien et le nouveau `HEAD` deviennent immédiatement indexés (*staged*).
2. **`--mixed` (Étapes 1 + 2)** : Déplace `HEAD` et écrase l'Index avec l'état du commit cible. Les deltas restent présents sur le disque sous forme de modifications non indexées (*unstaged*).
3. **`--hard` (Étapes 1 + 2 + 3)** : Déplace `HEAD`, réinitialise l'Index et écrase le Working Directory. Tout changement non commité en rapport avec la cible est supprimé du système de fichiers.

> **La Règle d'Or DevSecOps du Reset :**  
> **`git reset` est un outil d'hygiène strictement réservé aux branches locales non publiées.** Tout recours à `git reset` suivi d'un `git push --force` sur une branche partagée (`main`, `develop`, branches de release) constitue une rupture de la traçabilité cryptographique, brise les pipelines CI/CD des pairs et contrevient aux exigences d'auditabilité (SOC 2, ISO 27001). **Sanitisez localement avec `reset`, publiez immuablement avec `revert`.**

---

## 2. Provisionnement du Laboratoire

Initialisez un environnement d'expérimentation isolé :

```bash
# Nettoyage et initialisation du répertoire
rm -rf /tmp/git-reset-lab
mkdir -p /tmp/git-reset-lab && cd /tmp/git-reset-lab

# Création du dépôt local
git init core-platform-service
cd core-platform-service

# Configuration de l'identité
git config user.name "devsecops-engineer"
git config user.email "engineer@platform.local"

# Socle initial sur main
echo "# Core Security Platform" > README.md
git add README.md
git commit -m "chore: initial repository baseline"
git branch -M main
```

> **Conseil de surveillance :** Ouvrez un terminal adjacent et exécutez l'inspection continue de l'arbre :
> ```bash
> watch -n 1 -c git log --graph --oneline --all --decorate --color
> ```

---

## 3. Scénario 1 : `--soft` — Consolidation Atomique (Squash) Pré-Pull Request

### 3.1. Contexte Opérationnel
Un ingénieur développe un service de validation de jetons JWT. Durant la phase d'implémentation locale, il produit une série de micro-commits désordonnés et de corrections de syntaxe. Avant d'ouvrir sa Pull Request, les règles de gouvernance exigent de fusionner ces deltas en un commit atomique unique, conforme à la spécification des **Conventional Commits**.

---

### 3.2. Reproduction des Micro-Commits Locaux

```bash
git checkout -b feature/jwt-validator

# Commit 1 : Structure initiale
cat << 'EOF' > jwt_validator.py
class TokenValidator:
    def __init__(self, public_key):
        self.public_key = public_key
EOF
git add jwt_validator.py
git commit -m "wip: scaffold validator class"

# Commit 2 : Algorithme de vérification
cat << 'EOF' >> jwt_validator.py
    def verify(self, token):
        # Simulation validation signature
        return token.startswith("ey")
EOF
git add jwt_validator.py
git commit -m "wip: add token prefix checking"

# Commit 3 : Correction de documentation
echo "# Compliant with RFC 7519" >> jwt_validator.py
git add jwt_validator.py
git commit -m "docs: add rfc reference comment"
```

Inspection de l'historique pollué :
```bash
git log --oneline -4
```
```text
* d3e4f5a (HEAD -> feature/jwt-validator) docs: add rfc reference comment
* c2b3a4d wip: add token prefix checking
* b1a2c3d wip: scaffold validator class
* 1a2b3c4 (main) chore: initial repository baseline
```

---

### 3.3. Exécution du Reset Soft & Consolidation

Déplacez le pointeur `HEAD` au commit d'origine (`main`), sans toucher à la zone de transit :

```bash
git reset --soft main
```

Vérifiez l'état de l'arbre et du staging :
```bash
git status
```
**Sortie standard attendue :**
```text
On branch feature/jwt-validator
Changes to be committed:
  (use "git restore --staged <file>..." to unstage)
	new file:   jwt_validator.py
```

Inspectez le graphe :
```bash
git log --oneline -1
```
```text
* 1a2b3c4 (HEAD -> feature/jwt-validator, main) chore: initial repository baseline
```

`HEAD` pointe désormais sur le commit de base, mais **l'ensemble des modifications cumulées des 3 commits précédents est préservé dans l'Index**.

Finalisez la consolidation avec un commit propre et auditable :
```bash
git commit -m "feat(auth): implement RFC 7519 compliant JWT token validator"
```

```bash
git log --oneline -2
```
```text
* e8f9a0b (HEAD -> feature/jwt-validator) feat(auth): implement RFC 7519 compliant JWT token validator
* 1a2b3c4 (main) chore: initial repository baseline
```

---

## 4. Scénario 2 : `--mixed` — Dé-staging et Séparation des Préoccupations (SoC)

### 4.1. Contexte Opérationnel
Dans l'urgence, un développeur a commité au sein d'une même transaction deux modifications sémantiquement indépendantes : 
1. Une règle de durcissement d'infrastructure Terraform (`main.tf`).
2. Une mise à jour applicative non liée (`app.py`).

D'un point de vue DevSecOps (principe de moindre privilège et déclenchement ciblé des pipelines de déploiement), le pipeline Terraform ne doit pas s'exécuter sur une modification applicative. Il faut casser ce commit pour créer **deux commits distincts et atomiques**.

---

### 4.2. Reproduction du Commit Composite

```bash
git checkout main

# Ajout d'infrastructure
cat << 'EOF' > main.tf
resource "aws_s3_bucket" "audit_logs" {
  bucket = "prod-audit-logs"
  acl    = "private"
}
EOF

# Ajout applicatif
cat << 'EOF' > app.py
def healthcheck():
    return {"status": "healthy", "code": 200}
EOF

# Commit unifié (erreur de découpage architectural)
git add main.tf app.py
git commit -m "feat: add s3 audit bucket and healthcheck endpoint"
```

---

### 4.3. Exécution du Reset Mixed & Restructuration

Exécutez `git reset` (le mode `--mixed` est le comportement implicite) :

```bash
git reset HEAD~1
```

**Sortie attendue :**
```text
Unstaged changes after reset:
M	app.py
M	main.tf
```

Vérifiez l'Index :
```bash
git status
```
```text
On branch main
Untracked files:
  (use "git add <file>..." to include in what will be committed)
	app.py
	main.tf
```

#### Séparation et Double Commit Atomique :

```bash
# 1. Commit Infrastructure / DevSecOps
git add main.tf
git commit -m "infra(s3): provision private audit logs storage bucket"

# 2. Commit Applicatif
git add app.py
git commit -m "feat(api): expose liveness healthcheck endpoint"
```

Inspection finale :
```bash
git log --oneline -2
```
```text
* f1e2d3c (HEAD -> main) feat(api): expose liveness healthcheck endpoint
* a0b1c2d infra(s3): provision private audit logs storage bucket
```

> **Gain DevSecOps :** Le webhook CI détectant les changements sur `**/*.tf` ne déclenchera l'analyse Trivy/Checkov que pour le premier commit, isolant les responsabilités et facilitant les rollbacks futurs.

---

## 5. Scénario 3 : `--hard` — Purge Intégrale d'un Spike Défectueux

### 5.1. Contexte Opérationnel
Un ingénieur a mené une expérimentation locale (*spike*) pour tester une bibliothèque de chiffrement obsolète. L'espace de travail est pollué par des fichiers temporaires, du code corrompu et des dépendances cassées. L'ensemble doit être éradiqué pour restaurer l'état stable précédent sans laisser le moindre artefact résiduel.

---

### 5.2. Reproduction de la Corruption Locale

```bash
# Modification du code existant
echo "CORRUPTED MEMORY LEAK LOGIC" >> app.py

# Création d'artefacts non désirés
echo "insecure_mock_key = 12345" > debug_secret.key
git add app.py debug_secret.key
git commit -m "test(crypto): experiment with insecure legacy cipher"

# Modifications en cours non stagées
echo "malformed syntax line" >> app.py
```

Vérification de l'état altéré :
```bash
git status -s
```
```text
 M app.py
```

---

### 5.3. Exécution de la Purge Destructive

Réinitialisez intégralement l'Index et l'espace de travail au commit parent :

```bash
git reset --hard HEAD~1
```

**Sortie standard :**
```text
HEAD is now at f1e2d3c feat(api): expose liveness healthcheck endpoint
```

#### Vérification de la Complétude de la Purge :
```bash
git status
# "nothing to commit, working tree clean"

ls debug_secret.key
# ls: cannot access 'debug_secret.key': No such file or directory

cat app.py
# Vérification que la ligne corrompue a disparu
```

---

## 6. Scénario 4 : Récupération Forensique Post-`--hard` via `git reflog`

### 6.1. Contexte Opérationnel (Plan de Reprise d'Activité)
Un développeur junior a exécuté par mégarde `git reset --hard HEAD~1` sur sa branche locale, croyant purger une modification non désirée. Il a détruit un commit essentiel contenant l'implémentation d'une fonction de hachage sécurisée (`hash_data`). Ce commit n'a jamais été poussé sur le serveur distant. 

L'ingénieur DevSecOps intervient pour **sauvegarder les objets orphelins avant leur collecte par le ramasse-miettes (*Garbage Collector*)**.

---

### 6.2. Reproduction de l'Incident Critique

```bash
git checkout -b feature/sec-hashing

# Création du commit à valeur critique
cat << 'EOF' > hashing.py
import hashlib

def generate_sha256(data: str) -> str:
    return hashlib.sha256(data.encode('utf-8')).hexdigest()
EOF
git add hashing.py
git commit -m "feat(crypto): add deterministic sha256 hashing utility"

# Vérification de l'existence du commit
TARGET_LOST_SHA=$(git rev-parse --short HEAD)
echo "Commit à secourir: $TARGET_LOST_SHA"

# FAUSSE MANŒUVRE : Destruction via reset --hard
git reset --hard HEAD~1
```

Vérification : le fichier a disparu du disque et de l'historique :
```bash
ls hashing.py
# ls: cannot access 'hashing.py': No such file or directory
git log --oneline -1
```

---

### 6.3. Analyse Forensique et Extraction via `reflog`

Git conserve un journal d'audit de chaque mise à jour du pointeur `HEAD` dans `.git/logs/HEAD`. 

Exécutez l'interrogation du journal de référence :
```bash
git reflog -n 5
```

**Exemple de sortie :**
```text
f1e2d3c (HEAD -> feature/sec-hashing) HEAD@{0}: reset: moving to HEAD~1
9c4a8b1 HEAD@{1}: commit: feat(crypto): add deterministic sha256 hashing utility
f1e2d3c (HEAD -> feature/sec-hashing) HEAD@{2}: checkout: moving from main to feature/sec-hashing
```

Le commit n'est pas détruit : il est simplement déréférencé du graphe actif. Son SHA (ici `HEAD@{1}` ou `9c4a8b1`) existe toujours dans le magasin d'objets (`.git/objects`).

---

### 6.4. Reconstruction et Rétablissement de l'Intégrité

Pour restaurer l'état sans risque de collision, deux approches s'offrent à l'ingénieur :

#### Approche A : Réancrage Direct via `reset --hard`
```bash
git reset --hard HEAD@{1}
```

#### Approche B (Méthode Recommandée DevSecOps - Non Invasive) :
Matérialiser le commit orphelin sur une nouvelle branche d'investigation :
```bash
git branch rescue-hashing HEAD@{1}
git checkout rescue-hashing
```

#### Validation de la Récupération :
```bash
ls -la hashing.py
cat hashing.py
git log --oneline -1
```

```text
* 9c4a8b1 (HEAD -> rescue-hashing) feat(crypto): add deterministic sha256 hashing utility
```

> **Résultat d'Audit :** 100% du code et de l'historique cryptographique ont été recouvrés sans aucune altération de données.

---

## 7. Synthèse Matricielle pour la Gouvernance DevSecOps

| Paramètre d'Analyse               | `git reset --soft`                      | `git reset --mixed`                      | `git reset --hard`                   |
|:----------------------------------|:----------------------------------------|:-----------------------------------------|:-------------------------------------|
| **Pointeur `HEAD`**               | Réassigné au commit cible               | Réassigné au commit cible                | Réassigné au commit cible            |
| **Zone de Transit (Index)**       | **Préservée** (Conserve les deltas)     | **Réinitialisée** (Deltas exclus)        | **Écrasée** (Deltas supprimés)       |
| **Espace de Travail (Disque)**    | **Intact**                              | **Intact** (Fichiers non indexés)        | **Écrasé** (Fichiers réalignés)      |
| **Récupération des non-commités** | Totale                                  | Totale                                   | **Impossible** (Perte définitive)    |
| **Usage CI/CD et Collaboration**  | Local pré-push uniquement               | Local pré-push uniquement                | Runners CI / Environnements jetables |
| **Gouvernance de Sécurité**       | Haute sécurité (Nettoyage d'historique) | Haute sécurité (Découpage de privilèges) | Risque opérationnel élevé            |