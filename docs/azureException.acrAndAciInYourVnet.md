[retour](../index.md)

# Azure, Les exceptions qui font mal ! 

Voici le premier volet d'une série d'article basé sur des retours d'expériences malheureux avec Azure.
Il ne faut pas y voir une critique de la platforme cloud, mais plutôt comprendre cette série comme une occasion manqué et une possibilité pour Microsoft d'améliorer encore son cloud Azure. Rien n'est parfait !
J'espère que cette série vous permettra d'éviter les pièges dans lequels nous sommes tombés, moi et mon équipe.

## Chapitre 1 : Azure Container Registry et Azure Container Instance dans votre Vnet.

### L'architecture

Parlons un peu de l'architecture que nous avions envisagé de réaliser.
L'objectif est d'instancier des conteneurs via des **Azure Container Instance** à partir d'image déployé dans un **Azure Container Registry**. Tout cela doit être sécurisé au niveau reseau dans un **Virtual Network**.

D'un coté le service **Azure Container Registry** permet d'utiliser les **Private Endpoint** pour restreindre l'accès de son service de registre de conteneur à un réseau privé.
De l'autre, les **Azure Container Instance** peuvent être déployé dans un réseau privé virtuel.
Donc, sur le papier le schéma d'architecture ci-dessous fonctionne... Ce n'est pas le cas.

![archi 1](../img/azureException.acrAndAciWithVnet.svg)

### L'exception

Après quelques tests infructueux, nous avons fait une petite recherche et découvert rapidement que l'utilisation d'un **Private Endpoint** pour un **Azure Container Registry** comportait quelques exceptions et notamment celle-ci : 

_Les instances de services Azure, notamment Azure DevOps Services, Web Apps et Azure Container Instances, ne peuvent pas non plus accéder à un registre de conteneurs dont l’accès réseau est restreint._

### Contournement

Les 2 services Microsoft ne pouvant être utiliser conjointement dans un réseau privé, 3 solutions s'offrent à nous :
1. Ne plus restreindre l'accès réseau de notre **Azure Container Registry**.
   
   ![archi 2](../img/azureException.acrAndAciWithVnet1.svg)
2. Remplacer notre **Azure Container Registry** par un solution **Container Registry Self Hosted**
   
   ![archi 3](../img/azureException.acrAndAciWithVnet2.svg)
2. Remplacer nos **Azure Container Instance** par un cluster **Azure Kubernetes Services** par exemple.
   
   ![archi 3](../img/azureException.acrAndAciWithVnet3.svg)

### Conclusion

Il n'y a pour le moment pas de contournement idéal. 
Cependant, bien que la documentation Microsoft indique le contraire, il semblerait qu'il soit possible de tirer (pull) une image Linux via une **Azure Web App** depuis un **Azure Container Registry** en ajoutant dans les appsettings :

_WEBSITE_PULL_IMAGE_OVER_VNET=true_

C'est un début.

#### Références

- [Déployer des instance de conteneur dans un réseau virtuel Azure](https://docs.microsoft.com/fr-fr/azure/container-instances/container-instances-vnet)
- [Connexion privée à un registre de conteneurs Azure à l’aide d’Azure Private Link](https://docs.microsoft.com/fr-fr/azure/container-registry/container-registry-private-link)
- [Deploying Linux custom container from private Azure Container Registry](https://azure.github.io/AppService/2021/07/03/Linux-container-from-ACR-with-private-endpoint.html)
- [ACR with private endpoint - Web Apps](https://github.com/MicrosoftDocs/azure-docs/issues/78210)

#### Remerciement


_Rédigé par Philippe MORISSEAU, Publié le 09 Septembre 2021_

[retour](../index.md)