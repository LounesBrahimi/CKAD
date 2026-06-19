# CKAD Séance 1 · Pods — Clés techniques

---

# PARTIE 1 — QUIZ

---

## seance1-quiz-pending
**Pod Pending → quelle commande ?**

```bash
kubectl describe pod <name> -n <ns>
```

→ section **Events**.
Causes : ressources insuffisantes, taint non tolérée, `ImagePullBackOff`.
`logs` ne marche pas ici : le container n'a jamais démarré, donc aucun log à lire.

---

## seance1-quiz-restartpolicy
**restartPolicy Never vs OnFailure**

`Never` = ne redémarre jamais, succès ou échec.
`OnFailure` = redémarre si exit code ≠ 0.
`Always` = redémarre toujours (défaut).

Énoncé "ne jamais redémarrer même en cas d'erreur" → `Never` uniquement.

---

## seance1-quiz-apply-vs-create
**"created" = suffisant ?**

Non. `created` = API a accepté le manifest. Ne dit rien sur l'état réel.

```bash
kubectl get pod <name> -n <ns>
```

→ vérifier statut `Running`.

---

## seance1-quiz-create-vs-apply
**create vs apply**

`create` : échoue si la ressource existe déjà.
`apply` : crée ou met à jour, idempotent.
À l'exam : toujours `apply`.

---
---

# PARTIE 2 — TERMINAL

---

## seance1-term-crashloop
**CrashLoopBackOff — diagnostic**

```bash
kubectl config use-context k8s-staging
```
**Ce que ça fait :** bascule kubectl sur le cluster "k8s-staging" — toutes les commandes suivantes s'exécutent sur CE cluster, pas un autre.
**Pourquoi en premier :** si vous oubliez, vous diagnostiquez peut-être le mauvais cluster sans le savoir, et vous perdez du temps sur des résultats qui ne correspondent à rien.

```bash
kubectl get pod api-pod -n backend
```
**Ce que ça fait :** liste l'état résumé du pod — colonnes READY, STATUS, RESTARTS, AGE.
**Ce qu'on y lit :** `READY 0/1` (le container n'est pas prêt), `STATUS CrashLoopBackOff`, `RESTARTS > 0` (déjà redémarré plusieurs fois). C'est la première confirmation visuelle du problème.

```bash
kubectl get pod api-pod -n backend -o wide
```
**Ce que ça fait :** ajoute 4 colonnes au résultat précédent — IP du pod, nom du node où il tourne, nominated node, readiness gates.
**Indispensable ?** Non, optionnelle ici. Utile si vous avez plusieurs pods et voulez savoir lesquels sont sur le même node (utile pour des pannes liées à un node précis). Pour CE diagnostic, le `get` simple suffisait déjà.

```bash
kubectl describe pod api-pod -n backend
```
**Ce que ça fait :** affiche le détail complet du pod — bien plus que `get`. Inclut la configuration (image, ports, env), l'état actuel ET l'état précédent (`Last State`), et un historique d'`Events`.
**Ce qu'on y cherche :** section `Last State` → `Exit Code` (le code de sortie du dernier crash, souvent `1` = erreur générique). Section `Events` en bas → la raison lisible du redémarrage (`BackOff`, `Failed`, etc.).

```bash
kubectl logs api-pod -n backend
```
**Ce que ça fait :** affiche les logs du container ACTUEL — celui qui tourne (ou tente de tourner) en ce moment précis.
**Pourquoi c'est presque inutile ici :** dans un CrashLoopBackOff, le container vient de redémarrer il y a quelques secondes. Il n'a quasiment rien eu le temps d'écrire dans ses logs avant de re-crasher. Vous obtenez souvent une sortie vide ou juste "starting...".

