# Lab: Surgical Commit Grafting and Integrity Traceability with `git cherry-pick`

---

## 1. Fondements Architecturaux de `git cherry-pick` en DevSecOps

Dans le modèle de graphe orienté acyclique (DAG) de Git, **`git cherry-pick` est un moteur de transplantation chirurgicale de deltas**. Il permet de sélectionner un ou plusieurs commits arbitraires situés sur une branche source et de rejouer leurs modifications différentielles exactes sur la branche cible (`HEAD`), **sans fusionner la branche d'origine**.

```text
État Initial : Divergence des flux de développement
main (Production) :       [C1] <--- [C2] (HEAD)
                            \
feature/experimental :       [C3] <--- [C4 (Fix CVE-2026-X)] <--- [C5 (WIP instable)]

-----------------------------------------------------------------------------------------
Exécution : git checkout main && git cherry-pick C4

Résultat : Transplantation sélective et génération d'un nouveau nœud
main (Production) :       [C1] <--- [C2] <--- [C4' (Nouveau SHA - Fix CVE seul)] (HEAD)
                            \
feature/experimental :       [C3] <--- [C4] <--- [C5 (WIP instable)]
```

### 1.1. Mécanique Interne : La Fusion à Trois Voies Déportée
Lors de l'application de `git cherry-pick <commit-cible>`, Git ne copie pas bêtement des fichiers : il exécute un **algorithme de fusion à 3 branches (*3-way merge*)** calculé comme suit :
1. **Base commune ($O$)** : Le parent direct du commit ciblé (`<commit-cible>~1`).
2. **Notre branche ($A$)** : L'état actuel de la branche cible (`HEAD`).
3. **Leur branche ($B$)** : L'état exact du `<commit-cible>`.

Git extrait le différentiel $(B - O)$ et tente de l'appliquer sur $A$. Cette opération engendre **nécessairement un nouveau hash cryptographique (SHA)** pour le commit résultant ($C4'$), car son parent, son horodatage (*timestamp*) et son committer diffèrent du commit original ($C4$), bien que l'auteur initial et le patch soient préservés.

### 1.2. Justification DevSecOps & Conformité Réglementaire
Dans les architectures de CD et de gestion des vulnérabilités :
* **Isolation stricte des risques (Blast Radius)** : Lorsqu'une faille critique (CVE) est corrigée sur une branche de développement active aux côtés de fonctionnalités inachevées ou instables, il est proscrit de fusionner (`merge`) toute la branche vers la production. `cherry-pick` permet d'extraire **exclusivement** le correctif de sécurité.
* **Traçabilité de provenance (*Provenance Tracking*)** : La duplication de patches d'une branche à une autre sans métadonnées crée des commits "fantômes". Pour satisfaire aux exigences de conformité (**SOC 2, ISO 27001, SLSA L3**), chaque transplantation doit conserver la chaîne de traçabilité liant le nouveau commit à son commit parent source via l'option `-x`.

> **La Règle d'Or DevSecOps du Cherry-Pick :**  
> **N'appliquez jamais un cherry-pick sans traçabilité de provenance explicite.** Tout commit transplanté vers une branche de production doit inclure la référence cryptographique d'origine (`-x`) et être signé numériquement (`-S` ou `-s`) pour garantir la non-répudiation et préserver l'intégrité de la chaîne d'approvisionnement logicielle (*Software Supply Chain*).

---

## 2. Provisionnement du Labo

Initialisez un environnement de travail hermétique :

```bash
# Nettoyage et initialisation du répertoire
rm -rf /tmp/git-cherry-pick-lab
mkdir -p /tmp/git-cherry-pick-lab && cd /tmp/git-cherry-pick-lab

# Création du dépôt local
git init api-security-gateway
cd api-security-gateway

# Configuration de l'identité de l'auditeur
git config user.name "devsecops-lead"
git config user.email "lead@enterprise.internal"

# Socle initial de production (v1.0.0-stable)
cat << 'EOF' > server.py
import os

def handle_request(path):
    print(f"Routing to: {path}")
EOF

echo "v1.0.0-stable" > version.txt
git add server.py version.txt
git commit -m "COMMIT 1 :: chore: initial production baseline v1.0.0"
git branch -M main
```

> **Surveillance recommandée :** Dans un second terminal, inspectez la dynamique du DAG :
> ```bash
> watch -n 1 -c git log --graph --oneline --all --decorate --color
> ```

---

## 3. Scénario 1 : Rapatriement Chirurgical d'un Hotfix de Sécurité sans Charge Utile Instable

### 3.1. Contexte Opérationnel
L'équipe de R&D travaille sur une refonte expérimentale majeure de l'API (`feature/nextgen-api`). Durant leurs tests de fuzzing, ils identifient et corrigent une **vulnérabilité critique de Path Traversal**. 

