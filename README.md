![image](https://github.com/sitodav/sito_schedulables/assets/9033283/1c40a2b5-aadd-45f1-95de-79bcadf05f25)
# Sito-Schedulables

This documentation is still a work in progress. 
I will update/add more details as time goes by, so I am sorry if I am missing some information, or failing to give the proper explanation, but for now, it's intended  more like a README than a technical paper.


-----------------------------------------------------------------------------------


# Table of Contents
1. [Introduction](#-introduction)
2. [Demo Requirements](#-demo-requirements)
3. [Project Structure](#-projects-structure)
4. [Running the demo](#-running-the-demo)
5. [Distributed Commit limitations](#-distributed-commit-1-c-and-its-limitations)
6. [Output-Input Join limitations](#-output-as-input-and-its-limitations)
7. [Rest APIs examples](#payload-explanation-for-rest-api-examples-using-swagger-ui)
8. [Orchestrator Dashboard examples](#via-dashboard-orchestrator-and-how-it-works--the-angular-front-end-application)
9. [Task Status](#task-status)
10. [How to install the library](#how-to-install-the-library-for-a-microservices-architecture-of-spring-projects)
11. [How the system works](#how-the-system-works)


-----------------------------------------------------------------------------------


## > Introduction

A remote tasks composition/scheduling engine for microservices.

This demo shows the use of task composition, orchestration, and remote scheduling  for distributed systems.

The code is intended to be added *on top of already existing codebase*, with small interventions on its part, when you want to be able to create
dependencies between the microservices operations, in a schedulable and asynchronous manner.
a ta
Imagine that you have several operations (on different microservices or the same microservice and different APIs) and you want to couple them, in a scheduled manner
(start the next when one or more of the previous are completed, commit only when all are completed with success, stop everything and rollback when there is an error, and so on...). 
You deploy the orchestrator microservice (and the dashboard if you want), and you can orchestrate your existing code base with minimal additional effort.

Once attached, the orchestrator allows async microservice API execution.

This are the snapshot of a system that uses this library:

**1) You have some (one or more) microservices, and each microservice exposes one or more endpoints (rest APIs) . These endpoints are *synchronous* operations (that can return results or not): you call one endpoint and you wait until it's finished.**

**2) You install this scheduling engine (library interfaces and Spring components) on each microservice *->* now you can schedule the *asynchronous* execution of the endpoint flows (the ones that were synchronous)**
   
**3) You deploy the orchestrator (configuring its .yml so that it knows what *microservices->endpoints->task-types* it has to work with) *->* now you can create dependency trees of asynchronous tasks (and you can plug one task output into another input if you need to)**

The orchestrator will take the payloads for the normal existing operations (plus some parameters to control the dependencies) and will create a scheduled batch of executions.

The orchestrator exposes a swagger GUI for controlling (start, stop, restart...) orchestrated tasks.
There is a dashboard, written and deployed as angular application, that allows you to keep everything under control in a more user-friendly way.

![img](https://github.com/sitodav/sito_schedulables/blob/develop/images/img2.png "Optional title")

You can create a new orchestration of multiple tasks (distributed on different microservices or on the same one),
control the tasks' dependencies/status and stop the whole orchestration via REST APIs or the front end (like provided in the showcase ).

![img](https://github.com/sitodav/sito_schedulables/blob/develop/images/img.png "Optional title")

The angular front end shows, among other things, the trees for dependency hierarchy.
The trees are rendered using a custom typescript library (sito-tree : https://github.com/sitodav/sito_tree_lib )

![img](https://github.com/sitodav/sito_schedulables/blob/develop/images/tree1.jpg "Optional title")
  
Every task can be dependent on one or more tasks (this creates the computational/dependency tree).
The dependent tasks will start only if (and after) the dependencies are completed with success.
Otherwise, they will be stopped (and trigger a stop signal to all the dependent tasks down the branch tree).

The task-trees composition is created successfully (with the tasks on the microservices and their reference on the orchestrator), or every task is rolled back, both locally (to the orchestrator) and remotely (on the orchestrated microservices).

Remote tasks execution (dependencies, status, and percentage % of completion) can be accessed in real-time from the orchestrator service.

You can manually stop a task (this will stop all the dependent tasks).

The system support "simulated" 2PC (2-Phases Commit) to use a rollback/commit all policy, and it's database independent (because it's implemented at the application level, instead of being delegated to the application server/ dbms)

Moreover, the engine allows for the execution of tasks whose input depends on the output of previous tasks.

**IMPORTANT:** At the moment the library requires the microservices (you want to orchestrate) to expose REST APIs.

-----------------------------------------------------------------------------------


## > Demo requirements

### For the Microservices BE ###

- the code is intended to be compiled with **Java8** (but it's only for the sake of the demo, you can of course change your compiling/versioning via the POM and the IDE configuration) so you must install and configure it on your system.
- you need to have **Maven** installed and working on your environment.
- you need a running instance of **mysql** for the example to start. Of course, you can change according to your java version, and you can use different dbms (you will have to update the .properties).
  - no need to create the databases for the microservices or the orchestrator, they will be created automatically.
- **lombok** needs to be installed (https://projectlombok.org/)
- You will start the demo, locally, using the **loc** spring profile (the 3 projects are configured to be used with kubernetes too if you want, but this is out of the tutorial scope for now)
  
**IMPORTANT: remember to go into the "/resource" folder for every mvn project (microservice1, microservice2 and orchestrator) , open "application-loc.properties" file and change the username/password specifications ( using your dbms credentials )**


### For the Orchestrator Dashboard FE ###

- you need at least **angular15** and **node.js** (with npm) installed on your system
- In this demo you will start the dashboard front end application using the default **ng serve** command. This is configured (in the **orchestrator_fe/angular.json**) to use **development** as default profile. In this case the environment.ts will be used to configure
  orchestrator BE endpoints to contact.


-----------------------------------------------------------------------------------


## > Projects structure

This demo repository contains several folders:
- **/orchestrator_fe** : contains the angular dashboard
- **/microservice1** and **/microservice2** contain the orchestrated microservices. If you want to apply the library to your code base, use this as reference on how to.
  These represent the existing microservices to be orchestrated.
- **/orchestrator** This is the code for the orchestrator microservice itself. It has to be deployed as-is (with some modification, read below).

**microservice1, microservice2, orchestrator** are all maven projects. They are to be built (**mvn clean install**) and to be deployed as you want. 
For sake of simplicity we'll start them using the embedded tomcat.

**orchestrator_fe** is an angular/typescript app. It uses sito-tree (library) in the node dependencies, and it's pretty straightforward to use, once you configure the env variables.

Now let's dive in the projects structures.


### Microservice1 ### 

If you open it in your IDE (Eclipse for example) you can see that its formal name is **MicroserviceA**. 
- You can see in the */resources* folder that there are 2 .properties file : an application.properties (to be used when deploying in a k8s environment) and a application-loc.properties (to be used when starting locally, with the *loc* spring profile). We will use the latter.
  **Change the url/port and user/pw for your mysql local instance here!**
  **If you want to change the context-path for the microservice, you will have to update the application.yml on the orchestrator accordingly (do not change it for now)**
  **You don't have to manually create the database on the mysql instance, it will be created automatically**
- The microserviceA APIs expose 3 controllers. Tree different operations , to simulate an existing microservice with 3 different flows that start from the API and end on the database. We will map these 3 flows as 3 different task types in the microservices:
  - **BarNoResult** : it just takes a input payload and write it on the db with different indices, several times, as *BarEntity* (the controller returns a result but it's just a dummy string we are not interested in, so no real result returned from the service endpoint).
  - **BarResult** : like the previous one, but the service level returns the input payload attached to a string (if the input is bar: "Hello" the output will be "Hello result"). This is used to show how to call for a service with result, storing the output and using it as input for following tasks.
  - **FooNoResult** : like BarNoResult, it takes a different input and write it on the database as *FooEntity*
- Microservice 1 exposes its swagger GUI at **http://localhost:8080/sito-schedulables/microservice_a/swagger-ui.html** (the application-loc.properties doesn't specify a port, so the embedded tomcat will use the default one) and the base API url is **http://localhost:8080/sito-schedulables/microservice_a/**
- Code structure:
  - /api: it contains the 3 controllers
  - /config: it contains the database configuration (entity scan on entities package and jpa repository generation) , CORS filtering and Swagger configuration.
  - /dao,/dto,/entities: packages for dao repositories, dto pojos and spring jpa entities
  - /exceptions: contains a simple service exception
  - /service: it contains the 3 service interfaces and implementation (one for each of the three flow BarNoResult,BarResult,FooNoResult)
  - /sito_schdl: it contains the code for the scheduling activation, microservice-side.
  - the application.properties and application-loc.properties (the latter used for the demo, when running locally) contain the database configuration entries (they have to point to your mysql instance when running locally) and some others
    scheduling properties (do not modify start.schedule=true otherwise the scheduling engine will be turned off on the microservice) like **orchestrator.url** that must point to the orchestrator project once deployed.

**NB**: the microservice is configured to use additional tables . 
For simplicity , in the demo, it uses the same database where the business tables are contained (but it can be changed if you want!)
The database are all created automatically.
MicroserviceA creates a database called **microserviceA_DB** , with 2 business tables (TB_BAR and TB_FOO) for the legacy operations (these simulate the pre-existent logic for a microservice you want to orchestrate). 
On the same db then a table called **SCHDL_METADATA_SCHEDULEDTASKS_A** is created : this is the table for the scheduling mechanism to work on the microservice.


### Microservice2 ### 

Opening the project in the IDE it will be called **MicroserviceB**. 
- Everything is the same as microservice1, except it only has one controller (one flow to be mapped as task type) and it's configured (in the application-loc.properties) to run on port 8081.
  - The only controller it exposes is **AlphaNoResult** : it just takes a payload and write it on the database
- Its swagger GUI will be accessible at **http://localhost:8081/sito-schedulables/microservice_b/swagger-ui.html** and its API base url will be **http://localhost:8081/sito-schedulables/microservice_b**.
Once started the microservice automatically creates a database called **microserviceB_DB**, with a table **TB_ALPHA** for the business operation, and **SCHDL_METADATA_SCHEDULEDTASKS_B** for the scheduling mechanism to work.
**As stated before, you don't need to have the business tables and the scheduling one on the same database. You can configure several different datasources (read below)**

  
### Orchestrator ###  

The orchestrator project is deployed as service itself, exposes REST API (like the microservices ) and Swagger-gui.
- It contains the logic to orchestrate the microservices.
- Here you configure the (orchestrated)microservices urls, ports, context paths.
- You define for each task what (orchestrated)microservice contact, what type of payload etc...
- It exposes two different controllers/apis:
  - /notify : this is not intended to be invoked by the user directly. It's used by the (orchestrated)microservices (scheduling engine) to notify the orchestrator about their status/workflow.
  - **/compose** : this is the REST API exposed to create/delete/update/start/stop a group of tasks. It's where the magic starts (and it's the BE interface used by the orchestrator front-end in angular)
- The orchestrator must point to a valid DBMS to create its tables for scheduling metadata. In the example **we will run the orchestrator using the loc spring profile** so the information about the environment will be read from the
  application-loc.properties (you can see that it's configured to use a local mysql instance, creating the sitoschdl_orch_metadata db on startup). It then creates its own scheduling tables : SCHDL_METADATA_ORCHESTRATOR_RUNNING_NODES, 
SCHDL_METADATA_ORCHESTRATOR_ROOT_NODES, SCHDL_METADATA_ORCHESTRATOR_TASKS_DEPENDENCIES .
- /conf package : contains , among the other things , the spring data jpa configuration (entity scan and dao generation) .
- /dao, entity, exception are pretty straightforward.
- /service : contains the core logic and implementation (read below).
- /dto and utils/mappers are important : here you will add new (orchestrated)microservice information when you want to add a new schedulable type of task
- **application-loc.yml** and **application.yml** are very important: here you define the type of tasks (and on what url and payload-transformers) you want to orchestrate. The latter is for k8s environment. We will use the former, working with loc profile.
 
Here you can see what are the information provided to the orchestrator, about the microservices/tasks to orchestrate, in the application-loc.yml:

![img](https://github.com/sitodav/sito_schedulables/blob/develop/images/yml.jpg "Optional title")

We are telling that there are 4 different task types.
Every task type has a name (taskType) and is mapped on a given controller on the destination.

The **taskType** specified must be the same of the ENUM value contained in the orchestrator at **sito.dskij.utils.enums.ETaskType** . --> THIS IS FOR A SPECIFIC ENDPOINT IN A GIVEN ORCHESTRATED MICROSERVICE
**microserviceScheduleApiUrl**: this is the url where we installed the scheduling engine **on the microservice to be orchestrated** -->THIS IS FOR A WHOLE ORCHESTRATED MICROSERVICE
**microserviceRootApiSwaggerUrl** is the swagger-ui complete url for the whole (orchestrated) microservice. -->THIS IS FOR A WHOLE ORCHESTRATED MICROSERVICE
**microserviceRootApiUrl**: is the single operation (on the orchestrated ms) to be associated with that task. --> THIS IS FOR A SPECIFIC ENDPOINT IN A GIVEN ORCHESTRATED MICROSERVICE

As you can see while some properties are indicative of the whole microservice (for example you have to specify the whole destination microservice swagger-ui on every task type contained in that application)
others are task-specific.

So , going back to our application-loc.yml we can read it as it follows:


	taskType: MICROSERVICE_A_FOO_NO_RESULT **is the name for the first operation on MicroserviceA** 
	microserviceRootApiSwaggerUrl: http://localhost:8080/sito-schedulables/microservice_a/swagger-ui.html **is where the MicroserviceA has its swagger-ui**
	microserviceScheduleApiUrl:  http://localhost:8080/sito-schedulables/microservice_a/schedule **is where we installed the scheduling engine for the whole microserviceA**
	microserviceRootApiUrl:  http://localhost:8080/sito-schedulables/microservice_a/foo **is where we send the payload ONLY for the taskType MICROSERVICE_A_FOO_NO_RESULT**
	microserviceRequestTransformerClassname : sito.dskij.utils.mappers.ext_ms.ExternalMS_FooDTOMapper **this is the transformer we have to use for the payloads**

	taskType: MICROSERVICE_A_BAR_NO_RESULT **is the name for another operation , different from the previous one, but ALWAYS on MicroserviceA** 
	microserviceRootApiSwaggerUrl: http://localhost:8080/sito-schedulables/microservice_a/swagger-ui.html **is where the MicroserviceA has its swagger-ui, as you can see is the same one from the first taskType**
	microserviceScheduleApiUrl: http://localhost:8080/sito-schedulables/microservice_a/schedule **is where we installed the scheduling engine for the whole microserviceA. Again, is like the previous task because these two task types share the same microservice**
	microserviceRootApiUrl: http://localhost:8080/sito-schedulables/microservice_a/bar **Now this is different from the first task, because even if we are on the same microservice, we are defining a different endpoint to be called (it ends with /bar instead of /foo)**
	microserviceRequestTransformerClassname : sito.dskij.utils.mappers.ext_ms.ExternalMS_BarDTOMapper **and of course being the payload different from the first task we use a different class to transform the payloads from orchestrator to orchestrated*
	   
	taskType: MICROSERVICE_A_BAR_WITH_RESULT  **now we are creating a third task type, always on the first microserviceA** 
	microserviceRootApiSwaggerUrl: http://localhost:8080/sito-schedulables/microservice_a/swagger-ui.html **is where the MicroserviceA has its swagger-ui, as you can see is the same one from the first taskType**
	microserviceScheduleApiUrl: http://localhost:8080/sito-schedulables/microservice_a/schedule **is where we installed the scheduling engine for the whole microserviceA. Again, is like the previous task because these three task types share the same microservice**
	microserviceRootApiUrl: http://localhost:8080/sito-schedulables/microservice_a/bar_with_result **and of course a different final endpoint **
	microserviceRequestTransformerClassname : sito.dskij.utils.mappers.ext_ms.ExternalMS_BarDTOMapper **and different transformer**
	      
	taskType: MICROSERVICE_B_ALPHA_NO_RESULT **The fourth tasktype is on a completely different microservice (MicroserviceB). So will will point to another url with different port even for the swaggers** 
	microserviceRootApiSwaggerUrl: http://localhost:8081/sito-schedulables/microservice_b/swagger-ui.html **is where the MicroserviceB has its swagger-ui, as you can see is different from the first 3 tasks**
	microserviceScheduleApiUrl: http://localhost:8081/sito-schedulables/microservice_b/schedule **where we installed the scheduling engine on MicroserviceB**
	microserviceRootApiUrl: http://localhost:8081/sito-schedulables/microservice_b **this is where the MicroserviceB will respond only for the task type we are configuring. It has no additional context path after /microservice_b, like it happened on the previous tasks, because is the 	only flow on MicroserviceB.**
	microserviceRequestTransformerClassname : sito.dskij.utils.mappers.ext_ms.ExternalMS_AlphaDTOMapper **the transformer for this task type**


We will now use these default configuration for the demo.
Next I will show how to install new task types or configure on existing microservices from scratch.


### Orchestrator Front Ent (Dashboard) ###

It's a web app, written in Angular & Typescript, that uses a custom library to show what is happening on the orchestrator in real time.
Here **you don't have to configure anything** about task types, it just shows what the orchestrator knows.
Just configure (if you want to change orchestrator endpoints) the environment.ts
**The dashboard front end only knows about the orchestrator BE (and doesn't need to know anything about the orchestrated microservices on the other side of the orchestrator)**
 

-----------------------------------------------------------------------------------


## > Running the demo

- **microservice1** (MicroserviceA project) : Go into the root folder **/microservice1** containing the pom.xml for the project, open a shell and run **mvn clean install**. Once it's built, run **mvn spring-boot:run -Dspring-boot.run.profiles=loc** (this will start with the local profile). 
  If everything is ok, you should be able to see to the Swagger GUI for the microservice http://localhost:8080/sito-schedulables/microservice_a/swagger-ui.html
- **microservice2** (MicroserviceB project) : Go into the root folder **/microservice2** containing the pom.xml for the project, open a shell and run **mvn clean install**. Once it's built, run **mvn spring-boot:run -Dspring-boot.run.profiles=loc** (this will start with the local profile). 
  If everything is ok, you should be able to see to the Swagger GUI for the microservice http://localhost:8081/sito-schedulables/microservice_b/swagger-ui.html
- **orchestrator** : Go into the root folder **/orchestrator** containing the pom.xml for the project, open a shell and run **mvn clean install**. Once it's built, run **mvn spring-boot:run -Dspring-boot.run.profiles=loc** (this will start with the local profile). 
  If everything is ok, you should be able to see to the Swagger GUI for the microservice http://localhost:8082/sito-schedulables/orchestrator/swagger-ui.html
- **orchestrator_fe** : Go into the root folder **/orchestrator_fe** containing the tsconfig.json for the webapp, open a shell and run **npm install** and then **ng serve** (to start it on defautl port)
  If everything is ok, you should be able to see go to the dashboard at http://localhost:4200

Once you have compiled and started all the 4 projects (2 orchestrated ms BE, 1 orchestrator BE, 1  dashboard FE for the orchestrator) you can run the examples.


-----------------------------------------------------------------------------------


## > Distributed Commit (1-C) and its limitations

*Distributed Commit (1-C)* sets all the commits (for every task on every distributed microservice) to happen only when all the tasks in a dependency tree are **completed with success**, otherwise all the tasks are rolled back. 

It uses an implementation (at application level) of the *2PC (Two-Phases Commit)* algorithm.

When a task is completed (with success), if 1-C is enabled, it doesn't commit yet. It notifies the orchestrator, and it waits. Once all the tasks are completed, the orchestrator will tell the microservices to let the commits happen. Otherwise if one or more tasks are in error, the orchestrator does not only prevent the future tasks from starting, but makes the waiting ones rollback too.

This is different from both the *compensating transaction* used in SAGA pattern for distributed systems and *classic 2PC* usually implemented at DBMS level, that must be shared by the microservices and supported by the application server, while here the system is agnostic in relation to the DBMS (as long as is relational)

When *Distributed Commit (1-C)* is disabled, every task commits the results as soon as it's finished (with success). 

In this case, if a task fails, the orchestrator only prevents future (dependent) tasks from starting, but the one already completed won't rollback (as expected).

**To enable/disable *Distributed Commit (1-C)* use the parameter *distributedCommit: true* (false is the default) in the orchestrator API payload/input (read below)**

***Distributed Commit (1-C)* at application level, here, has some limitations:**

The system tries to be as agnostic as possible.

For *1-C enabled* distributed operations the system uses a *ISOLATION_READ_UNCOMMITTED* semantic for the transactions : a task can read results written on a DB (if shared) from another task, even if these rows are not committed yet. Why ? Because some future tasks may depend on results written on a shared table from previous completed tasks: in that case you want the future tasks to be able to read the data produced by the previous tasks (they depend on) even if this are not committed yet (why are they not committed yet ? Because we are using 1-C so we will commit only when all the tasks are completed).
But at some point the DBMS rules enter the game, so we must account for *native locking policy* of rows (some DBMS are more permissive then others).
At the point I am writing this guide there is only one case where the 1-C fails to work (and goes into error). The cases are explained below.

If two or more tasks have:
**Separated DBs (Non-shared)** : you can use *Distributed (1-C)* both on and off. No problem.

**Same DB, DIFFERENT tables**: same as before, you can use *Distributed (1-C)* on and off with no issues.

**Same DB, SAME table, one task has to update rows written by another task**: you **CAN'T** use *Distributed Commit (1-C)* : if you try to you will encounter a *PESSIMISTIC_LOCK* exception (on some DB) because even if one transaction can see the uncommitted rows from another task, the DBMS lock prevents them from being updated.

**Same DB, same table, all other operations (ex. both the task read from the table, one read and the other writes, etc..)**: you can use *Distributed Commit (1-C)* both on and off. 

So keep this in mind when you program your tasks trees.


-----------------------------------------------------------------------------------


## > Output-As-Input and its limitations

When we talked about tasks dependency, until now, we were referring to the order of execution: a task can only start when its dependencies (the other tasks it depends on) are (all) completed (with success).

So a task can depend on one or more other tasks according to the scheduling order: this is simply a order matter.

This has nothing to do with the fact that one task's output can become the next task's input.

So we can have dependent tasks whose input/output are independent.

But we can have the other case too: a task could depend on another , not just for a *decided* order, but even because the processed results from one's API must be sent as one other's APIs' input payload.

In the following examples we will show both cases.

When you call the orchestrator APIs, use the **forwardResult: true** (default false) parameter to set one task API result to be forwarded to next (dependent) tasks.

If for a task, the input must come from the previous (it depends on) use **readForwardedResult: true** (default false) .

Imagine you have one task, called *task1*, on which another *task2* depends: *task1 -> task2*.

*task1* will have *forwardResult:true* and *task2* will have *readForwardedResult:true* and so on.

Currently (at the time this guide is being written) **forwardResult/readForwardedResult (output-input) only supports 1:n tasks dependencies**.

This means that if you have a task X that depends on multiple previous tasks A,B,C..., you cannot send all their outputs (or partials) as X input.

But you can always send X's output result to all the tasks that depends **ONLY** on it.

**IMPORTANT: if a *original* microservice REST endpoints returns a result you can always keep output-input joining disabled if you want. In that case the result won't be forwarded to other tasks. Enabling output-input joining just forward one task result to the next**

-----------------------------------------------------------------------------------


## > Examples And Payload Semantics

### Payload explanation for Rest API (examples using Swagger-UI)

Now I will show you how to interact with the microservices endpoints (microserviceA : fooNoResult, barNoResult, barResult and microserviceB : alphaNoResult), like 
you would **before** installing anything from this library. These are the **synchronouos** calls to the existing microservices (the ones you want to make schedulable by using this library).

**Then** we 'll show how to call the same microservices, using the scheduling engine (once that we installed it on the microservice itself) : this allows for an existing endpoint to be executed **asynchronously**.

And **finally** we will show how to use the orchestrator microservice as *facade/proxy* to create **asynchronous** endpoints "execution" (like when calling the single microservices scheduling engine) but in a *tree-like dependency structure*, how to set distributed commit and how to join one task's output as one other's input.

All this is done using the *Swagger-UI* to contact the REST-APIs (but you can use PostMan, Curl or whatever you prefer, of course...).

-----------------------------------------------------------------------------------
#### Calling the "normal" synchronous REST APIs, one endpoint at the time

Let's start by visiting the first (orchestrated) microservice : *microserviceA* that once started (by default) exposes its Swagger-UI at

*http://localhost:8080/sito-schedulables/microservice_a/swagger-ui.html* .

The microserviceA's original endpoints (the one we *decored* with the scheduling engine) under Swagger-UI were:

*bar-controller-no-result : /bar*

*bar-controller-with-result : /bar_with_result*

*bar-controller-no-result: /foo* 

This are the endpoints that existed (on each microservice) before installing this library.

By installing this library you add the */schedule* endpoints under each controller (for each task/flow) and a single *Schedulable API* (the /schedule endpoints under each controller are used to schedule the asynchronous execution, while the *Schedulable API* is used to interact with the orchestrator. Keep reading for funther details...).

So let's start by contacting , using Swagger-UI, the rest api for */bar* : it's a REST endpoint that simply tasks a string as *bar parameter* (POST body) and returns nothing of relevance (just a fixed string message).
This is the simple , synchronous call to the API (as they were intended to work on the original system).

![img](https://github.com/sitodav/sito_schedulables/blob/develop/images/msA_barnores_plain.jpg "Optional title")

As you see, the API returns almost immediately (it's blocking and synchronous).

We can do the same thing with the */bar_with_result* : it works exactly as the */bar* , the only difference is that it returns an *interesting output*: the input string + "result".

So if we send *{bar : "hello world"}* we will get the result *"hello world result"* . 

This endpoint will be used to show how to take its output result and send it to another task as input.

Lastly you can call the */foo* endpoint, that works as */bar* : it takes in the body the *foo* parameter, a string. It returns nothing of importance.

These were the *synchronous* calls, in the *original system* : what existed already before installing this scheduling engine/library. The client called, and waited until the operation finished.

Now that we have installed the engine, we can call the **same APIs endpoints** but specifying a date for when **the tasks have to be executed**.

This will be done **by calling the APIS url (as before), but adding the */schedule* suffix**. 

-----------------------------------------------------------------------------------
#### Calling the (installed) scheduling engine, for asynchronous/scheduled in time execution

So now we want to call each microservice APIs (*/bar*, */bar_with_result* and */foo*) but setting a date in the future for the task to be executed.

The call will return almost immediately, because the task will only be executed once the specified (as input) scheduling date will be reached.

To do so we contact the original endpoint, but adding **/schedule** at the end of the url, and passing (in the body) the parameter **programmedAt: dd/MM/yyyy HH:mm:ss**  as string.

This is the payload new for the */bar/scheduled* API endpoint:

![img](https://github.com/sitodav/sito_schedulables/blob/develop/images/msA_barnores_scheduled.jpg "Optional title")

And the results contains info about the *programmed* task.

![img](https://github.com/sitodav/sito_schedulables/blob/develop/images/msA_barnores_result_scheduled.jpg "Optional title")

Now the task is programmed to be executed for the specified date, and once the /schedule endpoint returns it hasn't been executed yet!

We can do the same thing with the other two endpoints : */bar_with_result/schedule* and */foo/schedule*.

Now let's use the *Schedulable-Operation APIs*: they will give us control on task life-cycle (deletion, stop, date update etc...)

![img](https://github.com/sitodav/sito_schedulables/blob/develop/images/msA_schedulableOperationApi.jpg "Optional title")

As you can see in the screenshot you have several REST endpoints. The names on the right (for the methods) are quite self explanatory.

All the endpoints use the generated *taskId* (returned by the *../schedule* endpoint) to address to a specific task.

You can manually :
* delete a task
* get details of a task
* stop a task
* change the scheduled date of a task
* get all the tasks that are waiting for a distributed commit to be notified from the orchestrator
* manually commit a specific task (that it's waiting for a distributed commit)
* manually rollback a specific task (waiting for distributed commit)
* get all the percentages of completion for the running tasks
* reset a task to *CREATED* status
* change a task input payload
* get all the scheduled tasks

Now we have worked with only *microserviceA* (and its endpoints /bar, /bar_with_results, /foo).

We can do the same with the other *microserviceB*, and everything would be specular.

So, now that we know how to schedule (for the future) the various API endpoints executions, we will see how to use the orchestrator as facade for tree-like tasks dependencies composition (a task depending on one or more, and so on).

-----------------------------------------------------------------------------------
#### Using the orchestrator to compose *trees* of dependent tasks

The Orchestrator microservice (once started) exposes two groups of REST APIs:

![img](https://github.com/sitodav/sito_schedulables/blob/develop/images/orchestrator_api.jpg "Optional title")

The *Callback APIs* are not intended to be called directly: they are used from the *orchestrated microservices* (in this case *microserviceA* and *microserviceB*).

*Task Composer APIs* are the one you are supposed to work with.

Here you can see the list of endpoints:

![img](https://github.com/sitodav/sito_schedulables/blob/develop/images/orchestrator_api_taskcomposer.jpg "Optional title")

As before, reading the names on the right will be quite self-explanatory.

**Before continuing, I have to give you some definitions:**
* **A *TASKs GROUP* is composed of several *TASKs TREEs* (like a forest)**
* **Each *TREE* is composed of tasks. A *TREE* has only one *ROOT* task (node). The *ROOT* is the task that DOES NOT depend on any other task**
* **So a *GROUP* is composed of several *ROOTs*: every *ROOT* is the starting point of its *TREE***
* **When you create a *tasks-composition* you can define one or more *ROOT* (and you will have one or more *TREEs*) -> this is a GROUP (synonym for task-composition)**
* **The different *TREEs* in a *GROUP* (that start from different *ROOTs*) , are executed in parallel, because they do not share common *ROOTs* (or common paths on a *TREE*)**
* **When you create a task composition, you have to specify only the *SCHEDULING DATES* for the roots: when the date is reached for every root, its execution will start. Once the root execution is finished, the next (one or more) dependent task will start, and so on, until all the tasks in a tree are executed.**
* **If you enable *Distributed Commit (1-C)* this will cover all the trees in a group at the same time (this means that if one tree fails, even the other trees will fail too**

Using these APIs you can:
* get all task trees from all the groups
* create a new task composition (group) or adding some tasks to it
* get the task group for a given task-ID (all the trees that belong to the group where the task's tree is)
* delete a whole group of trees (of tasks)
* stop a whole group of trees (of tasks)
* get several infos about the groups (like percentages and such)
* restart a whole group of trees
* get all the roots tasks for a given group
* get details for a single task-ID
* stop a single task
* get all the mapped types for DTOs between orchestrator and orchestrated microservices (this is used by the FE angular app)

So let's start by creating a composition of several tasks. Each task will be of types representing the call to */bar, /bar_with_result, /foo, /alpha* .

You just have to specify the type for each task in the composition : the orchestrator will know what endpoint to call with the ending */schedule* url, to schedule the roots, or when to start the tasks that depends on one that has completed (with success).

To create a new task composition we use the *POST /compose* endpoint.

(payloads examples for the */compose* endpoint can be found under the **/resource/rest_example** folder in the orchestrator project).

*Example1 (no distributed commit, no output-input joining)*

Here we'll use one of the payload examples , and send it to the *POST /compose* endpoint:

```
{
  "dependencyList": [
    "-1",
    "0",
    "1",
    "2"
  ],
  "taskList": [
    {
      "programmedAt": "09/01/2023 12:03:00",
      "taskType": "MICROSERVICE_A_FOO_NO_RESULT",
      "foo": "i am foo"
    },
    {
      "taskType": "MICROSERVICE_A_BAR_NO_RESULT",
      "bar": "i am bar"
    },
    {
      "taskType": "MICROSERVICE_B_ALPHA_NO_RESULT",
      "alpha": "i am alpha"
    },
    {
      "taskType": "MICROSERVICE_A_FOO_NO_RESULT",
      "foo": "foo again"
    }
  ]
}
```

Explanation:
* **dependencyList**: tells the orchestrator what tasks are roots or dependents on which: the index in the list is referring to the same-index task defined in the **taskList** parameter. We use **-1** if it's a **root** or the **index** of the task it depends on (if a task depends on more then one task , we separate the indices with **/**.
* For example here we are telling that the task (in **taskList** parameter) of index 0, is a root (because we are using **-1**). We are telling that the task of index 1 (in taskList) depends on task of index 0. That task of index 2 (in taskList) depends on task of index 1 and that task of index 3 depends on task of index 2.
* In **taskList** we provide the tasks payloads like we did when calling the microservices endpoints APIs directly (except that we have to specify the **programmedAt** parameter for the tasks that are defined, on the **dependencyList** as roots, the ones with **-1** in the corresponding index-based position).
* So we are creating a group of just one tree (because there is only one root) with the following scheduling sequence:
* **[A_FOO:i am foo]->[A_BAR:i am bar] -> [B_ALPHA:i am alpha] -> [A_FOO:foo again]**

The *output* for the API call will be :

```
{
  "taskList": [
    {
      "taskId": "3183cfcf-0fad-43a0-9626-fca7ac446270",
      "createDate": null,
      "endDate": null,
      "startDate": null,
      "status": "CREATED",
      "programmedAt": null,
      "error": null,
      "taskType": "MICROSERVICE_A_FOO_NO_RESULT",
      "percentage": null,
      "payloadRequestJson": null,
      "microserviceRootSwaggerUrl": null,
      "distributedCommit": false,
      "forwardResult": false,
      "readForwardedResult": false,
      "serializedTaskResultAsString": null,
      "dependsOn": [
        null
      ],
      "children": [
        {
          "taskId": "4bfe4363-4661-451c-bc29-e469fdb89f13",
          "createDate": null,
          "endDate": null,
          "startDate": null,
          "status": "CREATED",
          "programmedAt": null,
          "error": null,
          "taskType": "MICROSERVICE_A_BAR_NO_RESULT",
          "percentage": null,
          "payloadRequestJson": null,
          "microserviceRootSwaggerUrl": null,
          "distributedCommit": false,
          "forwardResult": false,
          "readForwardedResult": false,
          "serializedTaskResultAsString": null,
          "dependsOn": [
            "#3183cfcf-0fad-43a0-9626-fca7ac446270"
          ],
          "children": [
            {
              "taskId": "fb4f7fae-f5bb-4082-8db9-16f6b82701db",
              "createDate": null,
              "endDate": null,
              "startDate": null,
              "status": "CREATED",
              "programmedAt": null,
              "error": null,
              "taskType": "MICROSERVICE_B_ALPHA_NO_RESULT",
              "percentage": null,
              "payloadRequestJson": null,
              "microserviceRootSwaggerUrl": null,
              "distributedCommit": false,
              "forwardResult": false,
              "readForwardedResult": false,
              "serializedTaskResultAsString": null,
              "dependsOn": [
                "#4bfe4363-4661-451c-bc29-e469fdb89f13"
              ],
              "children": [
                {
                  "taskId": "f5957581-d0ea-429f-83ab-5a34fb315df7",
                  "createDate": null,
                  "endDate": null,
                  "startDate": null,
                  "status": "CREATED",
                  "programmedAt": null,
                  "error": null,
                  "taskType": "MICROSERVICE_A_FOO_NO_RESULT",
                  "percentage": null,
                  "payloadRequestJson": null,
                  "microserviceRootSwaggerUrl": null,
                  "distributedCommit": false,
                  "forwardResult": false,
                  "readForwardedResult": false,
                  "serializedTaskResultAsString": null,
                  "dependsOn": [
                    "#fb4f7fae-f5bb-4082-8db9-16f6b82701db"
                  ],
                  "children": []
                }
              ]
            }
          ]
        }
      ]
    }
  ],
  "dependencyList": [
    null,
    "#3183cfcf-0fad-43a0-9626-fca7ac446270",
    "#4bfe4363-4661-451c-bc29-e469fdb89f13",
    "#fb4f7fae-f5bb-4082-8db9-16f6b82701db"
  ],
  "taskGroupUid": "7fa0f58c-c318-4520-9579-dca607a6d303",
  "distributedCommit": false
}
```
As you can see in the output, for every task there is a parameter called **dependsOn** that list all the task-IDs that the specific task depends on.

**children** parameter tells us what are the next tasks to be executed once the task is completed with success.

**forwardResult** and **readForwardedResult** tells us if there is an output-input join between the tasks (read further for an example where it's enabled).

Many parameters are not used here, because are needed for other APIs calls.

You can see at the end the **taskGroupUid** that defines the group, and if the **distributed commit** is enabled or not.

----------
*Example2 (no distributed commit, no output-input joining)*

Here we will create, with the same tasks, a different group structure

```
{
  "dependencyList": [
    "-1",
    "-1",
    "0",
    "1"
  ],
  "taskList": [
    {
      "programmedAt": "09/01/2023 12:03:00",
      "taskType": "MICROSERVICE_A_FOO_NO_RESULT",
      "foo": "i am foo"
    },
    {
      "taskType": "MICROSERVICE_A_BAR_NO_RESULT",
      "programmedAt": "09/01/2023 12:03:00",
      "bar": "i am bar"
    },
    {
      "taskType": "MICROSERVICE_B_ALPHA_NO_RESULT",
      "alpha": "i am alpha"
    },
    {
      "taskType": "MICROSERVICE_A_FOO_NO_RESULT",
      "foo": "foo again"
    }
  ]
}
```
Explanation:
* The first two tasks (in the **taskList**) are both roots.
* The third task depends on the first, and the fourth on the second.
* We are creating a group of tasks made of two trees (two roots) that are independent on each other. Every tree is composed of two task (a root and a dependent one).
* The group structure will be the following:
  
  **[A_FOO: i am foo] -> [B_ALPHA: i am alpha]**
  
  **[A_BAR: i am bar] -> [A_FOO: foo again]**

----------
*Example3 (no distributed commit, no output-input join)*

Here we will create, with the same tasks, a group made of one root, two tasks (second and third) that depend on the root, and a fourth task that depends on both the second and third.

```
{
  "dependencyList": [
    "-1",
    "0",
    "0",
    "1/2"
  ],
  "taskList": [
    {
      "programmedAt": "09/01/2023 12:03:00",
      "taskType": "MICROSERVICE_A_FOO_NO_RESULT",
      "foo": "i am foo"
    },
    {
      "taskType": "MICROSERVICE_A_BAR_NO_RESULT",
      "bar": "i am bar"
    },
    {
      "taskType": "MICROSERVICE_B_ALPHA_NO_RESULT",
      "alpha": "i am alpha"
    },
    {
      "taskType": "MICROSERVICE_A_FOO_NO_RESULT",
      "foo": "foo again"
    }
  ]
}
 
```
Explanation:
* The first task is the root.
* The second and third (index 1 and 2) both depends on the root
* The fourth task (index 3) depends on both the second and third (indices 1 and 2, separated by **/** in the **dependencyList** entry at index 3)
* The group structure will be the following:

  **[A_FOO:i am foo]**
                  **->[A_BAR:i am bar]**
  						**-> [A_FOO:foo again]**
                  **-> [B_ALPHA:i am alpha]**


Until now, in our examples, we did not enable **distributed commit**.

Enabling it is pretty straightforward.

----------
*Example4 (distributed commit ON, no output-input join)*

Here we are using the same group structure from *example 1*, but this time with **distributed commit (1-C)** enabled.

```
{
  "distributedCommit": true,
  "dependencyList": [
    "-1",
    "0",
    "1",
    "2"
  ],
  "taskList": [
    {
      "programmedAt": "09/01/2023 12:03:00",
      "taskType": "MICROSERVICE_A_FOO_NO_RESULT",
      "foo": "i am foo"
    },
    {
      "taskType": "MICROSERVICE_A_BAR_NO_RESULT",
      "bar": "i am bar"
    },
    {
      "taskType": "MICROSERVICE_B_ALPHA_NO_RESULT",
      "alpha": "i am alpha"
    },
    {
      "taskType": "MICROSERVICE_A_FOO_NO_RESULT",
      "foo": "foo again"
    }
  ]
}
```
*Explanation*:
* Just set the parameter **distributedCommit** at true, and the results will be written on all the distributed DBs only when all the tasks in the group are completed with success. Otherwise everything is rolledback.

Now it's time to compose a group using tasks with results. These results will become the input for the dependent (following) tasks.

----------
*Example5 (distributed commit ON, with output-input join between tasks)*

```
{
  "dependencyList": [
    "-1",
    "0",
    "1",
    "2"
  ],
  "taskList": [
    {
      "programmedAt": "09/01/2023 12:03:00",
      "taskType": "MICROSERVICE_A_BAR_WITH_RESULT",
      "bar": "ONE",
      "forwardResult": true
    },
    {
      "taskType": "MICROSERVICE_A_FOO_NO_RESULT",
      "readForwardedResult": true
    },
    {
      "taskType": "MICROSERVICE_A_BAR_WITH_RESULT",
      "bar": "THREE",
      "forwardResult": true
    },
    {
      "taskType": "MICROSERVICE_B_ALPHA_NO_RESULT",
      "readForwardedResult": true
    }
  ]
}
```
*Explanation*:
* Here the first task has results on which depends the second, and the third has results on which depends the input of the 4th
* As you can see the first has **forwardResult** true, the second has no real input payload (foo) but has **readForwardedResult** at true (so it reads from 1th output)
* the third has the "bar" input and has **forwardResult** true, and the 4th has no input payload but reads from the 3th because it has **readForwardedResult** at true
* The group structure is a sequence of dependent tasks (to be executed one after another) just like in example 1
* We are not using distributed commit (but we could, just by settings *distributedCommit* at true in the input payload)
   
   **[A_FOO:i am foo]->[A_BAR:i am bar] -> [B_ALPHA:i am alpha] -> [A_FOO:foo again]**

The rest of APIs are quite self explanatory (like stop of a group etc). We won't show their use via REST (Swagger-UI) calls, but we will use the dashboard for this.


### Via Dashboard Orchestrator (and how it works) : the Angular Front End Application ###

Until now we interacted with the orchestrator using the UI provided by *Swagger(UI)*.

Now we will use a dashboard written in *Angular* .

The *dashboard application* is composed of two main views: 

**main dashboard view**: used to check what groups have been created, what tasks are _running/stopped/failed/completed..., what are the payloads, what are the dependencies (etc...) . All with a **graphical representation for trees**.

**task-creation view**: allows for group (trees, tasks etc..) composition: you specify the task types, connect the nodes (tasks) to create the trees (using the tree graphical view), and send it to the orchestrator APIs (you can even save/restore groups as files for future use) .


----------
*How the **task-creation** view works*.

If you start the front end, with default configuration you can go to ***http://localhost:4200/dashboard***![img](https://github.com/sitodav/sito_schedulables/blob/develop/images/groupcomposition.jpg "Optional title") and you will be presented with an empty dashboard (if you have not created anything using the orchestrator APIs yet) .

Let's go to the group composition view by clicking the right (second) button on the upper left corner (or you can go to http:**//localhost:4200/taskCreation**) 

![img](https://github.com/sitodav/sito_schedulables/blob/develop/images/groupcomposition.jpg "Optional title")

This is what you will see 

![img](https://github.com/sitodav/sito_schedulables/blob/develop/images/dashboard2.jpg "Optional title")

On the right you can choose (from the dropdown ) what *task type* you want to add to the *group*.

![img](https://github.com/sitodav/sito_schedulables/blob/develop/images/dashboard3.jpg "Optional title")

The dashboard box (under the task type dropdown) will be populated with a valid json payload **according to the selected task type* : in this example we selected the type *MICROSERVICEB_ALPHA_NO_RESULT* so in the json we will have the *alpha* string parameter.

![img](https://github.com/sitodav/sito_schedulables/blob/develop/images/dashboard4.jpg "Optional title")

On top of the native parameters (every task type has its own) you will get : *distributedCommit*, *forwardResult* and *readForwardResult*.

These are used to enable/disable *distributedCommit*, and to enable result forwarding /reading for a given task .

When you have defined your json, you click *Add* .

The task will be added to the group, and you will see this:

![img](https://github.com/sitodav/sito_schedulables/blob/develop/images/dashboard5.jpg "Optional title")

As you can see the table in the middle show what task you have added.

We have only our first task (you can click on the Swagger-UI APIs and it will **redirect you to the Swagger-UI page for the given microservice associated with the given task type).

On the right we have the **tree representation**: at this point there is only one node (because we have only one task).

If you click on the node (in the tree representation) or on the table, you will see (on the left bottom) the defined json payload .

In the table you can click on a selected task to remove it (all its dependent children task will be removed, if any).

Let's try to add more tasks.

![img](https://github.com/sitodav/sito_schedulables/blob/develop/images/dashboard6.jpg "Optional title")

As you can see, now we have a total of 3 tasks.

But they are not in a tree form: this means that they are all roots (so we have formally tree separate independent tree, each made of just one task).

Let's say that one of the task must depend on another : **just drag a node and drop it on the task you want it to depend on** (if you drag X on Y , X will depend on Y and you will have Y->X)

![img](https://github.com/sitodav/sito_schedulables/blob/develop/images/dashboard7.jpg "Optional title")

As you can see now we have created a *cluster* (a tree of connected nodes/tasks: they will share the **same color**)

**The buttons on top of the tree graphical view allow for collapse/expansion of tree paths and to save/restore a given group of trees/tasks**.

If you click **Next** (on the bottom of the page) you will reach the final stage for task creation, when you have to set specific **scheduling dates** for all the roots.

In out case, we only had 3 tasks, with two roots , so we will have to define starting date for just two (root) tasks:

![img](https://github.com/sitodav/sito_schedulables/blob/develop/images/dashboard8.jpg "Optional title") 

On the right side of the recap page you will see the **whole json payload** that will be sent to the orchestrator API backend (just like we did in the Swagger-UI orchestrator example).

Before clicking the **Save** button (on the bottom right) you can **enable/disable distributed (1-C) commit** for the whole group (instead of defining it on each task json).

**Important** : the global 1-C settings overrides the specific tasks 1-C settings.

Once you click save you will be redirected to the other view (the main landing **dashboard**).

----------
*How the **main dashboard** view works*.

Now that we created a task group via task-creation view, our main dashboard view will be populated.

![img](https://github.com/sitodav/sito_schedulables/blob/develop/images/dashboard9.jpg "Optional title") 

In the upper table/list you can see all the created group (just one for now).

You can select/click on a group, and check (in the second table/list) the list of tasks belonging to that group.

The third view (accordion) will show you the graphical tree for that group.

You can select a task (both on the list and on the tree) and it will be highlighted.

![img](https://github.com/sitodav/sito_schedulables/blob/develop/images/dashboard10.jpg "Optional title") 

Selecting a task will give you (in the last section/accordion) the body detailed information for the payload of the task and the tree representation only for the task and its children (the *subpath*)

![img](https://github.com/sitodav/sito_schedulables/blob/develop/images/dashboard11.jpg "Optional title") 

For each group the **percentage of completion** is shown.

The same goes for each task percentage of completion.

You can **enable highlighting (on the second table) of fathers and children of a given tasks** (*blue arrow* means father and *red arrow* means children).

![img](https://github.com/sitodav/sito_schedulables/blob/develop/images/dashboard12.jpg "Optional title") 

**The status and nodes colors (on the graphical tree) represents task status.**

#### Task Status
There are several different status for a task/node:

* **CREATED**: a task created, waiting for its scheduling date so be reached (**grey**)
* **SCHEDULED**: a task has just reached its scheduling date, and the orchestrator picked it. It's about to be scheduled (**black**)
* **RUNNING**: a scheduled task, running in a separate thread (**blue**)
* **STOPPED**: a task stopped while it was running. All its children will have stopped (**orange**)
* **COMPLETED_SUCCESS_WAITING_TOCOMMIT**: if distributed commit is enabled, this task has completed with success, but the results are not committed yet (this means are written as UNCOMMITTED data to the database) (**black**)
* **COMPLETED_SUCCESS_COMMITTED**: when distributed commit is enabled, this is a completed task that has committed its results (because all the other tasks in the group are completed with success, and committed too) (**green**)
* **COMPLETED_ERROR**: task that failed. All its children will be stopped. (**red**)
* **COMPLETED_SUCCESS_ROLLEDBACK**: task that originally completed with success, but had to rollback because some other task in the group failed/was stopped. (**red**)

When the scheduling date (for the roots) will be reached, the first tasks will start running

![img](https://github.com/sitodav/sito_schedulables/blob/develop/images/dashboard14.jpg "Optional title") 

And at some point will be completed

![img](https://github.com/sitodav/sito_schedulables/blob/develop/images/dashboard15.jpg "Optional title") 


**You can stop (BEFORE COMPLETION) a whole group (using the stop button for a given group on the first table, the *groups list*) and this will stop all the running tasks for a group.**
**Or you can stop just a single task in a group (and this will stop only the task and its descendents).**

![img](https://github.com/sitodav/sito_schedulables/blob/develop/images/dashboard13.jpg "Optional title") 

**If a task is stopped, you can restart it (if possible) using the restart button on the task row (in the table)**


-----------------------------------------------------------------------------------


## > How to install the library for a microservices architecture of spring projects


### On the microservices you want to orchestrate ###

- From one of the orchestrated microservices in the demo (/microserviceA or /microserviceB ) copy the folder **/sito_schdl** 
  into your project.
- Add to the enum **ETaskType** in **/sito_schdl/core/misc** a new entry for each flow (of the microservice you are in ) that you want to make orchestrable.
  Every flow has to have a separate API entry point.
  The **ETaskType** enum on the orchestrated microservice must contain only its type (doesn't need to know about other existing microservices tasks, even if they
  can be codependant. It's orchestrator's job to know all the task types)
- In the application.properties of the microservices add the necessary properties for orchestration (like in the microserviceA or B of the example).
  - **start_scheduled** : enables the engine on startup (set to false if you want to disable the scheduling engine, with this the orchestration will be disabled).
  - **task.operator.secs.delay** : it's the scheduling engine frequency for checking updates/new tasks (in seconds)
  - **orchestrator_url** : is the url you will deploy the orchestrator to.
  
  IMPORTANT: make sure to enable automatic DDL creation in the application.properties for the engine tables so you won't have to make them manually on the DB
  (use **spring.jpa.generate-ddl=true** and **spring.jpa.hibernate.ddl-auto=update** ) . 
- Under **sito_schdl/conf/TaskSchedulerConfiguration** modify the basepackages to point to the copied folders under your projects packages structure.
- Change your service (the one for the flow you want to make schedulable) signature for the method you want to be async, adding the **String taskIdIfAsync** (like *BarService* or *FooService* in the example).
- Modify the input DTO so that implements the **SchedulableByDate** interface and add to two parameters: **programmedAt** and **percentage** (see the example).
  Then implement the getProgrammedAsDate method. (This could have been an abstract class without the need to declare the two parameters and the @Override for the method, but
  working with an interface is easier when having to apply to existing architectures/codebases)
- In the API (Controller) for the flow you want to make orchestrable:
  - Add the @Autowired for  the **SchedulingComponent**.
  - For the existing normal synchronous service call (in the ordinary api endpoint) use null as **taskIdIfAsync** (because from that point we are not calling the service asynchronously).
  - Add a new endpoint (like in the example on microserviceA or B, called **saveNewScheduledOperation(@RequestBody BarDTO dtoRequest)**), mapped with **/schedule** (in append to the ordinary api url), as POST method.
    This endpoint calls the **newScheduleTask** method of the SchedulingComponent, and pass the taskType to it (change the taskType accordingly).
	Use the code from the example (again, *microserviceA* or B under the api package) on your code (remember to change the **taskType** using the one you added to your enum).
- In the implementation for the service you want to make schedulable (reference in the example microserviceA/sito.dskij/service/impl/BarServiceImpl):
  - Make the serviceImpl extend **AbstractSchedulableOperationService** and create a constructor that calls the superclass constructor passing its task type.
  - Add the @Autowired **PercentageComponent**
  - Add the implementation for **doMsOperationAsync()**, like the code proviced in the example. This method will have to call your normal/old service method, passing the taskId.
  - In the body for the old/normal implementation of your service add a way to check for stop signal and to increment the percentage.
    Do this like the example code where it's commented with **/ADD THIS LINE OF CODE FOR SCHEDULING CHECKS/**
    
 
### For the orchestrator ###

- Modify the application.properties to enable DDL (to create the table for the orchestrator in update)
  Use **spring.jpa.generate-ddl=true** and **spring.jpa.hibernate.ddl-auto=update** ) .
- Modify the **ETaskType** and add **ALL** the task type (one for every flow on every orchestrated microservice).
  The names must correspond.
- Modify the class **GenericTaskInputDTO** adding all the properties for all the DTO of the API you want to orchestrate
  (this dto represents all the different type of requests).
- Create a specific mapper that transform from GenericTaskInputDTO to the specific DTO for a given flow (to be orchestrated) on a microservice
  (refer to the example from the demo orchestrator ).
  In the example I used a custom generic mapper class (**GenericAtomicMapper**) inside every mapper, to automatically map the atomic fields with the same name.
  This class only map atomic fields (so no lists or recursive mapping on composition). You must write your own mapping for nested objects/structured types
  (or use whatever library you want, like Dozer for example)
- Modify the application.yml to point to the orchestrated microservices. 
  - For the names use the exact values used in the **ETaskType** enum.
  - **microserviceScheduleApiUrl** : must point to the configured scheduled api engine on the microservice (if you have different orchestrated flows on the same microservice
    this will the same in the application.yaml, like in the orchestrator in the demo for the first two entries FOO and BAR).
	**microserviceRootApiUrl**: must point to the original api for the flow you want to orchestrate (the orchestrator will add **/schedule** in append to the provided url)


### For the orchestrator_fe (the dashboard angular application) ###

- Change the values in the **environment** file to point to the orchestrator url.

And that's pretty much it. Your dashboard (fe) should be able to contact the orchestrator.
The orchestrator will allow you to send dto payloads, with the task type, and multiplex the requests to the scheduling engine of the relative microservice, while handling the start/stop/update.


-----------------------------------------------------------------------------------


## How the system works ##

*TODO (generated tables explanation, column explanation, task status, diagram and explanation, adding 1-C and input/output)*



 