```bash
kubectl logs api-pod -n backend --previous
```
**Ce que ça fait :** affiche les logs du container PRÉCÉDENT — celui qui vient de se terminer (le crash qu'on cherche à comprendre).
**Pourquoi c'est LA commande clé :** le crash a eu lieu sur ce container disparu, pas sur le nouveau qui vient de démarrer. C'est ici, et seulement ici, qu'on trouve le vrai message d'erreur applicatif (ex: variable d'environnement manquante, connexion base de données refusée).

**Pourquoi `logs` puis `logs --previous` et pas l'inverse ou un seul des deux ?**
Un pod en CrashLoopBackOff redémarre en boucle automatiquement. À l'instant où vous tapez une commande, vous avez TOUJOURS deux containers à considérer : celui qui vient de redémarrer (actuel, presque aucun log) et celui qui a crashé juste avant (précédent, contient la cause réelle). Tester les deux dans cet ordre permet de comprendre cette mécanique en direct : le premier ne révèle rien, ça motive le `--previous`.

**Bonus diagnostic — pour creuser plus loin si besoin :**

```bash
kubectl get pod api-pod -n backend -o yaml
```
**Ce que ça fait :** sort le manifest complet du pod, tel qu'il existe réellement dans le cluster (pas votre fichier source — l'état stocké côté serveur, avec tous les champs que Kubernetes a ajoutés automatiquement).
**Utile pour :** vérifier une valeur précise sans deviner (image exacte utilisée, variables d'env réellement appliquées).

```bash
kubectl exec -it api-pod -n backend -- sh
```
**Ce que ça fait :** ouvre un shell interactif À L'INTÉRIEUR du container, comme si vous vous connectiez en SSH dedans.
**Limite importante :** si le container n'est pas dans un état "Running" au moment où vous tapez la commande (ce qui est probable en CrashLoopBackOff, puisqu'il alterne crash/redémarrage), la commande échoue — il n'y a personne à qui parler.

```bash
kubectl explain pod.spec.containers
```
**Ce que ça fait :** affiche la documentation officielle intégrée à kubectl pour ce champ précis du schéma Pod — pas de connexion internet nécessaire.
**Pourquoi le mentionner :** à l'examen CKAD, pas d'accès à Google. C'est votre seule source de référence si vous doutez d'un nom de champ ou de sa structure exacte.

---

## seance1-term-dryrun
**Dry-run speed run**

```bash
kubectl config use-context k8s-dev
```
**Ce que ça fait :** bascule sur le cluster "k8s-dev" avant toute action — même logique que pour l'exercice précédent.

```bash
kubectl run cache-pod --image=redis:7 -n infra --labels="tier=cache" --dry-run=client -o yaml > cache-pod.yaml
```
**Ce que ça fait, terme par terme :**
- `kubectl run cache-pod --image=redis:7` : génère la structure d'un Pod nommé "cache-pod" utilisant l'image redis:7 — sans cette commande, il faudrait écrire tout le YAML à la main.
- `-n infra` : place ce Pod dans le namespace "infra".
- `--labels="tier=cache"` : ajoute directement le label demandé, sans avoir à l'écrire après dans le fichier.
- `--dry-run=client` : NE CRÉE RIEN dans le cluster. Cette commande simule juste ce qui SERAIT créé, sans contacter le serveur Kubernetes pour de vrai.
- `-o yaml` : affiche le résultat au format YAML (au lieu du format tableau par défaut).
- `> cache-pod.yaml` : redirige cette sortie YAML vers un fichier sur disque, au lieu de juste l'afficher dans le terminal.

**Pourquoi `=client` est important :** sans cette précision, l'ancienne syntaxe `--dry-run` (sans valeur) est dépréciée dans les versions récentes de kubectl — elle peut afficher un avertissement ou se comporter différemment.

**Pourquoi le `>` est important :** sans la redirection vers un fichier, le YAML s'affiche juste dans le terminal et disparaît — vous ne pourrez pas le réutiliser avec `apply` à l'étape suivante.

```bash
kubectl apply -f cache-pod.yaml
```
**Ce que ça fait :** cette fois, CRÉE réellement le Pod dans le cluster, à partir du fichier YAML généré à l'étape précédente.
**Pourquoi cette étape est obligatoire :** `dry-run` à lui seul ne crée jamais rien — c'est uniquement un générateur de YAML. Sans cet `apply`, le Pod n'existe que sur votre disque, pas dans le cluster.

```bash
kubectl get pod cache-pod -n infra --show-labels
```
**Ce que ça fait :** vérifie que le Pod existe bien dans le cluster ET affiche ses labels dans une colonne supplémentaire.
**Pourquoi vérifier les labels précisément :** le critère de l'exercice porte spécifiquement sur le label "tier=cache" — `--show-labels` permet de confirmer visuellement qu'il est bien présent, pas juste de constater que le Pod tourne.

**Avec les alias configurés en début de séance (raccourci, même résultat) :**

```bash
alias k=kubectl
```
**Ce que ça fait :** crée un raccourci — taper `k` exécute la même chose que taper `kubectl`. Gagne du temps sur chaque commande tapée ensuite.

```bash
export do="--dry-run=client -o yaml"
```
**Ce que ça fait :** stocke la chaîne `--dry-run=client -o yaml` dans une variable nommée `do`. On peut ensuite l'insérer avec `$do` au lieu de retaper ces flags en entier.

```bash
k run cache-pod --image=redis:7 -n infra --labels="tier=cache" $do > cache-pod.yaml
```
**Ce que ça fait :** exactement la même commande que plus haut, mais `k` remplace `kubectl` et `$do` remplace `--dry-run=client -o yaml`. Résultat identique, moins de frappe — utile quand le temps presse à l'exam.

---

## seance1-term-replace
**Modifier un Pod sans le supprimer**

Pour un Pod seul, on ne peut pas modifier directement l'image avec un simple `kubectl set image` de façon fiable comme pour un Deployment. La méthode CKAD la plus sûre est : exporter le YAML, modifier l'image, puis remplacer le Pod avec `kubectl replace --force`.

```bash
kubectl config use-context k8s-prod
```
**Ce que ça fait :** bascule kubectl sur le cluster "k8s-prod" avant toute action.

```bash
kubectl get pod nginx-pod -n monitoring -o yaml > nginx-pod.yaml
```
**Ce que ça fait :** récupère le manifest RÉEL du Pod tel qu'il existe dans le cluster, et l'enregistre dans un fichier local — base de travail fidèle, pas besoin de la réécrire de mémoire.

```bash
sed -i 's|image: nginx:1.25|image: nginx:1.26|' nginx-pod.yaml
```
**Ce que ça fait :** remplace le texte "image: nginx:1.25" par "image: nginx:1.26" directement dans le fichier, sans ouvrir d'éditeur. `sed` est un outil de remplacement de texte en ligne de commande ; `-i` modifie le fichier sur place (in-place).
**Alternative :** vous pouvez ouvrir le fichier dans un éditeur (vi, nano) et faire la modification à la main — `sed` est juste plus rapide si vous êtes à l'aise avec.

```bash
kubectl replace --force -f nginx-pod.yaml
```
**Ce que ça fait :** SUPPRIME le Pod existant puis le RECRÉE immédiatement avec le contenu du fichier modifié — en une seule commande.
**Pourquoi cette méthode et pas `kubectl edit` ou `kubectl patch` :** un Pod seul (contrairement à un Deployment) a des champs souvent immutables une fois créé — l'image en fait partie. `edit` et `patch` tentent de modifier le Pod existant en place et peuvent être refusés. `replace --force` contourne le problème en ne modifiant rien : il supprime et recrée entièrement. **Contrepartie :** court instant de downtime, le temps de la recréation.

```bash
kubectl get pod nginx-pod -n monitoring -o jsonpath='{.spec.containers[0].image}{"\n"}'
```
**Ce que ça fait :** extrait UNIQUEMENT la valeur du champ image du premier container, sans rien d'autre autour. `jsonpath` est un langage de requête qui cible un champ précis dans une structure JSON/YAML — utile à l'exam pour vérifier un seul détail rapidement.

**Résultat attendu :**

```
nginx:1.26
```

---
---

# PARTIE 3 — YAML

---

## seance1-yaml-pod-simple
**Pod nginx + labels**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  namespace: monitoring
  labels:
    app: nginx
    env: prod
spec:
  restartPolicy: Always
  containers:
  - name: nginx
    image: nginx:1.25
```

Points clés :
- `apiVersion: v1` pour un Pod — PAS `apps/v1` (réservé aux Deployments)
- `namespace` dans `metadata` — jamais dans `spec`
- `containers` est une LISTE — le tiret `-` est obligatoire
- `restartPolicy: Always` est la valeur par défaut, mais l'écrire évite toute ambiguïté

Vérif :

```bash
kubectl apply -f pod.yaml
```

```bash
kubectl get pod nginx-pod -n monitoring
```
→ Running

```bash
kubectl get pod nginx-pod -n monitoring --show-labels
```
→ labels présents

---

## seance1-yaml-initcontainer
**Pod avec initContainer**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: worker
  namespace: workers
spec:
  initContainers:
  - name: init-task
    image: busybox
    command: ["echo", "init ok"]
  containers:
  - name: worker
    image: busybox
    command: ["/bin/sh"]
    args: ["-c", "echo running; sleep 3600"]
    env:
    - name: APP_ENV
      value: "production"
    resources:
      requests:
        memory: "64Mi"
        cpu: "125m"
      limits:
        memory: "128Mi"
        cpu: "250m"
```

Points clés :
- `initContainers` au MÊME niveau que `containers`, pas imbriqué dedans
- `command` = équivalent `ENTRYPOINT` (Dockerfile) · `args` = équivalent `CMD`
- `env` est une LISTE de `{name, value}`, pas un dictionnaire `clé: valeur`
- `cpu: 250m` = 0.25 vCPU. Sans le "m" → 250 vCPU entiers, Pod bloqué en `Pending`
- `limits` sans `requests` → Kubernetes copie limits dans requests, peut bloquer le scheduling

Vérif :

```bash
kubectl apply -f worker.yaml
```

```bash
kubectl get pod worker -n workers
```
→ `Init:0/1` pendant l'init, puis `Running`

```bash
kubectl describe pod worker -n workers
```
→ vérifier `APP_ENV` dans Environment, Limits/Requests dans Containers

---

## seance1-yaml-sidecar
**Pod multi-container sidecar**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-sidecar
spec:
  containers:
  - name: app
    image: busybox
    command: ["sleep", "3600"]
  - name: logger
    image: busybox
    command: ["sh", "-c", "while true; do date; sleep 5; done"]
```

Points clés :
- 2e container = sidecar, même Pod
- `kubectl logs` sans `-c` sur un Pod multi-container → erreur "specify container"
- `READY` doit afficher `2/2`
- Réseau partagé (`localhost`) — filesystem PAS partagé (sauf volume, séance 2)

```bash
kubectl logs app-with-sidecar -c logger
```

```bash
kubectl logs app-with-sidecar -c app
```

```bash
kubectl exec -it app-with-sidecar -c app -- sh
```

```bash
kubectl get pod app-with-sidecar
```
→ READY doit afficher 2/2

---

## seance1-yaml-bonus-complet
**Pod complet — toutes les contraintes**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: monitoring-agent
  namespace: payments
  labels:
    tier: monitoring
spec:
  restartPolicy: Always
  initContainers:
  - name: config-check
    image: busybox
    command: ["echo", "config check ok"]
  containers:
  - name: agent
    image: busybox
    command: ["/bin/sh"]
    args: ["-c", "while true; do echo monitoring active; sleep 30; done"]
    env:
    - name: ENVIRONMENT
      value: "production"
    - name: CONFIG_HOST
      value: "config.payments.svc"
    resources:
      requests:
        cpu: "250m"
        memory: "64Mi"
      limits:
        cpu: "500m"
        memory: "256Mi"
  - name: metrics-collector
    image: busybox
    command: ["/bin/sh"]
    args: ["-c", "while true; do echo metrics collected; sleep 15; done"]
```

Combine : initContainer + multi-container + env + resources + labels.
Resources s'appliquent uniquement au container `agent`, pas au `metrics-collector`.

---

## Hors-scope séance 1

| Sujet | Séance |
|---|---|
| Services | 2 |
| Volumes | 2 |
| Probes | 3 |
| ConfigMaps/Secrets | 3 |
| Deployments | 4 |
