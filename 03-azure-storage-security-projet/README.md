# 🔐 SÉCURISATION DU COMPTE DE STOCKAGE AZURE-Projet AZ-104

> **Auteur :** Serge TOGNON | Admin Cloud Azure Certifié | [LinkedIn](https://www.linkedin.com/in/serge-tognon-a63443187/?lipi=urn%3Ali%3Apage%3Ad_flagship3_profile_view_base_contact_details%3BiHxYnwbJRjOOflTaGqZasw%3D%3D )

![Azure](https://img.shields.io/badge/Azure-Storage-0078D4?style=for-the-badge&logo=microsoftazure&logoColor=white)
![AZ-104](https://img.shields.io/badge/Certif-AZ--104-0078D4?style=for-the-badge&logo=microsoft&logoColor=white)
![RHCSA](https://img.shields.io/badge/Candidat-RHCSA-EE0000?style=for-the-badge&logo=redhat&logoColor=white)
![Status](https://img.shields.io/badge/Status-Completed-28a745?style=for-the-badge)
![Level](https://img.shields.io/badge/Niveau-Intermédiaire-orange?style=for-the-badge)
![Abonnement](https://img.shields.io/badge/Azure-Étudiant%20Compatible-blue?style=for-the-badge&logo=microsoftazure)

---

## 📋 Table des matières

1. [🎯 Vue d'ensemble du projet](#-vue-densemble-du-projet)
2. [🏗 Architecture](#-architecture)
3. [📦 Prérequis](#-prérequis)
4. [🔵 Étape 1 — Création du compte de stockage](#-étape-1--création-et-configuration-de-base-du-compte-de-stockage)
5. [🌐 Étape 2 — Réseau virtuel dédié et contrôle d'accès réseau](#-étape-2--réseau-virtuel-dédié-et-contrôle-daccès-réseau)
6. [🔑 Étape 3 — RBAC et identités managées](#-étape-3--rbac-et-identités-managées)
7. [🔏 Étape 4 — Signatures d'accès partagé (SAS)](#-étape-4--signatures-daccès-partagé-sas)
8. [🛡 Étape 5 — Microsoft Defender pour le stockage](#-étape-5--microsoft-defender-pour-le-stockage)
9. [📋 Étape 6 — Audit, diagnostic et rotation des clés](#-étape-6--audit-diagnostic-et-rotation-des-clés)
10. [📊 Synthèse des bonnes pratiques](#-synthèse-des-bonnes-pratiques)
11. [🚨 Dépannage](#-dépannage-troubleshooting)
12. [💰 Optimisation des coûts](#-optimisation-des-coûts-azure)
13. [🎓 Compétences démontrées](#-compétences-démontrées)
14. [📁 Structure du dépôt](#-structure-du-dépôt-github)
15. [🔗 Ressources Microsoft officielles](#-ressources-microsoft-officielles)

---

## 🎯 Vue d'ensemble du projet

### Objectif

Ce projet démontre la mise en place d'une stratégie de sécurisation complète d'un compte de stockage Azure, en appliquant les meilleures pratiques Microsoft pour un environnement de production réaliste. Il couvre le chiffrement, le réseau virtuel dédié, le contrôle d'accès réseau, le RBAC, les SAS, la détection des menaces et l'audit ,le tout via le **Portail Azure** comme outil principal.

### 🏢 Contexte entreprise fictive

> **Entreprise :** FinSecure SA  Société de services financiers basée à Cotonou (Bénin), opérant dans 5 pays d'Afrique de l'Ouest.
>
> **Problématique :** FinSecure SA stocke des relevés clients, des rapports réglementaires et des données contractuelles dans Azure Blob Storage. Suite à un audit de conformité, la DSI exige une mise en conformité totale avec les standards de sécurité cloud (ISO 27001, RGPD) avant la fin du trimestre.
>
> **Mission :** En tant qu'Administrateur Cloud Azure , sécuriser le compte de stockage Azure en mettant en place un réseau virtuel dédié, des contrôles d'accès stricts, un chiffrement bout-en-bout, une surveillance active et un audit complet.

### 💼 Cas d'usage réels

| Cas d'usage | Description |
|---|---|
| 📂 Archivage réglementaire | Documents financiers soumis à des durées de conservation légales |
| 🔄 Partage sécurisé partenaires | Accès temporaire via SAS pour auditeurs et régulateurs |
| 🔍 Audit de conformité | Journalisation complète de tous les accès (RGPD, ISO 27001) |
| 🚨 Détection des menaces | Alertes automatiques en cas d'accès anormaux ou fichiers suspects |
| 🔐 Isolation réseau | Trafic Storage confiné dans un VNet privé dédié |

### 🛠 Outils utilisés

| Outil | Rôle | Priorité |
|---|---|---|
| **Portail Azure** | Configuration principale — toutes les étapes | 🥇 Principal |
| Azure Storage Explorer | Test d'accès et validation des SAS | 🔧 Complémentaire |
| Azure CLI | Scripts de référence et automatisation | 📚 Ressource complémentaire |
| PowerShell Az Module | Scripts Windows / CI-CD | 📚 Ressource complémentaire |

### ☁️ Services Azure utilisés

| Service Azure | Rôle dans le projet |
|---|---|
| Azure Blob Storage | Stockage principal des données FinSecure SA |
| **Azure Virtual Network** | **Isolation réseau dédiée — créé dans ce lab** |
| Microsoft Entra ID | Gestion des identités et RBAC (utilisateur : Serge TOGNON) |
| Microsoft Defender for Cloud | Détection des menaces avancée |
| Azure Monitor / Log Analytics | Surveillance, audit et requêtes KQL |
| Azure Private Endpoint | Accès privé sans exposition Internet |


## 🏗 Architecture

### Diagramme Vue globale du projet

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      FINSECURE SA — Azure Architecture                      │
│              Sécurisation du Compte de Stockage Azure (Lab AZ-104)          │
└─────────────────────────────────────────────────────────────────────────────┘

  INTERNET PUBLIC                    AZURE TENANT — ABONNEMENT ÉTUDIANT
  ┌─────────────────┐               ┌─────────────────────────────────────────┐
  │                 │               │  ┌───────────────────────────────────┐  │
  │  👤 Serge       │──HTTPS + SAS──┼─▶│       Microsoft Entra ID          │  │
  │  TOGNON         │               │  │   (Authentification + RBAC)       │  │
  │  (Admin Azure)  │               │  └─────────────────┬─────────────────┘  │
  │                 │               │                    │ Auth Entra ID       │
  │  🤝 Partenaire  │──SAS Token────┼────────────────────┼─────────────┐      │
  │  Externe        │               │                    │             │      │
  │  (Auditeur)     │               │  ┌─────────────────▼─────────────▼──┐  │
  └─────────────────┘               │  │   Resource Group                  │  │
                                    │  │   rg-storage-security             │  │
  ACCÈS BLOQUÉ                      │  │                                   │  │
  ┌─────────────────┐  ── ✗ ───────▶│  │  ┌─────────────────────────────┐ │  │
  │  ❌ IP inconnue │  Pare-feu IP   │  │  │  Virtual Network             │ │  │
  │  Non autorisée  │  Storage       │  │  │  vnet-finsecure-storage      │ │  │
  └─────────────────┘                │  │  │  Espace : 10.10.0.0/16      │ │  │
                                    │  │  │                             │ │  │
                                    │  │  │  ┌──────────────────────┐  │ │  │
                                    │  │  │  │ snet-storage-private │  │ │  │
                                    │  │  │  │ 10.10.1.0/24         │  │ │  │
                                    │  │  │  │  Service Endpoint    │  │ │  │
                                    │  │  │  │  Microsoft.Storage   │  │ │  │
                                    │  │  │  │       │              │  │ │  │
                                    │  │  │  └───────┼──────────────┘  │ │  │
                                    │  │  │          │ IP Privée        │ │  │
                                    │  │  │  ┌───────▼──────────────┐  │ │  │
                                    │  │  │  │  Storage Account     │  │ │  │
                                    │  │  │  │  stfinsecure1234   │  │ │  │
                                    │  │  │  │                      │  │ │  │
                                    │  │  │  │  🔒 HTTPS Only       │  │ │  │
                                    │  │  │  │  🔒 TLS 1.2 Min      │  │ │  │
                                    │  │  │  │  🔒 AES-256          │  │ │  │
                                    │  │  │  │  🔒 No Public Blob   │  │ │  │
                                    │  │  │  │  🔒 Soft Delete 7j   │  │ │  │
                                    │  │  │  │                      │  │ │  │
                                    │  │  │  │  [documents-finances] │  │ │  │
                                    │  │  │  └──────────────────────┘  │ │  │
                                    │  │  └─────────────────────────────┘ │  │
                                    │  │                                   │  │
                                    │  │  ┌──────────────────────────────┐ │  │
                                    │  │  │  Log Analytics Workspace     │ │  │
                                    │  │  │  law-storage-security        │◀┘  │
                                    │  │  │  (Logs Read/Write/Delete)    │    │
                                    │  │  └──────────────────────────────┘    │
                                    │  │                                       │
                                    │  │  ┌──────────────────────────────┐    │
                                    │  │  │  Microsoft Defender for Cloud│    │
                                    │  │  │  Plan : Stockage — Activé    │◀───┘
                                    │  │  └──────────────────────────────┘
                                    │  └───────────────────────────────────┘
                                    └─────────────────────────────────────────┘

  LÉGENDE :
  ──HTTPS──   Connexion chiffrée TLS 1.2+    🔒  Contrôle de sécurité actif
  ──SAS────   Signature d'accès partagé       ✗   Accès refusé par le pare-feu
  ◀───────    Flux de journalisation/audit    MI  Managed Identity
```

### Couches de sécurité (Défense en profondeur)

```
┌──────────────────────────────────────────────────────────────────┐
│              MODÈLE DÉFENSE EN PROFONDEUR — FINSECURE SA         │
├─────────┬───────────────────┬────────────────────────────────────┤
│ COUCHE  │ DOMAINE           │ CONTRÔLE MIS EN PLACE              │
├─────────┼───────────────────┼────────────────────────────────────┤
│   1     │ 🌐 Réseau         │ VNet dédié + Subnet privé          │
│         │                   │ Pare-feu IP + Service Endpoint     │
├─────────┼───────────────────┼────────────────────────────────────┤
│   2     │ 🔑 Identité       │ Microsoft Entra ID + RBAC          │
│         │                   │ Rôles Storage Data (pas Owner)     │
├─────────┼───────────────────┼────────────────────────────────────┤
│   3     │ 💾 Données        │ Chiffrement AES-256 au repos       │
│         │                   │ TLS 1.2+ en transit                │
├─────────┼───────────────────┼────────────────────────────────────┤
│   4     │ 🔏 Accès délégué  │ SAS avec permissions minimales     │
│         │                   │ Expiration courte + HTTPS only     │
├─────────┼───────────────────┼────────────────────────────────────┤
│   5     │ 🛡 Détection      │ Microsoft Defender for Storage     │
│         │                   │ Alertes anomalies + Malware scan   │
├─────────┼───────────────────┼────────────────────────────────────┤
│   6     │ 📊 Audit          │ Log Analytics + KQL                │
│         │                   │ Read/Write/Delete journalisés      │
└─────────┴───────────────────┴────────────────────────────────────┘
```

### Ressources Azure créées dans ce lab

| Ressource | Nom | Type | Étape |
|---|---|---|---|
| Groupe de ressources | `rg-storage-security` | Resource Group | Étape 1 |
| Compte de stockage | `stfinsecure1234` | StorageV2 Standard LRS | Étape 1 |
| Réseau virtuel | `vnet-finsecure-storage` | Virtual Network 10.10.0.0/16 | **Étape 2** |
| Sous-réseau | `snet-storage-private` | Subnet 10.10.1.0/24 | **Étape 2** |
| Espace de travail | `law-storage-security` | Log Analytics Workspace | Étape 6 |

---

## 📦 Prérequis

### Comptes et accès

| Prérequis | Détails |
|---|---|
| Abonnement Azure | Azure for Students  Compatible avec toutes les étapes |
| Compte utilisateur | **Serge TOGNON**  Rôle :  Propriétaire |
| Microsoft Entra ID | Accès au tenant (inclus avec l'abonnement étudiant) |
| Azure Storage Explorer | Optionnel pour tester les SAS visuellement |


## 🔵 Étape 1 — Création et configuration de base du compte de stockage

### 🎯 Objectif

Créer un compte de stockage Azure avec les paramètres de sécurité fondamentaux activés dès la création. Dans un environnement d'entreprise comme FinSecure SA, une configuration initiale incorrecte crée des vulnérabilités difficiles à corriger ultérieurement sans interruption de service.

**Lien AZ-104 :** Domaine *"Mettre en œuvre et gérer le stockage"*  Configuration sécurisée des comptes de stockage.

### 🧠 Concepts clés

| Concept | Explication |
|---|---|
| **StorageV2** | Type de compte recommandé : supporte tous les services (Blob, File, Queue, Table) et toutes les fonctionnalités récentes |
| **Transfert sécurisé** | Force HTTPS pour toutes les opérations  les requêtes HTTP sont rejetées avec une erreur 403 |
| **Redondance LRS** | Locally Redundant Storage : 3 copies dans un seul datacenter suffisant et économique pour le lab |
| **TLS 1.2 minimum** | Protocole de chiffrement en transit  TLS 1.0 et 1.1 sont dépréciés et vulnérables |
| **Accès public aux blobs** | Désactivé par défaut : aucun blob ne peut être lu publiquement sans authentification |
| **Suppression réversible** | Soft Delete : les blobs supprimés sont récupérables pendant 7 jours  protection contre les suppressions accidentelles |

### ⚙️ Procédure détaillée — Portail Azure

#### 1.1 — Accès au service

1. Ouvrir [https://portal.azure.com](https://portal.azure.com) et se connecter avec le compte **Serge TOGNON**
2. Dans la barre de recherche en haut, taper **`Comptes de stockage`**
3. Sélectionner **"Comptes de stockage"** dans les résultats
4. Cliquer sur le bouton **`+ Créer`** (en haut à gauche)

#### 1.2 — Onglet "Informations de base"

Remplir le formulaire avec les valeurs suivantes :

| Champ | Valeur à saisir |
|---|---|
| **Abonnement** | Votre abonnement étudiant Azure |
| **Groupe de ressources** | Cliquer "Créer" → saisir `rg-storage-security` → OK |
| **Nom du compte de stockage** | `stfinsecure1234' |
| **Région** | `(Europe) West Europe` *(ou la région la plus proche)* |
| **Performances** | `Standard` |
| **Redondance** | `Stockage localement redondant (LRS)` |

> ⚠️ **Contrainte de nommage :** Le nom du compte doit être **unique globalement** dans Azure, contenir entre 3 et 24 caractères, uniquement des **minuscules et des chiffres** — aucun trait d'union, aucune majuscule.

---
<img width="960" height="417" alt="A1" src="https://github.com/user-attachments/assets/2e1b78ad-9227-4331-8ccd-d7c2c9fe6a82" />
<img width="920" height="404" alt="A2" src="https://github.com/user-attachments/assets/f3b6762f-63cf-41eb-b18c-e8f9cc02d633" />

---

#### 1.3 — Onglet "Avancé" (paramètres de sécurité)

Cliquer sur l'onglet **`Avancé`** et configurer :

| Paramètre | Valeur | Importance |
|---|---|---|
| **Transfert sécurisé requis** | ✅ `Activé` | Critique — force HTTPS |
| **Autoriser l'accès à la clé de compte** | ✅ `Activé` | Laisser pour le lab (sera restreint via RBAC) |
| **Version TLS minimale** | `TLS 1.2` | Critique - rejette TLS 1.0/1.1 |
| **Accès public aux blobs** | ❌ `Désactivé` | Critique — aucun blob public |
| **Infrastructure de chiffrement double** | ✅ `Activé` si disponible | Recommandé pour données sensibles |

> 💡 **Pourquoi cette configuration ?** L'activation du transfert sécurisé et de TLS 1.2 garantit que toute communication avec le stockage est chiffrée. La désactivation de l'accès public aux blobs empêche toute exposition accidentelle de données financières de FinSecure SA.

---
<img width="960" height="392" alt="B1" src="https://github.com/user-attachments/assets/8de8b324-e16b-45c1-8d2d-e7ee7b455afd" />
<img width="949" height="386" alt="B2" src="https://github.com/user-attachments/assets/3c1bfda9-fca1-4954-8a06-bd7c5cc3d199" />
<img width="920" height="374" alt="B3" src="https://github.com/user-attachments/assets/76a10c85-fcdd-4334-90b4-f9923641b6e7" />
<img width="914" height="357" alt="C1" src="https://github.com/user-attachments/assets/63396c5e-c295-46ff-801e-0da4574c9972" />


---

#### 1.4 — Onglet "Protection des données"

Cliquer sur l'onglet **`Protection des données`** :

| Option | Valeur | Rétention |
|---|---|---|
| **Activer la suppression réversible pour les blobs** | ✅ Activé | 7 jours |
| **Activer la suppression réversible pour les conteneurs** | ✅ Activé | 7 jours |

#### 1.5 — Validation et déploiement

1. Cliquer sur **`Vérifier + créer`**
2. Vérifier le résumé de configuration — s'assurer qu'aucune erreur n'est signalée
3. Cliquer sur **`Créer`**
4. Attendre la notification **"Votre déploiement est terminé"** (30 à 60 secondes)
5. Cliquer sur **`Accéder à la ressource`**

---
<img width="929" height="430" alt="D" src="https://github.com/user-attachments/assets/999f55d8-c17a-4501-9bc2-240ba1ff67f2" />

---

### 🔒 Risques de sécurité évités

> 🔴 **Risque :** Laisser l'accès public aux blobs activé  des documents financiers pourraient être indexés par des moteurs de recherche.
>
> 🔴 **Risque :** Utiliser TLS 1.0/1.1  vulnérabilités connues (POODLE, BEAST) permettent des attaques man-in-the-middle.
>
> 🟢 **Bonne pratique Microsoft :** En production, activer la CMK (Customer-Managed Key) via Azure Key Vault pour le contrôle total des clés de chiffrement.

### 💻 Commandes de référence — Azure CLI

```bash
# Variables d'environnement 
export RESOURCE_GROUP="rg-storage-security"
export LOCATION="westeurope"
export STORAGE_ACCOUNT="stfinsecure1234"  # 

# Créer le groupe de ressources
az group create \
  --name $RESOURCE_GROUP \
  --location $LOCATION \
  --tags "projet=storage-security" "auteur=SergeTognon" "environnement=lab"

# Créer le compte de stockage avec tous les paramètres de sécurité
az storage account create \
  --name $STORAGE_ACCOUNT \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION \
  --sku Standard_LRS \
  --kind StorageV2 \
  --https-only true \
  --min-tls-version TLS1_2 \
  --allow-blob-public-access false \
  --require-infrastructure-encryption true

# Activer le soft delete pour les blobs (7 jours)
az storage blob service-properties delete-policy update \
  --account-name $STORAGE_ACCOUNT \
  --enable true \
  --days-retained 7

echo "✅ Compte de stockage créé : $STORAGE_ACCOUNT"
```

### ✅ Validation

Après création, vérifier la configuration via le portail :

1. Dans le compte de stockage créé, naviguer vers **`Sécurité + réseau`** > **`Configuration`**
2. Vérifier que **"Transfert sécurisé requis"** = `Activé`
3. Vérifier que **"Version TLS minimale"** = `TLS 1.2`
4. Naviguer vers **`Paramètres`** > **`Configuration`** > confirmer accès public blobs = `Désactivé`

**Résultat attendu :** Aucune alerte de sécurité dans Microsoft Defender for Cloud concernant le compte de stockage pour les paramètres configurés.

---

## 🌐 Étape 2 — Réseau virtuel dédié et contrôle d'accès réseau

### 🎯 Objectif

Créer un réseau virtuel Azure dédié pour isoler le trafic vers le compte de stockage, puis configurer le pare-feu du compte pour n'accepter que les connexions provenant de ce réseau ou de votre adresse IP. Par défaut, un compte de stockage est accessible depuis n'importe quelle adresse IP sur Internet inacceptable pour FinSecure SA.

**Lien AZ-104 :** Domaine *"Configurer et gérer les réseaux virtuels"*  VNet, Service Endpoints, Private Endpoints.

### 🧠 Concepts clés

| Concept | Explication |
|---|---|
| **Azure Virtual Network (VNet)** | Réseau privé isolé dans Azure : les ressources à l'intérieur communiquent de manière sécurisée |
| **Subnet (sous-réseau)** | Subdivision du VNet : permet de segmenter les ressources par fonction ou niveau de sécurité |
| **Service Endpoint** | Achemine le trafic Azure vers le service Storage directement via le backbone Microsoft sans passer par Internet public |
| **Pare-feu Storage** | Filtre les connexions entrantes par adresse IP ou par VNet autorisé |
| **Action par défaut = Refuser** | Tout trafic non explicitement autorisé est bloqué  principe du moindre privilège réseau |
| **Exceptions services approuvés** | Azure Backup, Monitor et d'autres services Azure de confiance peuvent nécessiter une exception |

### ⚙️ Procédure détaillée — Portail Azure

#### 2.1 — Créer le réseau virtuel dédié

1. Dans la barre de recherche Azure, taper **`Réseaux virtuels`** et sélectionner le service
2. Cliquer sur **`+ Créer`**

**Onglet "Informations de base" :**

| Champ | Valeur |
|---|---|
| **Abonnement** | Votre abonnement étudiant |
| **Groupe de ressources** | `rg-storage-security` *(sélectionner le groupe existant)* |
| **Nom** | `vnet-finsecure-storage` |
| **Région** | `West Europe` *(même région que le compte de stockage)* |

3. Cliquer sur l'onglet **`Adresses IP`**

**Onglet "Adresses IP" :**

| Champ | Valeur |
|---|---|
| **Espace d'adressage IPv4** | `10.10.0.0/16` *(supprimer l'espace existant et saisir celui-ci)* |

4. Cliquer sur **`+ Ajouter un sous-réseau`** et remplir :

| Champ | Valeur |
|---|---|
| **Nom du sous-réseau** | `snet-storage-private` |
| **Plage d'adresses du sous-réseau** | `10.10.1.0/24` |

5. Cliquer sur **`Ajouter`** pour valider le sous-réseau
6. Cliquer sur **`Vérifier + créer`** → **`Créer`**
7. Attendre la notification de déploiement réussi

---
<img width="890" height="380" alt="D1" src="https://github.com/user-attachments/assets/919b73cf-cdcf-4067-8dc7-02993566e321" />
<img width="905" height="318" alt="D2" src="https://github.com/user-attachments/assets/24385b03-d669-40b6-9b61-6d252ab22d38" />

<img width="908" height="344" alt="D3" src="https://github.com/user-attachments/assets/4d47230f-bc7d-4bd0-b8d1-3c00f76a0914" />

---

#### 2.2 — Activer le Service Endpoint sur le sous-réseau

Le Service Endpoint permet au sous-réseau de communiquer directement avec le service Storage via le réseau Microsoft, sans passer par Internet public.

1. Dans le VNet créé (`vnet-finsecure-storage`), cliquer sur **`Sous-réseaux`** dans le menu gauche
2. Cliquer sur le sous-réseau **`snet-storage-private`**
3. Dans la section **"Points de terminaison de service"**, cliquer sur la liste déroulante **"Services"**
4. Cocher **`Microsoft.Storage`**
5. Cliquer sur **`Enregistrer`**

---
<img width="902" height="410" alt="E1" src="https://github.com/user-attachments/assets/9229a656-27ae-4ba8-9019-a5004a5cc3fb" />

---

#### 2.3 — Configurer le pare-feu du compte de stockage

Maintenant que le VNet est prêt, configurer le compte de stockage pour n'accepter que les connexions autorisées.

1. Naviguer vers votre **compte de stockage** (`stfinsecure1234`)
2. Dans le menu gauche, cliquer sur **`Sécurité + réseau`** > **`Réseau`**
3. Dans **"Accès réseau public"**, sélectionner **`Activé depuis les réseaux virtuels et adresses IP sélectionnés`**

**Section "Réseaux virtuels" :**

4. Cliquer sur **`+ Ajouter un réseau virtuel existant`**
5. Sélectionner votre **abonnement étudiant**
6. Sélectionner le VNet **`vnet-finsecure-storage`**
7. Cocher le sous-réseau **`snet-storage-private`**
8. Cliquer sur **`Ajouter`**

**Section "Pare-feu" (votre IP de gestion) :**

9. Cocher **`Ajouter l'adresse IP de votre client`** Azure détecte et ajoute automatiquement votre IP publique actuelle

> 💡 Ceci permet à **Serge TOGNON** de continuer à gérer le compte depuis son poste de travail pendant le lab.

**Section "Exceptions" :**

10. Cocher ✅ **`Autoriser les services Azure de la liste des services approuvés à accéder à ce compte de stockage`**
11. Cocher ✅ **`Autoriser l'accès depuis Azure Resource Manager pour ce compte de stockage`**

12. Cliquer sur **`Enregistrer`** en haut de la page

---
<img width="925" height="317" alt="F1" src="https://github.com/user-attachments/assets/bcadc6fc-0da9-4ad6-8d40-f42e2757a298" />
<img width="872" height="386" alt="F2" src="https://github.com/user-attachments/assets/4ef32b04-7aa1-4976-a6d4-14226640f6a0" />
<img width="894" height="410" alt="F3" src="https://github.com/user-attachments/assets/656431a7-3481-40d7-83d9-6af3e2406b1c" />

---

### 🔒 Risques de sécurité évités

> 🔴 **Risque :** Laisser le stockage en mode "Tous les réseaux" n'importe qui sur Internet peut tenter des attaques par force brute sur les clés SAS ou exploiter des tokens exposés accidentellement.
>
> 🔴 **Risque :** Oublier les "services approuvés" Azure Backup et Azure Monitor ne peuvent plus accéder au stockage, cassant la supervision et les sauvegardes.
>
> 🟢 **Bonne pratique Microsoft :** En production, n'utiliser que le **Private Endpoint** (sans aucune exposition publique) associé à une zone DNS privée pour que les applications accèdent au stockage via une IP privée uniquement.

### 💻 Commandes de référence — Azure CLI

```bash
# Créer le réseau virtuel
az network vnet create \
  --name vnet-finsecure-storage \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION \
  --address-prefix 10.10.0.0/16 \
  --subnet-name snet-storage-private \
  --subnet-prefix 10.10.1.0/24

# Activer le Service Endpoint Microsoft.Storage sur le subnet
az network vnet subnet update \
  --name snet-storage-private \
  --resource-group $RESOURCE_GROUP \
  --vnet-name vnet-finsecure-storage \
  --service-endpoints Microsoft.Storage

# Récupérer l'IP publique actuelle
MY_IP=$(curl -s https://api.ipify.org)

# Configurer le pare-feu : Deny par défaut, autoriser le VNet + votre IP
az storage account update \
  --name $STORAGE_ACCOUNT \
  --resource-group $RESOURCE_GROUP \
  --default-action Deny \
  --bypass AzureServices Metrics Logging

az storage account network-rule add \
  --account-name $STORAGE_ACCOUNT \
  --resource-group $RESOURCE_GROUP \
  --vnet-name vnet-finsecure-storage \
  --subnet snet-storage-private

az storage account network-rule add \
  --account-name $STORAGE_ACCOUNT \
  --resource-group $RESOURCE_GROUP \
  --ip-address $MY_IP

echo "✅ Réseau configuré  VNet : vnet-finsecure-storage | IP autorisée : $MY_IP"
```

### ✅ Validation

1. Dans le compte de stockage > **`Sécurité + réseau`** > **`Réseau`**
2. Confirmer que "Accès réseau public" = **"Activé depuis les réseaux virtuels et adresses IP sélectionnés"**
3. Confirmer que le VNet `vnet-finsecure-storage` et votre IP apparaissent dans leurs sections respectives

**Test de rejet réseau :** Depuis un navigateur en navigation privée sur une connexion VPN (IP différente), tenter d'accéder à `https://stfinsecure1234.blob.core.windows.net/`  vous devez recevoir une erreur **403 This request is not authorized**.

---

## 🔑 Étape 3 — RBAC et identités managées

### 🎯 Objectif

Configurer le contrôle d'accès basé sur les rôles (RBAC) pour remplacer l'authentification par clé de compte (méthode héritée et risquée). Attribuer à **Serge TOGNON** les rôles appropriés pour gérer les données sans utiliser les clés partagées, et créer une identité managée pour les applications.

**Lien AZ-104 :** Domaine *"Gérer les identités et la gouvernance Azure"* Attribution de rôles, Managed Identities, principe du moindre privilège.

### 🧠 Concepts clés

| Concept | Explication |
|---|---|
| **RBAC Azure** | Contrôle qui peut faire quoi sur quelle ressource, via des rôles prédéfinis ou personnalisés |
| **Storage Blob Data Contributor** | Peut lire, écrire et supprimer des blobs ,opère sur les **données** |
| **Storage Blob Data Reader** | Lecture seule sur les blobs , idéal pour les auditeurs |
| **Storage Account Contributor** | Gère la configuration du compte n'accède pas aux données blob |
| **Identité managée** | Identité gérée par Azure  élimine la gestion manuelle des secrets et des rotations |
| **Scope RBAC** | L'attribution peut être faite au niveau : Abonnement > Resource Group > Ressource |

> ⚠️ **Distinction importante :** Les rôles `Storage Blob Data *` opèrent sur les **données** (blobs). Le rôle `Storage Account Contributor` opère sur la **gestion du compte** (configuration) mais ne donne pas accès direct aux données. Ne pas confondre ces deux niveaux.

### ⚙️ Procédure détaillée — Portail Azure

#### 3.1 — Attribuer le rôle à Serge TOGNON

1. Dans votre compte de stockage, cliquer sur **`Contrôle d'accès (IAM)`** dans le menu gauche
2. Cliquer sur **`+ Ajouter`** > **`Ajouter une attribution de rôle`**

**Onglet "Rôle" :**

3. Dans la barre de recherche, taper `Contributeur aux données Blob`
4. Sélectionner **`Contributeur aux données Blob du stockage`**
5. Cliquer sur **`Suivant`**

**Onglet "Membres" :**

6. **Accès attribué à :** sélectionner `Utilisateur, groupe ou principal de service`
7. Cliquer sur **`+ Sélectionner des membres`**
8. Rechercher **`Serge TOGNON`** dans le champ de recherche
9. Sélectionner votre compte dans les résultats
10. Cliquer **`Sélectionner`**

**Finalisation :**

11. Cliquer sur **`Vérifier + attribuer`**
12. Relire le résumé (rôle + membre + périmètre) et cliquer **`Vérifier + attribuer`** une seconde fois pour confirmer


#### 3.2 — Vérifier les attributions de rôles

1. Toujours dans **`Contrôle d'accès (IAM)`**, cliquer sur l'onglet **`Attributions de rôles`**
2. Dans le filtre "Étendue", sélectionner `Cette ressource` pour afficher uniquement les attributions directes
3. Vérifier que l'attribution pour **Serge TOGNON** apparaît bien

---

<img width="911" height="431" alt="G1" src="https://github.com/user-attachments/assets/252825c1-5429-4a38-8d6f-f1ad270b72ca" />


---

#### 3.3 — Créer une identité managée (pour les applications)

1. Dans la barre de recherche Azure, taper **`Identités managées`**
2. Cliquer sur **`+ Créer`**

| Champ | Valeur |
|---|---|
| **Abonnement** | Votre abonnement étudiant |
| **Groupe de ressources** | `rg-storage-security` |
| **Région** | `West Europe` |
| **Nom** | `mi-finsecure-app` |

3. Cliquer sur **`Vérifier + créer`** → **`Créer`**
4. Une fois créée, revenir dans **`Contrôle d'accès (IAM)`** du compte de stockage
5. Répéter le processus d'attribution de rôle mais cette fois :
   - **Accès attribué à :** `Identité managée`
   - **Membres :** Sélectionner `mi-finsecure-app`
   - **Rôle :** `Contributeur aux données Blob du stockage`

---
<img width="905" height="306" alt="G2" src="https://github.com/user-attachments/assets/1bc63947-ed77-43bd-bb03-7b9cc3028134" />
<img width="874" height="403" alt="G3" src="https://github.com/user-attachments/assets/832983f8-60ff-45ae-a99b-b583bd5476d0" />

---

### 🔒 Risques de sécurité évités

> 🔴 **Risque :** Coder les clés de compte (`AccountKey`) directement dans le code d'application — si le code est publié sur GitHub par erreur, les clés sont exposées publiquement (scénario malheureusement fréquent).
>
> 🔴 **Risque :** Attribuer `Owner` ou `Contributor` au niveau du Resource Group pour "simplifier" — donne des droits bien au-delà du nécessaire (violation du principe du moindre privilège).
>
> 🟢 **Bonne pratique Microsoft :** En production, utiliser **Microsoft Entra Privileged Identity Management (PIM)** pour les accès administrateurs — accès Just-In-Time avec durée limitée et approbation obligatoire.

### 💻 Commandes de référence — Azure CLI

```bash
# ID du compte de stockage (nécessaire pour le scope RBAC)
STORAGE_ID=$(az storage account show \
  --name $STORAGE_ACCOUNT \
  --resource-group $RESOURCE_GROUP \
  --query id -o tsv)

# ID objet de l'utilisateur courant (Serge TOGNON)
USER_OBJECT_ID=$(az ad signed-in-user show --query id -o tsv)

# Attribuer Storage Blob Data Contributor à Serge TOGNON
az role assignment create \
  --assignee $USER_OBJECT_ID \
  --role "Storage Blob Data Contributor" \
  --scope $STORAGE_ID

# Créer l'identité managée pour les applications
az identity create \
  --name mi-finsecure-app \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION

MI_PRINCIPAL_ID=$(az identity show \
  --name mi-finsecure-app \
  --resource-group $RESOURCE_GROUP \
  --query principalId -o tsv)

# Attribuer le rôle à l'identité managée
az role assignment create \
  --assignee $MI_PRINCIPAL_ID \
  --role "Storage Blob Data Contributor" \
  --scope $STORAGE_ID

echo "✅ RBAC configuré pour Serge TOGNON et l'identité managée mi-finsecure-app"

# Vérifier toutes les attributions de rôles sur le Storage Account
az role assignment list --scope $STORAGE_ID --output table
```

### ✅ Validation

1. Dans le compte de stockage > **`Contrôle d'accès (IAM)`** > **`Vérifier l'accès`**
2. Rechercher **`Serge TOGNON`**
3. Vérifier que le rôle **"Contributeur aux données Blob du stockage"** apparaît dans les attributions effectives

---

## 🔏 Étape 4 — Signatures d'accès partagé (SAS)

### 🎯 Objectif

Générer des SAS (Shared Access Signatures) permettant à des partenaires externes (auditeurs, régulateurs) d'accéder temporairement et de manière granulaire aux documents de FinSecure SA  sans partager les clés du compte ni créer des comptes Entra ID pour chaque partenaire.

**Lien AZ-104 :** Domaine *"Mettre en œuvre et gérer le stockage"*  SAS, politiques d'accès stockées, délégation d'accès.

### 🧠 Concepts clés

| Concept | Explication |
|---|---|
| **Account SAS** | Accès à plusieurs services (Blob, File, Queue, Table) — portée large |
| **Service SAS** | Accès limité à un seul service — plus granulaire |
| **User Delegation SAS** | Signé avec les credentials Entra ID de l'utilisateur — **méthode la plus sécurisée** |
| **Stored Access Policy** | Politique attachée à un conteneur permettant de révoquer un SAS avant son expiration |
| **Paramètres SAS** | `sp` (permissions), `se` (expiration), `spr` (protocole), `sv` (version) |
| **Principe du moindre accès** | Ne donner que les permissions strictement nécessaires pour la durée minimale requise |

### ⚙️ Procédure détaillée — Portail Azure

#### 4.1 — Créer le conteneur de documents

Avant de générer un SAS, créer le conteneur de blobs cible.

1. Dans le compte de stockage, cliquer sur **`Stockage des données`** > **`Conteneurs`**
2. Cliquer sur **`+ Conteneur`**

| Champ | Valeur |
|---|---|
| **Nom** | `documents-finances` |
| **Niveau d'accès public** | `Privé (pas d'accès anonyme)` |

3. Cliquer sur **`Créer`**

---

<img width="911" height="394" alt="H" src="https://github.com/user-attachments/assets/a34a38a0-d3f1-4b37-a85c-68c44e5a2a31" />


---

#### 4.2 — Générer un SAS au niveau du compte

1. Dans le compte de stockage, cliquer sur **`Sécurité + réseau`** > **`Signature d'accès partagé`**
2. Configurer les paramètres suivants :

| Section | Paramètre | Valeur |
|---|---|---|
| **Services autorisés** | Blob | ✅ Coché uniquement |
| | File, File d'attente, Table | ❌ Décochés |
| **Types de ressources autorisés** | Service | ❌ Décoché |
| | Conteneur | ✅ Coché |
| | Objet | ✅ Coché |
| **Permissions autorisées** | Lecture | ✅ Cochée |
| | Liste | ✅ Cochée |
| | Écriture, Suppression, etc. | ❌ Décochées |
| **Date/heure de début** | | Maintenant (laisser par défaut) |
| **Date/heure d'expiration** | | Dans **1 heure maximum** |
| **Protocoles autorisés** | | `HTTPS uniquement` |
| **Clé de signature** | | `key1` |

3. Cliquer sur **`Générer la chaîne de connexion et la SAP`**
4. Copier **l'URL de SAP du service Blob** (la 3ème URL générée)

> 🔒 **Sécurité :** Ne jamais sauvegarder le token SAS complet dans un dépôt Git ou un document partagé. Masquer les 20 derniers caractères sur les captures d'écran.

---

<img width="917" height="431" alt="I1" src="https://github.com/user-attachments/assets/d51ed6d3-e155-4fae-9a0b-e3841f2cd248" />
<img width="914" height="398" alt="I2" src="https://github.com/user-attachments/assets/f034cb31-1237-4597-acc7-489278b0ee17" />

---

#### 4.3 — Créer une Stored Access Policy (bonne pratique)

Une Stored Access Policy permet de révoquer instantanément un SAS sans attendre son expiration , essentiel en cas de compromission.

1. Dans le compte de stockage > **`Stockage des données`** > **`Conteneurs`**
2. Cliquer sur le conteneur **`documents-finances`**
3. Cliquer sur **`Stratégies d'accès`** dans le menu gauche du conteneur
4. Cliquer sur **`+ Ajouter une stratégie`**

| Champ | Valeur |
|---|---|
| **Identificateur** | `pol-lecture-auditeurs` |
| **Autorisations** | Cocher `Lecture` et `Liste` uniquement |
| **Heure de début** | Date actuelle |
| **Heure d'expiration** | Dans 30 jours |

5. Cliquer **`OK`** puis **`Enregistrer`**
----
<img width="923" height="308" alt="I3" src="https://github.com/user-attachments/assets/9c608727-fd33-482a-aed9-cba7e6fe4c55" />
<img width="923" height="301" alt="I4" src="https://github.com/user-attachments/assets/974a6cb2-cd07-4c7f-aa2a-1ba0ebf71df9" />


-----
### 🔒 Risques de sécurité évités

> 🔴 **Risque :** Générer un SAS avec toutes les permissions (`racwdl`)  un partenaire externe pourrait accidentellement ou malicieusement modifier ou supprimer des documents financiers.
>
> 🔴 **Risque :** Définir une expiration SAS à 1 an  si le token est exposé, l'attaquant dispose d'un accès prolongé.
>
> 🟢 **Bonne pratique Microsoft :** Utiliser les **User Delegation SAS** en production (basés sur Entra ID, révocables sans rotation de clés). Les Account SAS sont à réserver aux cas d'usage où Entra ID n'est pas disponible.

### 💻 Commandes de référence — Azure CLI

```bash
# Définir l'expiration à 1 heure
EXPIRY=$(date -u -d '+1 hour' '+%Y-%m-%dT%H:%MZ' 2>/dev/null || \
         date -u -v+1H '+%Y-%m-%dT%H:%MZ')

# Générer un SAS lecture seule sur le service Blob (1 heure)
SAS_TOKEN=$(az storage account generate-sas \
  --account-name $STORAGE_ACCOUNT \
  --services b \
  --resource-types co \
  --permissions rl \
  --expiry $EXPIRY \
  --https-only \
  --output tsv)

SAS_URL="https://$STORAGE_ACCOUNT.blob.core.windows.net/documents-finances?$SAS_TOKEN"
echo "✅ SAS généré — URL (token masqué) : ${SAS_URL:0:80}...[MASQUÉ]"

# Créer une Stored Access Policy sur le conteneur
az storage container policy create \
  --account-name $STORAGE_ACCOUNT \
  --container-name documents-finances \
  --name pol-lecture-auditeurs \
  --permissions rl \
  --expiry "2025-12-31T23:59Z" \
  --auth-mode login
```

### ✅ Validation

1. Copier l'URL SAS générée à l'étape 4.2
2. Coller l'URL dans un navigateur en navigation privée — vous devriez obtenir la liste XML des blobs du conteneur (HTTP 200)
3. Modifier manuellement `sp=rl` en `sp=w` dans l'URL et retenter — vous devriez obtenir une erreur **403 AuthorizationPermissionMismatch**

**Résultat attendu :** L'accès en lecture fonctionne, l'accès en écriture est refusé.
<img width="960" height="440" alt="J1" src="https://github.com/user-attachments/assets/70f70f4e-e92e-4c4b-b455-42884bfd78d2" />
<img width="959" height="203" alt="J2" src="https://github.com/user-attachments/assets/071ba3fa-d4b6-4d4a-bbb5-1f2c6aa078d7" />

---

## 🛡 Étape 5 — Microsoft Defender pour le stockage

### 🎯 Objectif

Activer la détection des menaces avancée sur le compte de stockage de FinSecure SA. Defender for Storage analyse les patterns d'accès et alerte automatiquement en cas d'activité suspecte : exfiltration massive de données, accès depuis des adresses IP Tor ou anonymes, upload de malwares, accès depuis une géolocalisation inhabituelle.

**Lien AZ-104 :** Domaine *"Surveiller et gérer les ressources Azure"*  Microsoft Defender for Cloud, sécurité des ressources.

### 🧠 Concepts clés

| Concept | Explication |
|---|---|
| **Defender for Storage** | Service de détection des menaces spécifique au Stockage Azure, analysant les accès en temps réel |
| **Activity Monitoring** | Détecte les comportements anormaux : volume élevé, géolocalisation inhabituelle, heure atypique |
| **Analyse des malwares** | Scan antivirus à l'upload des blobs  option payante, **à désactiver en lab** |
| **Détection données sensibles** | Identifie les PII (données personnelles) et données financières dans les blobs |
| **Alertes de sécurité** | Générées dans Defender for Cloud et transmissibles par email ou vers un SIEM |


### ⚙️ Procédure détaillée — Portail Azure

#### 5.1 — Activer Defender for Storage

1. Dans la barre de recherche Azure, taper **`Microsoft Defender pour le cloud`**
2. Dans le menu gauche, cliquer sur **`Paramètres d'environnement`**
3. Cliquer sur le nom de votre **abonnement étudiant** pour l'ouvrir
4. Dans la liste des **Plans Defender**, localiser la ligne **`Stockage`**
5. Basculer l'interrupteur de **`Désactivé`** vers **`Activé`** (il devient vert)

#### 5.2 — Configurer les options (budget étudiant)

6. Cliquer sur **`Paramètres`** à droite de la ligne "Stockage" :

| Option | Valeur (lab étudiant) | Raison |
|---|---|---|
| **Analyse des logiciels malveillants** | ❌ `Désactivé` | Coûteux ($0.15/Go scanné) |
| **Détection des données sensibles** | ✅ `Activé` | Gratuit - très utile |
| **Surveillance des activités** | ✅ `Activé` | Gratuit — essentiel |

7. Cliquer **`Continuer`** puis **`Enregistrer`** en haut de la page

---

<img width="922" height="437" alt="K1" src="https://github.com/user-attachments/assets/2e2a7ac4-c0fc-4c05-97eb-ef7bc537e968" />
<img width="913" height="410" alt="K2" src="https://github.com/user-attachments/assets/5eb72042-d692-42ff-9ea6-8ab8bfd1d0f7" />


---
#### 5.3 — Vérifier la protection depuis le compte de stockage

1. Revenir au compte de stockage (`stfinsecure1234`)
2. Dans le menu gauche, faire défiler jusqu'à **`Microsoft Defender pour le cloud`**
3. Vérifier que le statut de protection indique **"Protégé"** ou **"Activé"**

> ⚠️ La propagation peut prendre jusqu'à 24 heures après activation. Une mention "En attente" est normale immédiatement après la configuration.

---

<img width="885" height="403" alt="K3" src="https://github.com/user-attachments/assets/29dd93e6-efb1-4a37-99b7-f72330df83ec" />


---

### 🔒 Risques de sécurité évités

> 🔴 **Risque :** Sans Defender, une exfiltration de données peut passer inaperçue pendant des semaines — les données financières de FinSecure SA pourraient être volées sans aucune alerte.
>
> 🔴 **Risque :** Sans détection des données sensibles, les équipes peuvent involontairement stocker des données RGPD dans des conteneurs mal configurés.
>
> 🟢 **Bonne pratique Microsoft :** Connecter les alertes Defender to Cloud à **Microsoft Sentinel** (SIEM) pour corréler les incidents de sécurité Storage avec d'autres événements du tenant.

### 💻 Commandes de référence — Azure CLI

```bash
# Activer Microsoft Defender for Storage au niveau abonnement
az security pricing create \
  --name StorageAccounts \
  --tier Standard

# Vérifier le statut
az security pricing show \
  --name StorageAccounts \
  --query "{service: name, tier: pricingTier}" \
  --output table

# Configurer un contact de sécurité pour les alertes email
az security contact create \
  --name admin-securite \
  --email "serge.tognon@finsecure.bj" \
  --alert-notifications On \
  --alerts-to-admins On

echo "✅ Defender for Storage activé — alertes configurées pour serge.tognon@finsecure.bj"
```

### ✅ Validation

1. Dans le portail Azure > **`Microsoft Defender pour le cloud`** > **`Recommandations`**
2. Filtrer par ressource : votre compte de stockage
3. Les recommandations de sécurité déjà appliquées (comme "transfert sécurisé", "accès réseau restreint") doivent apparaître comme "Sain" (vert)

---

## 📋 Étape 6 — Audit, diagnostic et rotation des clés

### 🎯 Objectif

Mettre en place la journalisation complète de tous les accès au stockage (lecture, écriture, suppression) et effectuer une rotation des clés d'accès sans interruption de service. Ces deux composantes sont indispensables pour la conformité réglementaire RGPD et les investigations post-incident chez FinSecure SA.

**Lien AZ-104 :** Domaine *"Surveiller et gérer les ressources Azure"*  Azure Monitor, Log Analytics, Diagnostic Settings.

### 🧠 Concepts clés

| Concept | Explication |
|---|---|
| **Paramètres de diagnostic** | Configuration de l'envoi des logs d'une ressource vers une destination d'analyse |
| **StorageRead / Write / Delete** | Catégories de logs : toutes les opérations de lecture/écriture/suppression sur les blobs |
| **Log Analytics Workspace** | Service centralisé d'analyse de logs avec requêtes en langage KQL |
| **KQL (Kusto Query Language)** | Langage de requête Azure Monitor pour analyser et corréler les logs |
| **Rotation des clés** | Remplacement des clés d'accès partagées — les applications doivent être mises à jour |
| **Rétention des logs** | Durée de conservation recommandée : 90 jours minimum (RGPD), 1 an (PCI-DSS) |

### ⚙️ Procédure détaillée — Portail Azure

#### 6.1 — Créer un Log Analytics Workspace

1. Dans la barre de recherche Azure, taper **`Espaces de travail Log Analytics`**
2. Cliquer sur **`+ Créer`**

| Champ | Valeur |
|---|---|
| **Abonnement** | Votre abonnement étudiant |
| **Groupe de ressources** | `rg-storage-security` |
| **Nom** | `law-storage-security` |
| **Région** | `West Europe` |

3. Cliquer sur **`Vérifier + créer`** → **`Créer`**

---
<img width="925" height="422" alt="L1" src="https://github.com/user-attachments/assets/92b364c1-5db6-4473-b9bb-1b425e3bdb02" />

---

#### 6.2 — Configurer les paramètres de diagnostic

1. Revenir au compte de stockage (`stfinsecure1234`)
2. Dans le menu gauche, cliquer sur **`Surveillance`** > **`Paramètres de diagnostic`**
3. Vous voyez une liste avec les services : `blob`, `file`, `queue`, `table` — cliquer sur **`blob`**
4. Cliquer sur **`+ Ajouter un paramètre de diagnostic`**

| Section | Champ | Valeur |
|---|---|---|
| **Général** | Nom du paramètre | `diag-blob-complet` |
| **Catégories de journaux** | StorageRead | ✅ Coché |
| | StorageWrite | ✅ Coché |
| | StorageDelete | ✅ Coché |
| **Métriques** | Transaction | ✅ Coché |
| **Détails de destination** | Envoyer à Log Analytics | ✅ Sélectionner |
| | Abonnement | Votre abonnement |
| | Espace de travail | `law-storage-security` |

5. Cliquer sur **`Enregistrer`**

---
<img width="914" height="422" alt="L2" src="https://github.com/user-attachments/assets/10baf1d2-7c73-451c-91df-73b982292c87" />

---

#### 6.3 — Effectuer la rotation d'une clé d'accès

> ⚠️ **Avant de pivoter une clé en production :** S'assurer que toutes les applications utilisant `key1` ont été basculées sur `key2`. Chez FinSecure SA, cette opération se fait lors d'une fenêtre de maintenance planifiée.

1. Dans le compte de stockage > **`Sécurité + réseau`** > **`Clés d'accès`**
2. Cliquer sur **`Afficher les clés`** pour voir les clés (authentification MFA peut être requise)
3. Observer les deux clés (`key1` et `key2`) et leurs dates de dernière rotation
4. Cliquer sur **`Pivoter la clé`** en face de **`key1`**
5. Lire le message d'avertissement dans la boîte de dialogue
6. Confirmer en cliquant **`Oui`**

---
<img width="914" height="421" alt="M1" src="https://github.com/user-attachments/assets/86cac28f-a564-40c0-b0e3-ac6415d0a8c0" />
<img width="917" height="416" alt="M2" src="https://github.com/user-attachments/assets/9ab336ac-46b5-4362-96f1-2f0210705431" />



#### 6.4 — Requêtes KQL pour l'audit (Log Analytics)

Après quelques minutes (5–15 min pour la première propagation des logs), naviguer vers le workspace Log Analytics :

1. Dans le portail > **`Espaces de travail Log Analytics`** > `law-storage-security`
2. Cliquer sur **`Journaux`** dans le menu gauche
3. Coller et exécuter les requêtes suivantes :

```kql
// Requête 1 : Toutes les opérations des dernières 24 heures
StorageBlobLogs
| where TimeGenerated > ago(24h)
| summarize Nombre = count() by OperationName, StatusCode
| sort by Nombre desc
```
<img width="920" height="422" alt="N1" src="https://github.com/user-attachments/assets/30049067-f968-459c-8511-07760da0dccb" />

```kql
// Requête 2 : Tentatives d'accès échouées (alertes critiques FinSecure SA)
StorageBlobLogs
| where StatusCode in (401, 403)
| summarize Tentatives = count() by CallerIpAddress, bin(TimeGenerated, 1h)
| sort by Tentatives desc
```
<img width="907" height="400" alt="N2" src="https://github.com/user-attachments/assets/800977bd-2508-473d-b3e8-ad28f934b742" />

```kql
// Requête 3 : Suppressions de fichiers (audit réglementaire)
StorageBlobLogs
| where OperationName == "DeleteBlob"
|project TimeGenerated, CallerIpAddress, OperationName, ObjectKey
|sort by TimeGenerated desc 
---
<img width="917" height="400" alt="N3" src="https://github.com/user-attachments/assets/d1285241-cc7e-4724-93ef-c4fbe48345c9" />

---

### 🔒 Risques de sécurité évités

> 🔴 **Risque :** Sans logs de diagnostic, FinSecure SA ne peut pas répondre à la question "Qui a accédé à quel document le 15 janvier à 14h23 ?" — violation des exigences RGPD d'audit.
>
> 🔴 **Risque :** Ne jamais faire pivoter les clés — si une clé est compromise (fuite dans un log, un email, un dépôt Git), l'attaquant a un accès permanent au stockage.
>
> 🟢 **Bonne pratique Microsoft :** Automatiser la rotation des clés via **Azure Key Vault** avec une politique de rotation automatique tous les 90 jours, avec notification par email à Serge TOGNON.

### 💻 Commandes de référence — Azure CLI

```bash
# Créer le Log Analytics Workspace
az monitor log-analytics workspace create \
  --resource-group $RESOURCE_GROUP \
  --workspace-name law-storage-security \
  --location $LOCATION \
  --retention-time 90

WORKSPACE_ID=$(az monitor log-analytics workspace show \
  --resource-group $RESOURCE_GROUP \
  --workspace-name law-storage-security \
  --query id -o tsv)

# Configurer les paramètres de diagnostic pour le service Blob
STORAGE_ID=$(az storage account show --name $STORAGE_ACCOUNT --resource-group $RESOURCE_GROUP --query id -o tsv)

az monitor diagnostic-settings create \
  --name "diag-blob-complet" \
  --resource "${STORAGE_ID}/blobServices/default" \
  --logs '[
    {"category": "StorageRead", "enabled": true},
    {"category": "StorageWrite", "enabled": true},
    {"category": "StorageDelete", "enabled": true}
  ]' \
  --metrics '[{"category": "Transaction", "enabled": true}]' \
  --workspace $WORKSPACE_ID

# Rotation de key1
az storage account keys renew \
  --account-name $STORAGE_ACCOUNT \
  --resource-group $RESOURCE_GROUP \
  --key key1

echo "✅ Diagnostic configuré et key1 pivotée — journalisation active vers law-storage-security"
```

### ✅ Validation

1. Dans le compte de stockage, effectuer quelques opérations (uploader un fichier, le télécharger, le supprimer)
2. Attendre 5 à 10 minutes
3. Dans Log Analytics > `Journaux`, exécuter la requête 1 (Opérations 24h) — les opérations effectuées doivent apparaître

**Résultat attendu :** Les opérations `PutBlob`, `GetBlob`, `DeleteBlob` apparaissent dans les résultats avec leur IP source et leur code de statut HTTP.

---

## 📊 Synthèse des bonnes pratiques

| ✅ Recommandé | ❌ À éviter |
|---|---|
| Microsoft Entra ID + Identités managées pour les applications | Coder les clés de compte en dur dans le code source ou dans Git |
| Transfert sécurisé activé (HTTPS uniquement) | Autoriser les connexions HTTP non chiffrées |
| VNet dédié + Service Endpoint pour le trafic applicatif | Laisser le compte accessible depuis tous les réseaux publics |
| SAS avec permissions minimales + expiration courte (1h) | Partager les clés de compte avec des partenaires ou des tiers |
| Stored Access Policy pour pouvoir révoquer un SAS | Générer des SAS sans politique de révocation associée |
| Microsoft Defender for Storage activé dès la création | Ignorer les alertes de sécurité dans Defender for Cloud |
| Rotation des clés tous les 90 jours (automatisée via Key Vault) | Conserver la même clé indéfiniment |
| Log Analytics avec catégories Read/Write/Delete activées | Désactiver les journaux de diagnostic |
| Soft Delete activé (7 jours minimum) | Permettre la suppression définitive immédiate sans filet de sécurité |
| Rétention des logs 90 jours minimum | Logs avec rétention de moins de 30 jours |

---

## 🚨 Dépannage (Troubleshooting)

| Erreur rencontrée | Cause probable | Solution |
|---|---|---|
| `AuthorizationFailure (403)` | Votre IP n'est pas dans le pare-feu Storage | Ajouter l'IP dans Sécurité + réseau > Réseau |
| `BlobAccessTierNotSupported` | Tier incompatible avec le compte Standard | Utiliser Hot/Cool ou passer en StorageV2 |
| `InvalidResourceName` | Nom du compte avec majuscules ou caractères spéciaux | Respecter : 3-24 chars, minuscules et chiffres uniquement |
| Token SAS → erreur 403 | Token SAS expiré | Régénérer un nouveau SAS avec une nouvelle expiration |
| `PublicAccessNotPermitted` | Accès public désactivé (voulu) | Utiliser RBAC Entra ID ou un token SAS valide |
| `NoDiagnosticSettingsFound` | Paramètres de diagnostic non configurés | Refaire l'Étape 6 — vérifier que "blob" est sélectionné |
| Logs absents dans Log Analytics | Délai de propagation | Attendre 5 à 15 minutes après la première configuration |
| VNet non visible dans la liste pare-feu Storage | VNet dans une région différente du Storage | Créer le VNet dans la **même région** que le Storage Account |
| `Role assignment already exists` | Attribution de rôle déjà présente | Vérifier IAM > Attributions de rôles avant d'ajouter |
| Defender affiche "En attente" | Délai de propagation post-activation | Attendre jusqu'à 24 heures — comportement normal |

```bash
# Commande de diagnostic rapide — vérifier la configuration globale
az storage account show \
  --name $STORAGE_ACCOUNT \
  --resource-group $RESOURCE_GROUP \
  --query "{
    nom: name,
    httpsOnly: enableHttpsTrafficOnly,
    tlsMin: minimumTlsVersion,
    blobPublicAccess: allowBlobPublicAccess,
    networkDefault: networkRuleSet.defaultAction,
    nbReglesIP: length(networkRuleSet.ipRules)
  }" \
  --output table
```
### Nettoyage des ressources après le lab

```bash
# ⚠️ ATTENTION — Suppression irréversible de toutes les ressources du lab

# Supprimer tout le groupe de ressources (Storage + VNet + Log Analytics)
az group delete \
  --name rg-storage-security \
  --yes \
  --no-wait

echo "🗑️ Suppression en cours — vérifier dans le portail Azure dans 2-3 minutes"

# Vérifier la suppression
az group show --name rg-storage-security 2>&1 | grep -q "not found" && \
  echo "✅ Groupe de ressources supprimé avec succès" || \
  echo "⏳ Suppression encore en cours..."
```

### Bonnes pratiques budget étudiant

- Configurer une **alerte de budget à 5 €/mois** : Portail > Abonnements > Budgets > + Ajouter
- Utiliser `Standard_LRS` en lab (3× moins cher que GRS)
- **Supprimer les ressources après chaque session de lab** — ne jamais laisser tourner un lab 24h/24
- Vérifier régulièrement la page **"Cost Management + Billing"** dans le portail
- Utiliser des **tags** sur toutes les ressources : `"environnement=lab"` pour identifier les ressources temporaires

---

## 🎓 Compétences démontrées

### Domaines AZ-104

| Compétence | Niveau | Evidence dans ce projet |
|---|---|---|
| Configuration sécurisée des comptes de stockage | ⭐⭐⭐⭐⭐ | Étape 1 — HTTPS, TLS 1.2, Soft Delete |
| Création et configuration de VNet | ⭐⭐⭐⭐⭐ | Étape 2 — VNet, Subnet, Service Endpoint |
| Pare-feu et contrôle d'accès réseau Storage | ⭐⭐⭐⭐⭐ | Étape 2 — Règles IP + VNet |
| RBAC Azure et Managed Identities | ⭐⭐⭐⭐⭐ | Étape 3 — Rôles Storage Data, MI |
| Signatures d'accès partagé (SAS) | ⭐⭐⭐⭐⭐ | Étape 4 — SAS, Stored Access Policy |
| Microsoft Defender for Cloud | ⭐⭐⭐⭐ | Étape 5 — Plans Defender, alertes |
| Azure Monitor et Log Analytics | ⭐⭐⭐⭐⭐ | Étape 6 — Diagnostic Settings, KQL |
| Rotation des clés d'accès | ⭐⭐⭐⭐ | Étape 6 — Rotation sans interruption |

### Compétences transverses démontrées

| Compétence | Description |
|---|---|
| **Sécurité Cloud (Zero Trust)** | Modèle Zero Trust appliqué : vérification explicite, moindre privilège, hypothèse de violation |
| **Conformité réglementaire** | Audit, rétention et traçabilité répondant aux exigences RGPD, ISO 27001, PCI-DSS |
| **Défense en profondeur** | 6 couches de sécurité empilées (réseau, identité, données, accès, détection, audit) |
| **Infrastructure as Code** | Scripts CLI et PowerShell reproductibles, versionnables et déployables en CI/CD |
| **Documentation technique** | Documentation niveau ingénieur, structurée et prête pour GitHub portfolio |
| **Optimisation des coûts** | Choix délibérés des SKUs, nettoyage des ressources, alertes de budget |
| **Troubleshooting Azure** | Diagnostic méthodique et résolution des problèmes courants de stockage Azure |

---
## 🔗 Ressources Microsoft officielles

### Microsoft Learn — Parcours recommandés

| Ressource | Pertinence |
|---|---|
| [AZ-104 : Mettre en œuvre et gérer le stockage](https://learn.microsoft.com/fr-fr/training/paths/az-104-manage-storage/) | ⭐⭐⭐⭐⭐ Direct |
| [Sécuriser le stockage Azure — Recommandations](https://learn.microsoft.com/fr-fr/azure/storage/blobs/security-recommendations) | ⭐⭐⭐⭐⭐ Direct |
| [Gérer les identités et la gouvernance AZ-104](https://learn.microsoft.com/fr-fr/training/paths/az-104-manage-identities-governance/) | ⭐⭐⭐⭐⭐ Direct |
| [Configurer les réseaux virtuels Azure](https://learn.microsoft.com/fr-fr/training/modules/configure-virtual-networks/) | ⭐⭐⭐⭐ VNet Étape 2 |
| [Configurer Azure Monitor](https://learn.microsoft.com/fr-fr/training/modules/configure-azure-monitor/) | ⭐⭐⭐⭐ Étape 6 |
| [Microsoft Defender for Storage](https://learn.microsoft.com/fr-fr/azure/defender-for-cloud/defender-for-storage-introduction) | ⭐⭐⭐⭐ Étape 5 |

### Documentation officielle Microsoft

| Documentation | URL |
|---|---|
| Azure Storage — Bonnes pratiques sécurité | https://docs.microsoft.com/azure/storage/blobs/security-recommendations |
| RBAC Azure — Rôles intégrés Storage | https://docs.microsoft.com/azure/role-based-access-control/built-in-roles#storage |
| SAS — Vue d'ensemble et bonnes pratiques | https://docs.microsoft.com/azure/storage/common/storage-sas-overview |
| Service Endpoints pour Azure Storage | https://docs.microsoft.com/azure/virtual-network/virtual-network-service-endpoints-overview |
| Defender for Storage — Introduction | https://docs.microsoft.com/azure/defender-for-cloud/defender-for-storage-introduction |
| Log Analytics — KQL Reference | https://docs.microsoft.com/azure/data-explorer/kusto/query/ |
| Azure CLI — Référence Storage | https://docs.microsoft.com/cli/azure/storage |

### Frameworks de conformité couverts

| Framework | Contrôles adressés dans ce projet |
|---|---|
| **Microsoft Cloud Security Benchmark v3** | NS-1 (réseau), IM-1/IM-3 (identité), DP-3/DP-4 (données), LT-1/LT-3 (logs) |
| **ISO 27001:2022** | A.9 (contrôle d'accès), A.10 (chiffrement), A.12 (opérations), A.16 (incidents) |
| **RGPD** | Art. 25 (privacy by design), Art. 32 (mesures techniques), Art. 33 (notification) |
| **PCI-DSS v4.0** | Req. 7 (accès minimal), Req. 8 (authentification forte), Req. 10 (audit logs) |

---

<div align="center">

---

**Document rédigé par Serge TOGNON**

Administrateur Cloud Azure Certifié | Candidat RHCSA | Cotonou, Bénin

[![LinkedIn](https://img.shields.io/badge/LinkedIn-Serge_TOGNON-0077B5?style=for-the-badge&logo=linkedin&logoColor=white)]( https://www.linkedin.com/in/serge-tognon-a63443187/?lipi=urn%3Ali%3Apage%3Ad_flagship3_profile_view_base_contact_details%3BiHxYnwbJRjOOflTaGqZasw%3D%3D)
[![AZ-104](https://img.shields.io/badge/Microsoft-AZ--104_Certified-0078D4?style=for-the-badge&logo=microsoft&logoColor=white)](https://learn.microsoft.com/certifications/azure-administrator/)
[![RHCSA](https://img.shields.io/badge/Red_Hat-RHCSA_Candidat-EE0000?style=for-the-badge&logo=redhat&logoColor=white)](https://www.redhat.com/en/services/certification/rhcsa)

*Projet de démonstration des compétences Azure Administrator*

*"La sécurité n'est pas une fonctionnalité, c'est une culture."*

---

</div>
