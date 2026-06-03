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
(ISO 27001, RGPD) — toute VM critique doit être restaurable en
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
  --location westus \
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

📸 **Screenshot 1** : Resource group `vmbackups` visible dans
le portail Azure avec la région West US

---

### 2. Déployer les machines virtuelles

```bash
# VM Windows — déploiement asynchrone avec --no-wait
az vm create \
  --resource-group $RGROUP \
  --name FS-APP01 \
  --size Standard_DS1_v2 \
  --public-ip-sku Standard \
  --vnet-name FinSecureInternal \
  --subnet FinSecureInternal1 \
  --image Win2016Datacenter \
  --admin-username finsecureadmin \
  --no-wait \
  --admin-password <VotreMotDePasse123!>
```

**Explication** : `--no-wait` lance le déploiement en arrière-plan
sans bloquer le terminal. Permet d'enchaîner immédiatement la
création de la VM Linux.

```bash
# VM Linux RHEL 9 — authentification par clé SSH
az vm create \
  --resource-group $RGROUP \
  --name FS-RHEL01 \
  --size Standard_DS1_v2 \
  --image RedHat:RHEL:9-gen2:latest \
  --authentication-type ssh \
  --generate-ssh-keys \
  --vnet-name FinSecureInternal \
  --subnet FinSecureInternal1
```

**Explication** : `--generate-ssh-keys` crée automatiquement une
paire de clés RSA si elle n'existe pas encore dans `~/.ssh/`.
`--authentication-type ssh` désactive l'authentification par mot
de passe — bonne pratique de sécurité conforme à la politique
FinSecure SA.

📸 **Screenshot 2** : Les deux VMs `FS-APP01` et `FS-RHEL01`
visibles dans le portail, statut "Running"

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

📸 **Screenshot 3** : Volet de sauvegarde FS-RHEL01 avec statut
"Protection activée"

📸 **Screenshot 4** : Confirmation "Sauvegarder maintenant"
déclenchée

---

### 4. Activer la sauvegarde via CLI (FS-APP01)

```bash
# Créer le coffre Recovery Services FinSecure
az backup vault create \
  --resource-group vmbackups \
  --location westus \
  --name finsecure-backup
```

**Explication** : Le coffre Recovery Services est l'entité centrale
qui stocke toutes les données de sauvegarde. Azure gère les comptes
de stockage sous-jacents de façon transparente — FinSecure SA n'a
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
en plus des quotidiennes, et la sauvegarde sélective de disques —
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

📸 **Screenshot 5** : Output de `az backup job list` avec
`ConfigureBackup` = Completed et `Backup` = InProgress

---

### 5. Restaurer une VM (FS-APP01)

#### 5a. Créer le compte de stockage de staging

Portail → **Comptes de stockage** → **Créer**

| Paramètre | Valeur |
|---|---|
| Resource group | vmbackups |
| Nom | `finsecurestagingYYYYMMDD` |
| Région | West US |

📸 **Screenshot 6** : Compte de stockage de staging créé

#### 5b. Arrêter la VM avant restauration

```bash
az vm stop \
  --resource-group vmbackups \
  --name FS-APP01
```

> ⚠️ La VM doit être arrêtée avant toute restauration.
> Une tentative sur VM active retourne une erreur bloquante.

📸 **Screenshot 7** : VM FS-APP01 statut "Stopped (deallocated)"

#### 5c. Lancer la restauration depuis le portail

1. VM `FS-APP01` → **Opérations** → **Sauvegarde**
2. **Restaurer la VM** → **Sélectionner un point de restauration**
3. Choisir le point de récupération le plus récent
4. Configuration :
   - Type : **Remplacer l'existant**
   - Lieu de préparation : compte de stockage créé en 5a
5. Cliquer **Restaurer**

📸 **Screenshot 8** : Volet de restauration avec point de
récupération sélectionné

📸 **Screenshot 9** : Tâche de restauration en cours dans
"Tâches de sauvegarde"

📸 **Screenshot 10** : Statut final "Completed" dans le coffre
`finsecure-backup`

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
