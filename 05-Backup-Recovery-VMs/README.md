# 🛡️ Azure VM Backup & Recovery Lab 

> **Auteur :** Serge TOGNON | Administrateur Cloud Azure Certifié (AZ-104) | Candidat RHCSA
> [![LinkedIn](https://img.shields.io/badge/LinkedIn-Serge_TOGNON-0077B5?logo=linkedin)](https://www.linkedin.com/in/serge-tognon-a63443187/?lipi=urn%3Ali%3Apage%3Ad_flagship3_profile_view_base_contact_details%3BiHxYnwbJRjOOflTaGqZasw%3D%3D)
> [![Azure](https://img.shields.io/badge/Azure-AZ--104_Certified-0078D4?logo=microsoftazure)](https://learn.microsoft.com/certifications/azure-administrator/)

---

## 📋 Table des matières

- [🎯 Objectif du projet](#-objectif-du-projet)
- [🏗️ Architecture](#️-architecture)
- [✅ Prérequis](#-prérequis)
- [🚀 Étapes](#-étapes)
  - [1. Créer l'environnement](#1-créer-lenvironnement)
  - [2. Déployer les machines virtuelles](#2-déployer-les-machines-virtuelles)
  - [3. Activer la sauvegarde via le portail](#3-activer-la-sauvegarde-via-le-portail-fs-rhel01)
  - [4. Activer la sauvegarde via CLI](#4-activer-la-sauvegarde-via-cli-fs-app01)
  - [5. Restaurer une VM](#5-restaurer-une-vm-fs-app01)
- [📚 Concepts clés](#-concepts-clés)
- [🎓 Compétences démontrées](#-compétences-démontrées)
- [🧹 Nettoyage](#-nettoyage)

---

## 🎯 Objectif du projet

Démontrer la mise en place d'une stratégie de sauvegarde et de
restauration pour des machines virtuelles Azure (Windows et Linux)
en utilisant Azure Backup et un coffre Recovery Services.

Contexte métier : FinSecure SA doit satisfaire aux exigences de
continuité d'activité définies par sa politique de sécurité interne
(ISO 27001, RGPD) ,toute VM critique doit être restaurable en
moins de 24h.

Ce lab couvre :
- La création d'un environnement multi-VM (Windows + Linux RHEL 9)
- La configuration d'Azure Backup via le portail et la CLI
- Le déclenchement manuel d'une sauvegarde
- La restauration complète d'une VM depuis un point de récupération
- Le monitoring des jobs de sauvegarde

---

## 🏗️ Architecture

```
Resource Group : vmbackups (westus)
│
├── VNet : FinSecureInternal (10.0.0.0/16)
│   └── Subnet : FinSecureInternal1 (10.0.0.0/24)
│
├── VM Windows : FS-APP01 (Win2016Datacenter)
├── VM Linux   : FS-RHEL01 (RHEL 9)
│
├── Recovery Services Vault : finsecure-backup
│   ├── Backup Item : FS-APP01 (EnhancedPolicy)
│   └── Backup Item : FS-RHEL01 (DailyPolicy)
│
└── Storage Account : finsecurestagingYYYYMMDD
    └── (staging pour restauration de disque)
```

---

## ✅ Prérequis

- Abonnement Azure actif
- Azure CLI installé ou Azure Cloud Shell
- Droits Contributor sur le resource group

---

## 🚀 Étapes

### 1. Créer l'environnement

```bash
# Créer le resource group et capturer son nom dans une variable
RGROUP=$(az group create \
  --name vmbackups \
  --location canadaeast \
  --output tsv \
  --query name)
```

**Explication** : `--output tsv` retourne du texte brut au lieu
de JSON. `--query name` filtre uniquement le champ `name` du
résultat. La substitution `$(...)` capture la valeur dans `$RGROUP`
pour la réutiliser dans toutes les commandes suivantes.

```bash
# Créer le réseau virtuel et le sous-réseau FinSecure
az network vnet create \
  --resource-group $RGROUP \
  --name FinSecureInternal \
  --address-prefixes 10.0.0.0/16 \
  --subnet-name FinSecureInternal1 \
  --subnet-prefixes 10.0.0.0/24
```
<img width="560" height="85" alt="A1" src="https://github.com/user-attachments/assets/c64c9855-c78d-4469-a591-6a7272e29216" />
<img width="923" height="271" alt="A0" src="https://github.com/user-attachments/assets/5c9df88f-3425-4db0-9775-bd3863b79d8b" />
<img width="960" height="251" alt="A2" src="https://github.com/user-attachments/assets/bc4325e3-e434-4605-be6c-b60958a635b8" />




---

### 2. Déployer les machines virtuelles

```bash
# VM Windows — déploiement asynchrone avec --no-wait
az vm create \
  --resource-group $RGROUP \
  --name FS-APP01 \
  --size Standard_D2s_v5 \
  --public-ip-sku Standard \
  --vnet-name FinSecureInternal \
  --subnet FinSecureInternal1 \
  --image Win2016Datacenter \
  --admin-username Serge \
  --no-wait \
  --admin-password Serge97772885
```

**Explication** : `--no-wait` lance le déploiement en arrière-plan
sans bloquer le terminal. Permet d'enchaîner immédiatement la
création de la VM Linux.

```bash
# VM Linux RHEL 9 — authentification par clé SSH
az vm create \
  --resource-group $RGROUP \
  --name FS-RHEL01 \
  --size Standard_D2S_v5 \
  --image RedHat:RHEL:9-gen2:latest \
  --authentication-type ssh \
  --generate-ssh-keys \
  --vnet-name FinSecureInternal \
  --subnet FinSecureInternal1
```

**Explication** : `--generate-ssh-keys` crée automatiquement une
paire de clés RSA si elle n'existe pas encore dans `~/.ssh/`.
`--authentication-type ssh` désactive l'authentification par mot
de passe , bonne pratique de sécurité conforme à la politique
FinSecure SA.

<img width="960" height="100" alt="B1" src="https://github.com/user-attachments/assets/dc54b358-a9ae-48f7-8dad-cd016290e644" />
<img width="917" height="305" alt="B2" src="https://github.com/user-attachments/assets/bd27e920-fd8b-4ab2-b535-532eddde4b76" />


---

### 3. Activer la sauvegarde via le portail (FS-RHEL01)

1. Portail Azure → **Machines virtuelles** → `FS-RHEL01`
2. Onglet **Fonctionnalités** → **Sauvegarde**
3. Sélectionner **Standard**
4. Coffre : accepter le nom généré automatiquement
5. Politique : `DailyPolicy` (sauvegarde quotidienne 12h UTC,
   rétention 180 jours)
6. Cliquer **Activer la sauvegarde**
7. Une fois déployé → **Sauvegarder maintenant**

<img width="917" height="400" alt="C0" src="https://github.com/user-attachments/assets/39ece5b1-c45a-4a92-b3c2-a7fc6c96abbd" />
<img width="917" height="401" alt="C1" src="https://github.com/user-attachments/assets/04f46c68-9b77-43ad-9dfc-b76a61e5b58d" />
<img width="915" height="400" alt="C2" src="https://github.com/user-attachments/assets/554c66bf-4b87-408c-be0e-7fe88e94df6e" />


---

### 4. Activer la sauvegarde via CLI (FS-APP01)

```bash
# Créer le coffre Recovery Services FinSecure
az backup vault create \
  --resource-group vmbackups \
  --location canadaeast \
  --name finsecure-backup
```

**Explication** : Le coffre Recovery Services est l'entité centrale
qui stocke toutes les données de sauvegarde. Azure gère les comptes
de stockage sous-jacents de façon transparente. FinSecure SA n'a
pas à gérer l'infrastructure de stockage backup manuellement.

```bash
# Associer FS-APP01 au coffre avec la politique EnhancedPolicy
az backup protection enable-for-vm \
  --resource-group vmbackups \
  --vault-name finsecure-backup \
  --vm FS-APP01 \
  --policy-name EnhancedPolicy
```

**Explication** : `EnhancedPolicy` permet des sauvegardes horaires
en plus des quotidiennes, et la sauvegarde sélective de disques,
adapté aux environnements financiers critiques de FinSecure SA.

```bash
# Surveiller la progression de la configuration
az backup job list \
  --resource-group vmbackups \
  --vault-name finsecure-backup \
  --output table
```

Attendre que `ConfigureBackup` passe au statut `Completed`.

```bash
# Déclencher une sauvegarde immédiate
az backup protection backup-now \
  --resource-group vmbackups \
  --vault-name finsecure-backup \
  --container-name FS-APP01 \
  --item-name FS-APP01 \
  --retain-until 18-10-2030 \
  --backup-management-type AzureIaasVM
```

**Explication** : `--retain-until` définit la date d'expiration
du point de récupération. Conforme à la politique de rétention
10 ans exigée par la réglementation financière applicable à
FinSecure SA.
<img width="960" height="176" alt="D0" src="https://github.com/user-attachments/assets/0a59e5cd-c948-44e7-86ad-656474a2a264" />


---

### 5. Restaurer une VM (FS-APP01)

#### 5a. Créer le compte de stockage de staging

Portail → **Comptes de stockage** → **Créer**

| Paramètre | Valeur |
|---|---|
| Resource group | vmbackups |
| Nom | `finsecurestaging260604` |
| Région | canada east |

<img width="917" height="264" alt="D1" src="https://github.com/user-attachments/assets/16ceaef0-fe5b-442a-926e-ed65191ee534" />


#### 5b. Arrêter la VM avant restauration

```bash
az vm stop \
  --resource-group vmbackups \
  --name FS-APP01
```

> ⚠️ La VM doit être arrêtée avant toute restauration.
> Une tentative sur VM active retourne une erreur bloquante.

<img width="960" height="102" alt="E1" src="https://github.com/user-attachments/assets/94bf98ca-aeac-4aeb-874f-f7f87ca58ae4" />


#### 5c. Lancer la restauration depuis le portail

1. VM `FS-APP01` → **Opérations** → **Sauvegarde**
2. **Restaurer la VM** → **Sélectionner un point de restauration**
3. Choisir le point de récupération le plus récent
4. Configuration :
   - Type : **Remplacer l'existant**
   - Lieu de préparation : compte de stockage créé en 5a
5. Cliquer **Restaurer**

<img width="922" height="386" alt="F1" src="https://github.com/user-attachments/assets/329acf4a-6935-44e4-b51e-22c66c1fe5a1" />


<img width="914" height="397" alt="F2" src="https://github.com/user-attachments/assets/6f22082f-2c6d-4110-8749-84b75fbb2fe8" />

<img width="960" height="203" alt="F4" src="https://github.com/user-attachments/assets/c9088f11-350a-4559-bee8-b6f53aba02ef" />
<img width="918" height="283" alt="F3" src="https://github.com/user-attachments/assets/2f2a7207-93ca-4ba5-85ac-52bb5cf36586" />


---

## 📚 Concepts clés

| Concept | Explication |
|---|---|
| **Recovery Services Vault** | Entité centrale de stockage des backups. Azure gère les comptes de stockage sous-jacents automatiquement |
| **Snapshot tier** | Stockage local 0-5 jours. Restauration instantanée rapide |
| **Vault tier** | Stockage long terme dans le coffre. Rétention configurable sur des années |
| **EnhancedPolicy** | Politique avancée : sauvegardes horaires, sauvegarde sélective de disques |
| **Cohérence applicative** | VSS (Windows) ou scripts pre/post (Linux) pour capturer l'état complet en mémoire |
| **Staging storage** | Compte de stockage temporaire utilisé comme zone de transit lors d'une restauration de disque |

---

## 🎓 Compétences démontrées

- Azure CLI : création de ressources, variables bash, monitoring
- Azure Backup : configuration portail et CLI
- Recovery Services Vault : politique de sauvegarde, rétention
- Restauration VM : remplacement de disque existant
- Conformité : rétention alignée ISO 27001 / réglementation financière

---

## 🧹 Nettoyage

```bash
az group delete --name vmbackups --yes --no-wait
```

---

*Réalisé par **Serge TOGNON** — Administrateur Cloud Azure Certifié (AZ-104) | Candidat RHCSA*
*LinkedIn : [Serge TOGNON](https://www.linkedin.com/in/serge-tognon-a63443187/?lipi=urn%3Ali%3Apage%3Ad_flagship3_profile_view_base_contact_details%3BiHxYnwbJRjOOflTaGqZasw%3D%3D)*
```

---
