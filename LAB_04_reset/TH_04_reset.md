# `git reset` & la Manipulation des Trois Arbres

---

## 1. Fondations & Architecture Interne : L'Impact sur les « Trois Arbres »

### 1.1. Modèle mental : Le rôle de `git reset`
Dans l'architecture Git, votre code transite à travers trois états logiques (les « Trois Arbres ») :
1. **Le Répertoire de travail (*Working Directory*) :** Ce sont les fichiers physiques présents sur votre disque dur dans votre environnement local, éditables directement.
2. **L'Index (*Staging Area*) :** C'est le fichier binaire .git/index. Il préfigure exactement à quoi ressemblera le prochain commit. C'est la zone de préparation.
3. **Le Dépôt (*HEAD / Commit History*) :** C'est le pointeur qui référence le commit actif sur la branche courante. C'est l'état immuable consigné dans le graphe orienté acyclique (DAG).

```text
+-----------------------+     +-----------------------+     +-----------------------+
|  1. Working Tree      |     |  2. Index / Staging   |     |  3. HEAD (Repository) |
|  (Espace de Travail)  |     |  (Cache de Transit)   |     |  (Dernier Commit)     |
+-----------------------+     +-----------------------+     +-----------------------+
| Fichiers réels sur    |     | Instantané du         |     | Pointeur sur le nœud  |
| le système de fichier |     | prochain commit visé  |     | actif du DAG Git      |
+-----------------------+     +-----------------------+     +-----------------------+
            ^                             ^                             ^
            |                             |                             |
      Modifié par                   Modifié par                   Modifié par
    l'éditeur de code               "git add"                   "git commit"
```

La commande `git reset` est le **moteur de recul d'état**. Contrairement à `git checkout` (qui déplace la tête de lecture `HEAD` d'une branche à une autre), `git reset` **déplace la branche courante elle-même vers l'arrière** et répercute (ou non) ce recul sur l'Index et le Working Directory selon le mode choisi.

_Contrairement à ce que son nom suggère, git reset ne supprime pas de commits : il déplace un pointeur de branche vers l'arrière dans le DAG._

```text
Visualisation de l'action de git reset HEAD~1 :

Avant Reset :
  main ────► [Commit A] ──► [Commit B] ◄── HEAD

Après Reset (HEAD~1) :
  main ────► [Commit A] ◄── HEAD
                  \
                   └──► [Commit B] (Devient un commit orphelin / Dangling commit)
```
https://gitualize.com/commands/git-reset

---

Conceptuellement, `git reset` exécute jusqu'à **trois étapes séquentielles**, et les options (`--soft`, `--mixed`, `--hard`) déterminent simplement **à quelle étape Git s'arrête** :
```text
Étape 1 : Déplacer la référence HEAD (et la branche courante)      ---> [--soft s'arrête ici]
Étape 2 : Synchroniser l'Index avec la nouvelle HEAD             ---> [--mixed s'arrête ici (Défaut)]
Étape 3 : Synchroniser le Working Directory avec l'Index/HEAD     ---> [--hard s'arrête ici]
```

## 3. Analyse Détaillée des Trois Options

Prenons un état initial identique pour illustrer chaque mode :
* Vous avez un commit $C_1$ (stable).
* Vous avez créé un commit $C_2$ contenant une erreur ou prématuré.
* Votre `HEAD` pointe actuellement sur $C_2$ : `main -> C2`.
* Objectif : Revenir à l'état de $C_1$ (`git reset <options> C1` ou `git reset <options> HEAD~1`).

---

### 3.1. `git reset --soft` : L'Étape 1 Seule (Déplacement du Pointeur)

#### Mécanisme Interne :
Git déplace uniquement le pointeur de la branche courante et `HEAD` vers le commit cible. **L'Index et le Working Directory restent rigoureusement intacts.**

```text
[C1] <--- [C2]
  ^
  |-- HEAD (main)

État des 3 arbres après "git reset --soft C1" :
1. HEAD               : Positionné sur C1
2. Index              : Conserve le delta de C2 (fichiers déjà "staged")
3. Working Directory  : Conserve le delta de C2 (fichiers physiques inchangés)
```

#### Conséquence Opérationnelle :
Toutes les modifications introduites entre $C_1$ et $C_2$ se retrouvent immédiatement dans votre **Staging Area** (vertes dans `git status`). Pour Git, c'est comme si vous veniez de faire `git add` sur l'ensemble de ces modifications, prêtes à être commitées.

#### Cas d'Usage DevSecOps & Ingénierie :
* **Squash local pré-push** : Vous avez créé 5 micro-commits désordonnés localement (`HEAD~5`). Vous faites `git reset --soft HEAD~5`, puis un unique `git commit -m "feat(auth): implement secure OAuth2 flow"` propre et conforme aux standards Conventional Commits.
* **Reprise d'un message ou de métadonnées** : Permet de recommiter instantanément avec une signature GPG (`-S`) ou un message d'audit rectifié.

---

### 3.2. `git reset --mixed` : L'Étape 1 + Étape 2 (Mode par Défaut)

#### Mécanisme Interne :
Si aucune option n'est spécifiée, Git utilise `--mixed`. 
1. Git déplace `HEAD` vers le commit cible.
2. Git **écrase l'Index** pour qu'il reflète exactement l'état de ce commit cible.
3. Le **Working Directory reste totalement inchangé**.

