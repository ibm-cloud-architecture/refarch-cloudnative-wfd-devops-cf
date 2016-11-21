# refarch-cloudnative-wfd-devops-cf

This repository contains the DevOps Toolchain for managing and deploying the microservices making up the What's For Dinner app to Bluemix as Cloud Foundry apps.

# How to use

To create the toolchain included in this project, start by clicking on the "Create Toolchain" button.

[![Create wfd Deployment Toolchain](https://new-console.ng.bluemix.net/devops/graphics/create_toolchain_button.png)](https://new-console.ng.bluemix.net/devops/setup/deploy/?repository=https%3A//github.com/ibm-cloud-architecture/refarch-cloudnative-wfd-devops-cf)

1. The "What is for dinner Toolchain" creation view will open. 
2. In this window, click on the "Delivery Pipeline" icon. This opens the delivery pipeline settings. In the settings, assign a value to the variable named "Unique Identifier" (which by default is *unique-identifier*).
3. Click the Create button to complete the toolchain creation.
4. After creating the toolchain, make sure to deploy the microservices in the following order:
 4. The Eureka server. To do this, execute the wfd-eureka-cf-ad delivery pipeline
 5. The Config server, by running the wfd-config-cf-ad delivery pipeline
 6. All other microservices, by executing their delivery pipelines

# Details

This toolchain contains a github clone tool, and a delivery pipeline for each of the microservices.
The github clone tool creates a cloned repository for the microservice.
Each delivery pipeline consists of two stages:

1. Build. This stage has one job, which runs a gradle build of the microservice.
2. Delivery Pipeline. This stage has 4 jobs:
 1. **Deploy CF App**. This job deploys a new version of the microservice.
 2. **Active Deploy - Begin**. This job creates an active deployment for the microservice, and advances this to the Test phase.
 3. **Test New Version**. This job is empty by default. It can be populated with any required tests.
 4. **Active Deploy - Complete**. This job continues the active deployment for the microservice, and completes it. 

# Considerations
## Order of deployment
All microservices depend on Eureka for service registration and discovery. This is established by creating a service associated to the Eureka server that can be bound to the other microservices. This service is created the very first time the Eureka server is deployed, and it must exist before other services are deployed so they can bind it to them.

Dynamic configuration is implemented using the Config server. The Config server is accessed by microservices by binding them to the service that is associated with the Config server. This service is created on the first deployment of the Config server, and therefor is required to exist before any of the other microservices are deployed. 

## Eureka & Active Deploy

Active deploy switches from an old to a new version of an app by switching the route from an old version of an app to the new version of that app. Oftentimes this is fine, but in the case of Eureka there is a complication. Eureka keeps an in-memory database of all app that have registered with it. Any new version of the Eureka app will not immediately have the registrations the current version has, and until it does, its service cannot fully replace the service of the old version. This means that there will be some period of time in which Eureka's service will be degraded.

Microservices start registering with Eureka as soon as they can access the Eureka server, which in the context of Active Deploy is the end of the rampup phase. In order to make the process of registering with a new Eureka as fast as possible, we make the rampup phase as short as possible: one second (1s). In doing this, the period of time in which the Eureka service is degraded is  minimized.

By doing this, we have observed it to take anywhere from 50-80 seconds until the new Eureka contains the registrations of all microservices, and Eureka's service is fully restored.
