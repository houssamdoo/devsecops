# Lab DevSecOps : Collaboration Concurrente, Mécanique des Rejets Non-Fast-Forward et Résolution de Conflits de Fusion

---

## 1. Fondements Architecturaux de la Collaboration Distribuée sur Git

Git est un système de contrôle de version **pleinement distribué**. Contrairement aux architectures centralisées, chaque collaborateur possède un clone complet du graphe orienté acyclique (DAG) en local. Le serveur distant (*Remote*) n'est qu'un pair privilégié faisant office de point de synchronisation et de source de vérité (*Single Source of Truth*).

```text
               Topologie de Divergence et Collision Concurrente :

     [Base Commune: C1] 
           /        \
          v          v
   (Alice pousse)  (Bob commite localement)
       [C2]           [C3]
        |              |
        v              v
[origin/main: C2]  [local/main: C3]  <--- DIVERGENCE : Les historiques ne sont plus alignés
```

### 1.1. L'Invariable de Sécurité : Le Rejet Non-Fast-Forward
Lorsqu'un développeur exécute `git push`, le serveur distant applique une règle de garde-fou cryptographique stricte : **l'avance rapide (*Fast-Forward*)**.
* Un push est accepté en Fast-Forward **uniquement** si le commit au sommet du serveur distant est l'ancêtre direct du commit envoyé.
* Si le serveur contient des commits que le développeur local ne possède pas (divergence), Git **rejette catégoriquement le push**. Cela protège l'intégrité du dépôt : Git refuse d'écraser silencieusement le travail des pairs.

### 1.2. La Mécanique du Conflit de Fusion lors du `pull`
Pour pouvoir pousser, le développeur en retard doit d'abord incorporer l'état amont via `git pull`. 
`git pull` déclenche une fusion à 3 branches (*3-way merge*) entre :
1. **L'ancêtre commun ($O$)** : Le dernier commit partagé avant la divergence.
2. **La branche locale ($A$)** : Les modifications de l'utilisateur courant.
3. **La branche distante ($B$)** : Les modifications poussées par l'autre développeur.

> **Définition d'un Conflit de Contenu :**  
> Si $A$ et $B$ ont modifié **la même section des mêmes lignes** d'un fichier depuis $O$, ou si un côté a modifié un fichier que l'autre a supprimé, l'algorithme de fusion heuristique de Git ne peut pas décider de manière déterministe quelle version prévaut. Git suspend la transaction, inscrit des **marqueurs de conflit** dans le fichier physique et transfère l'arbitrage à l'humain.

---

## 2. Provisionnement du Laboratoire

Pour simuler fidèlement une infrastructure d'entreprise, nous allons instancier trois environnements dans `/tmp/git-collaboration-lab` :
1. **Le serveur central (`remote-central.git`)** : Dépôt nu (*Bare repository*) agissant comme passerelle d'entreprise.
2. **La station d'Alice (`workstation-alice`)** : Lead DevSecOps responsable du durcissement des accès.
3. **La station de Bob (`workstation-bob`)** : Développeur Backend responsable des performances des microservices.

```bash
# 1. Nettoyage et initialisation du banc d'essai
rm -rf /tmp/git-collaboration-lab
mkdir -p /tmp/git-collaboration-lab && cd /tmp/git-collaboration-lab

# 2. Création du serveur central (Bare Repository)
git init --bare --initial-branch=main remote-central.git

# OU, si la version de git ne supporte pas --initial-branch :
#git init --bare remote-central.git
#git -C remote-central.git symbolic-ref HEAD refs/heads/main

# 3. Provisionnement de la station d'Alice
git clone -b main remote-central.git workstation-alice
#git clone remote-central.git workstation-alice

cd workstation-alice
git config user.name "Alice-DevSecOps"
git config user.email "alice@enterprise.internal"

# Initialisation du socle d'infrastructure
cat << 'EOF' > gateway-config.yaml
server:
  port: 8443
  tls_enabled: true
  rate_limit: 100
  timeout_seconds: 30
EOF

git add gateway-config.yaml
git commit -m "COMMIT 1 :: chore: initial security gateway baseline"
git branch -M main
git push origin main
cd ..

# 4. Provisionnement de la station de Bob
git clone -b main remote-central.git workstation-bob
#git clone remote-central.git workstation-bob
cd workstation-bob
git config user.name "Bob-BackendDev"
git config user.email "bob@enterprise.internal"
cd ..
```

---

## 3. Scénario 1 : Le Flux Nominal Conforme (Fast-Forward)

### 3.1. Contexte Opérationnel
Alice ajoute une instruction d'audit. Bob récupère ensuite cette modification sans aucun conflit pour synchroniser sa copie de travail locale avec le serveur.

---

### 3.2. Reproduction & Validation

