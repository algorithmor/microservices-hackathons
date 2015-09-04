# Microservices examples


This is a small project to show how to build microservices and configure them to communicate using Eureka, Ribbon (and Prana when applicable) from Netflix OSS stack.

## Introduction

The microservices built are very simple, and its goal is to show the basic structure of an API microservice.

On each project, there is also a set of scripts that allow to build an Amazon AMI with **Packer**. So, at the end, we will have an Amazon image for each microservice we want to populate.

The AMI generation is done automatically using the **Travis-CI** tool.   
The `.travis.yml` file contains all the required steps to compile the code (pass the tests if it has), build the distro and generate the Amazon image with the created microservice.

## Project content

In this project you will find the template microservices we have developed. 

- `got-quotes-microservice-python`: Game of Thrones simple quotes provider written in Python using Flask. 
- `got-quotes-microservice-go`: Game of Thrones simple quotes provider written in Go.
- `got-quotes-microservice-java`: Game of Thrones simple quotes provider written in Java.
- `top-quotes-microservice-java`: Written also in Java, it asks for the most returned quote to the `got-*` services (notice that it can/will ask to any of the previous services, independently of the language).

> **Disclaimer:** the result of the *trending* (most returned) quote may vary depending on the microservice asked, since the counter is stored internally on each instance, not centralized into any database, for the sake of simplicity.

*On each available project, usually only the source code should be modified. None of the configuration files nor deployment ones.*

## How to clone the language microservice we want

Each repo will contain a microservice example which is one of the `got-*` projects available.

In case you want to change that microservice and use another of the example, or even download the `top-*` project because you want to communicate one service with another and are willing to use our example, please contact us and we will help you.


## Communication schema

All the projects that are *not* written in Java use the **Prana** library to expose their APIs and to communicate with NetflixOSS' **Eureka** service discovery server.

The `top-quote-microservice-java` project uses **Ribbon** to define and implement the interface of communication with the `got-*` projects. (In the ideal world, the `got-*` projects would be the ones exposing their API, but it needs more hardcore implementation not suitable for this purpose).

In the following schema, the basic communication intra microservices is exposed. Note the key point of the Eureka module (service discovery).

```                                                                                                 
              +------------------+                           
              | SERVICE          +---------------------+     
              | TOP QUOTE (JAVA) |                     |     
              |         +--------+                     |     
              |         | RIBBON |                     |     
              +---------+----+---+                     v     
                             |                   +----------+
                             |                   |          |
                             |                   |  EUREKA  |
      +----------------+-----+-----------+       |          |
      |                |                 |       +----------+
      |                |                 |          ^  ^  ^ 
      |                |                 |          |  |  |  
      v                v                 v          |  |  |  
+-----------+    +-----------+     +-----------+    |  |  |  
|           |    |  PRANA    +--+  | PRANA     +----+  |  |  
|           |    +-----------+  |  +-----------+       |  |  
| SERVICE   |    | SERVICE   |  |  | SERVICE   |       |  |  
| GOT QUOTE |    | GOT QUOTE |  |  | GOT QUOTE |       |  |  
| [JAVA]    |    | [PYTHON]  |  |  | [GO]      |       |  |  
+---------+-+    +-----------+  |  +-----------+       |  |  
          |                     |                      |  |  
          |                     +----------------------+  |  
          |                                               |  
          +-----------------------------------------------+  
```

## Available endpoints

### In `got-*` projects

**Get quote by id:**

```
GET 	/api/quote/<quoteid>

Response:
{
    "quote": STRING,
    "counter": INTEGER
}
```

**Get random quote:**

```
GET 	/api/quote/random

Response:
{
    "quote": STRING,
    "counter": INTEGER
}
```

**Get top quote: (most returned)**

```
GET 	/api/quote/top

Response:
{
    "quote": STRING,
    "counter": INTEGER
}
```

_**Error response**_: Response applied only in case the requested quote doesn't exist.

```
Response:
{
    "status": 404,
    "message": "Quote not found!"
}
```

### In `top-quotes-microservice-java`:

**Get trending quote: (most returned)**

```
GET 	/api/quote/trending

Response:
{
    "trend_quote": STRING,
    "counter": INTEGER
}
```


## Trying the microservices

### How to run it locally without Eureka

The first test we can execute is to run the both microservices locally, witouth using Eureka. For that we'll need to specify directly the host and port to connect from one service to another.

In this example we will be changing `GotQuotes` port and, therefore, configure `TopQuotes` to access to that service/port.

**Actions to follow to run it locally:**

