   # 🏢 Infrastructure Azure
   ### Lab Professionnel Entreprise 

<div align="center">

![Azure](https://img.shields.io/badge/Azure-AZ--104%20Certified-0078D4?style=for-the-badge&logo=microsoftazure&logoColor=white)


**Auteur : Serge TOGNON — AZ-104 Certified | Préparation RHCSA**

[![LinkedIn](https://img.shields.io/badge/LinkedIn-Serge%20TOGNON-0077B5?style=flat&logo=linkedin)](https://www.linkedin.com/in/serge-tognon-a63443187/?lipi=urn%3Ali%3Apage%3Ad_flagship3_profile_view_base_contact_details%3BiHxYnwbJRjOOflTaGqZasw%3D%3D )
![Status](https://img.shields.io/badge/Status-En%20cours-orange?style=flat)
![Level](https://img.shields.io/badge/Niveau-Junior%20Cloud%2FDevOps-blue?style=flat)

</div>

# 🌐 Lab Azure — Héberger une application web avec Azure App Service

> **Objectif du lab :** Déployer une application Python Flask sur Azure App Service (Linux).  

---

## 📋 Étapes réalisées

### Étape 1 — Créer l'environnement Python et l'application

```bash
python3 -m venv venv
source venv/bin/activate
pip install flask
```

```bash
mkdir ~/Projet_serge
cd ~/Projet_serge
# Créer application.py (voir fichier dans ce repo)
pip freeze > requirements.txt
```
<img width="871" height="440" alt="C" src="https://github.com/user-attachments/assets/f532d607-8413-469c-9d65-c8f87781e864" />
<img width="785" height="335" alt="D" src="https://github.com/user-attachments/assets/22c86401-5bb8-4146-a186-9fa500e285b5" />


---

### Étape 2 — Créer l'App Service sur le portail Azure

- Portail Azure → **Créer une ressource** → **Application web**
- Paramètres utilisés :

| Paramètre | Valeur |
|-----------|--------|
| Nom | `technovaapp` |
| Groupe de ressources | `rg-lab-azure` |
| Système d'exploitation | Linux |
| Runtime | Python 3.x |
| Plan | F1 (Gratuit) |
| Région | Central US |


<img width="960" height="450" alt="A" src="https://github.com/user-attachments/assets/fe6daa1c-2e7b-4490-b719-14d91d50dfcb" />

<img width="960" height="349" alt="B" src="https://github.com/user-attachments/assets/8f0bd0b4-4b1c-4dc4-b3f1-777fa456d404" />


---

### Étape 3 — Déployer avec Azure CLI

```bash
# Récupérer les infos automatiquement
export APPNAME=$(az webapp list --query "[0].name" --output tsv)
export APPRG=$(az webapp list --query "[0].resourceGroup" --output tsv)
export APPPLAN=$(az appservice plan list --query "[0].name" --output tsv)
export APPSKU=$(az appservice plan list --query "[0].sku.name" --output tsv)
export APPLOCATION=$(az appservice plan list --query "[0].location" --output tsv)

# Déployer
az webapp up \
  --name $APPNAME \
  --resource-group $APPRG \
  --plan $APPPLAN \
  --sku $APPSKU \
  --location "$APPLOCATION"
```

<img width="958" height="109" alt="E" src="https://github.com/user-attachments/assets/dc18ec5a-f8fb-436f-9582-de9372221047" />

<img width="959" height="447" alt="F" src="https://github.com/user-attachments/assets/1d3aa20e-6ea7-4749-b67e-99fb37ae40bf" />

---

### Étape 4 — Vérifier l'application en ligne

URL : `https://technovaapp-bxf6a7dsgaarcubf.centralus-01.azurewebsites.net`


<img width="957" height="514" alt="G" src="https://github.com/user-attachments/assets/660f9864-8544-4605-a20f-ce6a35ccf549" />
<img width="947" height="506" alt="H" src="https://github.com/user-attachments/assets/fef70738-2410-4671-ab16-6d23d88a042f" />
<img width="949" height="506" alt="I" src="https://github.com/user-attachments/assets/de8f29a4-70e6-426d-9a3d-c735061618bb" />

---

## 📁 Structure du projet

```
Marque_Serge/
├── application.py      # Application Flask (vitrine Marque Serge)
├── requirements.txt    # Dépendances Python
└── README.md           # Ce fichier
```

---

## 📜 Ressources

- [Microsoft Learn — Héberger une application web avec Azure App Service](https://learn.microsoft.com/fr-fr/training/modules/host-a-web-app-with-azure-app-service/)
- [Documentation Azure App Service](https://learn.microsoft.com/fr-fr/azure/app-service/)


**Serge TOGNON** · AZ-104 Certified · Préparation RHCSA · 2025

[![LinkedIn](https://img.shields.io/badge/LinkedIn-Serge%20TOGNON-0077B5?style=flat&logo=linkedin)]( https://www.linkedin.com/in/serge-tognon-a63443187/?lipi=urn%3Ali%3Apage%3Ad_flagship3_profile_view_base_contact_details%3BiHxYnwbJRjOOflTaGqZasw%3D%3D)



</div>