```bash
# Alice apporte une modification sur sa station
cd /tmp/git-collaboration-lab/workstation-alice
echo "audit_logging: enabled" >> gateway-config.yaml
git add gateway-config.yaml
git commit -m "COMMIT 2 :: sec(audit): enable mandatory audit logging"
git push origin main
```

Bob souhaite récupérer ce travail. Sur sa station :
```bash
cd /tmp/git-collaboration-lab/workstation-bob

# Inspection de l'Index avant synchronisation
git log --oneline -1

# Récupération et intégration
git pull origin main
```

**Sortie standard observée chez Bob :**
```text
Updating a1b2c3d..e4f5a6b
Fast-forward
 gateway-config.yaml | 1 +
 1 file changed, 1 insertion(+)
```

> **Analyse DevSecOps :** Le pointeur `main` de Bob s'est simplement déplacé vers l'avant le long de la ligne d'historique (*Fast-forward*). Aucun nœud de merge n'a été créé, la structure du DAG demeure linéaire.

---

## 4. Scénario 2 : Le Rejet Critique au Push (`non-fast-forward`)

### 4.1. Contexte Opérationnel
Alice et Bob travaillent simultanément. 
* **Alice** durcit la sécurité en réduisant le seuil de limitation de requêtes (`rate_limit`) pour atténuer une menace par déni de service (DDoS). Elle commite et pousse sur `origin/main`.
* **Bob**, sans avoir synchronisé sa station, modifie **la même directive** pour augmenter le débit dans le cadre de tests de montée en charge. Il commite localement et tente de pousser.

---

### 4.2. Reproduction de la Divergence

#### Étape 1 : Alice pousse sa politique de sécurité
```bash
cd /tmp/git-collaboration-lab/workstation-alice

# Modification de la ligne 4
sed -i 's/rate_limit: 100/rate_limit: 50/' gateway-config.yaml
git add gateway-config.yaml
git commit -m "COMMIT 3 :: sec(dos): restrict rate limit to 50 req/sec"
git push origin main
```

#### Étape 2 : Bob commite localement une modification concurrente
```bash
cd /tmp/git-collaboration-lab/workstation-bob

# Modification contradictoire sur la même ligne (rate_limit = 500)
sed -i 's/rate_limit: 100/rate_limit: 500/' gateway-config.yaml
git add gateway-config.yaml
git commit -m "COMMIT 4 :: perf(tuning): elevate rate limit to 500 req/sec for stress test"
```

Inspection du DAG local de Bob :
```bash
git log --oneline -2
```
```text
* 7b8c9d0 (HEAD -> main) perf(tuning): elevate rate limit to 500 req/sec for stress test
* e4f5a6b sec(audit): enable mandatory audit logging
```

---

### 4.3. Déclenchement de l'Interdiction au Push

Bob tente de publier son commit vers le serveur central :

```bash
git push origin main
```

**Sortie d'erreur émise par le démon Git :**
```text
To /tmp/git-collaboration-lab/remote-central.git
 ! [rejected]        main -> main (fetch first)
error: failed to push some refs to '/tmp/git-collaboration-lab/remote-central.git'
hint: Updates were rejected because the remote contains work that you do
hint: not have locally. This is usually caused by another repository pushing
hint: to the same ref. You may want to first integrate the remote changes
hint: (e.g., 'git pull ...') before pushing again.
hint: See the 'Note about fast-forwards' in 'git push --help' for details.
```

#### Diagnostic Technique :
Le serveur distant contient le commit d'Alice (`sec(dos)`). Ce nœud n'appartient pas à la lignée ancestrale du commit de Bob (`perf(tuning)`). Si Git acceptait le push de Bob, le commit d'Alice serait déréférencé et effacé de la branche de production. Git applique son principe de protection : **rejet strict de la mutation**.

---

## 5. Scénario 3 : Collision de Contenu au `pull` et Triage Manuel du Conflit

### 5.1. Contexte Opérationnel
Suite au rejet de son push, Bob doit synchroniser son dépôt avec l'amont avant de pouvoir pousser. Il exécute un `git pull`. Comme Alice et Bob ont altéré la même ligne, Git déclenche un **conflit de fusion textuel**.

---

### 5.2. Déclenchement du Conflit

Toujours sur la station de Bob :

```bash
cd /tmp/git-collaboration-lab/workstation-bob

# Déclenchement de l'intégration concurrente (stratégie de merge standard)
git pull --no-rebase origin main
```

**Sortie standard d'alerte :**
```text
Auto-merging gateway-config.yaml
CONFLICT (content): Merge conflict in gateway-config.yaml
Automatic merge failed; fix conflicts and then commit the result.
```