La branche de R&D contient du code hautement expérimental et non prêt pour la production. En tant qu'ingénieur DevSecOps, vous devez extraire **uniquement** le commit de correction de sécurité pour le déployer sur `main`, sans importer le code inachevé.

---

### 3.2. Reproduction de la Branche de Développement Divergente

```bash
git checkout -b feature/nextgen-api

# Commit 1 : Code expérimental instable (WIP)
cat << 'EOF' >> server.py
def unstable_async_worker():
    # Expérimentation non testée : fuite mémoire connue
    raise NotImplementedError("Work in progress")
EOF
git add server.py
git commit -m "COMMIT 2 :: chore : introduce experimental async worker pipeline"

# Commit 2 : Le correctif de vulnérabilité critique (CVE)
cat << 'EOF' > sanitizer.py
import re

def sanitize_path(path: str) -> str:
    # Blocage strict des attaques Path Traversal (../)
    return re.sub(r'\.\./', '', path)
EOF
git add sanitizer.py
git commit -m "COMMIT 3 :: sec(cve): mitigate path traversal vulnerability in request dispatcher"

# Commit 3 : Autre fonctionnalité non stabilisée
echo "DEBUG_TELEMETRY=true" >> config.env
git add config.env
git commit -m "COMMIT 4 :: chore(debug): enable raw memory profiling"
```

Inspection de la divergence sur `feature/nextgen-api` :
```bash
git log --oneline -4
```
```text
* 4c5d6e7 (HEAD -> feature/nextgen-api) chore(debug): enable raw memory profiling
* 3b4c5d6 sec(cve): mitigate path traversal vulnerability in request dispatcher
* 2a3b4c5 feat(worker): introduce experimental async worker pipeline
* 1a2b3c4 (main) chore: initial production baseline v1.0.0
```

---

### 3.3. Exécution de la Transplantation Chirurgicale

Identifiez dynamiquement le hash du commit de sécurité :
```bash
HOTFIX_COMMIT=$(git log --grep="mitigate path traversal" --format="%H")
echo "Commit de remédiation identifié : $HOTFIX_COMMIT"
```

Basculez sur la branche de production `main` et transplantez le commit :
```bash
git checkout main

# Application chirurgicale du commit
git cherry-pick $HOTFIX_COMMIT
```

#### Vérification de l'Intégrité de l'Arbre :
```bash
git log --oneline --graph -2
```
```text
* 7f8a9b0 (HEAD -> main) sec(cve): mitigate path traversal vulnerability in request dispatcher
* 1a2b3c4 chore: initial production baseline v1.0.0
```

Vérifiez que **seul** le composant de sécurité a été importé, à l'exclusion du code instable :
```bash
# Le composant de sécurité est bien présent
ls sanitizer.py
# Sortie : sanitizer.py

# Les artefacts instables (config.env, fuite mémoire dans server.py) SONT ABSENTS
ls config.env 2>/dev/null || echo "config.env est absent : CONFORME"
grep -q "unstable_async_worker" server.py || echo "Code expérimental exclu : CONFORME"
```

> **Gain DevSecOps :** Le rayon d'impact (*Blast Radius*) est strictement contenu. Le système de production est immunisé contre la CVE sans exposer les utilisateurs aux instabilités de la branche de R&D.

---

## 4. Scénario 2 : Auditabilité et Traçabilité Réglementaire avec `-x` et `--signoff`

### 4.1. Contexte Opérationnel
Le commit $7f8a9b0$ créé au scénario précédent pose un défi d'auditabilité : son hash a changé et rien dans son message n'indique formellement d'où provient ce patch dans le dépôt. Dans le cadre d'un audit de sécurité (ex. **SOC 2 Type II** ou **PCI-DSS**), vous devez prouver la généalogie du patch.

---

### 4.2. Application avec Enregistrement de Provenance

Nous allons annuler le cherry-pick précédent et le réappliquer selon les normes de gouvernance DevSecOps :

```bash
# Annulation du cherry-pick précédent
git reset --hard HEAD~1

# Ré-exécution conforme :
# -x       : injecte automatiquement la mention "(cherry picked from commit <SHA>)"
# -s / --signoff : injecte la mention formelle Signed-off-by (conformité DCO - Developer Certificate of Origin)
git cherry-pick -x -s $HOTFIX_COMMIT
#git commit --amend

#git cherry-pick -x -s -e $HOTFIX_COMMIT

#git cherry-pick -n -x -s $HOTFIX_COMMIT
#git commit -s -m "Mon message personnalisé incluant le correctif"
```

#### Examen Forensique des Métadonnées du Commit :
```bash
git log -1
```

