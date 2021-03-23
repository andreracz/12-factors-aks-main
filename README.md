12 Factors Apps em AKS
======================

Esse repositório foi criado para demonstrar como utilizar o AKS (Azure Kubernetes Services) para hospedar microserviços construíoas com os principios [12 factors](http//12factors.net).

A apresentação está aqui: [12 factors em AKS](12-Factors-em-AKS.pdf)

A explicação do repositório e dos princípios estará em um artigo que irei publicar, e linkar aqui, que ainda está em desenvolvimento.

Este exemplo utiliza outros dois repositórios, com o código do microserviço e o código do front-end em angular:

1. [Microserviço](https://github.com/andreracz/12-factors-aks-microservice)
2. [Front-end](https://github.com/andreracz/12-factors-aks-front)


Subindo o ambiente
------------------

Para subir o ambiente utilizado, os seguintes passos foram utilizados (assumindo que foi usado powershell e as ferramentas [Azure cli](https://docs.microsoft.com/pt-br/cli/azure/install-azure-cli), [kubectl](https://kubernetes.io/docs/tasks/tools/) e [helm](https://helm.sh/docs/intro/install/) estão instaladas e que o usuário está logado na Azure):

### Subir o AKS e ACR

Instruções básicas para subir o AKS, ACR e Container Insights.

1. Obter o id da subscrion atual: ```$subscription = (az account show | ConvertFrom-Json).id```
2. Escolha o nome do seu resource-group: ```$rgName="12-factors"```
3. Escolha um nome único para o seu ACR: ```$acrName="meuAcrUnico"```
4. Escolha um nome para seu containerInsights: ```$containerInsightsName="log-12-factors"```
5. Escolha um nome para seu AKS: ```$aksName = "aks-12-factors"```
6. Criar resource group: ```az group create --location eastus --name $rgName```
7. Criar ACR: ```az acr create -g $rgName --name $acrName --sku basic```
8. Criar AKS: ```az aks create -g $rgName --name $aksName --node-count 1 --attach-acr $acrName```
9. Criar um Azure Monitor Workspace: ```$contInsights = az monitor log-analytics workspace create -g $rgName --workspace-name $containerInsightsName```
10. Obter o id completo do Workspace: ```$workspaceID = ($contInsights | ConvertFrom-Json).id```
11. Ligar o AKS no Container Insights: ```az aks enable-addons -a monitoring -n $aksName -g $rgName --workspace-resource-id "$workspaceId"```

### Criação do service principal para usar nos github actions

1. Criar service principal: ```$servicePrincipal = az ad sp create-for-rbac --sdk-auth --skip-assignment```
2. Converter o Service principal em objeto para uso nos proximos comandos: ```$sp = $servicePrincipal |ConvertFrom-Json```
3. Atribuir o papel de Contributor para o Service Principal poder fazer deploy no AKS: ```az role assignment create --assignee $sp.clientId --scope /subscriptions/$subscription/resourcegroups/$rgName/providers/Microsoft.ContainerService/managedClusters/$aksName --role Contributor```
4. Atribuir o papel de AcrPush para o Service Principal poder fazer Push do Container no ACR: ```az role assignment create --assignee $sp.clientId --scope /subscriptions/$subscription/resourceGroups/$rgName/providers/Microsoft.ContainerRegistry/registries/$acrName --role AcrPush```

### Instalação de Ingress para rotear e permitir conexões externas com nosso AKS:

1. Conectar com o AKS localmente: ```az aks get-credentials -g $rgName --name $aksName```
2. Criar um namespace para instalar o Ingress: ```kubectl create namespace ingress-basic```
3. Adicionar o repositório do nginx no helm: ```helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx```
4. Instalar o ingress: ```helm install nginx-ingress ingress-nginx/ingress-nginx --namespace ingress-basic  --set controller.replicaCount=2 --set controller.nodeSelector."beta\.kubernetes\.io/os"=linux --set defaultBackend.nodeSelector."beta\.kubernetes\.io/os"=linux --set controller.admissionWebhooks.patch.nodeSelector."beta\.kubernetes\.io/os"=linux```
5. Obter o IP do Ingress para podermos acessar o cluster: ```kubectl --namespace ingress-basic get services -o wide -w nginx-ingress-ingress-nginx-controller```

### Criação de Namespaces para deployar as aplicações

Aqui iremos criar dois namespaces para rodar os ambientes de desenvolvimento e homologação, e criaremos um role e rolebinding nestes namespaces para permitir que a nossa aplicação tenha acesso aos jobs (utilizado para o principio [XII - admin processes](https://12factor.net/pt_br/admin-processes))

1. Conectar com o AKS localmente: ```az aks get-credentials -g $rgName --name $aksName```
2. Criar namespace de desenvolvimento: ```kubectl create namespace dev```
3. Criação de um role com permissão de acesso a jobs em dev:  ```kubectl apply -n dev -f .\role.yaml```
4. Criar namespace de homologação: ```kubectl create namespace hom```
5. Criação de um role com permissão de acesso a jobs em hom:   ```kubectl apply -n hom -f .\role.yaml```

### Deploy do postgres nos namespaces

1. Conectar com o AKS localmente: ```az aks get-credentials -g $rgName --name $aksName```
2. Adicionar o repositório helm do PostgreSQL: ```helm repo add bitnami https://charts.bitnami.com/bitnami```
3. Instalar em Dev: ```helm install -n dev postgres bitnami/postgresql --set postgresqlPassword=senhaPostgresDev```
4. Instalar em Hom: ```helm install -n hom postgres bitnami/postgresql --set postgresqlPassword=senhaPostgresHom```
5. Alternativamente, se quiser usar o Azure Postgres ao invés de rodar ele no AKS, pode-se usar um external name, com o comando (O arquivo externalPostgres.yaml está neste repositório): ```kubectl apply -n hom -f .\externalPostgres.yaml```

