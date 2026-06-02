# 🛡️ Azure NSG & Service Endpoints — Sécurité Réseau ERP

> **Auteur :** Serge TOGNON | Administrateur Cloud Azure Certifié (AZ-104) | Candidat RHCSA
> [![LinkedIn](https://img.shields.io/badge/LinkedIn-Serge_TOGNON-0077B5?logo=linkedin)](https://www.linkedin.com/in/serge-tognon-a63443187/?lipi=urn%3Ali%3Apage%3Ad_flagship3_profile_view_base_contact_details%3BiHxYnwbJRjOOflTaGqZasw%3D%3D  )
> [![Azure](https://img.shields.io/badge/Azure-AZ--104_Certified-0078D4?logo=microsoftazure)](https://learn.microsoft.com/certifications/azure-administrator/)

---

## 📋 Table des matières

1. [Architecture du projet](#-architecture-du-projet)
2. [Étape 1 — Création du VNet et des sous-réseaux](#-étape-1--création-du-vnet-et-des-sous-réseaux)
3. [Étape 2 — Création du NSG et des VMs](#-étape-2--création-du-nsg-et-des-vms)
4. [Étape 3 — Test de la connectivité par défaut](#-étape-3--test-de-la-connectivité-par-défaut)
5. [Étape 4 — Règle SSH entrante](#-étape-4--règle-ssh-entrante)
6. [Étape 5 — Règle HTTP : isoler le DataServer](#-étape-5--règle-http--isoler-le-dataserver)
7. [Étape 6 — Application Security Group (ASG)](#-étape-6--application-security-group-asg)
8. [Étape 7 — Service Endpoints pour Azure Storage](#-étape-7--service-endpoints-pour-azure-storage)
9. [Étape 8 — Test final et validation](#-étape-8--test-final-et-validation)
10. [Récapitulatif des règles NSG](#-récapitulatif-des-règles-nsg)
11. [Bonnes pratiques et compétences démontrées](#-bonnes-pratiques-et-compétences-démontrées)

---

## 🎯 Objectif du projet

Une entreprise manufacturière migre son système **ERP vers Azure**. Les serveurs d'application et de base de données doivent coexister dans le même réseau virtuel, mais avec une **isolation stricte des flux** : un serveur de base de données compromis ne doit jamais pouvoir initier de connexion vers les serveurs applicatifs. De plus, des fichiers de schémas techniques confidentiels sont stockés dans Azure Storage et ne doivent être accessibles **que depuis les serveurs de base de données**, jamais depuis les serveurs applicatifs ni depuis Internet.

Ce projet implémente cette architecture de bout en bout sur un abonnement Azure réel, en appliquant trois mécanismes de sécurité réseau complémentaires :

| Mécanisme | Rôle dans ce projet |
|---|---|
| **NSG (Network Security Group)** | Filtrer le trafic entre AppServer et DataServer au niveau port/protocole |
| **ASG (Application Security Group)** | Regrouper les serveurs DB par rôle pour des règles scalables sans gestion d'IP |
| **Service Endpoint** | Restreindre l'accès à Azure Storage au seul sous-réseau Databases, sans exposer les données sur Internet |

À l'issue du projet, les flux suivants sont **prouvés par des tests réels** :

```
✅ SSH entrant           → AppServer et DataServer (administration)
✅ HTTP AppServer        → DataServer (flux applicatif autorisé)
❌ HTTP DataServer       → AppServer (flux bloqué par NSG/ASG)
✅ DataServer            → Azure Storage (via Service Endpoint)
❌ AppServer             → Azure Storage (sous-réseau non autorisé)
```

---

## 🏗 Architecture du projet

```
┌─────────────────────────────────────────────────────────────────────┐
│                    VNet : ERP-servers (10.0.0.0/16)                  │
│                    NSG global : ERP-SERVERS-NSG                      │
│                                                                       │
│  ┌─────────────────────────────┐  ┌──────────────────────────────┐  │
│  │ Subnet : Applications        │  │ Subnet : Databases            │  │
│  │ 10.0.0.0/24                  │  │ 10.0.1.0/24                   │  │
│  │                              │  │                               │  │
│  │  ┌────────────────────┐      │  │  ┌─────────────────────┐     │  │
│  │  │  AppServer          │      │  │  │  DataServer          │     │  │
│  │  │  10.0.0.4           │      │  │  │  10.0.1.4            │     │  │
│  │  │  Ubuntu 22.04       │      │  │  │  Ubuntu 22.04        │     │  │
│  │  │  (Serveur web/app)  │      │  │  │  ASG: ERP-DB-SERVERS │     │  │
│  │  └────────────────────┘      │  │  │  Service Endpoint    │     │  │
│  │                              │  │  │  → Azure Storage     │     │  │
│  └─────────────────────────────┘  └──────────────────────────────┘  │
│                                                                       │
│         ❌ DataServer → AppServer (HTTP port 80 bloqué)              │
│         ✅ AppServer  → DataServer (HTTP port 80 autorisé)           │
│         ✅ DataServer → Azure Storage (via Service Endpoint)         │
│         ❌ AppServer  → Azure Storage (sous-réseau non autorisé)     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 🚀 Étape 1 — Création du VNet et des sous-réseaux

### Objectif
Créer l'infrastructure réseau de base : un VNet avec deux sous-réseaux (Applications et Databases).

### Prérequis
- Azure CLI installée ou Azure Cloud Shell
- Un groupe de ressources existant

### Commandes

```bash
# Définir le groupe de ressources
rg="rg-nsg-erp-lab"

# Créer le groupe de ressources (si besoin)
az group create \
  --name $rg \
  --location WestUS

# Créer le VNet + sous-réseau Applications
az network vnet create \
    --resource-group $rg \
    --name ERP-servers \
    --address-prefixes 10.0.0.0/16 \
    --subnet-name Applications \
    --subnet-prefixes 10.0.0.0/24

# Créer le sous-réseau Databases
az network vnet subnet create \
    --resource-group $rg \
    --vnet-name ERP-servers \
    --address-prefixes 10.0.1.0/24 \
    --name Databases
```

### Vérification

''bash
# Lister les sous-réseaux créés
az network vnet subnet list \
    --resource-group $rg \
    --vnet-name ERP-servers \
    --output table

-
<img width="960" height="265" alt="A1" src="https://github.com/user-attachments/assets/7824e615-1411-4104-b7bd-4a727340560b" />
<img width="916" height="293" alt="A2" src="https://github.com/user-attachments/assets/1cee7b5c-3791-488d-92d4-7870508c80aa" />
<img width="917" height="325" alt="A3" src="https://github.com/user-attachments/assets/4e18c979-c793-49d6-b04f-c840df8f5cbc" />
---

## 🖥️ Étape 2 — Création du NSG et des VMs

### Objectif
Créer le groupe de sécurité réseau `ERP-SERVERS-NSG` et déployer deux VMs Ubuntu associées à ce NSG.

### Commandes

```bash
# Créer le NSG
az network nsg create \
    --resource-group $rg \
    --name ERP-SERVERS-NSG

# Télécharger le fichier cloud-init (installe un serveur web nginx automatiquement)
wget -N https://raw.githubusercontent.com/MicrosoftDocs/mslearn-secure-and-isolate-with-nsg-and-service-endpoints/master/cloud-init.yml

# Créer AppServer dans le sous-réseau Applications
az vm create \
    --resource-group $rg \
    --name AppServer \
    --vnet-name ERP-servers \
    --subnet Applications \
    --nsg ERP-SERVERS-NSG \
    --image Ubuntu2204 \
    --size Standard_D2S_v3 \
    --generate-ssh-keys \
    --admin-username polo \
    --custom-data cloud-init.yml \
    --no-wait \
    --admin-password "<VotreMotDePasseComplexe>"

# Créer DataServer dans le sous-réseau Databases
az vm create \
    --resource-group $rg \
    --name DataServer \
    --vnet-name ERP-servers \
    --subnet Databases \
    --nsg ERP-SERVERS-NSG \
    --image Ubuntu2204 \
    --size Standard_D2S_v3 \
    --generate-ssh-keys \
    --admin-username serge \
    --custom-data cloud-init.yml \
    --no-wait \
    --admin-password "<VotreMotDePasseComplexe>"

# Vérifier que les deux VMs sont en cours d'exécution
az vm list \
    --resource-group $rg \
    --show-details \
    --query "[*].{Name:name, Provisioned:provisioningState, Power:powerState}" \
    --output table
```

### Résultat attendu

```
Name        Provisioned    Power
----------  -------------  ----------
AppServer   Succeeded      VM running
DataServer  Succeeded      VM running
```
<img width="891" height="371" alt="B0" src="https://github.com/user-attachments/assets/ef1049a9-4773-4154-a095-d14bac0920a7" />
<img width="960" height="230" alt="B1" src="https://github.com/user-attachments/assets/4b4b67ce-363c-467b-a056-45b4914cd802" />

<img width="911" height="414" alt="B2" src="https://github.com/user-attachments/assets/f619484e-5bca-4202-b33f-b395a0db814d" />

---

## 🔌 Étape 3 — Test de la connectivité par défaut

### Objectif
Démontrer que les règles par défaut bloquent toutes les connexions SSH entrantes depuis Internet.

### Commandes

```bash
# Récupérer les adresses IP publiques
az vm list \
    --resource-group $rg \
    --show-details \
    --query "[*].{Name:name, PrivateIP:privateIps, PublicIP:publicIps}" \
    --output table

# Stocker les IPs dans des variables
APPSERVERIP="$(az vm list-ip-addresses \
                 --resource-group $rg \
                 --name AppServer \
                 --query "[].virtualMachine.network.publicIpAddresses[*].ipAddress" \
                 --output tsv)"

DATASERVERIP="$(az vm list-ip-addresses \
                 --resource-group $rg \
                 --name DataServer \
                 --query "[].virtualMachine.network.publicIpAddresses[*].ipAddress" \
                 --output tsv)"

# Tester la connexion SSH vers AppServer → doit échouer
ssh polo@$APPSERVERIP -o ConnectTimeout=5

# Tester la connexion SSH vers DataServer → doit échouer
ssh serge@$DATASERVERIP -o ConnectTimeout=5
```

### Résultat attendu

```
ssh: connect to host X.X.X.X port 22: Connection timed out
```

> ✅ **Comportement normal :** La règle `DenyAllInBound (65500)` bloque tout trafic entrant depuis Internet, y compris SSH. C'est la posture de sécurité par défaut d'Azure.

<img width="960" height="239" alt="C0" src="https://github.com/user-attachments/assets/babe7d1b-15df-4240-9f2c-b189c1705a6f" />

---

## 🔑 Étape 4 — Règle SSH entrante

### Objectif
Créer une règle NSG autorisant les connexions SSH (port 22) depuis Internet, pour pouvoir administrer les VMs.

### Commandes

```bash
# Créer la règle AllowSSH (priorité 100)
az network nsg rule create \
    --resource-group $rg \
    --nsg-name ERP-SERVERS-NSG \
    --name AllowSSHRule \
    --direction Inbound \
    --priority 100 \
    --source-address-prefixes '*' \
    --source-port-ranges '*' \
    --destination-address-prefixes '*' \
    --destination-port-ranges 22 \
    --access Allow \
    --protocol Tcp \
    --description "Allow inbound SSH"

# Tester SSH vers AppServer (attendre 1-2 min si nécessaire)
ssh polo@$APPSERVERIP -o ConnectTimeout=5
# → Taper "yes" puis le mot de passe → puis "exit"

# Tester SSH vers DataServer
ssh serge@$DATASERVERIP -o ConnectTimeout=5
# → Taper "yes" puis le mot de passe → puis "exit"
```

### Résultat attendu

```
The authenticity of host 'X.X.X.X' can't be established.
Are you sure you want to continue connecting (yes/no)? yes
serge@AppServer:~$
```
<img width="916" height="407" alt="C1" src="https://github.com/user-attachments/assets/eaa4f382-800c-4ac3-a4da-4d2227b83b82" />
<img width="930" height="118" alt="C2" src="https://github.com/user-attachments/assets/f7d0170a-8f8e-477f-a8e4-c1cf7a5d0c40" />
<img width="953" height="394" alt="C3" src="https://github.com/user-attachments/assets/b7207265-77ee-4a68-899b-fa756d16db60" />
<img width="960" height="114" alt="C4" src="https://github.com/user-attachments/assets/df22edfe-62c4-4347-821c-6cd70d5d0978" />

---

## 🚫 Étape 5 — Règle HTTP : isoler le DataServer

### Objectif
Créer une règle NSG qui **bloque** les connexions HTTP (port 80) **depuis DataServer vers AppServer**, tout en laissant AppServer accéder à DataServer via HTTP.

### Logique de la règle

```
DataServer (10.0.1.4) → HTTP port 80 → AppServer (10.0.0.4)  ❌ BLOQUÉ
AppServer  (10.0.0.4) → HTTP port 80 → DataServer (10.0.1.4) ✅ AUTORISÉ
```

> 💡 La règle par défaut `AllowVnetInBound (65000)` autorise tout trafic intra-VNet. Notre règle `httpRule (150)` est plus prioritaire et vient **override** ce comportement pour ce flux spécifique.

### Commandes

```bash
# Créer la règle de blocage HTTP de DataServer vers AppServer
az network nsg rule create \
    --resource-group $rg \
    --nsg-name ERP-SERVERS-NSG \
    --name httpRule \
    --direction Inbound \
    --priority 150 \
    --source-address-prefixes 10.0.1.4 \
    --source-port-ranges '*' \
    --destination-address-prefixes 10.0.0.4 \
    --destination-port-ranges 80 \
    --access Deny \
    --protocol Tcp \
    --description "Deny from DataServer to AppServer on port 80"

# TEST 1 : AppServer → DataServer via HTTP (doit fonctionner → HTTP 200 OK)
ssh -t polo@$APPSERVERIP 'wget http://10.0.1.4; exit; bash'

# TEST 2 : DataServer → AppServer via HTTP (doit échouer → Connection timed out)
ssh -t serge@$DATASERVERIP 'wget http://10.0.0.4; exit; bash'
# Ctrl+C pour interrompre après quelques secondes
```

### Résultats attendus

```bash
# TEST 1 (AppServer → DataServer) : SUCCÈS
HTTP request sent, awaiting response... 200 OK

# TEST 2 (DataServer → AppServer) : ÉCHEC
Connecting to 10.0.0.4:80... Connection timed out.
```
<img width="926" height="416" alt="D" src="https://github.com/user-attachments/assets/73526580-d504-434d-81a9-50bde204b6c1" />
<img width="960" height="241" alt="D0" src="https://github.com/user-attachments/assets/995e096d-b58f-42d6-9b44-4f90a6c48646" />
<img width="960" height="119" alt="D1" src="https://github.com/user-attachments/assets/1f742d45-fe65-4369-86fe-d75dcfde9e98" />


## 👥 Étape 6 — Application Security Group (ASG)

### Objectif
Remplacer l'adresse IP source `10.0.1.4` dans la règle NSG par un **ASG** nommé `ERP-DB-SERVERS-ASG`. Ainsi, tous les futurs serveurs de base de données ajoutés à cet ASG hériteront automatiquement de la règle de blocage sans modifier le NSG.

### Pourquoi c'est mieux que l'IP ?

| Approche | Maintien si 10 serveurs DB ? |
|---|---|
| Règle par adresse IP | 10 règles NSG à créer et maintenir |
| Règle par ASG | 1 seule règle, ajouter les VMs à l'ASG |

### Commandes

```bash
# Créer l'ASG pour les serveurs de base de données
az network asg create \
    --resource-group $rg \
    --name ERP-DB-SERVERS-ASG

# Associer DataServer à cet ASG
az network nic ip-config update \
    --resource-group $rg \
    --application-security-groups ERP-DB-SERVERS-ASG \
    --name ipconfigDataServer \
    --nic-name DataServerVMNic \
    --vnet-name ERP-servers \
    --subnet Databases

# Mettre à jour la règle httpRule pour utiliser l'ASG au lieu de l'IP
az network nsg rule update \
    --resource-group $rg \
    --nsg-name ERP-SERVERS-NSG \
    --name httpRule \
    --direction Inbound \
    --priority 150 \
    --source-address-prefixes "" \
    --source-port-ranges '*' \
    --source-asgs ERP-DB-SERVERS-ASG \
    --destination-address-prefixes 10.0.0.4 \
    --destination-port-ranges 80 \
    --access Deny \
    --protocol Tcp \
    --description "Deny from DataServer to AppServer on port 80 using ASG"

# Retester (attendre 1-2 min pour propagation)
# TEST 1 : AppServer → DataServer (doit encore fonctionner)
ssh -t polo@$APPSERVERIP 'wget http://10.0.1.4; exit; bash'

# TEST 2 : DataServer → AppServer (doit encore être bloqué)
ssh -t serge@$DATASERVERIP 'wget http://10.0.0.4; exit; bash'
```
<img width="929" height="431" alt="E0" src="https://github.com/user-attachments/assets/3438176f-77ad-4fbc-b7f6-2bea19ce5163" />

<img width="926" height="406" alt="E3" src="https://github.com/user-attachments/assets/ea77eb14-da3a-4814-a753-441e352bd071" />
<img width="958" height="320" alt="E2" src="https://github.com/user-attachments/assets/29756b1e-3792-4dbb-b81e-68be7982ba7f" />

---

## 📦 Étape 7 — Service Endpoints pour Azure Storage

### Objectif
Créer un compte de stockage Azure et le restreindre **exclusivement** au sous-réseau `Databases` via un Service Endpoint. AppServer ne pourra pas y accéder. DataServer oui.

### Schéma de flux

```
DataServer (subnet Databases) ──Service Endpoint──→ Azure Storage ✅
AppServer  (subnet Applications) ────────────────→ Azure Storage ❌ (accès refusé)
```

### Commandes

```bash
# ── PARTIE A : Règles NSG sortantes ──────────────────────────────────

# Autoriser le trafic sortant vers Azure Storage
az network nsg rule create \
    --resource-group $rg \
    --nsg-name ERP-SERVERS-NSG \
    --name Allow_Storage \
    --priority 190 \
    --direction Outbound \
    --source-address-prefixes "VirtualNetwork" \
    --source-port-ranges '*' \
    --destination-address-prefixes "Storage" \
    --destination-port-ranges '*' \
    --access Allow \
    --protocol '*' \
    --description "Allow access to Azure Storage"

# Bloquer tout autre trafic Internet sortant
az network nsg rule create \
    --resource-group $rg \
    --nsg-name ERP-SERVERS-NSG \
    --name Deny_Internet \
    --priority 200 \
    --direction Outbound \
    --source-address-prefixes "VirtualNetwork" \
    --source-port-ranges '*' \
    --destination-address-prefixes "Internet" \
    --destination-port-ranges '*' \
    --access Deny \
    --protocol '*' \
    --description "Deny access to Internet"

# ── PARTIE B : Création du compte de stockage ────────────────────────

# Créer le compte de stockage 
STORAGEACCT=$(az storage account create \
                --resource-group $rg \
                --name engineeringdocs$RANDOM \
                --sku Standard_LRS \
                --query "name" | tr -d '"')

# Stocker la clé primaire
STORAGEKEY=$(az storage account keys list \
                --resource-group $rg \
                --account-name $STORAGEACCT \
                --query "[0].value" | tr -d '"')

# Créer un partage de fichiers Azure
az storage share create \
    --account-name $STORAGEACCT \
    --account-key $STORAGEKEY \
    --name "erp-data-share"

# ── PARTIE C : Activation du Service Endpoint ────────────────────────

# Activer Microsoft.Storage sur le sous-réseau Databases uniquement
az network vnet subnet update \
    --vnet-name ERP-servers \
    --resource-group $rg \
    --name Databases \
    --service-endpoints Microsoft.Storage

# Configurer le compte de stockage pour refuser tout accès par défaut
az storage account update \
    --resource-group $rg \
    --name $STORAGEACCT \
    --default-action Deny

# Autoriser uniquement le sous-réseau Databases
az storage account network-rule add \
    --resource-group $rg \
    --account-name $STORAGEACCT \
    --vnet-name ERP-servers \
    --subnet Databases
```
<img width="920" height="407" alt="F0" src="https://github.com/user-attachments/assets/c2d120ff-d89a-4c42-9fda-db298e903c9c" />
<img width="917" height="307" alt="F1" src="https://github.com/user-attachments/assets/75cb19c3-ad44-40bc-b84d-cc370f9c58c8" />
<img width="936" height="386" alt="F2" src="https://github.com/user-attachments/assets/6e4103f1-86ea-419c-a511-30af99742d37" />
<img width="917" height="382" alt="F3" src="https://github.com/user-attachments/assets/a8f899ba-c1a5-49bd-b89c-ec087b58cd32" />


---

## ✅ Étape 8 — Test final et validation

### Objectif
Valider que toute l'architecture fonctionne conformément aux règles définies.

### Tests de validation

```bash
# ── TEST 1 : AppServer tente de monter le partage Azure Storage → DOIT ÉCHOUER

ssh -t polo@$APPSERVERIP \
    "mkdir azureshare; \
    sudo mount -t cifs //$STORAGEACCT.file.core.windows.net/erp-data-share azureshare \
    -o vers=3.0,username=$STORAGEACCT,password=$STORAGEKEY,dir_mode=0777,file_mode=0777,sec=ntlmssp; \
    findmnt -t cifs; exit; bash"

# Résultat attendu : "mount error: No such file or directory"
# (accès refusé car AppServer est sur le sous-réseau Applications, sans Service Endpoint)


# ── TEST 2 : DataServer tente de monter le partage Azure Storage → DOIT RÉUSSIR

ssh -t serge@$DATASERVERIP \
    "mkdir azureshare; \
    sudo mount -t cifs //$STORAGEACCT.file.core.windows.net/erp-data-share azureshare \
    -o vers=3.0,username=$STORAGEACCT,password=$STORAGEKEY,dir_mode=0777,file_mode=0777,sec=ntlmssp; \
    findmnt -t cifs; exit; bash"

# Résultat attendu : Le montage réussit et findmnt affiche le point de montage CIFS
```

### Résultats attendus

```
# AppServer (ÉCHEC attendu) :
mount error(13): Permission denied
Refer to the mount.cifs(8) manual page (e.g. man mount.cifs) and kernel log messages (dmesg)

# DataServer (SUCCÈS attendu) :
TARGET      SOURCE                                          FSTYPE  OPTIONS
/home/azureuser/azureshare  //engineeringdocsXXXX.file.core.windows.net/erp-data-share  cifs    ...
```
<img width="959" height="258" alt="G0" src="https://github.com/user-attachments/assets/7c77a012-8955-4fb9-96aa-c09f544ac168" />

<img width="960" height="180" alt="G1" src="https://github.com/user-attachments/assets/15cf9c0a-318c-4612-97f7-06d1a0a4224a" />

---

## 📊 Récapitulatif des règles NSG

### Règles entrantes — ERP-SERVERS-NSG

| Priorité | Nom | Source | Destination | Port | Action | But |
|---|---|---|---|---|---|---|
| 100 | AllowSSHRule | * | * | 22 TCP | ✅ Autoriser | Administration SSH |
| 150 | httpRule | ASG: ERP-DB-SERVERS-ASG | 10.0.0.4 | 80 TCP | ❌ Refuser | Bloquer DB→App HTTP |
| 65000 | AllowVnetInBound | VirtualNetwork | VirtualNetwork | * | ✅ Autoriser | Défaut VNet |
| 65001 | AllowAzureLoadBalancerInBound | AzureLoadBalancer | * | * | ✅ Autoriser | Load Balancer |
| 65500 | DenyAllInBound | * | * | * | ❌ Refuser | Tout bloquer |

### Règles sortantes — ERP-SERVERS-NSG

| Priorité | Nom | Source | Destination | Port | Action | But |
|---|---|---|---|---|---|---|
| 190 | Allow_Storage | VirtualNetwork | Storage (tag) | * | ✅ Autoriser | Accès Azure Storage |
| 200 | Deny_Internet | VirtualNetwork | Internet (tag) | * | ❌ Refuser | Bloquer Internet sortant |
| 65000 | AllowVnetOutBound | VirtualNetwork | VirtualNetwork | * | ✅ Autoriser | Défaut VNet |
| 65001 | AllowInternetOutBound | * | Internet | * | ✅ Autoriser | Défaut Internet |
| 65500 | DenyAllOutBound | * | * | * | ❌ Refuser | Tout bloquer |

---
> ⚠️ **Important :** Penser à supprimer les ressources après le lab pour éviter des frais inutiles.

---

## 🎓 Bonnes pratiques et compétences démontrées

### Compétences AZ-104 démontrées

| Domaine AZ-104 | Compétence |
|---|---|
| Implement and manage virtual networking | ✅ Création VNet, sous-réseaux, NSG |
| Secure access to virtual networks | ✅ Règles NSG entrantes/sortantes |
| Configure network security | ✅ ASG, Service Tags, Service Endpoints |
| Monitor virtual networking | ✅ Tests de connectivité et validation |

### Compétences RHCSA démontrées (Linux)

| Commande | Contexte utilisé |
|---|---|
| `ssh` | Connexion aux VMs Ubuntu depuis Cloud Shell |
| `wget` | Test de connectivité HTTP entre VMs |
| `mount -t cifs` | Montage de partage Azure Files sur Linux |
| `findmnt` | Vérification des points de montage |

### Bonnes pratiques appliquées

- ✅ **Principe du moindre privilège** : ouverture uniquement des ports nécessaires
- ✅ **Défense en profondeur** : NSG sur le sous-réseau + validation des flux dans les deux sens
- ✅ **Scalabilité** : utilisation des ASG au lieu des IPs statiques
- ✅ **Isolation réseau PaaS** : Service Endpoints pour ne jamais exposer le stockage sur Internet
- ✅ **Service Tags** : utilisation de `Storage` et `Internet` au lieu d'IPs manuelles
- ✅ **Ségrégation des rôles** : sous-réseaux distincts par type de serveur

---


## 🔗 Ressources

- [Documentation NSG Azure](https://learn.microsoft.com/azure/virtual-network/network-security-groups-overview)
- [Application Security Groups](https://learn.microsoft.com/azure/virtual-network/application-security-groups)
- [Service Endpoints VNet](https://learn.microsoft.com/azure/virtual-network/virtual-network-service-endpoints-overview)
- [Module Microsoft Learn original](https://learn.microsoft.com/training/modules/secure-and-isolate-with-nsg-and-service-endpoints/)

---

*Réalisé par **Serge TOGNON** — Administrateur Cloud Azure Certifié (AZ-104) | Candidat RHCSA*
*LinkedIn : [Serge TOGNON]( https://www.linkedin.com/in/serge-tognon-a63443187/?lipi=urn%3Ali%3Apage%3Ad_flagship3_profile_view_base_contact_details%3BiHxYnwbJRjOOflTaGqZasw%3D%3D )*

