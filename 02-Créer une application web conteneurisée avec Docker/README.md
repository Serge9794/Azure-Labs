# 🚀 Création et Déploiement d'une Application Web Conteneurisée avec Docker et Azure

**Auteur :** Serge TOGNON — [Mon profil LinkedIn]( https://www.linkedin.com/in/serge-tognon-a63443187/?lipi=urn%3Ali%3Apage%3Ad_flagship3_profile_view_base_contact_details%3BiHxYnwbJRjOOflTaGqZasw%3D%3D)  
**Technologies :** WSL 2 · Docker Desktop · Git · .NET Core · Azure Container Registry (ACR) · Azure Container Instances (ACI)

---

## 📝 Description du Projet

Ce projet illustre le **cycle DevOps complet** : conteneurisation d'une API web .NET Core (système de réservation hôtelière) avec Docker, suivie de son déploiement cloud sur Microsoft Azure via un registre privé (ACR) et une instance de conteneur publique (ACI).

> 💡 **Compétences démontrées :** rédaction d'un Dockerfile multi-étapes, build et test local d'une image Docker, push vers un registre Azure privé, déploiement et exposition publique sur Internet.

---

## 🛠️ PARTIE 1 — Conteneurisation et test local

### Étape 1 — Récupération du code source

Depuis le terminal **WSL** (Windows Subsystem for Linux), clonage du dépôt GitHub officiel du projet :

```bash
git clone https://github.com/MicrosoftDocs/mslearn-hotel-reservation-system.git
cd mslearn-hotel-reservation-system/src
```
<img width="947" height="361" alt="1" src="https://github.com/user-attachments/assets/b39d2968-a408-436e-a951-b94f0fc3af48" />

> **Pourquoi WSL ?**  
> WSL permet d'exécuter un environnement Linux natif sous Windows. Docker Desktop s'y intègre parfaitement et offre les mêmes commandes qu'un serveur Linux en production — idéal pour un workflow DevOps professionnel.

---

### Étape 2 — Écriture du Dockerfile (build multi-étapes)

Création du fichier `Dockerfile` dans le répertoire `/src`. Le **multi-stage build** est une bonne pratique DevOps : on sépare la phase de **compilation** de la phase d'**exécution**, ce qui réduit considérablement la taille de l'image finale (pas de SDK dans l'image de prod).

```dockerfile
# ── ÉTAPE 1 : Compilation avec le SDK .NET complet ──────────────────────────
FROM mcr.microsoft.com/dotnet/core/sdk:2.2
WORKDIR /src

# Restauration des dépendances NuGet
COPY ["/HotelReservationSystem/HotelReservationSystem.csproj", "HotelReservationSystem/"]
COPY ["/HotelReservationSystemTypes/HotelReservationSystemTypes.csproj", "HotelReservationSystemTypes/"]
RUN dotnet restore "HotelReservationSystem/HotelReservationSystem.csproj"

# Compilation en mode Release
COPY . .
WORKDIR "/src/HotelReservationSystem"
RUN dotnet build "HotelReservationSystem.csproj" -c Release -o /app

# Publication de l'application
RUN dotnet publish "HotelReservationSystem.csproj" -c Release -o /app

# ── ÉTAPE 2 : Image d'exécution légère ──────────────────────────────────────
EXPOSE 80
WORKDIR /app
ENTRYPOINT ["dotnet", "HotelReservationSystem.dll"]
```
<img width="939" height="443" alt="2" src="https://github.com/user-attachments/assets/7508ab27-bf77-4ccf-b4f6-789921c86bca" />

> **Explication des instructions clés :**
> | Instruction | Rôle |
> |---|---|
> | `FROM` | Définit l'image de base (SDK .NET pour compiler) |
> | `WORKDIR` | Définit le répertoire de travail dans le conteneur |
> | `RUN dotnet restore` | Télécharge les packages NuGet (dépendances) |
> | `RUN dotnet publish` | Compile et prépare l'app pour la production |
> | `EXPOSE 80` | Déclare que le conteneur écoute sur le port 80 (HTTP) |
> | `ENTRYPOINT` | Commande lancée automatiquement au démarrage du conteneur |

---

### Étape 3 — Construction de l'image Docker

```bash
docker build -t reservationsystem .
```

> **Ce qui se passe :** Docker lit le `Dockerfile` ligne par ligne, exécute chaque instruction dans un calque (layer) isolé et empile ces calques pour former l'image finale. Le flag `-t` (tag) lui donne le nom `reservationsystem`.

Vérification de la présence de l'image dans le registre local :

```bash
docker image list
```

<img width="960" height="474" alt="3" src="https://github.com/user-attachments/assets/82a9e3a7-9d9f-431b-b555-c12abaac8173" />


---

### Étape 4 — Exécution et test local du conteneur

Lancement du conteneur en **mode détaché**, avec redirection de port :

```bash
docker run -p 8080:80 -d --name reservationscontainer reservationsystem
```

> **Décryptage des options :**
> | Option | Signification |
> |---|---|
> | `-p 8080:80` | Redirige le port 8080 du PC hôte vers le port 80 du conteneur |
> | `-d` | Mode détaché : le conteneur tourne en arrière-plan |
> | `--name reservationscontainer` | Donne un nom lisible au conteneur pour le gérer facilement |

Vérification du statut :

```bash
docker ps -a
```

<img width="950" height="313" alt="4" src="https://github.com/user-attachments/assets/7d6eb33d-c6cc-4c36-b7e0-06f20230f0d3" />
<img width="759" height="270" alt="4&#39;" src="https://github.com/user-attachments/assets/5d03bf3d-2cc2-42fb-8b65-fdc627d301b3" />


---

### ✅ Résultat du test local — API opérationnelle

En naviguant sur `http://localhost:8080/api/reservations/1`, l'API retourne le JSON suivant, confirmant que le conteneur fonctionne correctement en local :

<img width="960" height="218" alt="5" src="https://github.com/user-attachments/assets/8a1ef15d-cff7-4ac6-9473-59be07a0c24d" />


> **Analyse de la réponse :**  
> L'API renvoie un objet de réservation complet avec `reservationID: 1`, `customerID`, `hotelID: "Hôtel 1031518109"`, les dates de `checkin` / `checkout`, le nombre d'invités (`numberOfGuests: 4`) ainsi qu'un commentaire encodé en Base64 — preuve que l'ensemble de la logique métier de l'application fonctionne dans le conteneur.

Nettoyage du conteneur local après validation :

```bash
docker container stop reservationscontainer
docker rm reservationscontainer
```

---

## ☁️ PARTIE 2 — Déploiement sur Azure (ACR + ACI)

### Étape 1 — Création du groupe de ressources

Un **groupe de ressources** Azure est un conteneur logique qui regroupe toutes les ressources d'un même projet pour faciliter leur gestion, leur facturation et leur suppression.

Sur le **Portail Azure** (`portal.azure.com`) :

1. **"Créer une ressource"** → **"Groupe de ressources"**
2. Renseigner :
   - **Nom :** `rg-hotel-conteneur`
   - **Région :** `France Central` (ou la plus proche)
3. **"Vérifier + créer"** → **"Créer"**

> 💡 **Bonne pratique :** en regroupant toutes les ressources sous `rg-hotel-conteneur`, il suffira de supprimer ce seul groupe à la fin de l'exercice pour effacer toute l'infrastructure et éviter des frais inutiles.

---

### Étape 2 — Création du registre Azure Container Registry (ACR)

L'**ACR** est un registre Docker **privé** hébergé sur Azure. Contrairement à Docker Hub (public), les images y sont accessibles uniquement par les personnes et services autorisés.

1. Chercher **"Container Registry"** → **"Créer"**
2. Renseigner :
   - **Groupe de ressources :** `rg-hotel-conteneur`
   - **Nom du registre :** `hotelacr`
   - **SKU :** `Standard`
3. Après création, aller dans **Paramètres → Clés d'accès**
4. Activer **"Utilisateur Administrateur"** pour obtenir :
   - Serveur de connexion : `hotelacr.azurecr.io`
   - Nom d'utilisateur et mot de passe

<img width="960" height="310" alt="6" src="https://github.com/user-attachments/assets/b8982818-6a62-426c-bbbc-cb81ba560ed3" />
<img width="958" height="400" alt="7" src="https://github.com/user-attachments/assets/53418756-0dc3-41ed-ba30-0ffaf9402fda" />


> **Pourquoi activer l'utilisateur Admin ?**  
> Cela génère des identifiants permettant à la commande `docker login` de s'authentifier auprès du registre privé Azure pour y pousser ou tirer des images.

---

### Étape 3 — Publication de l'image vers Azure ACR

On **étiquette** (tag) l'image locale avec l'adresse complète du registre Azure, puis on l'envoie :

```bash
# 1. Tag de l'image avec l'URL complète du registre Azure
docker tag reservationsystem:latest hotelacr.azurecr.io/reservationsystem:latest

# 2. Authentification auprès du registre privé
docker login hotelacr.azurecr.io

# 3. Envoi de l'image vers Azure
docker push hotelacr.azurecr.io/reservationsystem:latest
```
<img width="899" height="389" alt="8" src="https://github.com/user-attachments/assets/916f812d-6f50-442c-bf07-f756afbde3d8" />
<img width="949" height="361" alt="9" src="https://github.com/user-attachments/assets/58cd1585-a697-433d-a02b-e485664388d5" />

> **Pourquoi le tag ?**  
> Docker identifie les registres de destination par le préfixe du nom d'image. En ajoutant `hotelcr.azurecr.io/` devant le nom, on indique explicitement à Docker où envoyer l'image lors du `push`.

Après le push, vérifier dans le portail : **ACR `hotelacr` → Référentiels** — l'image `reservationsystem:latest` doit y apparaître.

<img width="960" height="425" alt="10" src="https://github.com/user-attachments/assets/16f08331-50fb-465b-b8e3-d58723344ccf" />
<img width="959" height="449" alt="11" src="https://github.com/user-attachments/assets/07e8afa1-17a4-4048-84e5-e14a98efce85" />


---

### Étape 4 — Déploiement public via Azure Container Instances (ACI)

**Azure Container Instances** permet de déployer un conteneur directement dans le cloud Azure **sans gérer de serveur ni de cluster**. C'est la solution la plus rapide pour exposer une application conteneurisée publiquement.

1. Chercher **"Container Instances"** → **"Créer"**
2. **Onglet "Notions de base" :**
   - **Groupe de ressources :** `rg-hotel-conteneur`
   - **Nom du conteneur :** `hotel-app-instance`
   - **Source d'image :** Azure Container Registry
   - **Registre :** `hotelacr`
   - **Image :** `reservationsystem` · **Tag :** `latest`
   - **Système d'exploitation :** Linux
3. **Onglet "Mise en réseau" :**
   - **Type :** Public
   - **Étiquette de nom DNS :** `hotel-serge-tognon` *(nom unique de votre choix)*
   - **Port :** `80` (TCP)
4. **"Vérifier + créer"** → **"Créer"**

> **Qu'est-ce qu'une étiquette DNS ?**  
> Elle génère automatiquement un **FQDN** (nom de domaine complet) public sous la forme `hotel-serge-tognon.<région>.azurecontainer.io` — permettant d'accéder à l'application depuis n'importe quel navigateur dans le monde, sans configuration DNS manuelle.

<img width="960" height="408" alt="12" src="https://github.com/user-attachments/assets/506b384c-0063-49f3-ba09-7df7f3d8a9ba" />


---

## 🎯 Résultat Final — Validation Cloud

L'application est désormais accessible **publiquement depuis n'importe où dans le monde**.

URL d'accès publique générée par Azure :

```
http://hotel-serge-tognon.efa4gubtgee2frbc.francecentral.azurecontainer.io/api/reservations/1
```

<img width="960" height="290" alt="13" src="https://github.com/user-attachments/assets/f3601aaa-c4db-43f6-8797-1ba1800528fb" />


---

## 🧹 Nettoyage des ressources Azure

Pour éviter tout frais résiduel après l'exercice, supprimer le groupe de ressources entier :

```bash
az group delete --name rg-hotel-conteneur --yes --no-wait
```

Ou via le portail : **`rg-hotel-conteneur` → Supprimer le groupe de ressources**.

---

## 📊 Récapitulatif des ressources créées

| Ressource | Nom | Type Azure | Rôle |
|---|---|---|---|
| Groupe de ressources | `rg-hotel-conteneur` | Resource Group | Conteneur logique du projet |
| Registre de conteneurs | `sergetognonacr` | Azure Container Registry | Stockage privé de l'image Docker |
| Instance de conteneur | `hotel-app-instance` | Azure Container Instances | Hébergement public de l'application |

---

## 💡 Conclusion

Ce projet valide la maîtrise du **cycle DevOps complet** :

| Étape | Action |
|---|---|
| 1️⃣ Conteneurisation | Rédaction d'un Dockerfile multi-étapes optimisé |
| 2️⃣ Test local | Build, run et validation via Docker / WSL |
| 3️⃣ Registre privé | Push de l'image vers `sergetognonacr` (ACR) |
| 4️⃣ Déploiement cloud | Exposition publique via `hotel-app-instance` (ACI) |
| 5️⃣ Validation | API accessible mondialement et fonctionnelle ✅ |

> 🎓 **Compétences prouvées :** Docker · Dockerfile · WSL 2 · Azure Portal · ACR · ACI · Architecture microservices · CI/CD · Déploiement cloud sans infrastructure serveur.
--------------------------------------------------------------------------------------------------------------------------------------
🚧**Difficultés Rencontrées et Solutions Apportées**
Ce projet, bien que structuré et documenté, n'a pas été exempt d'obstacles techniques. Voici les deux principales difficultés rencontrées et la manière dont elles ont été surmontées.

⚠️ **Difficulté 1 — Échec du dotnet restore : erreur SSL lors du docker build**
**Symptôme** : Lors du premier docker build -t reservationsystem ., le build échouait à l'étape 8 du Dockerfile — précisément à l'instruction RUN dotnet restore. Le terminal affichait une cascade d'erreurs SSL :
**error** : Failed to retrieve information about 'Microsoft.AspNetCore...' 
The SSL connection could not be established.
Authentication failed because the remote party has closed the transport stream.
**ERROR**: failed to build: failed to solve: process "/bin/sh -c dotnet restore..." 
did not complete successfully: exit code: 1

<img width="956" height="284" alt="Capture d’écran 2026-05-28 105324" src="https://github.com/user-attachments/assets/a5ce2419-ed5d-416d-89bc-e36ab5f8ec5a" />


**Cause identifiée** : L'image de base mcr.microsoft.com/dotnet/core/sdk:2.2 est très ancienne (fin de vie en 2019). Lors de l'exécution du dotnet restore à l'intérieur du conteneur, celui-ci tentait de contacter api.nuget.org via HTTPS, mais les certificats TLS embarqués dans cette vieille image étaient expirés ou incompatibles avec les autorités de certification actuelles — provoquant l'échec de la connexion sécurisée.
Solution adoptée : La résolution est passée par la configuration DNS de Docker Desktop. En ajoutant les serveurs DNS publics de Google (8.8.8.8 et 8.8.8.4) dans les paramètres du moteur Docker (Docker Engine > fichier de configuration JSON), la résolution de nom et la connectivité réseau du daemon Docker ont été stabilisées, permettant au build de s'exécuter correctement en exploitant le cache de couches déjà résolu.

<img width="954" height="509" alt="Capture d’écran 2026-05-28 111652" src="https://github.com/user-attachments/assets/75c1ea88-d8b5-41c1-8298-b46c4cb3d9d3" />


json{
  "dns": ["8.8.8.8", "8.8.8.4"]
}
Après application de cette configuration et redémarrage du daemon Docker, le docker build s'est déroulé sans erreur, les 15 étapes (layers) s'exécutant jusqu'au FINISHED.

💡 **Leçon retenue**
ProblèmeCause racineSolutiondotnet restore — SSL failureImage SDK .NET 2.2 obsolète + DNS Docker instableConfiguration des DNS publics Google dans Docker EngineBuild bloqué à l'étape RUNConnectivité réseau du daemon Docker insuffisanteRedémarrage du daemon après mise à jour de la config JSON

Bonne pratique à retenir : Lorsqu'un docker build échoue sur une commande réseau (restore, apt-get, npm install…), la première piste à investiguer est la configuration DNS du daemon Docker — avant même de remettre en cause le Dockerfile lui-même. Un simple ajout de 8.8.8.8 dans daemon.json peut débloquer la situation en quelques secondes.


[LINKEDLN](https://www.linkedin.com/in/serge-tognon-a63443187/?lipi=urn%3Ali%3Apage%3Ad_flagship3_profile_view_base_contact_details%3BiHxYnwbJRjOOflTaGqZasw%3D%3D)
