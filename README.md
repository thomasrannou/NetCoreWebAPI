# Mettre à jour AKS

Ce répertoire contient les sources nécessaires à la réalisation de ce tutoriel :
Mettre à jour son cluster Kubernetes dans Azure - http://thomasrannou.azurewebsites.net/2020/04/14/mettre-a-jour-son-cluster-kubernetes-dans-azure-aks/

Pour être au niveau coté sécurité et fonctionnalité, il est important de rester à jour sur sa version de Kubernetes. 
Nous allons voir ici comment mettre à jour notre cluster AKS sans pour autant entraîner d’interruption de service.

## Initialisation
Tout d’abord, je vais utiliser le script Powershell pour provisionner une container registry et un cluster AKS pour héberger mon application.
J’utilise Visual Studio Code avec l’extension Powershell pour développer et tester mon script.
Dans ce script, vous pouvez paramétrer la localisation, le nom du ressource group, de la registry et du cluster. 
Il permet de déployer un cluster comprenant 3 nodes réparti chacun dans une zone de disponibilité de la région North Europe.
Une fois exécuté, le script vous fourni les informations sur votre cluster déployé.

## Déploiement
Je construit une image Docker pour mon projet. J’ai choisi comme projet de test une Web API .Net Core 3.1.
docker build -f "NetCoreWebApi/Dockerfile" . -t webapi

En parallèle, je me connecte à ma registry :
az acr login --name registryaks1004

Je tag mon image :
docker tag webapi registryaks1004.azurecr.io/webapi:latest

Et je la push :
docker push registryaks1004.azurecr.io/webapi:latest

Je déploie mon API :
kubectl create -f .\DeployWebApi.yaml

Je teste quelques commandes pour connaitre l’état de mon déploiement :
kubectl describe pod | select-string -pattern '^Name:','^Node:'

kubectl get service :
C’est OK, mon API est bien disponible à l’url indiquée par la commande jouée précédemment (External IP) 🙂

## Mise à jour du cluster
Je liste les mises à jour disponibles pour mon cluster, volontairement déployés lors de la phase d’initialisation en version 1.13.2 :
az aks get-upgrades --resource-group rgAks --name aksClusterMaj

Soyons fou, je vais essayer d’installer une des dernières versions, la 1.15.5 :
az aks upgrade --name aksClusterMaj --resource-group rgAks --kubernetes-version 1.15.5

Et bien non ! il n’est pas possible de “sauter” une version majeure. Par exemple, les mises à niveau 1.12.x -> 1.13.x ou 1.13.x -> 1.14.x sont autorisées, mais pas 1.12.x -> 1.14.x ! Je vais donc commencer par passer en 1.14.x.

Installons donc tout d’abord notre version 1.14.8 :
az aks upgrade --name aksClusterMaj --resource-group rgAks --kubernetes-version 1.14.8

Pendant la durée de la mise à jour mon application reste accessible.

Je peux maintenant passer en version 1.15.10 :
az aks upgrade --name aksClusterMaj --resource-group rgAks --kubernetes-version 1.15.10

Idem, mon application est comme précédemment imperturbable 🙂

az aks show --resource-group rgAks--name aks --output table 

## Les explications !
Comme vu précédemment, notre API est restée accessible tout du long de la mise à jour du cluster. Pourquoi ?
En fait ce n’est pas le node lui même qui est mis à jour. Il est plutôt remplacé par un node cible.
Pendant la mise à jour, AKS ajoute un nouveau nœud au cluster utilisant la version de Kubernetes indiquée. 
Une fois que le démarrage des pods d’application par le nouveau nœud est confirmée, l’ancien nœud est supprimé. Et l’opération se répétera pour l’ensemble des nœuds du cluster.

