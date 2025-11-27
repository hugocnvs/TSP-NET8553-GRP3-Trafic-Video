
# Validation du Slice 5G eMBB (NexSlice)

## 📘 Introduction

Ce projet vise à valider la capacité d’un slice 5G de type **eMBB (Enhanced Mobile Broadband)** à supporter un trafic vidéo lourd simulant un usage réel (streaming HD / 4K).
L’ensemble de l’architecture s’appuie sur un déploiement **Cloud-Native Kubernetes**, avec les fonctions 5G OAI, un UE simulé UERANSIM et un serveur vidéo dédié.

Les tests ont été réalisés afin de mesurer :

* La stabilité du débit sous contrainte (QoS à 5 Mbps)
* Le comportement en mode *Best Effort* (débit maximal)
* La corrélation des mesures via Grafana sur l’UPF
* La capacité du chemin utilisateur (User Plane 5G) à transporter un flux vidéo continu

---

# 📚 1. Contexte du Scénario Vidéo eMBB

## 1.1 Objectif et Choix Techniques

L'objectif de cette phase était de valider la capacité du slice eMBB (Enhanced Mobile Broadband) à supporter un flux applicatif lourd, simulant un usage réel de type "Streaming Vidéo HD/4K".

Compte tenu de l'environnement d'exécution du projet (Architecture Headless / VM sans interface graphique), j'ai opté pour une approche "Cloud-Native" simulant un streaming HTTP (Progressive Download), technologie utilisée par les plateformes de VOD majeures (Netflix, YouTube).

Au lieu de lancer une interface VLC graphique (incompatible avec l'environnement), j'ai mis en place l'architecture suivante :

* **Serveur de Contenu (Content Provider)** : Déploiement d'un pod nginx hébergeant un fichier vidéo haute définition simulé (500 Mo).
* **Client (UE)** : Utilisation de l'outil curl configuré pour simuler un lecteur vidéo, avec forçage du routage via l'interface 5G.

---

## 1.2 Architecture Déployée

J'ai procédé au déploiement des services applicatifs directement dans le cluster Kubernetes, aux côtés des fonctions réseau 5G.

On observe dans l’état des pods :

* Les fonctions du cœur 5G (AMF, SMF, UPF...) en statut Running.
* Le pod client **ueransim-ue1** (l'utilisateur simulé).
* Le pod **video-server** que j’ai déployé pour héberger le contenu.

![Pods NexSlice](images/pods.png)`

---

## 1.3 Réalisation des Tests et Résultats

### Scénario A : Simulation d’un Flux Streaming Régulé (QoS)

Pour ce premier test, j’ai limité le débit à **5 Mo/s (40 Mbps)**.

Observations :

* **UE** : `curl --limit-rate 5M` montre un débit parfaitement stable (5120k).
* **Grafana** : augmentation nette du trafic UPF, confirmant que le flux vidéo transite bien par le plan utilisateur 5G.

![Streaming 5Mbps](img/3.png)`

### Scénario B : Test de Capacité Maximale (Sans Limite)

En supprimant la limitation :

* **UE** : téléchargement à 12.6 Mo/s (~100 Mbps)
* **Grafana** : pic à 19.2 MB/s sur l’UPF

📷 *Image ici*
![Grafana Unlimited](img/4.png)

---

## 1.4 Conclusion

L'expérimentation valide fonctionnellement le cas d'usage eMBB.

* Débit garanti de 40 Mbps sans jitter pour du streaming 4K
* Débit maximal constaté ≈ 100 Mbps
* UPF correctement dimensionné et observé en charge réelle via Grafana

---

# ⚙️ 2. Commandes et Exécutions

## 2.1 Infrastructure complète

```bash
sudo k3s kubectl get pods -n nexslice -o wide
```
![Grafana 1](img/1.png)

---

## 2.2 Test Connectivité 5G

```bash
export UE1_POD=$(sudo k3s kubectl get pods -n nexslice | grep "ueransim-ue1" | awk '{print $1}')
clear
sudo k3s kubectl exec -it -n nexslice $UE1_POD -- ip a show uesimtun0
```
![Grafana 1](img/2.png)

---

## 2.3 Test QoS Streaming

### Variables

```bash
export UE1_POD=$(sudo k3s kubectl get pods -n nexslice | grep "ueransim-ue1" | awk '{print $1}')
export VIDEO_IP=$(sudo k3s kubectl get pod video-server -n nexslice -o jsonpath='{.status.podIP}')
```

### Streaming bridé (5 Mbps)

```bash
sudo k3s kubectl exec -it -n nexslice $UE1_POD -- curl --interface uesimtun0 http://$video_ip/movie.mp4 -o /dev/null --limit-rate 5M
```
![Grafana 1](img/5.png)

---

# 🗄️ 3. YAML du Serveur Vidéo

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: video-server
  namespace: nexslice
  labels:
    app: video-server
spec:
  containers:
    - name: nginx-streamer
      image: nginx:alpine
      ports:
        - containerPort: 80
      resources:
        limits:
          memory: "256Mi"
          cpu: "500m"
      lifecycle:
        postStart:
          exec:
            command:
              [
                "/bin/sh",
                "-c",
                "echo 'Génération du fichier vidéo HD...'; dd if=/dev/zero of=/usr/share/nginx/html/movie.mp4 bs=1M count=500; echo 'Vidéo prête.'"
              ]
```

---

# 🤖 4. Script Automatisé de Test

```bash
#!/bin/bash

echo "=================================================="
echo "   TEST AUTOMATISÉ : STREAMING VIDEO 5G (NexSlice)"
echo "=================================================="

echo "[1/3] Recherche du Pod UE..."
UE_POD=$(sudo k3s kubectl get pods -n nexslice | grep "ueransim-ue1" | awk '{print $1}')

if [ -z "$UE_POD" ]; then
    echo "ERREUR : Aucun UE trouvé. Vérifiez que le RAN est démarré."
    exit 1
fi
echo "      -> Client trouvé : $UE_POD"

echo "[2/3] Recherche de l'IP du Serveur Vidéo..."
VIDEO_IP=$(sudo k3s kubectl get pod video-server -n nexslice -o jsonpath='{.status.podIP}')

if [ -z "$VIDEO_IP" ]; then
    echo "ERREUR : Serveur vidéo introuvable."
    exit 1
fi
echo "      -> Cible trouvée : $VIDEO_IP"

echo "[3/3] Lancement du téléchargement via l'interface 5G (uesimtun0)..."
sudo k3s kubectl exec -it -n nexslice $UE_POD -- curl --interface uesimtun0 http://$VIDEO_IP/movie.mp4 -o /dev/null --limit-rate 5M

echo "=================================================="
echo "   FIN DU TEST"
echo "=================================================="
```

---

Si tu veux :
✔️ version PDF
✔️ version GitHub stylée
✔️ ajout d’icônes, couleurs, schémas

Je peux te la générer immédiatement.

