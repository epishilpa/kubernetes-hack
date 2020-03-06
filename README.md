# Deploying a test app to kubernetes
This README compiles instructions from this tutorial.
https://docs.microsoft.com/en-us/azure/aks/tutorial-kubernetes-prepare-app

All commands were run in the directory of the cloned repo.
## Step 1: Deploy an app to docker
Follow the steps at https://docs.microsoft.com/en-us/azure/aks/tutorial-kubernetes-prepare-app

## Step 2: Create an ACR
az acr create --resource-group shilpahackday --subscription "Social Dev" --name shilpahackdayacr

Login to ACR (needed to use the ACR)
az acr login --name shilpahackdayacr --subscription "Social Dev"

To get the ACR login server address:
az acr list --resource-group shilpahackday --subscription "Social Dev" --query "[].{acrLoginServer:loginServer}" --output table

Tag your image:
docker tag azure-vote-front shilpahackdayacr.azurecr.io/azure-vote-front:v1

Push image to registry:
docker push shilpahackdayacr.azurecr.io/azure-vote-front:v1
The push refers to repository [shilpahackdayacr.azurecr.io/azure-vote-front] with a label of v1

Verify: List image in the registry:
az acr repository list --name shilpahackdayacr --subscription "Social Dev" --output table

Verify: Tag for image in the registry looks ok:
az acr repository show-tags --name shilpahackdayacr --subscription "Social Dev" --repository azure-vote-front --output table

## Step 3: Create a cluster
Directory permission is needed for the current user to register the application. For how to configure, please refer 'https://docs.microsoft.com/azure/azure-resource-manager/resource-group-create-service-principal-portal'. Original error: Insufficient privileges to complete the operation.
Creting AD permissions painful, and creating a service principal is not something Ops gives us access to.

az login
az account set --subscription "73fe0a3d-8118-4e61-95a6-5588f241d42b"
kubectl get pods
(showed default k8s pods)
kubectl get all
kubectl get all -A
kubectl get namespaces
kubectl config set-context --current --namespace=shilpa
kubectl get pods
(now no pods as in shilpa nmpsc)

## Step 4: Deploy to kubernetes
Change .yaml file to point to your ACR server and then deploy using
kubectl apply -f azure-vote-all-in-one-redis.yaml

kubectl get pods
NAME                                READY   STATUS             RESTARTS   AGE
azure-vote-back-6d4b4776b6-lbvqp    1/1     Running            0          52m
azure-vote-front-6c8445ff95-98gjl   0/1     ImagePullBackOff   0          12m


kubectl describe pod azure-vote-front-7cb9bfc4b9-pk287
Events:
  Type     Reason     Age                   From                               Message
  ----     ------     ----                  ----                               -------
  Normal   Scheduled  17m                   default-scheduler                  Successfully assigned shilpa/azure-vote-front-6c8445ff95-98gjl to aks-agentpool-30058973-2
  Normal   Pulling    16m (x4 over 17m)     kubelet, aks-agentpool-30058973-2  Pulling image "shilpaacr.azurecr.io/azure-vote-front:v1"
  Warning  Failed     16m (x4 over 17m)     kubelet, aks-agentpool-30058973-2  Failed to pull image "shilpaacr.azurecr.io/azure-vote-front:v1": [rpc error: code = Unknown desc = Error response from daemon: Get https://shilpaacr.azurecr.io/v2/azure-vote-front/manifests/v1: unauthorized: Application not registered with AAD., rpc error: code = Unknown desc = Error response from daemon: Get https://shilpaacr.azurecr.io/v2/azure-vote-front/manifests/v1: unauthorized: authentication required]
  Warning  Failed     16m (x4 over 17m)     kubelet, aks-agentpool-30058973-2  Error: ErrImagePull
  Warning  Failed     7m27s (x42 over 17m)  kubelet, aks-agentpool-30058973-2  Error: ImagePullBackOff
  Normal   BackOff    2m22s (x64 over 17m)  kubelet, aks-agentpool-30058973-2  Back-off pulling image "shilpaacr.azurecr.io/azure-vote-front:v1"
PS C:\docs\Hackday\Kubernetes\azure-voting-app-redis>

Did try
PS C:\docs\Hackday\Kubernetes\azure-voting-app-redis> az aks update -n k8s-playground -g cb-epts --subscription "Microsoft Azure Enterprise" --attach-acr smjhackday
Waiting for AAD role to propagate[################################    ]  90.0000%Could not create a role assignment for ACR. Are you an Owner on this subscription?

Chris:
"If it turns out to be private, the direction for how you would add the connection secret to kubernetes is here:
Pull an Image from a Private Registrykubernetes.io
https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/"

### How to deploy an app to Kubernetes to pull an image from a private registry
#### Step 1: Authenticate with the ACR to create a auth token that gets added to the .docker/config.json file
C:\WINDOWS\system32>docker login shilpaacr.azurecr.io
Username: shilpaacr
Password:
Login Succeeded
(Get username and pwd from azure portal on the ACR resource)

#### Step 2: To view docker config.json
In GitBash, go to default location: C:\Users\shjo
$cat ~/.docker/config.json

#### Step 3: Create a secret in kubernetes
We need to import the credential for shilpaacr.azurecr.io ACR as a secret into kubernetes and reference the in the yaml (to get around the Failed to pull image "shilpaacr.azurecr.io/azure-vote-front:v1" error).
Create secret
PS C:\docs\Hackday\Kubernetes\azure-voting-app-redis> kubectl create secret docker-registry regcred --docker-server=shilpaacr.azurecr.io --docker-username=shilpaacr --docker-password=<getyourpwdfromazureportal>

Get secrets
PS C:\docs\Hackday\Kubernetes\azure-voting-app-redis> kubectl get secrets
PS C:\docs\Hackday\Kubernetes\azure-voting-app-redis> kubectl describe secret/regcred

#### Step 4: Point to the kubernetes secret in yaml
Update deployment .yaml file:
imagePullSecrets:
         - name: regcred
		 
PS C:\docs\Hackday\Kubernetes\azure-voting-app-redis> kubectl apply -f azure-vote-all-in-one-redis.yaml


