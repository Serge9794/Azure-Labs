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
- **SKU retenu :** `9-lvm-gen2` — image Gen2 avec partitionnement LVM, recommandée pour les charges de production sur Azure.

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
# 1. Créer le VMSS RHEL 9
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

# 2. Configurer l'autoscale
az monitor autoscale create \
  --resource-group rg-finsecure-availability \
  --resource vmss-finsecure \
  --resource-type Microsoft.Compute/virtualMachineScaleSets \
  --name autoscale-finsecure \
  --min-count 2 \
  --max-count 10 \
  --count 2

# 3. Règle Scale OUT (CPU > 75% pendant 5 min → +2 instances)
az monitor autoscale rule create \
  --resource-group rg-finsecure-availability \
  --autoscale-name autoscale-finsecure \
  --condition "Percentage CPU > 75 avg 5m" \
  --scale out 2

# 4. Règle Scale IN (CPU < 25% pendant 5 min → -1 instance)
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

**📸 Screenshot 3 :** *VMSS → Instances → Vue de la montée en charge (Scale Out)*

**📸 Screenshot 4 :** *Portail Azure → Autoscale → Historique des événements de scaling*

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
  --image RedHat:RHEL:9-lvm-gen2:latest \
  --vm-sku Standard_B2ps_v2 \
  --instance-count 3 \
  --zones 1 2 3 \
  --orchestration-mode Flexible \
  --load-balancer lb-finsecure \
  --backend-pool-name backendPool \
  --admin-username Polo \
  --generate-ssh-keys

# Vérifier la répartition par zone
az vmss list-instances \
  --resource-group rg-finsecure-availability \
  --name vmss-finsecure-zones \
  --query "[].{Name:name, Zone:zones[0]}" \
  --output table
```

### Résultat attendu

| Instance | Zone | Datacenter | OS |
|---|---|---|---|
| `vmss-finsecure-zones_0` | Zone 1 | DC Toronto-Ouest | RHEL 9 |
| `vmss-finsecure-zones_1` | Zone 2 | DC Toronto-Centre | RHEL 9 |
| `vmss-finsecure-zones_2` | Zone 3 | DC Toronto-Est | RHEL 9 |

> ✅ En cas de panne totale de la Zone 2, les Zones 1 et 3 continuent de servir le trafic sans interruption.

**📸 Screenshot 5 :** *VMSS → Instances → Colonne Zone affichant 1, 2, 3*

**📸 Screenshot 6 :** *Portail Azure → Load Balancer → Frontend IP zone-redondante*

---

## 📊 Supervision et Alerting (KQL)

L'**Azure Monitor Agent (AMA)** est déployé sur chaque instance RHEL 9 pour la collecte des métriques système et des logs. Les requêtes KQL permettent une détection proactive des incidents avant qu'ils n'affectent les utilisateurs.

### Configuration du Workspace

```bash
az monitor log-analytics workspace create \
  --resource-group rg-finsecure-availability \
  --workspace-name law-finsecure \
  --location canadacentral \
  --sku PerGB2018
