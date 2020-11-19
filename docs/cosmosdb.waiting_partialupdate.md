## CosmosDB : En attendant le partial update

Cela fait maintenant : 6 ans que cette idée a été remonté à Microsoft, 2 ans et demi que le sujet est dans la roadmap de Microsoft, 1 ans et demi que celui-ci est commencé et... toujours rien.

Pourtant, cette fonctionnalité pourrait tout changer ! J'ai eu l'occasion de rencontrer des clients qui pour cette unique raison préféraient partir sur MongoDB Atlas.

Mais pourquoi cette fonctionnalité change tout ?

### Constat : Atomicité des mises à jour DocumentDB

Donc pour faire une mise à jour dans un conteneur CosmosDB, il faut récupérer le document, le mettre à jour (merge entre la donnée reçu et celle exitant dans CosmosDB) et l'enregistrer dans le conteneur.

![Uml Diagram before](../img/UmlDiagram.before.svg)

Mais lorsque plusieurs composants sont suceptibles de mettre à jour cette mise à jour ou que vous utilisé des mécanismes de scalabilités horizontal pour vos composants, il y'a un risque d'écrasement de donnée.

En effet, prenons 2 composants A et B réalisant une mise à jour au même moment sur la même donnée.
- L'identifiant est lié à la proriété "id" (je mets de coté volontairement la notion de clé de partition),
- Le composant A met à jour la date de naissance ainsi que l'adresse email,
- Le composant B met à jour la date de naissance ainsi que le numéro de téléphone. 

| Data Composant A | Data Composant B | Data CosmosDB |
| - | - | - |
| {<br>&nbsp;&nbsp;"id": "99039816",<br>&nbsp;&nbsp;"firstname": "joe",<br>&nbsp;&nbsp;"lastname": "doe",<br>&nbsp;&nbsp;"birthdate": "1984-**06**-16T00:00:00Z",<br>&nbsp;&nbsp;**"email": "joe.doe@yopmail.com"**<br>} | {<br>&nbsp;&nbsp;"id": "99039816",<br>&nbsp;&nbsp;"firstname": "joe",<br>&nbsp;&nbsp;"lastname": "doe",<br>&nbsp;&nbsp;"birthdate": "1984-**07**-16T00:00:00Z",<br>&nbsp;&nbsp;**"phone": "+33.8.76.54.32.10"**<br>} | {<br>&nbsp;&nbsp;"id": "99039816",<br>&nbsp;&nbsp;"firstname": "joe",<br>&nbsp;&nbsp;"lastname": "doe",<br>&nbsp;&nbsp;"birthdate": "1984-05-16T00:00:00Z",<br>}

Dans un traitement classic suite à la mise à jour de la donnée par les 2 composants nous devrions avoir dans CosmosDB :

- Si le composant A est passé avant le composant B :
```
{
    "id": "99039816",
    "firstname": "joe",
    "lastname": "doe",
    "birthdate": "1984-07-16T00:00:00Z",
    "email": "joe.doe@yopmail.com",
    "phone": "+33.8.76.54.32.10"
}
```
- Si le composant B est passé avant le composant A :
```
{
    "id": "99039816",
    "firstname": "joe",
    "lastname": "doe",
    "birthdate": "1984-06-16T00:00:00Z",
    "email": "joe.doe@yopmail.com",
    "phone": "+33.8.76.54.32.10"
}
```

Imaginons maintenant le séquencement suivant :
1. Composant A : Fetch Data
2. Composant B : Fetch Data
3. Composant A : Merge data
4. Composant A : Upsert data
5. Composant B : Merge data
6. Composant B : Upsert data

Suite à la mise à jour de la donnée par les 2 composants nous aurons dans CosmosDB :
```
{
    "id": "99039816",
    "firstname": "joe",
    "lastname": "doe",
    "birthdate": "1984-07-16T00:00:00Z",
    "phone": "+33.8.76.54.32.10"
}
```
L'information adresse email est PERDU ! 
Le problème c'est que le composant A ne sais pas que le composant B réalise une opération de mise à jour au même moment. Et chaque composant réalise sont opération de mise à jour de son coté croyant qu'il est seul au monde !

Mais comment résoudre ce problème ?

Il s'agit d'un problème d'atomicité. Il va falloir s'assurer que les opérations de récupération, de fusion et d'enregistrement soient réalisée de manière atomique. 

Je reprend mes vieux cours d'informatique et la solution est "le verrou".

### Une solution : CosmosDB + Redis

Dans le cloud Azure, le service "cache redis" est bien pratique. Il ne sert pas qu'a faire du cache ! Il peut aussi gérer notre verrou.

##### Le principe

Chaque composant voulant réaliser une opération de mise à jour va systématiquement devoir poser un verrou dans Redis. Si le cache contient déjà ce verrou au moment où le composant souhaite poser son verrou, le composant saura alors qu'un autre composant est en train de réaliser une opération de mise à jour.
Ainsi, grace au verrou l'atomicité de l'operation de mise à jour est préservée.
Il faudra bien entendu penser à libérer le verrou une fois l'opération terminée.

![Uml Diagram after](../img/UmlDiagram.after.svg)

Il ne reste plus qu'à encapsuler cela dans une petite couche de technique (DAL) afin d'abstraire tout cela de mon traitement métier.

> Pour le verrou, il faut utiliser l'identifiant (la clé primaire) de la donnée en tant que clé. Par exemple pour notre exemple ci-dessous, on utilisera "99039816" comme clé. 

Parfait !

Mais que se passe-t-il s'il y a un verrou de posé ? On attends !

Oui, mais imaginons que l'on ai une vague de mise à jour sur la même donnée (c'est un cas extrème), ou que le cache redis ou que le comosdb ne réponde pas ? Mon composant ne va pas attendre éternellement ! 

C'est vrai. Pour cela, je conseille d'utiliser une file d'attente.

#### L'architecture

L'idée est de découper le traitement de mise à jour en 2 composants.
- Le premier composant (Métier) va pousser sa demande mise à jour dans une file d'attente.
- Le second composant (DAL) va lui traiter cette file d'attente et réaliser les mise à jour.
- En cas d'impossibilité de réaliser une mise à jour par le second composant, celui-ci va remettre la demande en file d'attente (celle-ci sera traité ultérieurement)  

![Schema CosmosDB-Redis](../img/cosmosdb.redis.svg)
```
1 - Demande de mise à jour
2 - Mise en file d'attente de la mise à jour par le composant métier
3 - Consommation de la file d'attente par le composant DAL
4 - Vérification du verrou
5 - Upsert de la donnée
5'- Remise en file d'attente en cas d'échec. 
```

#### Cas au limite

C'est bien cette histoire de file d'attente, mais du coup, j'ai un risque que les opérations de mise à jour soient réalisées dans le désordre.

C'est vrai.

Pour cela, on peut se baser sur le timestamp de la donnée (il faudra alors stocker cette information en plus dans CosmosDB). Si la mise à jour à traité est antiérieur à la dernière mise à jour dans CosmosDB, la mise à jour est abandonnée.

Ca reste imparfait.

### Conclusion

Il faut se doter de plusieurs ressources azure pour obtenir une solution capable de gérer proprement les mise à jour dans CosmosDB.

Vivement le Partial Update !

#### Références

- feedback Azure : https://feedback.azure.com/forums/263030-azure-cosmos-db/suggestions/6693091-be-able-to-do-partial-updates-on-document?page=4&per_page=20#{toggle_previous_statuses}
- wikipédia : https://fr.wikipedia.org/wiki/Propri%C3%A9t%C3%A9s_ACID
  
