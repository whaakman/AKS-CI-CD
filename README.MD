![VSTS](/doc/kubernetes_cicd.PNG)

#Introduction
Azure Kubernetes Service (AKS) is still in preview and it�s one of many  ways you can run and manage your containers within Microsoft Azure. There are multiple ways to deploy your containers to AKS but from a DevOps perspective I prefer the build and release through Visual Studio Team Services and of course integrate CI/CD concepts. 
Some basic knowledge regarding Visual Studio Team Services, Visual Studio and AKS / Containers are required. If you haven�t mastered the basics, the links will give you a push in the right direction.   

For the following to work make sure you have the following in place:
-	Working Azure Kubernetes Service (AKS) with Azure Container Registry (ACR). If you don�t have one running, Microsoft has a great tutorial on deploying your newly created containers to AKS. 
https://docs.microsoft.com/en-us/azure/aks/tutorial-kubernetes-prepare-app
-	Docker-compose file;
-	An image publish to ACR; 
-	Visual Studio Team Services account;
-	Empty project.

If you did not use the Microsoft Azure AKS tutorial (all the way through) and you don't have an image at hand you can download the code in this repository. This repository will have all the files you need to configure CI-CD with VSTS for Docker and Kubernetes. Make sure to configure your application insights (or remove it if you wish).

#Preparing VSTS
Firstly we need to prepare VSTS. Because we are using docker-compose and Kubernetes (KubeCTL) we need to install the VSTS extensions. From the VSTS browse to extensions (https://[yourvstsaccount].visualstudio.com/_admin/_extensions) and open the marketplace. Install the following extensions:
-	Docker Integration;
-	Kubernetes Extension.

![Extensions](/doc/Extensions.PNG)

When the extensions are installed, commit the code to your project. If you don't know how this works you can look it up here: https://docs.microsoft.com/en-us/vsts/git/share-your-code-in-git-vs-2017?view=vsts. 

##tagging
If you are not using the files in this repository, you will need to create an extra file to store the image name of the image(s) you are tagging. To clarify: it is best practice to tag your images (versions) and have some form of versioning on your images stored in ACR. 

Create the tags.txt in the root folder of your project and store the name(s) of images your will tag before pushing them to ACR. For example
*acrdemo01.azurecr.io/demo-netcoreweb-web*

![Tags](/doc/tags.PNG)

Save the file and make sure you can find it later :)

##Service Endpoints
Before we configure the build definition you are required to create the endpoints for your Azure Subscription and AKS.
Go to *Services* and add a new Service Endpoint. Search for *Azure Resource Manager*. If your account already has access to a Microsoft Azure Subscription where ACR is located, the automated configuration will do.

Additionally you can configure the connection to use a different subscription using a Service Principal (click *use the full version of the endpoint dialog*). **Make sure your ACR is located within this subscription.**

The easiest way to configure a Service Principal manually is to use the Azure CLI (az ad sp): https://docs.microsoft.com/en-us/cli/azure/create-an-azure-service-principal-azure-cli?view=azure-cli-latest
Make sure you verify the connection.

![SP](/doc/service_principal.png)


Create another endpoint, scroll down until you see *Kubernetes* and add the endpoint. 
Configure the following:

Connection Name: AKS
Server URL: The Address of your Kubernetes Master, you can retrieve this by running *kubectl cluster-info* Look for *Kubernetes master is running at*
KubeConfig: Paste the contents of your Kubernetes config file. If you used the Microsoft Tutorial mentioned earlier it will be stored in c:\user\<yourusername>\.kube\config.

Once again verify the connection and create the endpoint.

![SP](/doc/service_principal2.png)


# Build Definition
Create a new build definition (*Build and release* --> *New*) for your project use the *master* branch or a different branch if you chose to use one earlierr. The build definition is pretty straight forward.

1.	From the process definition make sure you select the �Hosted Linux Preview� as an agent queue (unless you are running Windows containers). 
![Agent](/doc/Agent.PNG)
2.	From Phase 1, make sure the agent queue is set to *<inherit from definition>*

3.	Add the first step and search for *Docker Compose*
From the *Docker Compose File* browse to the compose file in your project. Configure te *Action* to run a Docker Compose command. The command we will run is *build*.

![compose build](compose_build.PNG)

4.	Add a new step and search for *Docker*
Set the action to *Tag images* and for the *Image Names Path* browse to your tags.txt. Additionally we will configure a tag to correspond with your VSTS build number. As an additional image tag set *$(Build.BuildNumber)*

![Tagging](/doc/tagging.PNG)

5.	As a third step, search for *Docker* once more and configure the following:

Container Registry Type: Azure Container Registry
Azure Subscription: <Select your newly created Service Endpoint>
Azure Container Registry: <Select your ACR here>
Action: Push an image
Image Name: <your image name as defined in your docker-compose.yaml with the ACR prefix>
Additional Image tag: $(Build.BuildNumber)

![Push](/doc/Push_Image.png)

6.	Navigate to the *Triggers* tab and tick the box *Enable Continuous Integration*. Configure the Branch Filter to include the branch *Master*

![CI](/doc/CI.PNG)

7.	Save and queue your build and wait for it to complete

#Release
Once your build is successful you can create a release process.
1.	Navigate to release and add a new release. 

2.	Add your build definition to the artifacts.

3.	Add an environment, a new blade will appear, search for *Kubernetes*

![CI](/doc/searchkub.PNG)

4.	Once the environment has been added navigate to *tasks*. You will see that *kubectl apply* is configured by default. Because we want to update our already running container on AKS, we will change this step to use the *set image* command. 

Configure the following settings:

Display name: *Set image*
Kubernetes Service Connection: <Select your Kubernetes service endpoint here>
Namespace: empty for default or enter your namespace if your configured one
Command: *set*
Arguments: **example** *image deployment demo-netcoreweb-web demo-netcoreweb-web=acrdemo01.azurecr.io/demo-netcoreweb-web:$(Build.BuildNumber)*
**Configure this to correspond with your own running image, build and pushed image and tag**

![Release](/doc/release.PNG)

5.	Save the release and run it

And that's it. Once you commit new updates to your repository your container will build and release using your Azure Kubernetes Service.






