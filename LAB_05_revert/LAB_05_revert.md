# Lab DevSecOps : Stratégies de Rétro-Ingénierie Non Destructive avec `git revert`

## 1. Fondements Architecturaux de `git revert`

Contrairement à `git reset` — qui est une opération destructive réécrivant le graphe orienté acyclique (DAG) en déplaçant le pointeur de branche vers l'arrière et en orphelinisant des commits — **`git revert` est une opération de propagation vers l'avant (Forward-Roll)**. 

D'un point de vue mathématique et structurel, `git revert` calcule le **diff inversé** (l'anticommutateur) d'un ou plusieurs commits ciblés et injecte une **nouvelle mutation** au sommet de l'arbre (`HEAD`).

```text
Modèle Destructif (Interdit sur les branches partagées) :
[C1] <-- [C2] <-- [C3 (Bug/Faille)] <-- [HEAD: main]
                   |
             git reset --hard HEAD~1
                   v
[C1] <-- [C2 (HEAD: main)]   --- [C3 est orphelin / Histoire réécrite]

Modèle Non-Destructif / DevSecOps (git revert) :
[C1] <-- [C2] <-- [C3 (Bug/Faille)] <-- [C4: Revert "C3" (HEAD: main)]
                                            ^
                       Inverse exact des deltas de C3
```

### 1.1. Mécanique Interne du Calcul de Diff Inversé
Lorsque Git exécute `git revert <commit-cible>`, il applique un algorithme de fusion à 3 branches (*3-way merge*) :
1. **Base commune ($O$)** : L'état de l'arbre au commit ciblé.
2. **Notre branche ($A$)** : L'état actuel de `HEAD`.
3. **Leur branche ($B$)** : Le parent immédiat du commit ciblé (`<commit-cible>~1`).

Git transpose la transition de $O \to B$ sur l'état courant $A$.

### 1.2. Justification DevSecOps & Conformité Réglementaire
Dans les cadres d'audit stricts (**SOC 2 Type II, ISO/IEC 27001, PCI-DSS v4.0**), l'effacement d'un commit par `reset` ou `rebase` sur une branche protégée constitue une rupture de la **chaîne de traçabilité cryptographique** (*Chain of Custody*). 

> **La Règle d'Or DevSecOps du Revert :**  
> **Tout artefact déployé ou partagé ayant traversé une passerelle de conformité (CI/CD) doit être neutralisé par propagation positive (`revert`), jamais par oblitération temporelle (`reset`). L'historique des vulnérabilités et des régressions fait partie intégrante de l'audit d'intégrité du système.**

---

## 2. Provisionnement du Laboratoire

Initialisez un espace de travail isolé pour l'ensemble des scénarios :

```bash
# Nettoyage et initialisation du répertoire de laboratoire
rm -rf /tmp/git-revert-lab
mkdir -p /tmp/git-revert-lab && cd /tmp/git-revert-lab

# Création du dépôt local
git init core-security-service
cd core-security-service

# Configuration de l'identité de l'auditeur
git config user.name "devsecops-auditor"
git config user.email "auditor@enterprise.internal"

# Initialisation de la branche principale et du socle
echo "# Secure Authentication Service" > README.md
git add README.md
git commit -m "chore: initial production baseline"
git branch -M main
```

> **Recommandation opérationnelle :** Ouvrez un second terminal et lancez la surveillance dynamique du DAG :
> ```bash
> watch -n 1 -c git log --graph --oneline --all --decorate --color
> ```

---

## 3. Scénario 1 : Remédiation Atomique d'une Régression Critique en Production

### 3.1. Contexte Opérationnel
Un ingénieur a accidentellement désactivé la vérification des certificats TLS dans la configuration du client HTTP. Ce commit a franchi la chaîne CI/CD et a été fusionné sur `main`. L'équipe de sécurité ordonne un retour immédiat à la conformité sans interrompre le travail des pipelines distants.

---

### 3.2. Reproduction de l'Incident

Simulez l'introduction de la vulnérabilité sur la branche `main` :

