# refarch-cloudnative-wfd-devops-cf

This repository contains the DevOps toolchain for managing and deploying the microservices making up the What's For Dinner app to Bluemix as Cloud Foundry apps.

# How to use

To create the toolchain included in this project, start by clicking on the "Create Toolchain" button.

[![Create wfd Deployment Toolchain](https://new-console.ng.bluemix.net/devops/graphics/create_toolchain_button.png)](https://new-console.ng.bluemix.net/devops/setup/deploy/?repository=https%3A//github.com/ibm-cloud-architecture/refarch-cloudnative-wfd-devops-cf)

1. The "What is for dinner Toolchain" creation view will open.
2. In this window, you will be asked to specify the following properties for your toolchain:
 1. Specify a name for your toolchain that is unique among all toolchains on your Bluemix namespace DevOps section.
 2. Click on the "GitHub" icon. This opens the GitHub settings. Please, give your desired name to each of the repos that will be cloned.
 3. Click on the "Delivery Pipeline" icon. This opens the delivery pipeline settings:
   * Specify the Bluemix domain for your apps' routes (by default: mybluemix.net).
    * Specify the build branch you would like your delivery pipelines to build the code from (by default: master).
     * Specify a unique identifier which must make your apps' names __unique in Bluemix public__ (by default: toolchain's creation timestamp).
3. Click the Create button to complete the toolchain creation.
4. After creating the toolchain, make sure to deploy the microservices in the following order:
 1. The Eureka server. To do this, execute the wfd-eureka-cf-ad delivery pipeline
 2. The Config server, by running the wfd-config-cf-ad delivery pipeline
 3. All other microservices, by executing their delivery pipelines

# Details

This toolchain contains a github clone tool and a delivery pipeline for each of the microservices.
The github clone tool creates a cloned repository for the microservice.
The delivery pipeline will carry out a red/black deployment of the microservice by making use of the IBM Active Deploy tool.
Each delivery pipeline consists of two stages:

1. Build - The input for this stage will be the code on the branch of the GitHub repository of the microservice. This stage has only one job, which runs a gradle build on the code specified as the input.
2. Deploy - The input for this stage will be the output of the previous one. That is, it will take the build output. This stage is made up of 4 jobs:
 1. **Deploy CF App**. This job deploys a new version of the microservice. It deploys the microservice without a public route mapped to it since this will be taken care in the following steps. If needed, this job also binds the microservice to the appropriate User Provided Services.
 2. **Active Deploy - Begin**. This job uses the IBM Active Deploy tool to create an active red/black deployment for the microservice. It routes half the public traffic to the new version of the microservice as long as the ramp-up period has been defined. Finally, it routes all the traffic to the new version of the microservice and advances the deployment to the Test phase.
 3. **Test New Version**. This job is empty by default but it should carry out any sort of testing needed onto the new version of the microservice in order to make sure the new version works as expected. Finally, it advances the deployment to the next phase.
 4. **Active Deploy - Complete**. This job gets rid of old versions of the microservices. This is done based on the concurrent versions parameter of the IBM Active Deploy tool.

 **IMPORTANT:** If any error or failure occurs during any of the IBM Active Deploy tool phases, the tool itself executes a rollback procedure whereby traffic is routed back to the microservice version running before the deployments got kicked off.

# Considerations
## Order of deployment
All microservices depend on Eureka for communicating and collaborating among them. For doing so, microservices need to implement the service registration and discovery pattern with the Eureka Server. In order for microservices to register, they need to know at startup time Eureka Server's location. This is done by creating a *User Provided Service* (UPS) associated to the Eureka server. This UPS will store Eureka Server's location. By binding the UPS to the other microservices afterwards but before they are started up, microservices will read Eureka Server's location from their VCAP SERVICES. The UPS is created the very first time the Eureka server is deployed (getting updated anytime a new Eureka version is rolled out), and it must exist before other services are deployed so they can bind it to them.

Dynamic configuration is implemented using the Config Server. In order for microservices to access the Config Server, they also need to know Config Server's location. Again, this is achieved by implementing the same approach as the one described for Eureka Server.

## Eureka & active deploy

Active deploy switches from an old to a new version of an app by switching the route from an old version of an app to the new version of that app. Oftentimes this is fine, but in the case of Eureka there is a complication. Eureka keeps an in-memory database of all app that have registered with it. Any new version of the Eureka app will not immediately have the registrations the current version has, and until it does, its service cannot fully replace the service of the old version. This means that there will be some period of time in which Eureka's service will be degraded.

Microservices start registering with Eureka as soon as they can access the Eureka server, which in the context of Active Deploy is the end of the rampup phase. In order to make the process of registering with a new Eureka as fast as possible, we make the rampup phase as short as possible: one second (1s). In doing this, the period of time in which the Eureka service is degraded is  minimized.

By doing this, we have observed it to take anywhere from 50-80 seconds until the new Eureka contains the registrations of all microservices, and Eureka's service is fully restored.
