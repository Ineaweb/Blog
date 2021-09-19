[retour](../index.md)

# Azure, Les exceptions qui font mal ! 

Voici le premier volet d'une série d'articles basé sur des retours d'expériences malheureux avec Azure.
Il ne faut pas y voir une critique de la plateforme cloud, mais plutôt comprendre cette série comme une occasion manquée et une possibilité pour Microsoft d'améliorer encore son cloud Azure. Rien n'est parfait !
J'espère que cette série vous permettra d'éviter les pièges dans lesquels nous sommes tombés, moi et mon équipe.

## Chapitre 1 : Azure Container Registry et Azure Container Instance dans votre Vnet.

### L'architecture

Parlons un peu de l'architecture que nous avions envisagé de réaliser.
L'objectif est d'instancier des conteneurs via des **Azure Container Instance** à partir d'image déployé dans un **Azure Container Registry**. Tout cela doit être sécurisé au niveau reseau dans un **Virtual Network**.

D'un coté le service **Azure Container Registry** permet d'utiliser les **Private Endpoint** pour restreindre l'accès de son service de registre de conteneur à un réseau privé.
De l'autre, les **Azure Container Instance** peuvent être déployés dans un réseau privé virtuel.
Donc, sur le papier le schéma d'architecture ci-dessous fonctionne... Ce n'est pas le cas.

![archi 1](../img/azureException.acrAndAciWithVnet.svg)

### L'exception

Après quelques tests infructueux, nous avons fait une petite recherche et découvert rapidement que l'utilisation d'un **Private Endpoint** pour un **Azure Container Registry** comportait quelques exceptions et notamment celle-ci : 

_Extrait [docs.microsoft.com](https://docs.microsoft.com/fr-fr/azure/container-registry/container-registry-private-link) : Les instances de services Azure, notamment Azure DevOps Services, Web Apps et Azure Container Instances, ne peuvent pas non plus accéder à un registre de conteneurs dont l’accès réseau est restreint._

### Contournement

Les 2 services Microsoft ne pouvant être utilisés conjointement dans un réseau privé, 3 solutions s'offrent à nous :
1. Ne plus restreindre l'accès réseau de notre **Azure Container Registry**.
   
   ![archi 2](../img/azureException.acrAndAciWithVnet1.svg)
2. Remplacer notre **Azure Container Registry** par une solution **Container Registry Self Hosted**
   
   ![archi 3](../img/azureException.acrAndAciWithVnet2.svg)
2. Remplacer nos **Azure Container Instance** par un cluster **Azure Kubernetes Services** par exemple.
   
   ![archi 3](../img/azureException.acrAndAciWithVnet3.svg)

Et il ne faut pas compter sur l'utilisation d'un **Service Endpoint** (en preview) puisque celui-ci ne permet pas non plus de récupérer l'image depuis un **Azure Container Instance**.

_Extrait de [docs.microsoft.com](https://docs.microsoft.com/fr-fr/azure/container-registry/container-registry-vnet#preview-limitations) : Les instances de services Azure, notamment Azure DevOps Services, Web Apps et Azure Container Instances, ne peuvent pas non plus accéder à un registre de conteneurs dont l’accès réseau est restreint._

### Conclusion

Il n'y a pour le moment pas de contournement idéal. 
Cependant, bien que la documentation Microsoft indique le contraire, il semblerait qu'il soit possible de tirer (pull) une image Linux via une **Azure Web App** depuis un **Azure Container Registry** en ajoutant dans les appsettings :

_WEBSITE_PULL_IMAGE_OVER_VNET=true_

C'est un début.

#### Dans la série

- [Chapitre 2 : Azure Container Instance et Windows dans votre Vnet](../docs/azureException.aciWindowsWithVnet.md)

#### Références

- [Déployer des instance de conteneur dans un réseau virtuel Azure](https://docs.microsoft.com/fr-fr/azure/container-instances/container-instances-vnet)
- [Connexion privée à un registre de conteneurs Azure à l’aide d’Azure Private Link](https://docs.microsoft.com/fr-fr/azure/container-registry/container-registry-private-link)
- [Restreindre l’accès à un registre de conteneurs à l’aide d’un point de terminaison de service dans un réseau virtuel Azure](https://docs.microsoft.com/fr-fr/azure/container-registry/container-registry-vnet#preview-limitations)
- [Deploying Linux custom container from private Azure Container Registry](https://azure.github.io/AppService/2021/07/03/Linux-container-from-ACR-with-private-endpoint.html)
- [ACR with private endpoint - Web Apps](https://github.com/MicrosoftDocs/azure-docs/issues/78210)

#### Remerciement

- [Etienne Louise](https://www.linkedin.com/in/etienne-louise-78154063/) : pour la relecture
- [David Dubourg](https://www.linkedin.com/in/dubourg-david-7413779/) : pour la relecture
- [Laurent Mondeil](https://www.linkedin.com/in/laurent-mondeil-0a87a743/) : pour la relecture

_Rédigé par Philippe MORISSEAU, Publié le 06 Septembre 2021_

[retour](../index.md)