```bash
# Commit 1 : Feature légitime
cat << 'EOF' > client.py
import requests

def fetch_telemetry():
    return requests.get("https://api.internal/telemetry", verify=True)
EOF
git add client.py
git commit -m "feat(telemetry): implement mutual tls telemetry consumer"

# Commit 2 : La vulnérabilité (Désactivation du TLS verify pour tests locaux, commité par erreur)
sed -i 's/verify=True/verify=False/' client.py
git add client.py
git commit -m "fix(telemetry): bypass ssl verification during latency spikes"
```

Inspection du DAG avant remédiation :
```bash
git log --oneline -3
```
```text
* a1b2c3d (HEAD -> main) fix(telemetry): bypass ssl verification during latency spikes
* 8e9f0a1 feat(telemetry): implement mutual tls telemetry consumer
* 4f5e6d7 chore: initial production baseline
```

---

### 3.3. Exécution du Revert & Validation

Identifiez le hash du commit vulnérable et appliquez le rollback :

```bash
# Annuler le commit sans réécrire l'historique
git revert HEAD --no-edit
```

Git génère automatiquement un commit ayant pour message : `Revert "fix(telemetry): bypass ssl verification during latency spikes"`.

#### Vérification du code source :
```bash
cat client.py
```
```python
import requests

def fetch_telemetry():
    return requests.get("https://api.internal/telemetry", verify=True)
```

#### Inspection du DAG Post-Remédiation :
```bash
git log --oneline --graph -4
```
```text
* f3e2d1c (HEAD -> main) Revert "fix(telemetry): bypass ssl verification during latency spikes"
* a1b2c3d fix(telemetry): bypass ssl verification during latency spikes
* 8e9f0a1 feat(telemetry): implement mutual tls telemetry consumer
* 4f5e6d7 chore: initial production baseline
```

> **Analyse DevSecOps :** Le commit `a1b2c3d` reste présent dans l'historique d'audit. Les rapports de conformité peuvent prouver la détection de la vulnérabilité à l'instant $T$ et sa remédiation formelle à l'instant $T+1$ via le commit `f3e2d1c`. Aucun historique n'a été corrompu.

---

## 4. Scénario 2 : Rollback par Lot Sans Bruit d'Historique (`--no-commit`)

### 4.1. Contexte Opérationnel
Une suite de 3 commits implémentant une passerelle d'autorisation non conforme (IAM RBAC) a été poussée. Pour éviter de générer trois commits de revert distincts (ce qui polluerait la traçabilité et déclencherait plusieurs builds CI), l'équipe DevSecOps doit regrouper l'annulation de cette plage de commits dans une **seule transaction atomique**.

---

### 4.2. Reproduction de la Chaîne Défectueuse

```bash
# Micro-commit 1
echo "class RBAC: roles = []" > auth.py
git add auth.py
git commit -m "feat(iam): define role structure"

# Micro-commit 2
echo "    def evaluate(self): return True" >> auth.py
git add auth.py
git commit -m "feat(iam): scaffold access evaluator"

# Micro-commit 3
echo "DEFAULT_POLICY = 'PERMISSIVE'" >> auth.py
git add auth.py
git commit -m "feat(iam): expose insecure permissive policy"
```

Inspection de la plage à annuler :
```bash
git log --oneline -4
```
```text
* 7a8b9c0 (HEAD -> main) feat(iam): expose insecure permissive policy
* 6b7c8d9 feat(iam): scaffold access evaluator
* 5c6d7e8 feat(iam): define role structure
* f3e2d1c Revert "fix(telemetry): bypass ssl verification during latency spikes"
```

---

### 4.3. Exécution d'un Revert Atomique Consolidé

Utilisez l'option `-n` (ou `--no-commit`) pour inverser la plage de commits directement dans la zone de transit (*staging area*) :

```bash
# Inverser les 3 derniers commits (de HEAD~3 exclu à HEAD inclus)
git revert --no-commit HEAD~3..HEAD
```

Vérifiez l'état de la copie de travail et de la staging area :
```bash
git status
git diff --name-only --cached
git diff --cached
```
```text
On branch main
Changes to be committed:
  (use "git restore --staged <file>..." to unstage)
	deleted:    auth.py
```

