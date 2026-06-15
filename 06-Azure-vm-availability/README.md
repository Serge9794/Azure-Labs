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
  --query "[].{Name:name, Zone:zones[0]}" \
  --output table
```

### Résultat attendu

| Instance | Zone | Datacenter | OS |
|---|---|---|---|
| `vmss-finsecure-zones_0` | Zone 1 | DC Toronto-Ouest | RHEL 9 |
| `vmss-finsecure-zones_1` | Zone 2 | DC Toronto-Centre | RHEL 9 |
| `vmss-finsecure-zones_2` | Zone 3 | DC Toronto-Est | RHEL 9 |

<img width="960" height="223" alt="1O" src="https://github.com/user-attachments/assets/ce330648-1b69-4c40-9ab1-47b6e1eac594" />

> ✅ En cas de panne totale de la Zone 2, les Zones 1 et 3 continuent de servir le trafic sans interruption.

<img width="892" height="374" alt="11" src="https://github.com/user-attachments/assets/1fbb776c-93a5-45b2-8e75-ab4435819bab" />


<img width="899" height="389" alt="12" src="https://github.com/user-attachments/assets/994efec4-5905-44b0-bc5e-6ec1799d7c2f" />


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
