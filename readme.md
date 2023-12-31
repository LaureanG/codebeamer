# Install Docker locally

- Run multi-container application locally to make sure both codebeamer and the db work. 
- Edit the sample docker-compose.yaml file from here (https://codebeamer.com/cb/wiki/18076086)
to add appropiate certificates and build the container images.
- Check the logs in Docker to make sure containers have started up and are ready to handle requests
- docker-compose up --build -d

### Check codebeamer instance
- https://localhost:9000/

### Stop the application and remove the containers.
- docker-compose down

### See the created images:
- docker images

### Tag the images
- docker tag intland/mysql:debian-8.0.23-utf8mb4 codebeameracr.azurecr.io/codebeamer-db
- docker tag intland/codebeamer:21.09-lts codebeameracr.azurecr.io/codebeamer-app

### Install or update Azure CLI using CMD (admin privileges) 
- az --version
- az --upgrade

### Create a resource group
- az group create --name codebeamer-lau-rg --location westeurope

### Create an Azure container registry 
- The container registry name must be unique within Azure, and contain 5-50 alphanumeric characters.
- az acr create --resource-group codebeamer-lau-rg --name codebeamerACR --sku Basic

### Log in to container registry
- az acr login --name codebeamerACR

- If tenant not found
- az login --tenant <tenant_id>

### Push images to container registry
- docker-compose push

### Notes
- CodeBeamer is shipped with a default system administrator user (SysAdmin) with the name bond and 007 password. 
https://codebeamer.com/cb/wiki/86294#section-Change+System+Administrator%27s+Name+and+Password

# Deploy Codebeamer to Azure

## 0. Azure Resources
- codebeamer-lau-rg
- codebeamerACR
- codebeamerstorageaccount
- codebeamer-fs

## 1. Using Azure Container Instances (ACI)
### Create Azure context. 
- This context associates Docker with an Azure subscription and resource group 
- docker login azure --tenant-id <your tenant id>
- docker context create aci codebeamer_aci_context

### View current context
- docker context ls

### Deploy application to Azure Container Instances (ACI)
- docker context use codebeamer_aci_context

### Use the alternate _docker-compose.yaml file
- docker compose up

- Note: Docker Compose's integration for ECS and ACI will be retired in November 2023. Learn more: 
- https://docs.docker.com/go/compose-ecs-eol/

## 2. Using Azure Kubernetes Service (AKS)

### Install AKS CLI
- az aks install-cli
- set PATH=%PATH%;"C:\Users\laurean.gheorghiu\.azure-kubectl" 

### Link CLI to AKS
- az aks get-credentials --resource-group codebeamer-lau-rg --name codebeamer-aks

### Get AKS nodes
- kubectl get nodes

### Get AKS services
- kubectl get services

### Get AKS pods
- kubectl get pods

### Create a storage class
- kubectl apply -f azure-file-sc.yaml

### Create a persistent volume claim
- kubectl apply -f azure-file-pvc.yaml

### Create Azure secret
- kubectl apply -f azure-secret.yaml

### Retrieve storage classes
- kubectl get storageclasses

### Create deployment.yaml
- kubectl apply -f deployment.yaml

### Convert docker-compose.yaml to AKS deployment yaml files
- Go to docker-compose location
- kompose convert --out ./k8s
- Will generate a k8s folder will all the yaml files
- Manual editing of the files is required
- eg. pv, pvc, deployment, service, but no ingress

### Make service externally available
- Change in codebeamer-app-service from
status:
  loadBalancer: {}
to
type: LoadBalancer

### Reconnect to AKS
- az aks get-credentials --resource-group codebeamer-lau-rg --name codebeamer-aks

- kompose convert --out ./k8s
- docker-compose build
- kubectl apply -f k8s
- kubectl delete -f k8s
- kubectl get svc codebeamer-app

- kubectl get nodes
- kubectl get services
- kubectl get pods

- kubectl logs codebeamer-app-6bf8b7c5f5-zq4ps

- kubectl create -f codebeamer-app-configmap.yaml
- kubectl create configmap [cm-name] --from-file=[path-to-file]

### Encode the certificate file
- Using powershell:
- -  cat example.crt | base64
[convert]::ToBase64String((Get-Content -path "domain.crt" -Encoding byte))

- Edit Secret: codebeamer-app-certificate-secret
domain.crt 
- Using powershell:
- -  [convert]::ToBase64String((Get-Content -path "domain.p12" -Encoding byte))
- Using bash:
- - cat ../certificates/test_certs/domain.p12 | base64 -w 0
- eg. domain.key -> U3dxYTEyMzQh

### Encode the key file
- Using powershell:
- - cat example.key | base64
[convert]::ToBase64String((Get-Content -path "domain.key" -Encoding byte))

### Create a Kubernetes Secret
- kubectl create secret generic <secret_name> --from-literal=key1=value1 --from-literal=key2=value2
- kubectl create secret generic <secret_name> --from-file=<path_to_file>
- kubectl create secret generic example-secret \
  --from-file=example.crt=path/to/encoded/certificate.crt \
  --from-file=example.key=path/to/encoded/key.key

- kubectl create secret generic codebeamer-app-certificate-secret --from-file=domain.crt=C:\Users\laurean.gheorghiu\work\certificates\test_certs\domain.crt.b64.txt --from-file=domain.key=C:\Users\laurean.gheorghiu\work\certificates\test_certs\domain.key.b64.txt

### View a Kubernetes Secret
- kubectl get secrets
- kubectl get secret <secret-name> -o yaml

### Export secret to yaml file
- kubectl get secret codebeamer-app-certificate-secret -o yaml > codebeamer-app-certificate-secret.yaml

### Changes in codebeamer-app-deployment
- name: TOMCAT_CONNECTOR_KEYSTORE_FILE
  value: /home/appuser/ssl/keystore.p12
- name: TOMCAT_CONNECTOR_KEYSTORE_PASS
  value: Swqa1234!

### Forward port to pod
- kubectl port-forward codebeamer-db-68c9bb4d7d-8r6zq 3306 

### Attach to pod
- kubectl exec -it codebeamer-app-65c64ff999-4tdbd -n default -- bash

### Check the persistance on host node
- eg /home/appuser/codebeamer/repository/ directory for laurean.txt

### Test CodeBeamer is working from inside the pod
- curl -I http://localhost:8090

### View nodes in your cluster along with their resource utilization
- kubectl top nodes
- - DS2_v2 had 7Gb RAM and crashed at login, now using D4s that has 16Gb RAM, using 13Gb
- Codebeamer usage 

NAME | CPU(cores) | CPU% | MEMORY(bytes) | MEMORY%
--- | --- | ---| --- | ---
aks-agentpool-36809342-vmss000000 | 215m | 5%| 12490Mi | 99%
aks-agentpool-36809342-vmss000000 | 1281m | 33% | 4281Mi | 34%
aks-agentpool-36809342-vmss000000 | 210m  | 5%  |  3853Mi | 30%
aks-agentpool-36809342-vmss000000 | 1434m | 37% | 809Mi   | 54% 

### Other users
- connect to aks instance locally
  - az aks get-credentials --resource-group codebeamer-lau-rg --name codebeamer-aks
- Copy files from/to pod
  - kubectl cp <pod_name>:<file_path> <destination_path>
  - kubectl cp codebeamer-app-bd848f5c7-9btlq:/home/appuser/codebeamer/logs/f34a42ea-b9ca-4b1b-b267-dcbdcf6ea2a1/logs/errors.txt errors.txt  
  - kubectl cp codebeamer-app-bd848f5c7-9btlq:/home/appuser/codebeamer/logs/f34a42ea-b9ca-4b1b-b267-dcbdcf6ea2a1/logs logs                 

### Create an alias
- in Windows create an alias for kubectl using powershell:
-  New-Alias -Name "k" kubectl

### Using a privater registry
- kubectl create secret docker-registry regcred --docker-server=YOUR_REGISTRY_SERVER --docker-username=YOUR_REGISTRY_USERNAME --docker-password=YOUR_REGISTRY_PASSWORD --docker-email=YOUR_EMAIL
- kubectl delete secret <SECRET_NAME>
- kubectl create secret docker-registry codebeamer-regcred --docker-server=codebeameracr.azurecr.io --docker-username=codebeamerACR --docker-password=KBKLvDMqZhh2U25APlPDAEpVPd1Z/fRFBF2HRvGXYU+ACRCk9Ql/ 
- kubectl delete secret codebeamer-regcred

### Run a command in a pod, by modifying the deployment yaml file
- command: ["sh", "-c", "cp -r /source_folder/. /destination_folder"]
          
# Troubleshoot Docker
- stop and remove all the running services and volumes
  - docker-compose down -v
- completely remove volumes and cache
  - docker system prune --volumes
- pulls and creates the services again
  - docker-compose up -d
  - Deleted Networks: minikube
- run mysql instance locally
  - docker run -it -p 3306:3306 -e MYSQL_USER="user" -e MYSQL_PASSWORD="pass" -e MYSQL_DATABASE="codebeamer" -e MYSQL_ROOT_PASSWORD="password" -e MYSQL_MAX_ALLOWED_PACKET="1024M" -e MYSQL_INNODB_BUFFER_POOL_SIZE="1G" -e MYSQL_INNODB_LOG_FILE_SIZE="256M" -e MYSQL_INNODB_LOG_BUFFER_SIZE="256M" --mount type=bind,source="C:\codebeamer\codebeamer-db-data",destination=/var/lib/mysql intland/mysql:debian-8.0.32-utf8mb4
- Error causing mysql container to stop: Failed to initialize DD Storage Engine
  - delete and recreate the shared volume folder: 
  - codebeamer-db-data
- Get the User ID and the Group ID for a user
  - id -u root -> 0
  - id -u appuser -> 1001
  - id -g root -> 0
  - id -g appuser -> 1001
- Retrieve the list of folders together with details, including the owner
  - ls -la
- Download logs from container
  - cp codebeamer-app-bd848f5c7-9btlq:/home/appuser/codebeamer/logs/f34a42ea-b9ca-4b1b-b267-dcbdcf6ea2a1/logs logs
  - kubectl cp codebeamer-app-bd848f5c7-9btlq:/home/appuser/codebeamer/logs/f34a42ea-b9ca-4b1b-b267-dcbdcf6ea2a1/logs/errors.txt errors.txt
- Create a terminal alias, for the duration of the window
  - New-Alias -Name "k" kubectl 
- Download customization from container
  - kubectl cp codebeamer-app-697c6758c5-p682x:/home/appuser/codebeamer/repository/config/customization/js js
- Start CB
  -./startup
- List processes
  - ps aux
  - ps aux | grep 'R'

# Deploy to new AKS cluster
- az aks get-credentials --resource-group codebeamer-rg --name aks-new
- kubectl get nodes
- kubectl create secret docker-registry codebeamer-regcred --docker-server=codebeameracr.azurecr.io --docker-username=codebeamerACR --docker-password=KBKLvDMqZhh2U25APlPDAEpVPd1Z/fRFBF2HRvGXYU+ACRCk9Ql/
- kubectl create secret generic codebeamer-certificate --from-file=.\ssl\keystore.p12
- kubectl get secrets
- kubectl get secret codebeamer-certificate -o yaml
- kubectl apply -f .\k8s\ers\laurean.gheorghiu\work\k8s\codebeamer>
- kubectl exec -it codebeamer-app-769579897b-2d6z6  -n default -- sh
- cd codebeamer/bin
- ./status should return something like: codebeamer (pid: 894) is running

# Create new AKS cluster with properly configured subnet and NSG 

- create a vnet with a cluster subnet, then an aks using that vnet and subnet.
- create a NSG
- kubectl config get-contexts
- kubectl config use-context aks-new
- kubectl get nodes
- kubectl create secret docker-registry codebeamer-regcred --docker-server=codebeameracr.azurecr.io --docker-username=codebeamerACR --docker-password=KBKLvDMqZhh2U25APlPDAEpVPd1Z/fRFBF2HRvGXYU+ACRCk9Ql/
- kubectl create secret generic codebeamer-certificate --from-file=.\ssl\keystore.p12
- kubectl get secrets
- az aks get-credentials --resource-group codebeamer-lau-rg --name codebeamer-aks-01
- New-AzAksCluster -ResourceGroupName codebeamer-lau-rg -Name codebeamer-lau-aks -NetworkPlugin 
kubenet -NodeVnetSubnetID codebeamer-lau-subnet

## Troubleshoot MySQL pod
- kubectl config get-contexts
- kubectl config use-context codebeamer-aks-01
- kubectl describe pod/codebeamer-db-7b99c46d44-fwt46
- change to latest db image: codebeameracr.azurecr.io/codebeamer-db:3
- kubectl apply -f .\k8s\codebeamer-db-deployment.yaml
- - Reason:       OOMKilled
- cat /etc/os-release
- Debian 11
- here https://codebeamer.com/cb/wiki/5562876#section-How+to+add+%28or+override%29+files+to+the+docker+images it mentions the image is based on OpenShift RHE but there isn't any Debian version there.
- here the Docker image: https://hub.docker.com/layers/intland/mysql/debian-8.0.34-utf8mb4/images/sha256-83e8fe00c25653513a6c66ee7546b26361a5bea88c98f9097e0182a7ba67b8a9?context=explore
- copy /etc/mysql/ 
- copy /usr/local/bin/ 
- cat docker-entrypoint.sh
- copy /docker-entrypoint-initdb.d

# Connect to MySQL
- mysql -u root -p
- mysql -h localhost -u user -p codebeamer
- mysql -u user -ppassword codebeamer
- mysql -u user -p codebeamer
- mysql -u root
- SHOW SCHEMAS;
- show databases;
- create database lau;

- kubectl delete -f .\k8s\codebeamer-db-service.yaml
- kubectl delete -f .\k8s\codebeamer-db-deployment.yaml
- kubectl delete -f .\k8s\codebeamer-db-mysql-pvc.yaml
- kubectl delete -f .\k8s\codebeamer-db-standard-ssd-class.yaml

- kubectl apply -f .\k8s\codebeamer-db-standard-ssd-class.yaml
- kubectl apply -f .\k8s\codebeamer-db-mysql-pvc.yaml
- kubectl apply -f .\k8s\codebeamer-db-service.yaml
- kubectl apply -f .\k8s\codebeamer-db-deployment.yaml

-  cd docker-entrypoint-initdb.d
/tmp/mysql-data/$MYSQL_INIT_FILENAME
/var/lib/mysql
cp codebeamer-app-bd848f5c7-9btlq:/home/appuser/codebeamer/logs/f34a42ea-b9ca-4b1b-b267-dcbdcf6ea2a1/logs logs
cd var/lib/mysql/data


- kubectl exec -it codebeamer-db-84db85b44f-mfnbt  -n default -- sh
- mysql -u root (only one that worker on v34)
- mysql -u user -p (worked on v23 with pass)