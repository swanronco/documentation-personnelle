# Guide de dépannage : ImagePullBackOff

## 1. Erreur d'Authentification (Registry Privé)
* **Symptôme :** `Unauthorized` ou `authentication required` dans `kubectl describe pod`.
* **Solution :** Créer un secret et l'ajouter au déploiement.
  ```bash
  # Création du secret
  kubectl create secret docker-registry mon-secret-registry \
    --docker-server=<REGISTRY_URL> \
    --docker-username=<USER> \
    --docker-password=<TOKEN>

  # Ajouter au YAML (spec.template.spec) :
  imagePullSecrets:
    - name: mon-secret-registry

## 2. Erreur de Nom ou de Tag
* **Symptôme :** `manifest unknown` ou `repository does not exist`.
* **Solution :** Vérifier le nom exact et le tag sur le registre.
  ```bash
  # Vérifier l'image locale puis corriger le YAML
  docker pull <IMAGE>:<TAG> # Si ça échoue, l'image est introuvable

## 3. Problème Réseau / DNS
* **Symptôme :** `connection timed out` ou `could not resolve host`.
* **Solution :**
  * Vérifier que le cluster a accès à Internet (nat gateway, DNS).
  * Tester la connectivité depuis un pod de test :
  ```bash
  kubectl run net-test --rm -it --image=alpine -- sh
  # Dans le pod :
  nslookup <REGISTRY_URL>

## 4. Incompatibilité d'architecture (ARM/AMD64)
* **Symptôme :** `exec format error`.
* **Solution :** Recompiler l'image pour la bonne architecture.
  ```bash
  # Exemple Docker pour build multi-arch
  docker buildx build --platform linux/amd64 -t <IMAGE>:<TAG> .

## 5. Rate Limiting (DockerHub)
* **Symptôme :** `429 Too Many Requests`.
* **Solution :** S'authentifier avec un compte DockerHub via `imagePullSecrets` (même pour les images publiques) ou utiliser un miroir de registre.
