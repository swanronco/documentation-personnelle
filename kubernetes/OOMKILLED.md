# 📑 Fiche Incident : OOMKilled (Out Of Memory)

## 🚨 Symptôme
Lors d'un `kubectl get pods -n <namespace>`, le statut d'un ou plusieurs Pods affiche **`OOMKilled`** et la colonne **`RESTARTS`** augmente.

---

## 🔍 Arbre de Diagnostic (Les 4 Scénarios)

### 📌 Cas n°1 : Le Pod a dépassé sa propre limite (Sous-calibrage / Fuite)

- **Comment le repérer :**

```bash
kubectl describe pod <nom-du-pod> -n <namespace>
```

Regarder la section `Last State: Terminated` -> `Reason: OOMKilled` -> `Exit Code: 137`. Cela prouve que le conteneur a franchi sa propre barrière invisible (`limits.memory`).

- **Action :**
  - Analyser la courbe de RAM sur Grafana. Si elle monte en ligne droite diagonale : **Fuite mémoire** -> Corriger le code.
  - Si elle fait un plateau propre mais sature lors d'un pic de trafic : **Sous-calibrage** -> Augmenter la ligne `limits.memory` dans le fichier YAML du Deployment.

---

### 🚀 Cas n°2 : Effet de meute (Saturation de la machine physique / Worker)

- **Comment le repérer :**
  Ton Pod est tué alors qu'il n'a pas atteint sa limite de RAM personnelle, ou plusieurs Pods différents meurent en même temps sur la même machine.

```bash
kubectl get nodes
kubectl describe node <nom-du-noeud>
```

Vérifier si le nœud affiche la condition : `MemoryPressure = True`.

- **Action :**
  Trouver les voisins fantômes (les environnements de test oubliés par les collègues qui tournent pour rien et accumulent la RAM).

```bash
kubectl top pods -A --sort-by=memory
```

**Résolution :** Sortir le balai 🧹. Supprimer les namespaces inutilisés pour libérer l'immeuble.

---

### 🚀 Cas n°3 : Le mur du Quota (Blocage logique au démarrage)

- **Comment le repérer :**
  Le Pod ne passe même pas en `OOMKilled`, il refuse de démarrer et reste bloqué en statut `Pending`.

```bash
kubectl describe deployment <nom-du-deployment> -n <namespace>
```

Message d'erreur : `Forbidden: exceeded quota`.

- **Action :**
  Le Namespace entier a atteint la limite maximale autorisée par l'infrastructure via un *ResourceQuota*. Faire le ménage dans ce namespace spécifique ou demander aux Ops d'augmenter le quota.

---

### 🚀 Cas n°4 : Crash du Système (L'outil Kubernetes devient fou)

- **Comment le repérer :**
  Le nœud physique passe au statut `NotReady` et ne répond plus du tout à aucune commande `kubectl`.

- **Action :**
  Fuite mémoire rare sur les composants vitaux (`containerd` ou `kubelet`). Le système Linux de la machine commet un "suicide managé" pour survivre. Nécessite une intervention infra (reboot de la VM).

---

### 🔧 Commandes Réflexes (Cheat Sheet)

| Commande | Objectif |
|---|---|
| `kubectl describe pod <pod> -n <ns>` | Inspecter l'**Exit Code 137** (La preuve du OOM) |
| `kubectl top nodes` | Voir instantanément si un serveur physique est à 99% de RAM |
| `kubectl top pods -n <ns>` | Classer les applications les plus gourmandes de ton projet |
| `kubectl get pods -n <ns> -w` | Surveiller en temps réel (watch) les crashs et les redémarrages |