Le fichier non conforme a été purgé de la zone de staging sans commit immédiat. Consignez la remédiation unifiée en respectant les standards conventionnels :

```bash
git commit -m "security(iam): revert non-compliant RBAC implementation

Rollback commits:
- 5c6d7e8 (feat(iam): define role structure)
- 6b7c8d9 (feat(iam): scaffold access evaluator)
- 7a8b9c0 (feat(iam): expose insecure permissive policy)

Reason: Failed architecture review due to overly permissive defaults."
```

#### Validation de l'arbre :
```bash
git log --oneline -2
```
```text
* 9c8b7a6 (HEAD -> main) security(iam): revert non-compliant RBAC implementation
* 7a8b9c0 feat(iam): expose insecure permissive policy
```

---

## 5. Scénario 3 : Neutralisation d'un Merge Commit et le Piège de la Réintégration

### 5.1. Contexte Opérationnel
Une Pull Request volumineuse (`feature/billing`) a été fusionnée via un **Merge Commit non-fast-forward**. Dès l'activation en pré-production, un memory leak critique est détecté. Vous devez annuler la fusion.  
*Problème :* Un merge commit possède au moins **deux parents**. Git ne sait pas quelle branche constitue la ligne directrice (*mainline*).

---

### 5.2. Reproduction de la Fusion

```bash
# Création et divergence de la feature branch
git checkout -b feature/billing
echo "class BillingEngine: pass" > billing.py
git add billing.py
git commit -m "feat(billing): introduce billing engine"

# Évolution parallèle de main
git checkout main
echo "v1.2.0-stable" > version.txt
git add version.txt
git commit -m "chore(release): bump version to 1.2.0"

# Fusion explicite (création du commit de merge)
git merge --no-ff feature/billing -m "merge: integrate feature/billing into main"
```

Visualisation de la topologie du DAG :
```bash
git log --graph --oneline -3
```
```text
*   e4d5c6b (HEAD -> main) merge: integrate feature/billing into main
|\  
| * a9b8c7d (feature/billing) feat(billing): introduce billing engine
* | 3f2e1d0 chore(release): bump version to 1.2.0
|/
```

---

### 5.3. Exécution avec Spécification de la Ligne Directrice (`-m`)

Tentez d'abord un revert simple :
```bash
git revert HEAD
```
```text
error: commit e4d5c6b is a merge but no -m option was given.
fatal: revert failed
```

#### Diagnostic Technique :
Git refuse l'opération car il ne sait pas quelle branche préserver.
* **Parent 1 (`-m 1`)** : La branche sur laquelle le merge a eu lieu (`main`).
* **Parent 2 (`-m 2`)** : La branche apportée par le merge (`feature/billing`).

Pour annuler les changements de la feature branch tout en conservant l'état de `main`, désignez le parent 1 :

```bash
git revert -m 1 HEAD --no-edit
```

Vérification :
```bash
ls -la billing.py
# Résultat : ls: cannot access 'billing.py': No such file or directory
```

```bash
git log --oneline -2
```
```text
* 8a7b6c5 (HEAD -> main) Revert "merge: integrate feature/billing into main"
* e4d5c6b merge: integrate feature/billing into main
```

---

### 5.4. L'Écueil DevSecOps : La Réintroduction Ultérieure de la Feature
Si, plus tard, l'équipe corrige le memory leak sur `feature/billing` et tente de re-fusionner la branche sur `main`, **Git ignorera silencieusement les commits d'origine**. Pourquoi ? Parce que le commit `8a7b6c5` a déclaré formellement que l'introduction des deltas de `feature/billing` était annulée.

#### La Solution : Le "Re-Revert"
Pour réintégrer la branche ultérieurement, la procédure standard consiste à inverser le revert :

```bash
# 1. Annuler l'annulation
git revert 8a7b6c5 -m 1 --no-edit  # (si le revert était un merge) ou simplement :
git revert 8a7b6c5 --no-edit

# 2. Les fichiers de billing réapparaissent proprement sur main !
ls -la billing.py
```

---

## 6. Scénario 4 : Conflits de Dépendance Temporelle et Garde-Fous (`--abort`)