- In *GotQuotes'* `AppServer.java` code, comment two modules:
 -  `ShutdownModule.class`
 -  `KaryonWebAdminModule.class`

```java
@Modules(include = {
//        ShutdownModule.class,
//        KaryonWebAdminModule.class,
	      KaryonEurekaModule.class,
          AppServer.KaryonRestRouterModuleImpl.class
})
```

> **Note**: The code is commented for simpliciy, but could be configured to be running for the both services in the same machine, only the ports must be specified for each module.

- In *GotQuotes'* `AppServer.java` code, change the `DEFAULT_PORT` to `8081`.
- In *TopQuotes'* `AppServer.properties` file, comment the following property line:
 - `got-quotes-service-client.ribbon.NIWSServerListClassName`

```
# If uncommented, the service discovery Eureka will be used.
# If commented, ribbon will look for the other service at "listOfServers" specified urls:ports.
# got-quotes-service-client.ribbon.NIWSServerListClassName=com.netflix.niws.loadbalancer.DiscoveryEnabledNIWSServerList
```

- In *TopQuotes'* `AppServer.properties` file, configure the GotQuotes service address/port:

```
# Initial list of servers, can be changed via Archaius dynamic property at runtime
got-quotes-service-client.ribbon.listOfServers=localhost:8081
```

Once everything is set up, we can launch both services using the following command for each one:

```bash
$ ./gradlew runApp
```

and test them locally accessing to their available endpoints (i.e. `http://localhost:8081/api/quote/random`)


### How to run it locally with Eureka

We can also try to run both microservices locally but relying on Eureka for the service discovery.
The Eureka used will be the hackathon one, which is you can see it's console on: `http://eu-west-1.eureka.hackathon.schibsted.io:8080/eureka-server/`

> **Note**: the console shows which microservices are running and their amazon instances identification.
 
**Actions to follow to run it locally:**

- In *GotQuotes'* `AppServer.java` code, comment two modules:
 -  `ShutdownModule.class`
 -  `KaryonWebAdminModule.class`

```java
@Modules(include = {
//        ShutdownModule.class,
//        KaryonWebAdminModule.class,
          KaryonEurekaModule.class,
          AppServer.KaryonRestRouterModuleImpl.class
})
```

- In *GotQuotes'* and *TopQuotes'* `eureka-client.properties`:
 - `eureka.serviceUrl.default` variable should be set to `http://${EC2_REGION}.eureka.${CLOUD_ENVIRONMENT}.schibsted.io:8080/eureka-server/v2/`

- In *TopQuotes'* `AppServer.properties`:
 - The variable `got-quotes-service-client.ribbon.listOfServers` defined to `localhost:8081` can be ignored since it won't be used.
 - The variable `got-quotes-service-client.ribbon.NIWSServerListClassName` must be **uncommented** and defined as `com.netflix.niws.loadbalancer.DiscoveryEnabledNIWSServerList`
 
> Note: we rely on Eureka's service discovery, so we don't need to specify any direct host URL as we did in the previous local test.

Once everything is set up, we can launch both services using the following command for each one:

```bash
$ ./gradlew runApp
```

After starting the application, the system needs some time to be alive and registered into Eureka's server. It takes around 90 seconds (3 Eureka heartbets correctly answered in a row).  
The trace will look something similar to:

