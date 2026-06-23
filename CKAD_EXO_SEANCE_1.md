# 🎯 CKAD : Séance 1, Pods
### Récap des exercices + solutions : à garder pour réviser !



## 🧠 Exercice 1 : Quiz de raisonnement

### 📋 Énoncé

**S1.** Ton Pod est en `Pending` depuis 3 minutes. Quelle commande tu lances, et qu'est-ce que tu cherches dans la sortie ?

**S2.** L'énoncé dit "le Pod doit s'exécuter une seule fois et ne jamais redémarrer, même en cas d'erreur". Quelle `restartPolicy` utilises-tu ?

**S3.** `kubectl apply -f pod.yaml` affiche `pod/my-pod created`. Est-ce suffisant pour valider ta réponse à l'exam ?

**S4.** Quelle différence entre `kubectl create` et `kubectl apply` ? Lequel utiliser à l'exam ?



### ✅ Solution

**S1.** 👉
```bash
kubectl describe pod <name> -n <ns>
```
🔍 Regarde la section **Events** en bas de la sortie. Causes fréquentes : pas assez de ressources sur les nodes, taint non tolérée, image impossible à puller.

**S2.** 👉 `restartPolicy: Never`
⚠️ Pas `OnFailure`, celui-là redémarre si le container plante (exit ≠ 0), donc il pourrait redémarrer. `Never` ne redémarre **jamais**, quoi qu'il arrive.

**S3.** 👉 **Non !** 🚫
`created` veut juste dire que l'API a accepté ton fichier, pas que le Pod tourne vraiment. Toujours vérifier après :
```bash
kubectl get pod my-pod -n <ns>
# tu veux voir : Running ou Completed
```

**S4.** 👉
- `create` 🙅 échoue si la ressource existe déjà
- `apply` 🔁 crée ou met à jour, sans jamais planter

💡 **À l'exam : toujours `apply`.** Si tu resoumets par erreur, ça passe quand même.



## 🛠️ Exercice 2 : Boîte à outils kubectl

### 📋 Énoncé

> Cluster `k8s-staging`. Le Pod `api-pod` (namespace `backend`) est en `CrashLoopBackOff` 💥. Sans le modifier, trouve la cause du crash et note l'exit code.



### ✅ Solution

**1️⃣ Switch de contexte d'abord, toujours :**
```bash
kubectl config use-context k8s-staging
```

**2️⃣ Enquête pas à pas :**
```bash
kubectl get pod api-pod -n backend
kubectl get pod api-pod -n backend -o wide      # 🖥️ sur quel node ?
kubectl describe pod api-pod -n backend         # 🔎 Events + Last State
kubectl logs api-pod -n backend
kubectl logs api-pod -n backend --previous      # 🕐 logs AVANT le crash
```

**3️⃣ Aller plus loin :**
```bash
kubectl get pod api-pod -n backend -o yaml      # voir tout le manifest réel
kubectl exec -it api-pod -n backend -- sh       # entrer dans le container
kubectl explain pod.spec.containers             # 📖 la doc intégrée (pas de Google à l'exam !)
```

**4️⃣ Pod de debug jetable :**
```bash
kubectl run debug --image=busybox \
  --restart=Never --rm -it \
  -n backend -- sh
# --rm = disparaît tout seul quand tu sors 🧹
```

⚠️ **Piège classique** : `kubectl logs` sans `--previous` ne renvoie rien si le container a déjà redémarré après son crash.



## ✍️ Exercice 3 : Écrire un Pod simple

### 📋 Énoncé

> Cluster `k8s-prod`. Crée un Pod nginx dans le namespace `monitoring`, avec les labels `app=nginx` et `env=prod`. Il doit redémarrer automatiquement s'il s'arrête.



### ✅ Solution

**Avant tout, le réflexe context switch :**
```bash
kubectl config use-context k8s-prod
kubectl get ns monitoring   # existe-t-il déjà ?
kubectl create ns monitoring   # sinon, on le crée
```

**Le YAML :**
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

**Vérification, ne JAMAIS sauter cette étape ✋ :**
```bash
kubectl apply -f pod.yaml
kubectl get pod nginx-pod -n monitoring                # → Running ?
kubectl get pod nginx-pod -n monitoring --show-labels  # → labels présents ?
```

