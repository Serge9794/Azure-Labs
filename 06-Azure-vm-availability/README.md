# 🏛️ Azure VM Availability Lab — FinSecure SA

> **Projet Portfolio Cloud & Sécurité | AZ-104**
> Déploiement d'une architecture hautement disponible (SLA 99,99 %) avec redondance zonale, autoscaling et serveurs Linux Red Hat Enterprise.

---

![Azure](https://img.shields.io/badge/Azure-VM%20Availability-0078D4?style=for-the-badge&logo=microsoftazure&logoColor=white)
![AZ-104](https://img.shields.io/badge/Certification-AZ--104-0078D4?style=for-the-badge&logo=microsoft&logoColor=white)
![RHEL](https://img.shields.io/badge/OS-RHEL%209-EE0000?style=for-the-badge&logo=redhat&logoColor=white)
![Status](https://img.shields.io/badge/Statut-Complété-28a745?style=for-the-badge)

---

## 🛡️ Sécurité, Conformité et Gouvernance (ISO 27001)

*Ce projet est structuré selon les exigences de la norme ISO 27001, garantissant la résilience et la traçabilité des systèmes.*

| Domaine ISO | Implémentation technique |
|---|---|
| **Disponibilité (A.17)** | Availability Sets et Availability Zones pour garantir la continuité de service |
| **Gestion des accès (A.9)** | Principe du moindre privilège via NSG + authentification SSH par clé (sans mot de passe) |
| **Traçabilité (A.12.4)** | Collecte centralisée des logs dans un Log Analytics Workspace via Azure Monitor Agent |
| **Gestion des changements** | Automatisation du cycle de vie via Azure CLI (approche Infrastructure as Code) |
| **Durcissement OS (A.12.6)** | RHEL 9 avec SELinux enforcing, firewalld et mises à jour de sécurité intégrées |

> **Note de sécurité :** *Dans ce lab, l'authentification SSH par clé (`--generate-ssh-keys`) est utilisée à la place des mots de passe, conformément aux bonnes pratiques. En production, les clés et secrets sont gérés via Azure Key Vault.*

---

## 📋 Table des matières

- [Contexte et objectifs](#-contexte-et-objectifs)
- [Choix technologiques](#-choix-technologiques)
- [Architecture technique](#-architecture-technique)
- [Technologies et services](#-technologies-et-services)
- [Phase 1 — Availability Sets + Load Balancer](#-phase-1--availability-sets--load-balancer)
- [Phase 2 — VM Scale Sets + Autoscale](#-phase-2--vm-scale-sets--autoscale)
- [Phase 3 — Availability Zones](#-phase-3--availability-zones)
- [Supervision et Alerting (KQL)](#-supervision-et-alerting-kql)
- [Tests de résilience](#-tests-de-résilience)
- [Résultats et métriques](#-résultats-et-métriques)
- [Compétences démontrées](#-compétences-démontrées)
- [Auteur](#-auteur)

---

## 🏢 Contexte et objectifs

**FinSecure SA** est une société de services financiers dont l'infrastructure héberge une **application web critique** soumise à un SLA de **99,99 % de disponibilité**.

À la suite d'une interruption de service non planifiée ayant causé une indisponibilité de **4 heures**, la direction technique a mandaté la mise en place d'une architecture **hautement disponible, résiliente et scalable** sur Azure.

**Objectif majeur :** Éliminer tout point de défaillance unique  via une approche multi-couches couvrant le rack physique, le datacenter et la couche applicative.

---

## 💡 Choix technologiques

### Pourquoi RHEL 9 ?

**Red Hat Enterprise Linux 9** a été retenu comme système d'exploitation pour l'ensemble des VMs du lab, pour les raisons suivantes :

- **Niveau entreprise :** RHEL 9 est le standard de facto dans les environnements financiers et réglementés (PCI-DSS, ISO 27001). Sa certification FIPS 140-2 et son cycle de support long  en font un choix stratégique.
- **Sécurité native :** SELinux en mode `enforcing` par défaut, firewalld intégré, et politique de mots de passe renforcée out-of-the-box.
- **Cohérence avec l'infrastructure existante :** FinSecure SA dispose de serveurs RHEL 9 on-premises gérés via Azure Arc; cette homogénéité simplifie les opérations, la supervision et la gestion des patches.
- **Support Azure de première classe :** RedHat et Microsoft maintiennent un partenariat officiel. RHEL 9 est disponible en image certifiée (`RedHat:rhel-arm64:9_8-arm64:latest`) directement depuis la marketplace Azure.
- **SKU retenu :** Standard_B2ps_v2 avec partitionnement LVM, recommandée pour les charges de production sur Azure.

---

## 🏗️ Architecture technique

L'infrastructure est déployée dans la région `canadacentral` avec trois niveaux de redondance imbriqués :

```
                   ┌──────────────────────────────────────────────────────┐
                   │              Azure Region : canadacentral            │
                   │                                                      │
                   │  ┌────────────┐  ┌────────────┐  ┌───────────────┐  │
                   │  │   Zone 1   │  │   Zone 2   │  │    Zone 3     │  │
                   │  │            │  │            │  │               │  │
                   │  │ ┌────────┐ │  │ ┌────────┐ │  │ ┌───────────┐ │  │
                   │  │ │RHEL 9  │ │  │ │RHEL 9  │ │  │ │VMSS RHEL9 │ │  │
                   │  │ │VM-AS-1 │ │  │ │VM-AS-2 │ │  │ │(2→10 inst)│ │  │
                   │  │ │FD:0 UD:0│ │  │ │FD:1 UD:1│ │  │           │ │  │
                   │  │ └───┬────┘ │  │ └───┬────┘ │  │ └─────┬─────┘ │  │
                   │  └─────┼──────┘  └─────┼──────┘  └───────┼───────┘  │
                   │        │               │                  │          │
                   │        └───────┬───────┘                  │          │
                   │                │                          │          │
                   │       ┌────────▼─────────┐                │          │
                   │       │  Azure Load      │◄───────────────┘          │
                   │       │  Balancer (L4)   │                           │
                   │       │  (Public IP)     │                           │
                   │       └────────┬─────────┘                           │
                   │                │                                      │
                   │       ┌────────▼─────────┐                           │
                   │       │  Azure Monitor   │                           │
                   │       │  + Log Analytics │                           │
                   │       │  + AMA + Alertes │                           │
                   │       └──────────────────┘                           │
                   └──────────────────────────────────────────────────────┘
                                    │
                             Internet / Clients
                             FinSecure SA
```

**Trois niveaux de redondance :**
- **Couche Calcul :** VMs RHEL 9 réparties sur Fault Domains, Update Domains et Availability Zones
- **Couche Réseau :** Load Balancer L4 avec Health Probes assurant la répartition de charge
- **Couche Supervision :** Azure Monitor + AMA assurant la visibilité et l'alerting proactif

---

## 🛠️ Technologies et services

| Service Azure | Rôle dans le projet |
|---|---|
| **Azure Virtual Machines** | Compute  RHEL 9 (`rhel-arm64:9_8-arm64:latest`) sur `Standard_B2ps_v2` |
| **Availability Sets** | Redondance intra-datacenter (2 FD + 5 UD) |
| **Availability Zones** | Redondance inter-datacenter (Zone 1, 2, 3) |
| **VM Scale Sets (VMSS)** | Scaling horizontal automatique (2 → 10 instances RHEL 9) |
| **Azure Load Balancer (L4)** | Distribution du trafic avec Health Probes |
| **Azure Monitor + AMA** | Supervision des métriques et des logs |
| **Log Analytics Workspace** | Centralisation et analyse via KQL |
| **Action Groups** | Notification e-mail/SMS à l'équipe d'astreinte |
| **Azure CLI** | Provisionnement et automatisation (IaC) |

**Région :** `canadacentral`
**Resource Group :** `rg-finsecure-availability`
**OS :** Red Hat Enterprise Linux 9  `RedHat:rhel-arm64:9_8-arm64:latest`
**SKU VM :** `Standard_B2ps_v2`

---

## 🔵 Phase 1 — Availability Sets + Load Balancer

### Objectif

Déployer deux VMs **RHEL 9** dans un Availability Set configuré avec 2 Fault Domains et 5 Update Domains, puis distribuer le trafic entrant via un Azure Load Balancer. Les Fault Domains protègent contre les pannes matérielles (rack, alimentation, switch) ; les Update Domains garantissent qu'Azure ne redémarre jamais toutes les VMs simultanément lors d'une maintenance.

### Commandes Azure CLI


1️⃣ Créer le groupe de ressources

Le Resource Group est le conteneur logique qui regroupe toutes les ressources Azure du projet. C'est le premier élément à créer ,toutes les ressources suivantes (VMs, réseau, Load Balancer) y seront rattachées. Sans lui, aucune ressource ne peut être déployée. On le place dans canadacentral pour bénéficier d'une bonne capacité de SKUs disponibles.
```bash
az group create \
  --name rg-finsecure-availability \
  --location canadacentral

2️⃣ Créer le VNet et le Subnet

Le Virtual Network (VNet) est le réseau privé isolé dans lequel toutes les VMs vont communiquer entre elles. Le Subnet est une subdivision de ce réseau qui regroupe les VMs d'une même couche applicative. Sans réseau virtuel, les VMs n'ont pas de canal de communication interne sécurisé. Le préfixe 10.0.0.0/16 offre jusqu'à 65 536 adresses IP privées, et le subnet 10.0.1.0/24 en réserve 256 pour nos VMs.
```bash
az network vnet create \
  --resource-group rg-finsecure-availability \
  --name vnet-finsecure \
  --address-prefix 10.0.0.0/16 \
  --subnet-name subnet-finsecure \
  --subnet-prefix 10.0.1.0/24 \
  --location canadacentral

3️⃣ Créer l'Availability Set

L'Availability Set est un mécanisme Azure qui garantit que les VMs sont distribuées sur des infrastructures physiques distinctes au sein du même datacenter. Les 2 Fault Domains signifient que les VMs sont sur des racks physiques différents (alimentation et réseau indépendants). Les 5 Update Domains garantissent qu'Azure ne redémarre jamais toutes les VMs simultanément lors d'une maintenance planifiée. Il faut impérativement créer l'Availability Set avant les VMs, car une VM ne peut y être ajoutée qu'à sa création.

```bash
az vm availability-set create \
  --resource-group rg-finsecure-availability \
  --name avset-finsecure \
  --location canadacentral \
  --platform-fault-domain-count 2 \
  --platform-update-domain-count 5

4️⃣ Créer VM 1 (RHEL 9)

La première machine virtuelle est déployée sous Red Hat Enterprise Linux 9 (RHEL 9 LVM Gen2), choix cohérent avec la préparation RHCSA en cours. Elle est placée dans l'Availability Set créé à l'étape 3, dans le subnet privé, sans IP publique (sécurité renforcée , l'accès se fera via le Load Balancer uniquement). Le flag --no-wait permet de lancer la création en arrière-plan sans bloquer le terminal, afin d'enchaîner immédiatement avec la VM 2.

bash
az vm create \
  --resource-group rg-finsecure-availability \
  --name vm-finsecure-1 \
  --availability-set avset-finsecure \
  --image RedHat:rhel-arm64:9_8-arm64:latest \
  --size Standard_B2ps_v2 \
  --admin-username polo \
  --generate-ssh-keys \
  --vnet-name vnet-finsecure \
  --subnet subnet-finsecure \
  --public-ip-address ""
  --no-wait

5️⃣ Créer VM 2 (RHEL 9)

La deuxième VM est identique à la première en termes de configuration (même image, même taille, même réseau) mais Azure la place automatiquement sur un Fault Domain différent (FD:1) et un Update Domain différent (UD:1) grâce à l'Availability Set. C'est ce mécanisme qui garantit que si vm-finsecure-1 tombe (panne de rack, maintenance), vm-finsecure-2 continue de servir les requêtes sans interruption.

```bash
az vm create \
 --resource-group rg-finsecure-availability \
  --name vm-finsecure-2 \
  --availability-set avset-finsecure \
  --image RedHat:rhel-arm64:9_8-arm64:latest \
  --size Standard_B2ps_v2 \
  --admin-username polo \
  --generate-ssh-keys \
  --vnet-name vnet-finsecure \
  --subnet subnet-finsecure \
  --public-ip-address ""
  --no-wait
  

6️⃣ Créer le Load Balancer

L'Azure Load Balancer Standard est le point d'entrée unique du trafic externe vers les VMs. Il expose une IP publique unique côté internet (frontendIP) et distribue les requêtes vers le Backend Pool qui contient les deux VMs. Le SKU Standard est obligatoire pour supporter les Availability Zones et offre des fonctionnalités avancées (haute disponibilité des ports, métriques Azure Monitor). Sans Load Balancer, les VMs en mode haute disponibilité ne seraient pas accessibles depuis l'extérieur.

```bash
az network lb create \
  --resource-group rg-finsecure-availability \
  --name lb-finsecure \
  --sku Standard \
  --frontend-ip-name frontendIP \
  --backend-pool-name backendPool

7️⃣ Créer le Health Probe (port 80)

Le Health Probe est le mécanisme de surveillance de santé des VMs par le Load Balancer. Toutes les quelques secondes, Azure envoie une sonde TCP sur le port 80 de chaque VM du backend pool. Si une VM ne répond plus (arrêt, panne, surcharge), le Load Balancer la retire automatiquement du pool et cesse de lui envoyer du trafic jusqu'à ce qu'elle réponde à nouveau. C'est le cerveau de la haute disponibilité ; sans Health Probe, le LB continuerait d'envoyer du trafic vers une VM morte.

```bash
az network lb probe create \
  --resource-group rg-finsecure-availability \
  --lb-name lb-finsecure \
  --name healthProbe \
  --protocol tcp \
  --port 80

8️⃣ Créer la règle de load balancing

La règle de load balancing définit comment le trafic entrant est distribué entre les VMs du backend pool. Elle lie le frontendIP (IP publique) au backendPool (les VMs) sur le port 80 (HTTP). Elle référence le healthProbe de l'étape 7 pour n'envoyer du trafic qu'aux VMs en bonne santé. C'est la règle de routage centrale du Load Balancer ; sans elle, le trafic entrant n'a aucune instruction sur comment être distribué vers les VMs.

```bash
az network lb rule create \
  --resource-group rg-finsecure-availability \
  --lb-name lb-finsecure \
  --name lbRule-HTTP \
  --protocol tcp \
  --frontend-port 80 \
  --backend-port 80 \
  --frontend-ip-name frontendIP \
  --backend-pool-name backendPool \
  --probe-name healthProbe

```

### Résultat attendu

| VM | Fault Domain | Update Domain | OS |
|---|---|---|---|
| `vm-finsecure-1` | FD 0 | UD 0 | RHEL 9 |
| `vm-finsecure-2` | FD 1 | UD 1 | RHEL 9 |

> ✅ Si le rack hébergeant FD 0 tombe en panne, seule `vm-finsecure-1` est impactée. `vm-finsecure-2` continue de traiter les requêtes sans interruption.

<img width="915" height="402" alt="1" src="https://github.com/user-attachments/assets/8926e9e0-55a2-4942-a70b-a295f6540217" />


<img width="918" height="287" alt="2" src="https://github.com/user-attachments/assets/9be52947-5de3-4a69-8cce-894807b2ce15" />


---

## 🟢 Phase 2 — VM Scale Sets + Autoscale

### Objectif

Déployer un VM Scale Set **RHEL 9** capable d'adapter automatiquement sa capacité selon la charge CPU, entre 2 instances minimum et 10 instances maximum. Cette élasticité répond aux pics de charge tout en optimisant les coûts (FinOps) durant les périodes creuses.

### Commandes Azure CLI

```bash
1️⃣ Créer le VM Scale Set (VMSS) RHEL 9

Le VM Scale Set est un groupe de machines virtuelles identiques et gérées ensemble comme une seule unité. Contrairement aux VMs individuelles de la Phase 1, le VMSS permet à Azure d'ajouter ou supprimer automatiquement des instances selon la charge sans intervention manuelle. On utilise ici l'image RHEL 9 ARM64 associée à la taille Standard_B2ps_v2 (architecture Ampere ARM64) , une combinaison cohérente. Le mode Flexible permet une gestion fine des instances. Le VMSS est directement rattaché au Load Balancer et au Backend Pool créés en Phase 1, ce qui lui permet de recevoir immédiatement du trafic dès qu'une instance est prête. On démarre avec 2 instances comme base de départ.

bash
az vmss create \
  --resource-group rg-finsecure-availability \
  --name vmss-finsecure \
  --image RedHat:rhel-arm64:9_8-arm64:latest \
  --vm-sku Standard_B2ps_v2 \
  --instance-count 2 \
  --orchestration-mode Flexible \
  --vnet-name vnet-finsecure \
  --subnet subnet-finsecure \
  --load-balancer lb-finsecure \
  --backend-pool-name backendPool \
  --admin-username polo \
  --generate-ssh-keys

2️⃣ Configurer le moteur d'Autoscale

L'Autoscale est le cerveau du scaling automatique , c'est lui qui surveille en permanence les métriques du VMSS et décide quand ajouter ou retirer des instances. Cette commande crée le profil de base de l'autoscale en définissant trois limites fondamentales : le minimum (2 instances toujours actives pour garantir la haute disponibilité, même en période creuse), le maximum (10 instances pour éviter une explosion des coûts en cas de pic prolongé), et le nombre par défaut (2 instances au démarrage). Sans ce profil, les règles Scale OUT et Scale IN des étapes 3 et 4 n'ont aucun cadre pour s'exécuter.

bash
az monitor autoscale create \
  --resource-group rg-finsecure-availability \
  --resource vmss-finsecure \
  --resource-type Microsoft.Compute/virtualMachineScaleSets \
  --name autoscale-finsecure \
  --min-count 2 \
  --max-count 10 \
  --count 2

3️⃣ Règle Scale OUT — Montée en charge

La règle Scale OUT définit le déclencheur d'ajout automatique d'instances quand la charge augmente. Ici, si le CPU moyen de toutes les instances dépasse 75% pendant 5 minutes consécutives, Azure ajoute automatiquement +2 instances au VMSS. On ajoute 2 instances d'un coup (et non 1) pour absorber rapidement une montée en charge sans attendre un second déclenchement. La fenêtre de 5 minutes évite les faux positifs — un pic CPU de 30 secondes ne déclenche pas inutilement un scaling. C'est cette règle qui protège l'application FinSecure SA contre les surcharges et garantit la continuité de service lors des pics de trafic.

bash
az monitor autoscale rule create \
  --resource-group rg-finsecure-availability \
  --autoscale-name autoscale-finsecure \
  --condition "Percentage CPU > 75 avg 5m" \
  --scale out 2

4️⃣ Règle Scale IN — Réduction de charge

La règle Scale IN définit le déclencheur de suppression automatique d'instances quand la charge diminue. Si le CPU moyen descend sous 25% pendant 5 minutes consécutives, Azure supprime 1 instance du VMSS. On retire intentionnellement 1 seule instance à la fois (prudemment) pour éviter de supprimer trop rapidement et de se retrouver sous-dimensionné si la charge remontait brusquement. Cette règle est essentielle pour optimiser les coûts sans elle, le VMSS resterait à 10 instances même à 3h du matin avec zéro trafic. C'est l'équilibre entre performance (Scale OUT agressif) et économie (Scale IN progressif) qui caractérise une architecture cloud mature.

bash
az monitor autoscale rule create \
  --resource-group rg-finsecure-availability \
  --autoscale-name autoscale-finsecure \
  --condition "Percentage CPU < 25 avg 5m" \
  --scale in 1
```

### Tableau des règles d'autoscale

| Règle | Condition | Action | Objectif |
|---|---|---|---|
| **Scale OUT** | CPU > 75 % (moy. 5 min) | +2 instances RHEL 9 | Absorber la demande |
| **Scale IN** | CPU < 25 % (moy. 5 min) | -1 instance RHEL 9 | Optimisation des coûts (FinOps) |
| **Limite MIN** | — | 2 instances toujours actives | Garantir la disponibilité de base |
| **Limite MAX** | — | 10 instances maximum | Maîtriser les coûts |

<img width="914" height="397" alt="3" src="https://github.com/user-attachments/assets/2d74fac2-ac6d-4be5-9685-d927e630a5ec" />

<img width="918" height="408" alt="4" src="https://github.com/user-attachments/assets/62429490-f4ac-443f-a69b-4ec57a7fa2b4" />

<img width="821" height="410" alt="5" src="https://github.com/user-attachments/assets/a787fff2-4429-4039-81ca-7eeceb8abf52" />


<img width="899" height="395" alt="9" src="https://github.com/user-attachments/assets/f9aa4234-3f24-40a6-86f7-e6673c07c4ce" />

---

## 🟣 Phase 3 — Availability Zones

### Objectif

Distribuer les instances RHEL 9 du Scale Set sur les **3 Availability Zones** de `canadacentral` pour assurer une résilience maximale. Cette configuration garantit que même si un datacenter complet est hors ligne (incendie, panne électrique majeure), le service reste disponible via les deux autres zones sans intervention manuelle.

### Commandes Azure CLI

```bash
# Créer un VMSS RHEL 9 multi-zones (Zone 1, 2 et 3)
az vmss create \
  --resource-group rg-finsecure-availability \
  --name vmss-finsecure-zones \
  --image RedHat:rhel-arm64:9_8-arm64:latest\
  --vm-sku Standard_B2ps_v2 \
  --instance-count 3 \
  --zones 1 2 3 \
  --orchestration-mode Flexible \
  --load-balancer lb-finsecure \
  --backend-pool-name backendPool \
  --admin-username polo \
  --generate-ssh-keys

# Vérifier la répartition par zone
az vmss list-instances \
  --resource-group rg-finsecure-availability \
  --name vmss-finsecure-zones \
  --output table
```

### Résultat attendu

<img width="644" height="170" alt="Zone" src="https://github.com/user-attachments/assets/dd895bcd-9717-469f-9f13-45c35bfe5d4b" />


> ✅ En cas de panne totale de la Zone 2, les Zones 1 et 3 continuent de servir le trafic sans interruption.
>
<img width="687" height="214" alt="Z2" src="https://github.com/user-attachments/assets/cbcbce80-0815-4de2-9296-d1c781b8bc0c" />

<img width="914" height="295" alt="13a" src="https://github.com/user-attachments/assets/d7418b70-6d7b-4f35-a4d3-66de6b09dc25" />

---

## 📊 Supervision et Alerting (KQL)

> **Module de supervision | Azure VM Availability Lab**
> Détection proactive des incidents sur infrastructure RHEL 9 via Azure Monitor, Log Analytics et KQL.

---

## 🎯 Contexte

Dans une infrastructure de services financiers comme **FinSecure SA**, détecter un incident **avant** qu'il n'impacte les utilisateurs finaux est une exigence non négociable. Cette phase met en place une couche de supervision complète reposant sur trois piliers :

- **Azure Monitor Agent (AMA):**  collecte les métriques et logs sur chaque instance RHEL 9
- **Log Analytics Workspace:** centralise et stocke toutes les données pour les rendre interrogeables
- **KQL + Alertes:** analyse les données en temps réel et notifie l'équipe dès qu'un seuil critique est franchi

L'objectif est simple : **zéro incident silencieux**. Chaque anomalie , VM qui ne répond plus, CPU qui s'emballe, scaling qui échoue , doit être détectée, tracée et signalée automatiquement, sans intervention humaine.

---

## 💡 Pourquoi cette approche ?

Une infrastructure haute disponibilité sans supervision est une infrastructure **aveugle**. Les Availability Sets et le VMSS protègent contre les pannes, mais ils ne préviennent pas l'équipe quand quelque chose se passe mal. La surveillance vient compléter la résilience technique par une **intelligence opérationnelle**  elle transforme des données brutes (CPU, heartbeat, événements) en informations actionnables pour l'équipe on-call.

---

## 🏗️ Architecture de supervision

```
Instances RHEL 9 (VMSS + VMs)
        │
        │  métriques + logs (toutes les 60s)
        ▼
Azure Monitor Agent (AMA)
        │
        ▼
┌─────────────────────────────────────────┐
│   Log Analytics Workspace               │
│   law-finsecure / canadacentral         │
│                                         │
│   Tables :                              │
│   ├── Heartbeat   (disponibilité VM)    │
│   ├── Perf        (CPU, mémoire)        │
│   └── AzureActivity (events scaling)   │
└──────────────┬──────────────────────────┘
               │
               ├──► Requêtes KQL ──► Tableaux de bord
               │
               └──► Règles d'alerte
                       │
                       ├── CPU > 80% (5min)
                       │     └──► ✉️  Email équipe infra
                       │
                       └── Heartbeat absent (5min)
                             └──► ✉️  Email + 📱 SMS on-call
```

---

## 📋 Ce que surveille ce lab

| Métrique surveillée | Table KQL | Seuil d'alerte | Sévérité | Réaction |
|---|---|---|---|---|
| **Disponibilité VM** | `Heartbeat` | Absence > 5 min | Severity 1 — Critical | SMS on-call |
| **Charge CPU** | `Perf` | Moyenne > 80% / 5 min | Severity 2 — Warning | Email infra |
| **Événements scaling** | `AzureActivity` | Toute opération VMSS | — | Audit / traçabilité |
| **Mémoire disponible** | `Perf` | Analyse manuelle | — | Tableau de bord |

---

## 🔧 Étape 1 — Log Analytics Workspace

> Le **Log Analytics Workspace** est la base de données centralisée dans laquelle tous les logs et métriques collectés par l'AMA sont stockés et interrogeables via KQL. Sans ce workspace, aucune requête n'est possible et aucune alerte ne peut être configurée ,c'est le socle de toute la chaîne de supervision. Le SKU `PerGB2018` facture uniquement selon le volume de données ingérées, ce qui est idéal pour un lab où les volumes restent faibles.

```bash
az monitor log-analytics workspace create \
  --resource-group rg-finsecure-availability \
  --workspace-name law-finsecure \
  --location canadacentral \
  --sku PerGB2018
```

**Vérification :**

```bash
az monitor log-analytics workspace show \
  --resource-group rg-finsecure-availability \
  --workspace-name law-finsecure \
  --query "{Nom:name, ID:customerId, Statut:provisioningState}" \
  --output table
```
<img width="529" height="145" alt="13a" src="https://github.com/user-attachments/assets/4241a9b2-c375-44b3-9a47-e27d7953244a" />

<img width="917" height="374" alt="13b" src="https://github.com/user-attachments/assets/243e5dcc-f637-45a5-9a37-ea1b40e59c52" />


---

## 📋 Étape 2 — Requêtes KQL
---

**Étape 1 — Installer l'Azure Monitor Agent sur les VMs**

**Sur vm-finsecure-1**
```bash
az vm extension set \
  --resource-group rg-finsecure-availability \
  --vm-name vm-finsecure-1 \
  --name AzureMonitorLinuxAgent \
  --publisher Microsoft.Azure.Monitor \
  --version 1.0 \
  --enable-auto-upgrade true
```
**Sur vm-finsecure-2**
```bash
az vm extension set \
  --resource-group rg-finsecure-availability \
  --vm-name vm-finsecure-2 \
  --name AzureMonitorLinuxAgent \
  --publisher Microsoft.Azure.Monitor \
  --version 1.0 \
  --enable-auto-upgrade true
```
**Installer AMA sur toutes les instances du VMSS**
az vmss extension set \
  --resource-group rg-finsecure-availability \
  --vmss-name vmss-finsecure-zones \
  --name AzureMonitorLinuxAgent \
  --publisher Microsoft.Azure.Monitor \
  --version 1.0 \
  --enable-auto-upgrade true
 
**Étape 2 — Lier les VMs au Log Analytics Workspace**

**Récupérer l'ID du workspace**
```bash
LAW_ID=$(az monitor log-analytics workspace show \
  --resource-group rg-finsecure-availability \
  --workspace-name law-finsecure \
  --query id --output tsv)
```
**Lier vm-finsecure-1**
```bash
az monitor data-collection rule association create \
  --name dcra-vm1-finsecure \
  --resource /subscriptions/$(az account show --query id -o tsv)/resourceGroups/rg-finsecure-availability/providers/Microsoft.Compute/virtualMachines/vm-finsecure-1 \
  --rule-id $LAW_ID
```
**Lier vm-finsecure-2**
```bash
az monitor data-collection rule association create \
  --name dcra-vm2-finsecure \
  --resource /subscriptions/$(az account show --query id -o tsv)/resourceGroups/rg-finsecure-availability/providers/Microsoft.Compute/virtualMachines/vm-finsecure-2 \
  --rule-id $LAW_ID
```


 **Étape 3 — Attendre 10-15 minutes**
 
L'AMA a besoin de temps pour démarrer et envoyer les premiers heartbeats au workspace.

# Étape 4 — Exécuter la requête KQL dans le portail


# Portail Azure
→ law-finsecure
→ Menu gauche : Logs
→ Coller la requête KQL
→ Cliquer Run

---

### 🔍 Requête 1 — Disponibilité des VMs RHEL 9 (Heartbeat)

> La table **Heartbeat** enregistre un signal toutes les 60 secondes pour chaque VM supervisée par l'AMA. Cette requête récupère le **dernier heartbeat de chaque VM sur les 24 dernières heures** et détermine si elle est actuellement en ligne ou hors ligne. La logique est simple : si le dernier signal date de moins de 5 minutes → `✅ En ligne` ; au-delà → `❌ Hors ligne`. C'est la requête de **tableau de bord de santé** ,elle donne en un coup d'œil l'état de toute l'infrastructure FinSecure SA.

```kql
Heartbeat
| where TimeGenerated > ago(24h)
| summarize LastHeartbeat = max(TimeGenerated) by Computer
| extend IsAvailable = iff(LastHeartbeat > ago(5m), "✅ En ligne", "❌ Hors ligne")
| project Computer, LastHeartbeat, IsAvailable
| sort by LastHeartbeat desc
```

**Résultat attendu :**
<img width="917" height="407" alt="14d" src="https://github.com/user-attachments/assets/80dcd65b-5acd-40cf-8ef2-2e63b2f2e354" />



---

### 🔍 Requête 2 — Alerte Heartbeat manquant (VM hors ligne)

> Cette requête est conçue pour être utilisée comme **condition d'une alerte Azure Monitor**. Elle cherche uniquement les VMs dont le dernier heartbeat remonte à plus de 5 minutes , ce qui signifie qu'elles ne répondent plus. Contrairement à la Requête 1 qui donne une vue globale, celle-ci retourne un résultat **uniquement s'il y a un problème**. Si elle retourne des lignes, l'alerte se déclenche et envoie immédiatement une notification à l'équipe on-call. C'est la sentinelle silencieuse du lab ,elle ne parle que quand quelque chose va mal.

```kql
Heartbeat
| where TimeGenerated > ago(5m)
| summarize LastHeartbeat = max(TimeGenerated) by Computer
| where LastHeartbeat < ago(5m)
// Alerte déclenchée si aucune réponse depuis plus de 5 minutes
```


<img width="921" height="402" alt="14a" src="https://github.com/user-attachments/assets/b551c9c9-084c-48e5-a6bf-a831f7ebbdbc" />

---

### 🔍 Requête 3 — Analyse CPU par instance RHEL 9

> Cette requête interroge la table **Perf** qui collecte les compteurs de performance système via l'AMA. Elle filtre sur le compteur `% Processor Time`, calcule la **moyenne et le pic CPU** par VM sur des fenêtres de 5 minutes, et ne remonte que les instances dont la moyenne dépasse **70%**. C'est la requête d'**investigation de performance** , elle identifie quelle instance RHEL 9 est sous pression, depuis combien de temps, et à quel niveau de pic elle a atteint. Couplée aux règles d'autoscale, elle permet de valider que le Scale OUT s'est bien déclenché au bon moment.

```kql
Perf
| where TimeGenerated > ago(1h)
| where ObjectName == "Processor" and CounterName == "% Processor Time"
| summarize AvgCPU = avg(CounterValue), MaxCPU = max(CounterValue) by Computer, bin(TimeGenerated, 5m)
| where AvgCPU > 70
| project TimeGenerated, Computer, AvgCPU = round(AvgCPU, 2), MaxCPU = round(MaxCPU, 2)
| sort by TimeGenerated desc
```

<img width="806" height="386" alt="df" src="https://github.com/user-attachments/assets/fb23257f-7bde-476d-a2e1-ab19fad6b56d" />


---

### 🔍 Requête 4 — Événements de scaling VMSS

> La table **AzureActivity** enregistre toutes les opérations effectuées sur les ressources Azure via le plan de contrôle. Cette requête filtre les opérations d'écriture sur le VMSS sur les **7 derniers jours**, ce qui correspond exactement aux événements Scale OUT et Scale IN déclenchés par les règles d'autoscale. Elle retourne le timestamp, le type d'opération, son statut (Succeeded/Failed) et l'identité du déclencheur (autoscale engine ou action manuelle). C'est le **journal d'audit du scaling**  il prouve que l'infrastructure s'est bien adaptée automatiquement à la charge.

```kql
AzureActivity
| where TimeGenerated > ago(7d)
| where ResourceGroup == "rg-finsecure-availability"
| where OperationNameValue has "Microsoft.Compute/virtualMachineScaleSets/write"
| project TimeGenerated, OperationNameValue, ActivityStatusValue, Caller
| sort by TimeGenerated desc
```



---

### 🔍 Requête 5 — Mémoire disponible par instance

> Cette requête surveille la **mémoire disponible** sur chaque instance RHEL 9 via le compteur `Available MBytes`. Elle trie les résultats par ordre croissant pour identifier immédiatement les instances les plus proches de la saturation mémoire. Complémentaire à l'analyse CPU, elle offre une vision complète de la santé des instances et permet d'anticiper les problèmes de mémoire avant qu'ils n'affectent les performances applicatives.

```kql
Perf
| where TimeGenerated > ago(1h)
| where ObjectName == "Memory" and CounterName == "Available MBytes"
| summarize AvgMemMB = avg(CounterValue) by Computer, bin(TimeGenerated, 5m)
| project TimeGenerated, Computer, AvgMemMB = round(AvgMemMB, 0)
| sort by AvgMemMB asc
```



---

## 🔔 Étape 3 — Action Group

> L'**Action Group** est le groupe de destinataires et d'actions à déclencher quand une alerte se produit. Il faut impérativement le créer **avant** les règles d'alerte car celles-ci y font référence. Un Action Group peut contenir plusieurs canaux de notification : email, SMS, webhook, Azure Function, ITSM. Ici, on configure un email vers l'équipe infra FinSecure SA.

```bash
az monitor action-group create \
  --resource-group rg-finsecure-availability \
  --name ag-finsecure-infra \
  --short-name finsec \
  --action email infra-alert admin@finsecure.bj
```

**Vérification :**

```bash
az monitor action-group show \
  --resource-group rg-finsecure-availability \
  --name ag-finsecure-infra \
  --query "{Nom:name, Email:emailReceivers[0].emailAddress}" \
  --output table
```

<img width="912" height="134" alt="14b" src="https://github.com/user-attachments/assets/a739ab95-c062-44b7-a9ef-8ef2c0ee69fb" />


<img width="916" height="386" alt="14c" src="https://github.com/user-attachments/assets/e1947549-5ebe-40d8-9b72-cbc99d847290" />


---

## 🚨 Étape 4 — Alerte CPU élevé (Severity 2 — Warning)

> Cette alerte surveille le **CPU moyen du VMSS** toutes les minutes sur une fenêtre de 5 minutes. Si le seuil de **80%** est dépassé de manière soutenue, l'Action Group est déclenché et un email est envoyé immédiatement à l'équipe infra. La sévérité 2 (Warning) indique une situation nécessitant attention sans être critique , l'autoscale peut encore absorber la charge, mais l'équipe doit être informée. On récupère d'abord l'ID du VMSS dynamiquement pour éviter toute erreur de saisie manuelle.

```bash
# Récupérer et afficher l'ID d'abord
az vmss show \
  --resource-group rg-finsecure-availability \
  --name vmss-finsecure-zones \
  --query id \
  --output tsv

# Créer l'alerte CPU
az monitor metrics alert create \
  --resource-group rg-finsecure-availability \
  --name alert-cpu-high-finsecure \
  --scopes "/subscriptions/ID/resourceGroups/rg-finsecure-availability/providers/Microsoft.Compute/virtualMachineScaleSets/vmss-finsecure-zones" \
  --condition "avg Percentage CPU > 80" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --severity 2 \
  --description "CPU VMSS FinSecure > 80% depuis 5 min" \
  --action ag-finsecure-infra
```

**Paramètres de l'alerte :**

| Paramètre | Valeur | Explication |
|---|---|---|
| `--condition` | `avg CPU > 80` | Moyenne CPU supérieure à 80% |
| `--window-size` | `5m` | Calcul sur une fenêtre de 5 minutes |
| `--evaluation-frequency` | `1m` | Vérification toutes les minutes |
| `--severity` | `2` | Warning — attention requise |
| `--action` | `ag-finsecure-infra` | Déclenche l'email à l'équipe infra |

<img width="914" height="409" alt="81" src="https://github.com/user-attachments/assets/2f6b5479-e0c3-4fe6-be8a-67d096470322" />


---

## 🚨 Étape 5 — Alerte VM hors ligne (Severity 1 — Critical)

> Cette alerte de type **Log Alert** exécute la requête KQL Heartbeat toutes les 5 minutes. Si une VM RHEL 9 ne répond plus depuis plus de 5 minutes, l'alerte se déclenche au niveau **Severity 1 — Critical**, le plus urgent dans Azure Monitor. Elle notifie l'équipe on-call par email et SMS pour une intervention immédiate. On récupère l'ID du workspace Log Analytics dynamiquement avant de créer la règle.


**Récupérer l'ID du workspace**

```bash
LAW_ID=$(az monitor log-analytics workspace show \
  --resource-group rg-finsecure-availability \
  --workspace-name law-finsecure \
  --query id \
  --output tsv)
```

**Récupérer l'ID de l'Action Group**
```bash
AG_ID=$(az monitor action-group show \
  --resource-group rg-finsecure-availability \
  --name ag-finsecure-infra \
  --query id \
  --output tsv)
```
**Créer l'alerte Heartbeat**
```bash
az monitor scheduled-query create \
  --resource-group rg-finsecure-availability \
  --name "alert-vm-down-finsecure" \
  --scopes "$LAW_ID" \
  --condition-query "Heartbeat | where TimeGenerated > ago(5m) | summarize LastHeartbeat = max(TimeGenerated) by Computer | where LastHeartbeat < ago(5m)" \
  --condition-time-aggregation "Count" \
  --condition-operator "GreaterThan" \
  --condition-threshold 0 \
  --evaluation-frequency "PT5M" \
  --window-size "PT5M" \
  --severity 1 \
  --description "VM RHEL 9 FinSecure hors ligne depuis plus de 5 min" \
  --action-groups "$AG_ID" \
  --location canadacentral
```

**Paramètres de l'alerte :**

| Paramètre | Valeur | Explication |
|---|---|---|
| `--condition-query` | Requête KQL Heartbeat | Détecte les VMs silencieuses |
| `--condition-threshold` | `0` | Se déclenche dès la première VM hors ligne |
| `--evaluation-frequency` | `5m` | Vérifie toutes les 5 minutes |
| `--severity` | `1` | Critical-intervention immédiate requise |
| `--action-groups` | `ag-finsecure-infra` | Email + SMS on-call |

<img width="911" height="395" alt="82" src="https://github.com/user-attachments/assets/d2031efd-79b9-47c5-88b5-72ca9354214d" />


<img width="731" height="384" alt="83" src="https://github.com/user-attachments/assets/0bbaf2d9-cdd2-4ac7-b5f6-bacab835580f" />


---

## ✅ Résultat final

### Tableau récapitulatif des alertes

| Alerte | Type | Condition | Sévérité | Action |
|---|---|---|---|---|
| `alert-cpu-high-finsecure` | Métrique | CPU > 80% / 5 min | ⚠️ Severity 2 — Warning | Email équipe infra |
| `alert-vm-down-finsecure` | Log (KQL) | Heartbeat absent > 5 min | 🔴 Severity 1 — Critical | Email + SMS on-call |

### Tableau récapitulatif des requêtes KQL

| # | Requête | Table | Usage |
|---|---|---|---|
| 1 | Disponibilité des VMs | `Heartbeat` | Tableau de bord de santé |
| 2 | VM hors ligne | `Heartbeat` | Condition d'alerte Monitor |
| 3 | Analyse CPU | `Perf` | Investigation performance |
| 4 | Événements scaling | `AzureActivity` | Journal d'audit VMSS |
| 5 | Mémoire disponible | `Perf` | Surveillance mémoire |

### Chaîne complète de supervision

```
VM RHEL 9 silencieuse depuis > 5 min
        │
        ▼
Requête KQL Heartbeat s'exécute (toutes les 5 min)
        │
        ▼
Résultat > 0 lignes → Condition vraie
        │
        ▼
Azure Monitor déclenche alert-vm-down-finsecure
        │
        ▼
Action Group ag-finsecure-infra notifié
        │
        ├──► ✉️  Email → admin@finsecure.bj
        └──► 📱 SMS → Équipe on-call
```



## 💡 Compétences démontrées

| # | Compétence |
|---|---|
| 1 | **Azure Virtual Machines**  Déploiement RHEL 9, configuration et haute disponibilité |
| 2 | **Azure Load Balancer**  Distribution de charge L4, Health Probes, Backend Pools |
| 3 | **VM Scale Sets & Autoscale**  Scaling horizontal automatique sur instances RHEL 9 |
| 4 | **Azure Monitor & Log Analytics**  AMA, KQL, alertes métriques et logs |
| 5 | **Résilience d'infrastructure** Availability Sets, Zones, Fault/Update Domains |
| 6 | **Linux Administration**  RHEL 9 en environnement cloud Azure (cohérent avec RHCSA) |
| 7 | **Gouvernance & Conformité**  Alignement ISO 27001, traçabilité, moindre privilège |

---

## 👤 Auteur

**Serge TOGNON**
*Cloud & Linux System Administrator | AZ-104 Certified | Candidat RHCSA (EX200)*

---

> *Ce projet démontre ma capacité à concevoir des environnements cloud robustes, sécurisés et optimisés en coûts (FinOps), avec une maîtrise des systèmes Linux Red Hat en environnement Azure.*
>
> *Environnement : Azure Pay-As-You-Go | Région : canadacentral | OS : RHEL 9*