### 6.1. Contexte Opérationnel
Un ingénieur tente d'annuler un commit ancien alors que des modifications ultérieures ont été appliquées sur les mêmes lignes de code. Git ne peut pas appliquer le diff inverse de manière déterministe, provoquant un conflit de revert.

---

### 6.2. Reproduction du Conflit

```bash
# Étape 1 : Fichier socle
cat << 'EOF' > security.conf
tls_version=1.2
cipher_suite=ECDHE-RSA-AES128-GCM-SHA256
hsts=false
EOF
git add security.conf
git commit -m "config(tls): configure baseline security ciphers"

# Étape 2 : Commit fautif ciblé (Activation HSTS)
sed -i 's/hsts=false/hsts=true/' security.conf
git add security.conf
git commit -m "feat(security): enable hsts header"
TARGET_COMMIT=$(git rev-parse HEAD)

# Étape 3 : Évolution ultérieure du fichier (durcissement TLS)
sed -i 's/tls_version=1.2/tls_version=1.3/' security.conf
sed -i 's/cipher_suite=.*/cipher_suite=TLS_AES_256_GCM_SHA384/' security.conf
git add security.conf
git commit -m "sec(hardening): mandate tls 1.3 only"
```

---

### 6.3. Déclenchement de la Collision et Résolution

Tentez d'annuler spécifiquement l'activation d'HSTS ($TARGET\_COMMIT$) :

```bash
git revert $TARGET_COMMIT
```

**Sortie Attendue :**
```text
Auto-merging security.conf
CONFLICT (content): Merge conflict in security.conf
error: could not revert TARGET_COMMIT... feat(security): enable hsts header
hint: Resolve all conflicts manually, mark them as resolved with
hint: "git add/rm <conflicted_files>", then run "git revert --continue".
hint: You can instead skip this commit with "git revert --skip".
hint: To abort and get back to the state before "git revert", run "git revert --abort".
```

#### Inspection des Marqueurs de Conflit :
```bash
git diff
```
```text
<<<<<<< HEAD
tls_version=1.3
cipher_suite=TLS_AES_256_GCM_SHA384
hsts=true
=======
tls_version=1.2
cipher_suite=ECDHE-RSA-AES128-GCM-SHA256
hsts=false
>>>>>>> parent of 1b2c3d4 (feat(security): enable hsts header)
```

#### Arbitrage et Clôture :
Nous souhaitons préserver le durcissement TLS 1.3 (`HEAD`) tout en révoquant HSTS (`hsts=false`) :

```bash
cat << 'EOF' > security.conf
tls_version=1.3
cipher_suite=TLS_AES_256_GCM_SHA384
hsts=false
EOF

# Marquer la résolution
git add security.conf

# Finaliser la transaction
git revert --continue --no-edit
```

---

### 6.4. Procédure d'Échappement d'Urgence (`--abort`)
Dans les incidents de production à haute criticité sous pression temporelle, si la complexité d'un conflit dépasse la fenêtre de maintenance autorisée, le rétablissement immédiat de l'intégrité de la session s'effectue via :

```bash
git revert --abort
```
Cette commande réinitialise immédiatement l'index et le répertoire de travail à l'état exact précédant l'appel initial de `git revert`, éliminant tout risque de commit corrompu en production.

---

## 7. Synthèse Comparée pour l'Architecture DevSecOps

| Critère de Gouvernance                | `git revert`                                     | `git reset --hard`                                    |
|:--------------------------------------|:-------------------------------------------------|:------------------------------------------------------|
| **Mutation de l'Arbre**               | Ajout strict ($N+1$)                             | Réduction destructive ($N-1$)                         |
| **Auditabilité & Traçabilité**        | **Totale** (Préserve la signature SHA d'origine) | **Nulle** (Orphelinise et masque l'erreur)            |
| **Impact sur les Branches Partagées** | Aucun conflit de push (`Fast-Forward`)           | Nécessite un `push --force` (Destructeur)             |
| **Applicabilité CI/CD**               | Idéal pour le Rollback automatisé                | Interdit par les stratégies de protection de branches |