Constat de blocage de l'Index :
```bash
git status
```
```text
On branch main
Your branch and 'origin/main' have diverged,
and have 1 and 1 different commits each, respectively.
  (use "git pull" if you plan to merge the remote branch with yours)

Unmerged paths:
  (use "git add <file>..." to mark resolution)
	both modified:   gateway-config.yaml
```

---

### 5.3. Anatomie Forensique des Marqueurs de Conflit

Examinez directement le contenu altéré du fichier sur le disque :

```bash
cat gateway-config.yaml
```

**Sortie des marqueurs de collision Git :**
```yaml
server:
  port: 8443
  tls_enabled: true
<<<<<<< HEAD
  rate_limit: 500
=======
  rate_limit: 50
>>>>>>> b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1
  timeout_seconds: 30
audit_logging: enabled
```

#### Décodage des Marqueurs :
* `<<<<<<< HEAD` : Délimite le début des modifications de la station locale (le commit de Bob, `rate_limit: 500`).
* `=======` : Séparateur central marquant la frontière entre les deux versions concurrentes.
* `>>>>>>> <SHA>` : Délimite la fin des modifications reçues depuis le serveur distant (le commit d'Alice, `rate_limit: 50`).

---

### 5.4. Résolution du Conflit et Finalisation du Commit de Fusion

Dans un cadre DevSecOps, la résolution d'un conflit n'est pas un acte syntaxique, mais une **décision d'arbitrage de sécurité**. 
* La politique d'Alice vise à empêcher une saturation des serveurs.
* Le besoin de Bob est d'absorber des pics de charge.
* **Compromis d'ingénierie validé :** La configuration de production sera fixée à `rate_limit: 150`, accompagnée d'une note de traçabilité.

#### Étape 1 : Nettoyer le fichier et supprimer les marqueurs
Bob édite le fichier pour remplacer la zone de collision par l'état arbitré :

```bash
cat << 'EOF' > gateway-config.yaml
server:
  port: 8443
  tls_enabled: true
  rate_limit: 150
  timeout_seconds: 30
audit_logging: enabled
EOF
```

#### Étape 2 : Valider la résolution auprès de l'Index
```bash
# Vérification du diff résolu
git diff

# Marquer le fichier comme résolu
git add gateway-config.yaml
```

Inspectez l'état de la zone de transit :
```bash
git status
```
```text
On branch main
All conflicts fixed but you are still merging.
  (use "git commit" to conclude merge)
```

#### Étape 3 : Sceller le commit de fusion (*Merge Commit*)
```bash
git commit -m "COMMIT 5 :: merge(config): reconcile rate-limiting thresholds (arbitrated to 150 req/sec)"
```

#### Étape 4 : Publication sécurisée sur le serveur central
Maintenant que le conflit est résolu, l'historique local de Bob englobe formellement les commits du serveur. Le push s'exécute sans encombre :

```bash
git push origin main
```

#### Étape 5 : Inspection du DAG résultant
```bash
git log --graph --oneline -4
```
```text
*   9a8b7c6 (HEAD -> main, origin/main) merge(config): reconcile rate-limiting thresholds (arbitrated to 150 req/sec)
|\  
| * 3c4d5e6 sec(dos): restrict rate limit to 50 req/sec
* | 7b8c9d0 perf(tuning): elevate rate limit to 500 req/sec for stress test
|/  
* e4f5a6b sec(audit): enable mandatory audit logging
```

> **Résultat Architectural :** Le commit `9a8b7c6` est un nœud à deux parents qui réconcilie formellement les branches d'Alice et de Bob dans l'historique public.

---

## 6. Scénario 4 : L'Alternative Professionnelle — Résolution par `git pull --rebase`

### 6.1. Contexte Opérationnel
La création répétée de commits de merge (`merge: integrate...`) pollue la lisibilité du DAG et complique les analyses de bissection automatisée (`git bisect`). 

Dans les équipes appliquant une gouvernance d'historique linéaire (*Trunk-Based Development*), l'intégration des modifications amont s'effectue via **`git pull --rebase`**.

---

### 6.2. Reproduction d'une Nouvelle Divergence

```bash
# 1. Alice modifie le paramètre de timeout sur le serveur
cd /tmp/git-collaboration-lab/workstation-alice
git pull origin main
sed -i 's/timeout_seconds: 30/timeout_seconds: 15/' gateway-config.yaml
git add gateway-config.yaml
git commit -m "sec(timeout): reduce idle connection timeout to 15s"
git push origin main

# 2. Simultanément, Bob modifie le même paramètre en local
cd /tmp/git-collaboration-lab/workstation-bob
sed -i 's/timeout_seconds: 30/timeout_seconds: 45/' gateway-config.yaml
git add gateway-config.yaml
git commit -m "perf(timeout): increase idle timeout to 45s for batch jobs"
```

---

### 6.3. Intégration par Rebase et Gestion de Collision

Bob synchronise son dépôt avec l'instruction de rebase :

```bash
cd /tmp/git-collaboration-lab/workstation-bob
git pull --rebase origin main
```

**Sortie attendue : Suspension du Rejeu :**
```text
Auto-merging gateway-config.yaml
CONFLICT (content): Merge conflict in gateway-config.yaml
error: could not apply 1a2b3c4... perf(timeout): increase idle timeout to 45s for batch jobs
hint: Resolve all conflicts manually, mark them as resolved with
hint: "git add/rm <conflicted_files>", then run "git rebase --continue".
hint: You can instead skip this commit with "git rebase --skip".
hint: To abort and get back to the state before "git rebase", run "git rebase --abort".
```

#### Diagnostic du Mécanisme :
Git a d'abord déplacé le pointeur local de Bob sur le commit d'Alice (`origin/main`), puis a tenté de **rejouer** le commit local de Bob par-dessus. La collision interrompt le rejeu.

#### Résolution du Conflit sous Rebase :
Examinez les marqueurs :
```bash
git diff
```
```text
<<<<<<< HEAD (Version d'Alice issue de origin/main)
  timeout_seconds: 15
=======
  timeout_seconds: 45
>>>>>>> 1a2b3c4 (Version locale de Bob)
```

Arbitrage de compromis : nous adoptons la valeur `timeout_seconds: 20` :

```bash
cat << 'EOF' > gateway-config.yaml
server:
  port: 8443
  tls_enabled: true
  rate_limit: 150
  timeout_seconds: 20
audit_logging: enabled
EOF

# Marquer le conflit comme résolu
git add gateway-config.yaml

# Reprendre le rejeu séquentiel (NE PAS FAIRE DE COMMIT ICI !)
git rebase --continue
```

Git applique le commit rejoué avec succès :
```text
Applying: perf(timeout): increase idle timeout to 45s for batch jobs
```

Bob publie son travail linéarisé :
```bash
git push origin main
```

Inspection du DAG final de Bob :
```bash
git log --graph --oneline -3
```
```text
* d5e6f7a (HEAD -> main, origin/main) perf(timeout): increase idle timeout to 45s for batch jobs
* c4b3a2d sec(timeout): reduce idle connection timeout to 15s
* 9a8b7c6 merge(config): reconcile rate-limiting thresholds (arbitrated to 150 req/sec)
```

> **Victoire DevSecOps :** Aucun commit de merge superflu n'a été créé. Le commit de Bob a été transplanté de manière fluide au sommet du commit d'Alice, préservant une chaîne d'audit strictement linéaire et facilitant l'investigation forensique.

---

## 7. Synthèse des Stratégies d'Arbitrage pour la Gouvernance Git

| Stratégie de Résolution | Commande                                    | Topologie du DAG                                | Avantages                                                             | Inconvénients / Risques                                                                  |
|:------------------------|:--------------------------------------------|:------------------------------------------------|:----------------------------------------------------------------------|:-----------------------------------------------------------------------------------------|
| **Merge Standard**      | `git pull --no-rebase`                      | Non-linéaire (Arborescent avec nœuds de fusion) | Préserve l'ordre chronologique réel de chaque commit                  | Pollue l'arbre de commits avec des messages `Merge branch...` récurrents                 |
| **Rebase Linéaire**     | `git pull --rebase`                         | **Strictement Linéaire**                        | Historique immaculé, simplifie `git bisect` et les audits automatisés | Réécrit les SHA locaux avant push ; peut obliger à résoudre des conflits étape par étape |
| **Abandon d'Urgence**   | `git merge --abort`<br>`git rebase --abort` | Inaltérée                                       | Restaure instantanément l'état stable précédent sans corruption       | Le travail d'intégration reste en suspens                                                |

### Les Règles d'Or de la Collaboration Sécurisée :
1. **Évitez le travail direct sur la branche centrale partagée** : Dans les entreprises régies par SOC 2 ou l'ISO 27001, les pushs directs sur `main` sont révoqués par des stratégies de protection de branches (*Branch Protection Rules*). Tout passage s'effectue via des **Pull Requests (PR)** ou **Merge Requests (MR)** soumises à revues obligatoires par les pairs.
2. **Synchronisez avant de concevoir** : Exécutez toujours un `git fetch` ou un `git pull --rebase` avant d'initier une nouvelle série de modifications locales pour minimiser l'écart de divergence par rapport à l'amont.
3. **Ne forcez jamais un push en aveugle** : Si Git rejette votre push (`non-fast-forward`), l'usage de `git push --force` sans analyse préalable peut détruire des heures de travail collaboratif ou masquer une remédiation de faille critique injectée par un pair.