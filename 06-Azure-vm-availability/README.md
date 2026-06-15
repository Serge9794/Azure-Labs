# 🏛️ Azure VM Availability Lab — FinSecure SA

> **Projet Portfolio Azure | AZ-104**
> Mise en œuvre complète de la haute disponibilité et du scaling automatique des machines virtuelles Azure dans un contexte d'entreprise financière.

---

![Azure](https://img.shields.io/badge/Azure-VM%20Availability-0078D4?style=for-the-badge&logo=microsoftazure&logoColor=white)
![AZ-104](https://img.shields.io/badge/Certification-AZ--104-0078D4?style=for-the-badge&logo=microsoft&logoColor=white)
![Status](https://img.shields.io/badge/Statut-Complété-28a745?style=for-the-badge)
![Lab](https://img.shields.io/badge/Type-Hands--on%20Lab-orange?style=for-the-badge)

---

## 📋 Table des matières

- [Contexte du projet](#-contexte-du-projet)
- [Objectifs techniques](#-objectifs-techniques)
- [Architecture déployée](#-architecture-déployée)
- [Technologies et services](#-technologies-et-services)
- [Phase 1 — Availability Sets + Load Balancer](#-phase-1--availability-sets--load-balancer)
- [Phase 2 — VM Scale Sets + Autoscale](#-phase-2--vm-scale-sets--autoscale)
- [Phase 3 — Availability Zones](#-phase-3--availability-zones)
- [Supervision — Azure Monitor + KQL](#-supervision--azure-monitor--kql)
- [Alertes configurées](#-alertes-configurées)
- [Résultats et métriques](#-résultats-et-métriques)
- [Compétences démontrées](#-compétences-démontrées)
- [Auteur](#-auteur)

---

## 🏢 Contexte du projet

**FinSecure SA** est une société de services financiers basée à Cotonou, Bénin, dont l'infrastructure héberge une **application web critique**  devant respecter un SLA de **99,99 % de disponibilité**.

Suite à une interruption de service non planifiée ayant entraîné une indisponibilité de 4 heures, la direction technique a mandaté la mise en place d'une architecture **hautement disponible, résiliente et scalable** sur Azure.

Ce lab documente l'ensemble du déploiement : Availability Sets, VM Scale Sets avec Autoscale, et Availability Zones, ainsi que la supervision via Azure Monitor.

---

## 🎯 Objectifs techniques

- [x] Déployer un **Availability Set** avec Update Domains et Fault Domains pour la redondance intra-datacenter
- [x] Configurer un **Azure Load Balancer** distribué sur les VMs du set
- [x] Déployer un **VM Scale Set (VMSS)** en mode Flexible avec règles d'autoscaling CPU
- [x] Distribuer les VMs sur plusieurs **Availability Zones** (Zone 1, 2, 3)
- [x] Activer **Azure Monitor Agent** et configurer des **règles d'alerte**
- [x] Rédiger des requêtes **KQL** pour analyser les métriques de disponibilité
- [x] Valider la résilience par simulation de panne

---

## 🏗️ Architecture déployée

```
                        ┌─────────────────────────────────────────────────────┐
                        │              Azure Region : francecentral            │
                        │                                                     │
                        │  ┌─────────────┐  ┌─────────────┐  ┌────────────┐  │
                        │  │   Zone 1    │  │   Zone 2    │  │   Zone 3   │  │
                        │  │             │  │             │  │            │  │
                        │  │ ┌─────────┐ │  │ ┌─────────┐ │  │┌─────────┐ │  │
                        │  │ │ VM-AS-1 │ │  │ │ VM-AS-2 │ │  ││ VMSS    │ │  │
                        │  │ │(FD:0,   │ │  │ │(FD:1,   │ │  ││Instances│ │  │
                        │  │ │ UD:0)   │ │  │ │ UD:1)   │ │  ││(2→10)   │ │  │
                        │  │ └────┬────┘ │  │ └────┬────┘ │  │└────┬────┘ │  │
                        │  └──────┼──────┘  └──────┼──────┘  └─────┼──────┘  │
                        │         │                 │               │         │
                        │         └────────┬────────┘               │         │
                        │                  │                        │         │
                        │         ┌────────▼────────┐               │         │
                        │         │  Azure Load     │               │         │
                        │         │  Balancer (L4)  │◄─────────────┘         │
                        │         │  (Public IP)    │                         │
                        │         └────────┬────────┘                         │
                        │                  │                                   │
                        │         ┌────────▼────────┐                          │
                        │         │  Azure Monitor  │                          │
                        │         │  + Log Analytics│                          │
                        │         │  + Alertes      │                          │
                        │         └─────────────────┘                          │
                        └─────────────────────────────────────────────────────┘
                                           │
                                    Internet / Clients
                                    FinSecure SA
```

---

## 🛠️ Technologies et services

| Service Azure | Rôle dans le projet |
|---|---|
| **Azure Virtual Machines** | Compute — VMs Windows Server 2022 |
| **Availability Sets** | Redondance intra-datacenter (FD + UD) |
| **Availability Zones** | Redondance inter-datacenter (Zone 1, 2, 3) |
| **VM Scale Sets (VMSS)** | Scaling horizontal automatique |
| **Azure Load Balancer (L4)** | Distribution du trafic entre VMs |
| **Azure Monitor** | Supervision des métriques et logs |
| **Log Analytics Workspace** | Centralisation des données de monitoring |
| **Azure Monitor Agent (AMA)** | Collecte des métriques depuis les VMs |
| **Action Groups** | Notification par email lors des alertes |
| **Azure CLI** | Provisionnement et automatisation |

**Région :** `francecentral`
**Resource Group :** `rg-finsecure-availability`
**Abonnement :** Azure Pay-As-You-Go (actif)

---

## 🔵 Phase 1 — Availability Sets + Load Balancer

### Objectif
Déployer deux VMs dans un **Availability Set** avec 2 Fault Domains et 5 Update Domains, puis distribuer le trafic via un **Azure Load Balancer**.

### Commandes Azure CLI

```bash
# 1. Créer le groupe de ressources
az group create \
  --name rg-finsecure-availability \
  --location francecentral

# 2. Créer l'Availability Set
az vm availability-set create \
  --resource-group rg-finsecure-availability \
  --name avset-finsecure \
  --location francecentral \
  --platform-fault-domain-count 2 \
  --platform-update-domain-count 5

# 3. Créer les VMs dans l'Availability Set
for i in 1 2; do
  az vm create \
    --resource-group rg-finsecure-availability \
    --name vm-finsecure-$i \
    --availability-set avset-finsecure \
    --image Win2022AzureEditionCore \
    --size Standard_B2s \
    --admin-username Polo \
    --admin-password "FinSecure@2025!" \
    --vnet-name vnet-finsecure \
    --subnet subnet-finsecure \
    --public-ip-address ""
done

# 4. Créer le Load Balancer
az network lb create \
  --resource-group rg-finsecure-availability \
  --name lb-finsecure \
  --sku Standard \
  --frontend-ip-name frontendIP \
  --backend-pool-name backendPool

# 5. Créer la règle de load balancing (port 80)
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

# 6. Créer le Health Probe
az network lb probe create \
  --resource-group rg-finsecure-availability \
  --lb-name lb-finsecure \
  --name healthProbe \
  --protocol tcp \
  --port 80
```

### Résultat attendu

- VM `vm-finsecure-1` → Fault Domain 0, Update Domain 0
- VM `vm-finsecure-2` → Fault Domain 1, Update Domain 1
- Si FD 0 tombe, seule `vm-finsecure-1` est affectée → `vm-finsecure-2` continue de servir les requêtes

**📸 Screenshot 1 :** *Vue du portail Azure → Availability Set → Répartition FD/UD des VMs*

**📸 Screenshot 2 :** *Load Balancer → Backend Pool → Membres actifs et état de santé*

---

## 🟢 Phase 2 — VM Scale Sets + Autoscale

### Objectif
Déployer un **VM Scale Set** capable de s'adapter automatiquement à la charge CPU, de 2 instances minimum à 10 instances maximum.

### Commandes Azure CLI

```bash
# 1. Créer le VMSS
az vmss create \
  --resource-group rg-finsecure-availability \
  --name vmss-finsecure \
  --image Ubuntu2204 \
  --vm-sku Standard_B2s \
  --instance-count 2 \
  --orchestration-mode Flexible \
  --vnet-name vnet-finsecure \
  --subnet subnet-finsecure \
  --load-balancer lb-finsecure \
  --backend-pool-name backendPool \
  --admin-username azureadmin \
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

| Règle | Condition | Action | Cooldown |
|---|---|---|---|
| **Scale OUT** | CPU > 75% (moy. 5 min) | +2 instances | 5 min |
| **Scale IN** | CPU < 25% (moy. 5 min) | -1 instance | 5 min |
| **Limite MIN** | — | 2 instances toujours actives | — |
| **Limite MAX** | — | 10 instances maximum | — |

**📸 Screenshot 3 :** *VMSS → Instances → Vue de la montée en charge (Scale Out)*

**📸 Screenshot 4 :** *Portail Azure → Autoscale → Historique des événements de scaling*

---

## 🟣 Phase 3 — Availability Zones

### Objectif
Distribuer les VMs du Scale Set sur les **3 Availability Zones** de `francecentral` pour une résilience maximale contre la panne d'un datacenter entier.

### Commandes Azure CLI

```bash
# Créer un VMSS multi-zones (Zone 1, 2 et 3)
az vmss create \
  --resource-group rg-finsecure-availability \
  --name vmss-finsecure-zones \
  --image Ubuntu2204 \
  --vm-sku Standard_B2s \
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

| Instance | Zone | Datacenter |
|---|---|---|
| `vmss-finsecure-zones_0` | Zone 1 | DC Paris-Ouest |
| `vmss-finsecure-zones_1` | Zone 2 | DC Paris-Centre |
| `vmss-finsecure-zones_2` | Zone 3 | DC Paris-Est |

> ✅ Si Zone 2 tombe en panne totale → Zone 1 et Zone 3 continuent de servir le trafic sans interruption.

**📸 Screenshot 5 :** *VMSS → Instances → Colonne Zone affichant 1, 2, 3*

**📸 Screenshot 6 :** *Portail Azure → Load Balancer → Zone-redundant Frontend IP*

---

## 📊 Supervision — Azure Monitor + KQL

### Configuration du Workspace

```bash
# Créer le Log Analytics Workspace
az monitor log-analytics workspace create \
  --resource-group rg-finsecure-availability \
  --workspace-name law-finsecure \
  --location francecentral \
  --sku PerGB2018
```

### Requêtes KQL

#### 🔍 Requête 1 — Disponibilité des VMs (dernières 24h)

```kql
Heartbeat
| where TimeGenerated > ago(24h)
| summarize LastHeartbeat = max(TimeGenerated) by Computer
| extend IsAvailable = iff(LastHeartbeat > ago(5m), "✅ En ligne", "❌ Hors ligne")
| project Computer, LastHeartbeat, IsAvailable
| sort by LastHeartbeat desc
```

#### 🔍 Requête 2 — Analyse CPU par VM (pic de charge)

```kql
Perf
| where TimeGenerated > ago(1h)
| where ObjectName == "Processor" and CounterName == "% Processor Time"
| summarize AvgCPU = avg(CounterValue), MaxCPU = max(CounterValue) by Computer, bin(TimeGenerated, 5m)
| where AvgCPU > 70
| project TimeGenerated, Computer, AvgCPU = round(AvgCPU, 2), MaxCPU = round(MaxCPU, 2)
| sort by TimeGenerated desc
```

#### 🔍 Requête 3 — Événements de scaling VMSS

```kql
AzureActivity
| where TimeGenerated > ago(7d)
| where ResourceGroup == "rg-finsecure-availability"
| where OperationNameValue has "Microsoft.Compute/virtualMachineScaleSets/write"
| project TimeGenerated, OperationNameValue, ActivityStatusValue, Caller
| sort by TimeGenerated desc
```

#### 🔍 Requête 4 — Mémoire disponible par VM

```kql
Perf
| where TimeGenerated > ago(1h)
| where ObjectName == "Memory" and CounterName == "Available MBytes"
| summarize AvgMemMB = avg(CounterValue) by Computer, bin(TimeGenerated, 5m)
| project TimeGenerated, Computer, AvgMemMB = round(AvgMemMB, 0)
| sort by AvgMemMB asc
```

**📸 Screenshot 7 :** *Log Analytics → Query Explorer → Résultat de la requête Heartbeat*

**📸 Screenshot 8 :** *Azure Monitor → Metrics → Graphe CPU des instances VMSS*

---

## 🚨 Alertes configurées

### Alert 1 — CPU élevé sur VMSS

| Paramètre | Valeur |
|---|---|
| **Nom** | alert-cpu-high-finsecure |
| **Ressource cible** | vmss-finsecure |
| **Métrique** | Percentage CPU |
| **Seuil** | > 80% (moy. 5 min) |
| **Sévérité** | Severity 2 — Warning |
| **Action** | Email à l'équipe infra |

```bash
# Créer l'Action Group
az monitor action-group create \
  --resource-group rg-finsecure-availability \
  --name ag-finsecure-infra \
  --short-name finsec \
  --action email infra-alert admin@finsecure.bj

# Créer la règle d'alerte CPU
az monitor metrics alert create \
  --resource-group rg-finsecure-availability \
  --name alert-cpu-high-finsecure \
  --scopes /subscriptions/<SUB_ID>/resourceGroups/rg-finsecure-availability/providers/Microsoft.Compute/virtualMachineScaleSets/vmss-finsecure \
  --condition "avg Percentage CPU > 80" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --severity 2 \
  --action ag-finsecure-infra
```

### Alert 2 — VM non disponible (Heartbeat manquant)

| Paramètre | Valeur |
|---|---|
| **Nom** | alert-vm-down-finsecure |
| **Type** | Log Alert (KQL) |
| **Sévérité** | Severity 1 — Critical |
| **Requête KQL** | Heartbeat manquant > 5 min |
| **Action** | Email + SMS équipe on-call |

```kql
Heartbeat
| where TimeGenerated > ago(5m)
| summarize LastHeartbeat = max(TimeGenerated) by Computer
| where LastHeartbeat < ago(5m)
```

**📸 Screenshot 9 :** *Azure Monitor → Alerts → Liste des alertes actives et déclenchées*

**📸 Screenshot 10 :** *Email de notification reçu lors du test de simulation de panne*

---

## ✅ Résultats et métriques

### SLA atteint par configuration

| Configuration | SLA théorique | Scénario de panne couvert |
|---|---|---|
| VM seule (sans dispo) | ~99,9% | Maintenance uniquement |
| Availability Set (2 VMs) | **99,95%** | Panne matérielle intra-rack |
| Availability Zones (2+ zones) | **99,99%** | Panne d'un datacenter entier |
| VMSS + Autoscale | Variable | Scaling sous charge |

### Tests de résilience réalisés

- ✅ **Simulation arrêt VM-1** → Load Balancer bascule sur VM-2 en < 30 secondes
- ✅ **Test charge CPU > 80%** → VMSS scale out de 2 → 4 instances en 3 minutes
- ✅ **Test charge CPU < 20%** → VMSS scale in de 4 → 2 instances en 5 minutes
- ✅ **Alerte CPU** → Email reçu dans les 2 minutes suivant le dépassement du seuil
- ✅ **Requêtes KQL** → Heartbeat et métriques visibles dans Log Analytics

---

## 💡 Compétences démontrées

| # | Compétence |
|---|---|
| 1 | **Azure Virtual Machines** — Déploiement, configuration, haute disponibilité |
| 2 | **Azure Load Balancer** — Distribution de charge L4, Health Probes, Backend Pools |
| 3 | **VM Scale Sets & Autoscale** — Scaling horizontal automatique basé sur métriques |
| 4 | **Azure Monitor & Log Analytics** — Supervision, KQL, alertes métriques et logs |
| 5 | **Infrastructure Résilience** — Availability Sets, Zones, Fault/Update Domains |

---

---

## 👤 Auteur

**Serge TOGNON**

---
> *Lab réalisé dans le cadre du développement d'un portfolio
> cloud & Linux professionnel.
> Certifié AZ-104 | Candidat RHCSA (EX200) en cours de préparation.
> Environnement : Azure Pay-As-You-Go | Région : francecentral*