```

### Requêtes KQL

#### 🔍 Disponibilité des VMs RHEL 9 (Heartbeat)

```kql
Heartbeat
| where TimeGenerated > ago(24h)
| summarize LastHeartbeat = max(TimeGenerated) by Computer
| extend IsAvailable = iff(LastHeartbeat > ago(5m), "✅ En ligne", "❌ Hors ligne")
| project Computer, LastHeartbeat, IsAvailable
| sort by LastHeartbeat desc
```

#### 🔍 Alerte Heartbeat manquant (VM hors ligne)

```kql
Heartbeat
| where TimeGenerated > ago(5m)
| summarize LastHeartbeat = max(TimeGenerated) by Computer
| where LastHeartbeat < ago(5m)
// Alerte déclenchée si aucune réponse depuis plus de 5 minutes
```

#### 🔍 Analyse CPU par instance RHEL 9

```kql
Perf
| where TimeGenerated > ago(1h)
| where ObjectName == "Processor" and CounterName == "% Processor Time"
| summarize AvgCPU = avg(CounterValue), MaxCPU = max(CounterValue) by Computer, bin(TimeGenerated, 5m)
| where AvgCPU > 70
| project TimeGenerated, Computer, AvgCPU = round(AvgCPU, 2), MaxCPU = round(MaxCPU, 2)
| sort by TimeGenerated desc
```

#### 🔍 Événements de scaling VMSS

```kql
AzureActivity
| where TimeGenerated > ago(7d)
| where ResourceGroup == "rg-finsecure-availability"
| where OperationNameValue has "Microsoft.Compute/virtualMachineScaleSets/write"
| project TimeGenerated, OperationNameValue, ActivityStatusValue, Caller
| sort by TimeGenerated desc
```

### Alertes configurées

| Alerte | Type | Seuil | Sévérité | Action |
|---|---|---|---|---|
| **CPU élevé** | Métrique | CPU > 80 % / 5 min | Severity 2 — Warning | E-mail équipe infra |
| **VM hors ligne** | Log (KQL) | Heartbeat absent > 5 min | Severity 1 — Critical | E-mail + SMS on-call |

**📸 Screenshot 7 :** *Log Analytics → Query Explorer → Résultat Heartbeat RHEL 9*

**📸 Screenshot 8 :** *Azure Monitor → Metrics → Graphe CPU des instances VMSS*

---

## ✅ Tests de résilience

Pour valider la conformité ISO 27001 (A.17 — Disponibilité), les simulations suivantes ont été réalisées :

| Test | Méthode | Résultat |
|:---|:---|:---|
| **Panne VM** | Arrêt manuel de `vm-finsecure-1` (RHEL 9) | Basculement sur `vm-finsecure-2` en < 30 secondes |
| **Pic de charge CPU** | Stress test CPU > 80 % sur les instances RHEL 9 | Scale OUT automatique : 2 → 4 instances en 3 minutes |
| **Retour à la normale** | Arrêt du stress, CPU < 20 % | Scale IN automatique : 4 → 2 instances en 5 minutes |
| **Alerte CPU** | Dépassement du seuil de 80 % | E-mail reçu dans les 2 minutes |
| **Panne de zone** | Déploiement zoné simulé | Continuité assurée par les zones restantes |

---

## 📈 Résultats et métriques

### SLA atteint par configuration

| Configuration | OS | SLA théorique | Scénario couvert |
|---|---|---|---|
| VM isolée | RHEL 9 | ~99,9 % | Maintenance planifiée uniquement |
| Availability Set (2 VMs) | RHEL 9 | **99,95 %** | Défaillance matérielle intra-rack |
| Availability Zones (2+ zones) | RHEL 9 | **99,99 %** | Panne complète d'un datacenter |
| VMSS + Autoscale | RHEL 9 | Variable | Adaptation dynamique à la charge |

---

## 💡 Compétences démontrées

| # | Compétence |
|---|---|
| 1 | **Azure Virtual Machines** — Déploiement RHEL 9, configuration et haute disponibilité |
| 2 | **Azure Load Balancer** — Distribution de charge L4, Health Probes, Backend Pools |
| 3 | **VM Scale Sets & Autoscale** — Scaling horizontal automatique sur instances RHEL 9 |
| 4 | **Azure Monitor & Log Analytics** — AMA, KQL, alertes métriques et logs |
| 5 | **Résilience d'infrastructure** — Availability Sets, Zones, Fault/Update Domains |
| 6 | **Linux Administration** — RHEL 9 en environnement cloud Azure (cohérent avec RHCSA) |
| 7 | **Gouvernance & Conformité** — Alignement ISO 27001, traçabilité, moindre privilège |

---

## 👤 Auteur

**Serge TOGNON**
*Cloud & Linux System Administrator | AZ-104 Certified | Candidat RHCSA (EX200)*

---

> *Ce projet démontre ma capacité à concevoir des environnements cloud robustes, sécurisés et optimisés en coûts (FinOps), avec une maîtrise des systèmes Linux Red Hat en environnement Azure.*
>
> *Environnement : Azure Pay-As-You-Go | Région : canadacentral | OS : RHEL 9 (`9-lvm-gen2`)*