**Sortie d'Audit Conforme :**
```text
commit 8d9e0f1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e (HEAD -> main)
Author: devsecops-lead <lead@enterprise.internal>
Date:   Mon May 18 10:00:00 2026 +0000

    sec(cve): mitigate path traversal vulnerability in request dispatcher

    (cherry picked from commit 3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2c)

    Signed-off-by: devsecops-lead <lead@enterprise.internal>
```

> **Résultat de Conformité :** La traçabilité cryptographique est assurée. Un auditeur peut corréler le commit déployé en production avec la pull request, les tests d'assurance qualité et les discussions associées sur la branche d'origine.

---

## 5. Scénario 3 : Transplantation d'une Plage de Commits (`range`) et Résolution de Conflits

### 5.1. Contexte Opérationnel
Un sous-système de validation de schéma de sécurité a été implémenté en deux commits séquentiels sur une branche tierce (`feature/schema-guard`). Le premier configure le parseur, le second applique un durcissement strict des règles. Lors du cherry-pick de cette séquence sur `main`, une modification concurrente sur le fichier de configuration provoque un conflit d'intégration.

---

### 5.2. Reproduction de la Chaîne et de la Collision

```bash
# Évolution parallèle sur main
cat << 'EOF' >> server.py
# Security middleware configuration
SECURITY_PROFILE = "STRICT_AUDIT"
EOF
git add server.py
git commit -m "COMMIT 6 :: sec(audit): mandate audit logging level"

# Création de la branche de validation de schéma
git checkout -b feature/schema-guard main~1

# Commit 1 de la plage : Configuration du schéma
cat << 'EOF' >> server.py
# Security middleware configuration
SECURITY_PROFILE = "PERMISSIVE_VALIDATION"
SCHEMA_STRICTNESS = 1
EOF
git add server.py
git commit -m "COMMIT 7 :: feat(schema): initialize schema validation baseline"

# Commit 2 de la plage : Durcissement
sed -i 's/SCHEMA_STRICTNESS = 1/SCHEMA_STRICTNESS = 3/' server.py
git add server.py
git commit -m "COMMIT 8 :: sec(schema): elevate schema validation strictness level"
```

---

### 5.3. Exécution du Cherry-Pick en Éventail (*Range*)

Basculez sur `main`. Nous allons transplanter les 2 commits de `feature/schema-guard` en spécifiant la plage syntaxique `A..B` (où $A$ est exclu et $B$ inclus) :

```bash
git checkout main

# Déclenchement de la transplantation de la plage
git cherry-pick -x main~1..feature/schema-guard
```

**Sortie attendue : Déclenchement du Conflit :**
```text
Auto-merging server.py
CONFLICT (content): Merge conflict in server.py
error: could not apply 2f3a4b5... feat(schema): initialize schema validation baseline
hint: After resolving the conflicts, mark them with
hint: "git add/rm <pathspec>", then run
hint: "git cherry-pick --continue".
hint: You can instead skip this commit with "git cherry-pick --skip".
hint: To abort and get back to the state before "git cherry-pick",
hint: run "git cherry-pick --abort".
```

#### Diagnostic et Résolution Déterministe :
Inspectez la collision :
```bash
git diff
```
```text
<<<<<<< HEAD
# Security middleware configuration
SECURITY_PROFILE = "STRICT_AUDIT"
=======
# Security middleware configuration
SECURITY_PROFILE = "PERMISSIVE_VALIDATION"
SCHEMA_STRICTNESS = 1
>>>>>>> 2f3a4b5 (feat(schema): initialize schema validation baseline)
```

Arbitrage de conformité DevSecOps : nous conservons `SECURITY_PROFILE = "STRICT_AUDIT"` tout en intégrant la nouvelle directive `SCHEMA_STRICTNESS = 1` :

```bash
cat << 'EOF' > server.py
import os

def handle_request(path):
    print(f"Routing to: {path}")

# Security middleware configuration
SECURITY_PROFILE = "STRICT_AUDIT"
SCHEMA_STRICTNESS = 1
EOF

# Marquer le conflit comme résolu
git add server.py

# Poursuivre la séquence
git cherry-pick --continue --no-edit

git diff

cat << 'EOF' > server.py
import os

def handle_request(path):
    print(f"Routing to: {path}")

# Security middleware configuration
SECURITY_PROFILE = "STRICT_AUDIT"
SCHEMA_STRICTNESS = 3
EOF

git add server.py
git cherry-pick --continue --no-edit


```


```bash
git log --oneline -3
```
```
* e1f2a3b (HEAD -> main) COMMIT 8 :: sec(schema): elevate schema validation strictness level
* d0e1f2a COMMIT 7 :: feat(schema): initialize schema validation baseline
* c9d0e1f COMMIT 6 :: sec(audit): mandate audit logging level
```

