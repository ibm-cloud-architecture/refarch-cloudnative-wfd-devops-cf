###### refarch-cloudnative-wfd-appetizer

## Microservices Reference Application - What's For Dinner Toolchain (Cloud Foundry)

This repository contains the DevOps toolchain for managing and deploying the Java microservices making up the What's For Dinner app to Bluemix as Cloud Foundry apps.

### How to use

In order to create a toolchain for the What's For Dinner Microservices Reference Application, please click below on the __Create toolchain__ button and follow the instructions.

[![Create wfd Deployment Toolchain](https://new-console.ng.bluemix.net/devops/graphics/create_toolchain_button.png)](https://new-console.ng.bluemix.net/devops/setup/deploy/?repository=https%3A//github.com/ibm-cloud-architecture/refarch-cloudnative-wfd-devops-cf.git&branch=RESILIENCY)

1. The "What is for dinner Toolchain" creation view will open.
2. In this window, you will be asked to specify the following properties for your toolchain:
 1. Specify a name for your toolchain that is __unique__ among all toolchains on your Bluemix namespace DevOps section.
 2. Click on the __GitHub__ icon. This opens the GitHub settings. Please, give your desired name to each of the repos that will be cloned.
 3. Click on the __Delivery Pipeline__ icon. This opens the delivery pipeline settings:
   * Specify the __Bluemix domain__ where your app will be hosted *(by default: mybluemix.net)*.
    * Specify the __build branch__ you would like your delivery pipelines to build the code from *(by default: RESILIENCY)*.
     * Specify your __app and APIs endpoints__ which must be __unique__ within Bluemix public *(by default: "mymenu" and "menu-apis" respectively)*.
      * Specify a __unique identifier__ which will be used to make the What's For Dinner microservices and their routing __unique within Bluemix public__ *(by default: toolchain's creation timestamp)*.
3. Click the Create button to complete the toolchain creation.
4. After creating the toolchain, make sure to deploy the What's For Dinner microservices in the same order as the delivery pipelines are numerated.

### Details

This RESILIENCY branch of the What's For Dinner DevOps GitHub repository not only implements the same toolchain as the master branch but also includes the new resiliency artifacts for the What's For Dinner app in its implementation.

These new elements are the Netflix OSS component Hystrix (metrics and circuit breaker) as well as the appropriate CloudAMQP service (Managed HA RabbitMQ Server) for integration. For further detail on these new components as well as on this new architecture that includes the resiliency pieces for the What's For Dinner app, please read the resiliency branch main app's [readme](https://github.com/ibm-cloud-architecture/refarch-cloudnative-netflix/tree/RESILIENCY).

Microservices implementing the Hystrix's Circuit Breaker pattern need the RabbitMQ Server credentials beforehand, so that they can establish connection to the server during their startup. Therefore, the CloudAMQP delivery pipeline must be executed before the Menu and Menu UI microservices are deployed (pipelines are numbered so that they are executed in the right order).

The CloudAMQP delivery pipeline will create a new CloudAMQP service. If there is an existing CloudAMQP service already, the CloudAMQP delivery pipeline will unbind it from any microservice and then delete it before creating the new one. Finally, it will bind the new CloudAMQP service to the previous bound microservices and re-stage them, so that these microservices use the new CloudAMQP service.

The Netflix OSS component Hystrix gets deployed after the Menu and Menu UI microservices and it is bound to the previously created CloudAMQP service in order to read Menu's and Menu UI's messages.

IMPORTANT: The Netflix OSS component Hystrix used for the What's For Dinner app has been modified in order to use web sockets rather than long HTTP polling.