```text
[C1] <--- [C2]
  ^
  |-- HEAD (main)

État des 3 arbres après "git reset --mixed C1" :
1. HEAD               : Positionné sur C1
2. Index              : Réaligné sur C1 (vide de tout delta)
3. Working Directory  : Conserve le delta de C2 sur le disque (fichiers "unstaged")
```

#### Conséquence Opérationnelle :
Les modifications entre $C_1$ et $C_2$ existent toujours sur votre disque dur, mais elles sont **dé-stagées** (rouges dans `git status`). Rien n'est perdu, mais rien n'est prêt à être commité.

#### Cas d'Usage DevSecOps & Ingénierie :
* **Atomisation des commits** : Vous avez commité en bloc plusieurs fonctionnalités non liées (ex. durcissement de configuration et refactoring de code). `git reset --mixed HEAD~1` remet tout dans votre espace de travail.
* **Annulation d'un `git add` accidentel** : `git reset HEAD <fichier>` utilise la logique `--mixed` pour sortir un fichier de l'Index sans toucher au fichier physique.

---

### 3.3. `git reset --hard` : Les 3 Étapes (Alignement Destructif)

#### Mécanisme Interne :
Git applique la séquence complète :
1. Déplacement de `HEAD` vers le commit cible.
2. Écrasement de l'Index avec l'état du commit cible.
3. **Écrasement du Working Directory** : Git force vos fichiers physiques sur le disque à correspondre exactement au snapshot du commit cible.

```text
[C1] <--- [C2 (orphelin)]
  ^
  |-- HEAD (main)

État des 3 arbres après "git reset --hard C1" :
1. HEAD               : Positionné sur C1
2. Index              : Réaligné sur C1
3. Working Directory  : Réaligné sur C1 (TOUTES les modifications de C2 sont écrasées)
```

#### Conséquence Opérationnelle :
**Toutes les modifications non commitées et tous les commits situés après la cible sont effacés de votre vue de travail.** 
> ⚠️ **Avertissement de criticité :** Si des modifications étaient présentes dans votre Working Directory sans avoir jamais été commitées ni stagées, **elles sont définitivement irrécupérables**.

#### Cas d'Usage DevSecOps & Ingénierie :
* **Purge d'expérimentation locale** : Vous avez mené un spike technique local qui s'avère non viable et vous souhaitez réinitialiser votre environnement à l'état strict de la branche d'origine.


---

## 4. Synthèse Matricielle Comparative

| Option                   | Déplace `HEAD` ? | Modifie l'Index (Staging) ? |  Modifie le Working Tree ?   | Degré de Risque                               |
|:-------------------------|:----------------:|:---------------------------:|:----------------------------:|:----------------------------------------------|
| **`--soft`**             |     **Oui**      |   Non *(conserve staged)*   |        Non *(intact)*        | **Nul** (zéro perte de code)                  |
| **`--mixed`** *(défaut)* |     **Oui**      |    **Oui** *(dé-stage)*     |  Non *(intact sur disque)*   | **Faible** (code conservé)                    |
| **`--hard`**             |     **Oui**      |           **Oui**           | **Oui** *(écrase le disque)* | **Critique** (perte définitive du non-tracké) |

---

## 5. Gouvernance DevSecOps : Risques et Garde-fous

### 5.1. Pourquoi le `reset` est Proscrit sur les Branches Partagées
Si vous effectuez un `git reset` sur un commit qui a déjà été propagé (`push`) vers un dépôt central (ex. `main` ou `develop`) :
1. Votre historique local diverge de l'historique distant.
2. Git refusera tout push ultérieur sans l'option `--force` (ou `--force-with-lease`).
3. L'utilisation du `--force` réécrit l'historique pour **tous** les collaborateurs et les pipelines, orphelinisant les commits sur lesquels d'autres équipes ont pu baser leur travail.
4. **Perte de traçabilité d'audit** : Les signatures cryptographiques et les identifiants SHA originaux sont rompus, ce qui viole les exigences de non-répudiation (SOC 2, ISO 27001).

> **Axiome :**  
> * Local non publié : **`git reset`** (pour l'hygiène et la propreté de l'arbre).  
> * Distant / Partagé : **`git revert`** (pour la non-régression et l'auditabilité).

### 5.2. Filet de Sécurité : La Récupération Post-`--hard` via `git reflog`
Si un développeur exécute par erreur un `git reset --hard` sur des commits déjà enregistrés, ces commits ne sont pas immédiatement détruits : ils deviennent des **commits orphelins** (inaccessibles via les branches, mais encore présents dans la base d'objets de Git jusqu'au prochain passage du *Garbage Collector*).

Git enregistre chaque mouvement de `HEAD` dans le journal de référence (**`reflog`**) :

```bash
# 1. Consulter l'historique des déplacements de pointeur
git reflog

# Exemple de sortie :
# 1a2b3c4 HEAD@{0}: reset: moving to HEAD~1
# 9f8e7d6 HEAD@{1}: commit: feat(crypto): introduce aes-gcm encryption

# 2. Restaurer immédiatement l'état perdu (sauvetage d'urgence) :
git reset --hard HEAD@{1}
# ou créer une branche de secours sur ce hash :
git branch rescue-branch 9f8e7d6
```

Cet instrument garantit qu'en environnement DevSecOps, un incident lié à une manipulation locale erronée reste réversible tant que les données ont fait l'objet d'au moins un commit initial.

