#### Procédure d'Échappement d'Urgence :
Si un conflit survenu en incident de production s'avère insoluble dans la fenêtre impartie, le retour immédiat à l'état stable s'effectue sans aucune pollution résiduelle via :
```bash
git cherry-pick --abort
```

---

## 6. Scénario 4 : Analyse Statique Pré-Vol et Dry-Run (`-n` / `--no-commit`)

### 6.1. Contexte Opérationnel
Dans un pipeline de défense en profondeur (*Shift-Left Security*), vous devez valider un patch avant qu'il ne devienne un commit formel dans l'historique local. En utilisant l'option `-n` (ou `--no-commit`), `git cherry-pick` applique les modifications directement dans la **Zone de Transit (Index)** et le **Working Directory**, permettant l'exécution d'outils SAST (SonarQube, Bandit, Semgrep) ou d'un formateur de code *avant* la ratification du commit.

---

### 6.2. Reproduction et Application Non-Commitée

Créons un commit sur une branche d'audit contenant une variable potentiellement risquée :

```bash
git checkout -b audit/crypto-patch
cat << 'EOF' > crypto_service.py
import secrets

def get_random_token():
    return secrets.token_hex(32)

# Debug variable left by accident
DEBUG_SEED = 123456
EOF
git add crypto_service.py
git commit -m "COMMIT 9 :: feat(crypto): implement cryptographically secure token generator"
CRYPTO_COMMIT=$(git rev-parse HEAD)
```

Basculez sur `main` et transplantez avec `-n` :

```bash
git checkout main

# Application du delta dans l'Index sans créer de commit
git cherry-pick -n $CRYPTO_COMMIT
#-n : --no-commit : applique les modifications dans l'Index et le Working Directory sans créer de commit
```

#### Inspection de la Zone de Transit :
```bash
git status
```
```text
On branch main
Changes to be committed:
  (use "git restore --staged <file>..." to unstage)
	new file:   crypto_service.py
```

Aucun commit n'a été créé :
```bash
git log --oneline -1
# Affiche toujours le dernier commit précédent sur main
```

#### Simulation de la Passerelle de Contrôle SAST :
L'ingénieur exécute un contrôle de politique de sécurité local avant commit :

```bash
# Détection de variable de debug non conforme
if grep -q "DEBUG_SEED" crypto_service.py; then
    echo "[SECURITY SCAN ALERT] Hardcoded debug variable detected. Sanitizing..."
    sed -i '/DEBUG_SEED/d' crypto_service.py
    git add crypto_service.py
fi
```

Consignez le commit assaini et vérifié :
```bash
git commit -m "COMMIT 10 :: feat(crypto): implement sanitized secure token generator"
```

```bash
cat crypto_service.py
```
```python
import secrets

def get_random_token():
    return secrets.token_hex(32)
```

> L'rtefact résiduel de débogage a été neutralisé en mémoire tampon *avant* son inscription dans la chaîne de blocs de Git, évitant la divulgation de métadonnées sensibles et prévenant tout échec de conformité dans le pipeline CI central.

---

## 7. Synthèse Architecturale Comparative pour la Gouvernance Git

| Vecteur d'Analyse               | `git cherry-pick`                                      | `git merge`                                      | `git rebase`                                                |
|:--------------------------------|:-------------------------------------------------------|:-------------------------------------------------|:------------------------------------------------------------|
| **Périmètre d'Application**     | **Chirurgical** (Commit unitaire ou plage choisie)     | **Global** (Intégralité de la branche)           | **Linéaire séquentiel** (Rejeu de l'historique complet)     |
| **Structure du DAG**            | Linéaire (Ajoute de nouveaux nœuds isolés)             | Non-linéaire (Crée un nœud de merge à 2 parents) | Linéaire (Réécrit la base des commits)                      |
| **Identifiants SHA**            | **Nouveaux SHA** générés pour chaque delta             | Préserve les SHA d'origine + 1 SHA de merge      | **Nouveaux SHA** pour tous les commits rejoués              |
| **Traçabilité de Provenance**   | Nécessite `-x` pour éviter les commits fantômes        | Nativement traçable via le graphe                | Masquée (Simule un historique linéaire)                     |
| **Cas d'Usage DevSecOps Idéal** | **Backporting de correctifs CVE / Hotfixes critiques** | Fusion de versions majeures validées             | Hygiène locale pré-Pull Request                             |
| **Risque Opérationnel**         | Duplication de deltas si mal documenté                 | Intégration accidentelle de code instable        | Destruction d'arbres si appliqué sur des branches distantes |