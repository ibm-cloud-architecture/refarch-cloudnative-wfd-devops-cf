###### refarch-cloudnative-wfd-appetizer

## Microservices Reference Application - What's For Dinner Toolchain (Cloud Foundry)

This repository contains the DevOps toolchain for managing and deploying the Java microservices making up the What's For Dinner app to Bluemix as Cloud Foundry apps.

_This application has been developed and designed to run in the **IBM Bluemix us-south public region**. Changes may be required if it is to run on a different IBM Bluemix public region or on a local/dedicated environment._

![Application Architecture](https://rawgit.com/ibm-cloud-architecture/refarch-cloudnative-wfd-devops-cf/master/.bluemix/arch.svg)

### How to use

In order to create a toolchain for the What's For Dinner Microservices Reference Application, please click below on the __Create toolchain__ button and follow the instructions.

[![Create wfd Deployment Toolchain](https://new-console.ng.bluemix.net/devops/graphics/create_toolchain_button.png)](https://new-console.ng.bluemix.net/devops/setup/deploy/?repository=https%3A//github.com/ibm-cloud-architecture/refarch-cloudnative-wfd-devops-cf)

1. The "What is for dinner Toolchain" creation view will open.
2. In this window, you will be asked to specify the following properties for your toolchain:
 1. Specify a name for your toolchain that is __unique__ among all toolchains on your Bluemix namespace DevOps section.
 2. Click on the __GitHub__ icon. This opens the GitHub settings. Please, give your desired name to each of the repos that will be cloned.
 3. Click on the __Delivery Pipeline__ icon. This opens the delivery pipeline settings:
   * Specify the __Bluemix domain__ where your app will be hosted *(by default: mybluemix.net)*.
    * Specify the __build branch__ you would like your delivery pipelines to build the code from *(by default: master)*.
     * Specify your __app and APIs endpoints__ which must be __unique__ within Bluemix public *(by default: "mymenu" and "menu-apis" respectively)*.
      * Specify a __unique identifier__ which will be used to make the What's For Dinner microservices and their routing __unique within Bluemix public__ *(by default: toolchain's creation timestamp)*.
3. Click the Create button to complete the toolchain creation.
4. After creating the toolchain, make sure to deploy the What's For Dinner microservices in the following order:
 1. The Eureka server, by running the Eureka CF delivery pipeline.
 2. The Config server, by running the Config Server CF delivery pipeline.
 3. All other microservices, by executing their delivery pipelines.

### Details

This toolchain contains a github clone tool, and a delivery pipeline for each of the microservices.
The github clone tool creates a cloned repository for the microservice.
Each delivery pipeline consists of two stages:

1. __Build Java Projects__. This stage has only one job which runs a gradle build of the microservice.
2. __Deploy Microservice__. This stage has 4 jobs: - The input for this stage will be the output of the previous one. That is, it will take the build output. This stage is made up of 4 jobs:
 1. **Deploy CF App**. This job deploys the recently built new version of the Java microservice.
 2. **Active Deploy - Begin**. This job creates a new Active Deploy job to upgrade the microservice to a newer version. Traffic is routed to the new version of the microservice during the rampup phase and advances the Active Deploy job to the Test phase.
 3. **Test**. This job is empty by default. It can be edited to perform any required tests.
 4. **Active Deploy - Complete**. This job advances the Active Deploy job to its rampdown phase, where the old microservice version will be removed and the Active Deploy job will be marked completed if the previous test phase succeeded. Otherwise, the upgrade will be rolled back and the Active Deploy job marked as failed.

![Common pipeline](static/imgs/common.png?raw=true)

## Considerations
### Order of deployment
All microservices depend on Eureka for service registration and discovery. This is established by creating a User Provided Service (UPS) associated to the Eureka server that will hold Eureka parameters needed by the rest of the microservices such as its url. This service is created the very first time the Eureka server is deployed and it must exist before other microservices are deployed. When the rest of the microservices are deployed, they will bind to the Eureka UPS so that they can access Eureka parameters through their VCAP Services. Likewise, dynamic configuration is implemented using the Config server whose location need to known by the rest of the microservices during their deployment. This is done, again, by creating another UPS which is associated to the Config Server.

Because of the need of the User Provided Services, the Eureka and Config Server __Deploy Microservice__ delivery pipeline stage contain an extra jobs and look like the following:

<img src="static/imgs/eureka.png?raw=true" hspace="250">

### Eureka & active deploy

Active deploy switches from an old to a new version of a microservice by switching the route from an old version of a microservice to the new version of that microservice. Oftentimes this is fine, but in the case of Eureka there is a complication. Eureka keeps an in-memory database of all microservices that have registered with it. Any new version of the Eureka microservice will not immediately have the registrations the current version has, and until it does, its service cannot fully replace the service of the old version. This means that there will be some period of time in which Eureka's service will be degraded.

After the deployment of the new version of the Eureka Server, it will get mapped to itself the same route as the old Eureka Server version, which means that both versions will now receive traffic from the Go Router. As soon as the new version gets the route mapped to it, the Active Deploy job will begin its rampup phase where traffic will be evenly directed to both the old and the new version. Since having traffic evenly directed to both Eureka Server versions degrades its service, we make the rampup phase as short as possible (1 second) so that all traffic is directed to the new version as soon as possible.

The Eureka client on the microservice side keeps a heartbeat with the Eureka server in order to maintain an up-to-date list of registered and alive microservices. This heartbeat is sent every X seconds, being X adjustable. As a result, it will take some time (and will depend on each microservice) for the new version of the Eureka Server to get the heartbeat from all the microservices making up the What's For Dinner app. Until the Eureka Server does not received the heartbeat from each microservice, the What's For Dinner application will be unavailable (we have observed it to take anywhere from 50-80 seconds until the new Eureka contains the registrations of all microservices, and Eureka's service is fully restored).

### Multiple What's For Dinner apps

There is a limitation on the number of What's For Dinner apps you can deploy onto the same space to one. The reason for this is that the UPS created will be called likewise so then the delivery pipelines from the different What's For Dinner apps will overwrite each other's values. Hence there is a limitation of one What's For Dinner app per Bluemix Space.