```
...
13671 [main] INFO com.netflix.appinfo.providers.EurekaConfigBasedInstanceInfoProvider - Setting initial instance status as: STARTING
13677 [main] WARN com.netflix.config.util.ConfigurationUtils - file:/Users/rsole/schibsted/hackathons/microservices-java/got-quotes/build/resources/main/eureka-client.properties is already loaded
14504 [main] INFO com.netflix.discovery.DiscoveryClient - Disable delta property : false
14504 [main] INFO com.netflix.discovery.DiscoveryClient - Single vip registry refresh property : null
14504 [main] INFO com.netflix.discovery.DiscoveryClient - Force full registry fetch : false
14504 [main] INFO com.netflix.discovery.DiscoveryClient - Application is null : false
14504 [main] INFO com.netflix.discovery.DiscoveryClient - Registered Applications size is zero : true
14504 [main] INFO com.netflix.discovery.DiscoveryClient - Application version is -1: true
15117 [main] INFO com.netflix.discovery.DiscoveryClient - Getting all instance registry info from the eureka server
15201 [main] INFO com.netflix.discovery.DiscoveryClient - The response status is 200
15204 [main] INFO com.netflix.discovery.DiscoveryClient - Starting heartbeat executor: renew interval is: 30
12877 [main] INFO com.netflix.discovery.DiscoveryClient - Starting heartbeat executor: renew interval is: 30
42983 [pool-3-thread-1] INFO com.netflix.discovery.DiscoveryClient - DiscoveryClient_TOP_QUOTES/192.168.1.105 - Re-registering apps/TOP_QUOTES
42984 [pool-3-thread-1] INFO com.netflix.discovery.DiscoveryClient - DiscoveryClient_TOP_QUOTES/192.168.1.105: registering service...
43047 [pool-3-thread-1] INFO com.netflix.discovery.DiscoveryClient - DiscoveryClient_TOP_QUOTES/192.168.1.105 - registration status: 204
55988 [DiscoveryClient-2] INFO com.schibsted.hackatons.example.topquotes.common.health.HealthCheck - Health check invoked.
55988 [DiscoveryClient-2] INFO com.netflix.discovery.DiscoveryClient - DiscoveryClient_TOP_QUOTES/192.168.1.105 - retransmit instance info with status UP
55988 [DiscoveryClient-2] INFO com.netflix.discovery.DiscoveryClient - DiscoveryClient_TOP_QUOTES/192.168.1.105: registering service...
56050 [DiscoveryClient-2] INFO com.netflix.discovery.DiscoveryClient - DiscoveryClient_TOP_QUOTES/192.168.1.105 - registration status: 204
73145 [pool-3-thread-1] INFO com.netflix.discovery.DiscoveryClient - DiscoveryClient_TOP_QUOTES/192.168.1.105 - Re-registering apps/TOP_QUOTES
73146 [pool-3-thread-1] INFO com.netflix.discovery.DiscoveryClient - DiscoveryClient_TOP_QUOTES/192.168.1.105: registering service...
73200 [pool-3-thread-1] INFO com.netflix.discovery.DiscoveryClient - DiscoveryClient_TOP_QUOTES/192.168.1.105 - registration status: 204
92148 [DiscoveryClient-1] INFO com.schibsted.hackatons.example.topquotes.common.health.HealthCheck - Health check invoked.
131255 [DiscoveryClient-0] INFO com.schibsted.hackatons.example.topquotes.common.health.HealthCheck - Health check invoked.
170382 [DiscoveryClient-3] INFO com.schibsted.hackatons.example.topquotes.common.health.HealthCheck - Health check invoked.
209486 [DiscoveryClient-1] INFO com.schibsted.hackatons.example.topquotes.common.health.HealthCheck - Health check invoked.
248590 [DiscoveryClient-2] INFO com.schibsted.hackatons.example.topquotes.common.health.HealthCheck - Health check invoked.
287684 [DiscoveryClient-0] INFO com.schibsted.hackatons.example.topquotes.common.health.HealthCheck - Health check invoked.
312751 [DiscoveryClient-3] INFO com.netflix.discovery.DiscoveryClient - Updating the serviceUrls as they seem to have changed from [http://ec2-52-16-39-174.eu-west-1.compute.amazonaws.com:8080/eureka-server/v2/, http://ec2-52-17-161-151.eu-west-1.compute.amazonaws.com:8080/eureka-server/v2/, http://ec2-52-16-128-229.eu-west-1.compute.amazonaws.com:8080/eureka-server/v2/] to [http://ec2-52-16-39-174.eu-west-1.compute.amazonaws.com:8080/eureka-server/v2/, http://ec2-52-16-128-229.eu-west-1.compute.amazonaws.com:8080/eureka-server/v2/, http://ec2-52-17-161-151.eu-west-1.compute.amazonaws.com:8080/eureka-server/v2/] 
326776 [DiscoveryClient-0] INFO com.schibsted.hackatons.example.topquotes.common.health.HealthCheck - Health check invoked.
365889 [DiscoveryClient-1] INFO com.schibsted.hackatons.example.topquotes.common.health.HealthCheck - Health check invoked.
...
```

...after this point, we have the services up and running and detected by Eureka.

**Important**: The `eureka-client.properties` file's port must match with the defined in the `DEFAULT_PORT` at `AppServer.java`.:
 
- `eureka-client.properties`:  

```
...
#The port where the service will be running and serving requests
eureka.port=8080
...
```

- `AppServer.java`:  

```
...
static final int DEFAULT_PORT = 8081;
...
```

### How to run it in AWS with

XXXXX XXXXX XXXXXXXXXX XXXXX XXXXXXXXXX XXXXX XXXXXXXXXX XXXXX XXXXX  
XXXXX XXXXX XXXXXXXXXX XXXXX XXXXXXXXXX XXXXX XXXXXXXXXX XXXXX XXXXX  
Execute autodeploy script????


