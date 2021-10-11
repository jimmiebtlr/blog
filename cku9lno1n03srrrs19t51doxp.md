## 9 tips for writing better micro-services

### 1 - Stateless services

Stateless services are trivial to scale compared to stateful services.  Be mindful of where you persist state and why.   To achieve this you'll generally store state in a database. 

Don't write to the file system in your services unless that's the sole purpose of your service (as in your service is a database).


### 2 - Infrastructure as Code

Micro-services can turn into a nightmare to manage at an infrastructure level.  To document this and help manage it, use infrastructure as code. 

Infrastructure as code is a way of specifying how your service is deployed in a machine readable format, that can then be used to re-deploy on entirely new infrastructure, make updates, and otherwise just make it easier for the next guy.

Currently I prefer to use Terraform, however there are many great options including Palumi, AWS CDK, Google Deployment manager.



### 3 - Fail fast when missing environmental variable

Don't set defaults in your code that apply for development envs only.  Instead fail the service immediately when a variable that would be required in production is missing, and use something outside the code to set defaults for your environment.  

Crashing the service when missing variables allows your infrastructure to notice and keep a previous version of the service running until you can fix it.

Otherwise you may end up with a service in production looking in the wrong place for your new database or service.



### 4 - Backward compatible API's

Don't break everything calling your service when making changes.  To cleanup API's, this should be done by having an old and new version of your API so other services can be changed to the new API without interruption.



### 5 - Isolated state storage

You inevitably have state to store however, it's best to isolate where you store your state per service.  This helps by making it easier to 

- Scale your state storage
- Make appropriate database choices (you may even want to opt for several databases used by one services).



### 6 - Health endpoint

Where you deploy your service should check that the service is live before directing traffic to it, as well as checking that it's still alive periodically.  To do this you'll need a health endpoint that can tell your infrastructure that the service is still running and ready to serve traffic.

### 7 - Deployment infrastructure choice

There are many great options for this, really the only wrong choice would be to throw them on a VM somewhere, undocumented, unmaintained, without monitoring, log collection, etc.

I prefer services that work with docker over cloud specific containerization.   My favorite are google cloud run or kubernetes (preferably gke).



### 8 - Use message queues (but not too much)   📞vs📨

When you can, communicate service-service with message queues. This helps make your system more robust by letting services manage how quickly they're processing requests, and storing the data for later if they go down entirely.

Don't be over zealous however, as there are plenty of times you'll need synchronous responses.


### 9 - Database selection

The best database is the one you and your team know how to use and scale to the level you need.  This said, some of the managed cloud databases can make things much easier to manage in production, eg google datastore.




#### Comment below with what I've missed!