⚠️ **Pièges à éviter :**
- `apiVersion: v1` pour un Pod, pas `apps/v1` (ça c'est pour les Deployments) 🙅
- `namespace` se met dans `metadata`, pas dans `spec`
- `containers` est une **liste** → tiret `-` obligatoire devant `name`



## ⚡ Exercice 4 : Speed run dry-run

### 📋 Énoncé

> Cluster `k8s-dev`. Crée en moins de **2 minutes** ⏱️ un Pod `cache-pod` (image `redis:7`, namespace `infra`, label `tier=cache`), interdit d'écrire le YAML à la main !



### ✅ Solution

```bash
kubectl config use-context k8s-dev

kubectl run cache-pod --image=redis:7 \
  -n infra --labels="tier=cache" \
  --dry-run=client -o yaml > cache-pod.yaml

kubectl apply -f cache-pod.yaml
kubectl get pod cache-pod -n infra --show-labels
```

🚀 **Astuce gain de temps** : configure ces alias en début de séance/exam pour aller encore plus vite :
```bash
alias k=kubectl
export do="--dry-run=client -o yaml"

k run cache-pod --image=redis:7 -n infra --labels="tier=cache" $do > cache-pod.yaml
```

🎁 **Bonus utile**, copier un fichier dans/depuis un Pod :
```bash
kubectl cp cache-pod:/data/fichier.txt ./local.txt -n infra
```

⚠️ **Pièges à éviter :**
- `--dry-run` sans `=client` est dépassé → toujours `--dry-run=client`
- Oublier `-o yaml > fichier.yaml` → rien n'est sauvegardé sur le disque !
- `dry-run` ne crée RIEN dans le cluster, il faut quand même faire `kubectl apply` après 😅



## 🧩 Exercice 5 : Le YAML complet

### 📋 Énoncé

> Cluster `k8s-prod`. L'app `worker` (image `busybox`) ne doit démarrer qu'**après** qu'une tâche d'init soit terminée (image `busybox`, commande `echo "init ok"`). L'app lance `/bin/sh -c "echo running; sleep 3600"`. Variable d'env : `APP_ENV=production`. Limite mémoire **128Mi**, CPU **250m**. Namespace : `workers`.



### ✅ Solution

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

🧠 **`command` vs `args`, le truc à retenir :**

| Dockerfile | YAML |
|||
| `ENTRYPOINT` | `command` |
| `CMD` | `args` |

**Vérification :**
```bash
kubectl apply -f worker.yaml
kubectl get pod worker -n workers
# tu verras d'abord : Init:0/1 (la tâche d'init tourne)
# puis : Running (l'app a démarré)

kubectl describe pod worker -n workers
# cherche APP_ENV dans Environment ✅
```

⚠️ **Pièges qui piègent tout le monde :**
- `initContainers` (avec un **s**) est au même niveau que `containers`, **pas dedans** 🙅
- `env` est une **liste** de `{name, value}`, pas un dictionnaire `clé: valeur`
- `250m` = 0.25 vCPU. Si tu oublies le `m` → `250` vCPU et ton Pod ne sera **jamais** schedulé 💀
- `limits` sans `requests` → Kubernetes copie automatiquement les limits dans requests



## 🎁 Bonus 1 : Modifier un Pod qui tourne déjà

### 📋 Énoncé

> Le Pod `nginx-pod` (namespace `monitoring`) tourne avec `nginx:1.25`. Passe-le en `nginx:1.26` sans le supprimer à la main.



### ✅ Solution : 3 façons de faire

**1. `edit` (rapide mais limité) :**
```bash
kubectl edit pod nginx-pod -n monitoring
# change l'image dans vim, sauvegarde
```

**2. `patch` (une ligne, efficace) :**
```bash
kubectl patch pod nginx-pod -n monitoring \
  -p '{"spec":{"containers":[{"name":"nginx","image":"nginx:1.26"}]}}'
```

**3. `replace --force` (la plus fiable à l'exam 🏆) :**
```bash
kubectl get pod nginx-pod -n monitoring -o yaml > pod.yaml
# modifier l'image dans pod.yaml
kubectl replace --force -f pod.yaml
```

⚠️ **Piège :** `kubectl edit` refuse souvent de changer l'image directement (champ immutable) → bascule sur `replace --force`.



## 🎁 Bonus 2 : Pod avec un sidecar

### 📋 Énoncé

> Crée un Pod `app-with-sidecar` avec 2 containers : `app` (busybox, `sleep 3600`) et un sidecar `logger` (busybox, `sh -c "while true; do date; sleep 5; done"`).



### ✅ Solution

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

**Inspecter chaque container :**
```bash
kubectl logs app-with-sidecar -c logger     # 📝 toujours préciser -c
kubectl exec -it app-with-sidecar -c app -- sh
```

✅ **Vérification rapide :**
```bash
kubectl get pod app-with-sidecar
# READY doit afficher 2/2 : les deux containers tournent 🎉
```

⚠️ **Piège :** sans `-c <nom>` sur un Pod multi-container, `kubectl logs` plante avec "specify container".



## 📌 Les 5 trucs à retenir de cette séance

1. 🔁 **`apply`** toujours, pas `create`
2. ✅ **Vérifier après chaque `apply`**, `created` ne veut pas dire `Running`
3. 🗂️ **`-n <namespace>`** sur (presque) toutes les commandes
4. ⚡ **`--dry-run=client -o yaml`** pour ne jamais écrire de YAML à la main
5. 📖 **`kubectl explain`** = ta doc à l'exam, pas Google



*Séance 1 · Pods · Garde ce doc sous la main pour réviser avant la prochaine séance 💪*
