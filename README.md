# 3scale / Fuse SOAP and SOAP to REST demo


# TODO
# TODO - delete my URLS


## In this demo we 
- setup 3scale on OpenShift
- setup a simple SOAP service on OpenShift
- manage that SOAP service in 3scale - coarse grained
- manage that SOAP service in 3scale - fine grained
- setup a RESTful interface to our SOAP service (through Fuse transformation)  on OpenShift
- manage that REST API in 3scale

Let's get started.

----------------------------------------------------------------------------------------------------

## Setup 3scale on OpenShift

First let's install 3scale 2.9 on OpenShift. There are various options around 
- operator or template
- storage

For simplicity, I'll be using the 3scale operator and S3 for storage. I will provide requisite instructions to do this on this README, but for more details see [deploying-threescale-using-the-operator](https://access.redhat.com/documentation/en-us/red_hat_3scale_api_management/2.9/html/installing_3scale/install-threescale-on-openshift-guide#deploying-threescale-using-the-operator) 
### Pre-requisites
- in my case, an S3 implemenation. I'll use Amazon S3.
- in my case, I'll use the Red Hat producised 3scale, for which you'll need an account at [https://access.redhat.com](https://access.redhat.com/) to pull the supported, productised images. You can alternatively use the Community operator for which no Red Hat credentials are required.
- an OpenShift 4 cluster - in my case 4.5 with Administrative access.
- the _oc_ client installed locally (e.g. on your laptop) logged in as an Administrator to OpenShift.
- this repo cloned - and _cd_ into it.

Setup this environment variable to be the home of this repo on your laptop.
```
export REPO_HOME=`pwd`
```

### 3scale setup instructions
Execute the following
```
oc new-project 3scale
```
Modify $REPO_HOME/1-3scale-setup/secret-s3.yaml with your actuals under _stringData_ and execute:
```
oc apply -f $REPO_HOME/1-3scale-setup/secret-s3.yaml
```
For more on S3 for 3scale storage see [S3 3scale storage](https://access.redhat.com/documentation/en-us/red_hat_3scale_api_management/2.9/html/installing_3scale/install-threescale-on-openshift-guide#amazon_simple_storage_service_3scale_emphasis_filestorage_emphasis_installation)


Create a secret using your [https://access.redhat.com](https://access.redhat.com/) credentials
```
oc create secret docker-registry threescale-registry-auth \
--docker-server=registry.redhat.io \
--docker-username="yourusername" \
--docker-password="yourpassword"
```

On the OpenShift web console, select the 3scale project. Navigate to _Operators->Operator Hub_. Search for _3scale_ and select the _Red Hat Integration - 3scale_ operator

![](https://github.com/tnscorcoran/3scale-soap-2-rest/blob/master/_images/2-3scale-operator.png)

Install this operator, going with the defaults on the next screen.
![](https://github.com/tnscorcoran/3scale-soap-2-rest/blob/master/_images/3-install-3scale-operator.png)

Your display will change to _Installed Operators_. A couple of minutes later the staus should be _Succeeded_.

Select the _Red Hat Integration - 3scale_ operator.
![](https://github.com/tnscorcoran/3scale-soap-2-rest/blob/master/_images/4-installed-3scale-operator.png)

Click _Create Instance_ on the _API Manager_ box. You need to overwrite the yaml that's on the screen. Overwrite with your modified __REPO_HOME/1-3scale-setup/threescale.yaml__ . You'll just need to modify what's highlighted - which you can get in the address on your brower tab that's logged into OpenShift:
![](https://github.com/tnscorcoran/3scale-soap-2-rest/blob/master/_images/5-threescale.yaml.png)

Copy it in, overwriting what's there and Click _Create_. Then navigate to _Workloads->Pods_. A few minutes later all pods should be Running and 1/1 or 3/3 under Ready.

Now you need to retrieve your credentials for 3scale. Go to _Workloads->Secrets_. Open the _system-seed_ secret, click _Reveal Values_ on the right and see your ADMIN_PASSWORD. Your ADMIN_USER will also be needed but it will be _admin_. Keep note of both.
![](https://github.com/tnscorcoran/3scale-soap-2-rest/blob/master/_images/6-reveal-system-seed-secret.png)

Now time to open the 3scale Admin console. Go to _Networking->Routes_ and open the _zync-3scale-provider-xxxxxxx_ Route. 
![](https://github.com/tnscorcoran/3scale-soap-2-rest/blob/master/_images/7-3scale-admin-route.png)

Use your ADMIN_USER and ADMIN_PASSWORD credentials from the previous step and log in.

----------------------------------------------------------------------------------------------------

## Setup a simple SOAP service on OpenShift

We use a different repo for the SOAP and Fuse projects:
__https://github.com/weimeilin79/3scale_development_labs.git__

Clone that repo - and _cd_ into it.

Overwrite this environment variable to be the home of this __new__ repo on your laptop.
```
export REPO_HOME=`pwd`
```

We're going to need to use our OpenShift wildcard domain - so let's export a variable for it. It will be something like __apps.[your openshift hostname].com__ and will be retriavable from the address bar of the browser tab where you're logged into OpenShift. Execute this with yours:
```
export OCP_WILDCARD_DOMAIN=apps.[your openshift hostname].com
```

Setup a new project:
```
oc new-project soap-rest
```

Import the stores-api template into your OpenShift environment:
```
oc create -f $REPO_HOME/templates/stores-api.json
```

Create the new application using the stores-api template:
```
oc new-app --template=stores-soap --param HOSTNAME_HTTP=stores-api.$OCP_WILDCARD_DOMAIN
```

Run this occasionally (maybe every 30 seconds) until: 1)stores-soap-1-xxx and 2)storesdb-1-xxx are both Ready 1/1:
```
oc get pods
```

Test Stores SOAP Service by navigating to _Networking->Routes_ in the OpenShift web console. Copy the location URL and append _/StoresWS?wsdl_ to it and test that in a browser. e.g. mine was:

_http://stores-api.apps.cluster-datagrid-7970.datagrid-7970.sandbox961.opentlc.com/StoresWS?wsdl_

The WSDL for this service should appear in your browser.

There are various services out there for generating sample SOAP requests from a WSDL file. You can use one of those but for simplicity, here is one for the _getAllStores_ operation on this SOAP service:
```
<soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/" xmlns:stor="http://www.rhmart.com/Stores/">
<soapenv:Header/>
<soapenv:Body><stor:getAllStores/></soapenv:Body>
</soapenv:Envelope>
```
Test it out in your favour SOAP API testing tool. I use postman:
![](https://github.com/tnscorcoran/3scale-soap-2-rest/blob/master/_images/8-postman-soap-direct.png)


----------------------------------------------------------------------------------------------------

## Manage that SOAP service in 3scale - coarse grained

First we need to create the API Gateway Staging Route. To begin with we'll set an environmental variable to hold our 3scale OpenShift project:
```
export GW_PROJECT=3scale
```

====================In OpenShift, select the 3scale project
In 3scale, which you should already have logged into above, select _Dashboard_ then _Backends_

![](https://github.com/tnscorcoran/3scale-soap-2-rest/blob/master/_images/10-3scale-dashboard-backends.png)

Click NEW BACKEND. Populate the form with the following details:
```
Name:               Stores SOAP Policy Backend
System Name:        stores-soap-policy-backend
Description:        Stores SOAP Policy Backend
Private Endpoint:   Populate with the output of the following:
                    echo -en "\n\nhttp://stores-soap.$OCP_USERNAME-stores-api.svc.cluster.local:8080\n"

```






















----------------------------------------------------------------------------------------------------



----------------------------------------------------------------------------------------------------

- Spring Boot
- Quarkus in JVM mode
- Quarkus in Native mode
We also briefly demonstrate Quarkus' live code updating capabilities.

We then push our native application as container to the popular Quay container registry and from there we pull it into a KNative Serverless application running on OpenShift.

## Prerequisites
To run the first part of this demo, you'll need Java (I use version 8), Maven and GraalVM installed.
To run the second part, you'll need an OpenShift 4.4 cluster. I recommend [Codeready Containers](https://developers.redhat.com/products/codeready-containers/overview) for a local cluster or [try.openshift.com](https://www.openshift.com/try) for a full production ready cluster. You'll also need access to a public container registry. I use the excellent free one from [http://quay.io](http://quay.io).


## Part 1 - Spring Boot / Quarkus comparison

## Steps

Clone this repo and change to its directory. Then: 
```
export REPO_HOME=`pwd`
```

## 1 - Spring Boot
```
cd $REPO_HOME/springboot-hello-world
mvn package
java -jar target/springboot-hello-world-1.0.0.jar
```

Take note of the startup time - you can see mine took 2.258 seconds
![](https://github.com/tnscorcoran/springboot-quarkus-serverless/blob/master/images/01-springboot-startup.png)

Also take note of the memory consumption - under rss (Resident Set Size) in the following command
```
ps -o pid,rss,command -p $(pgrep -f springboot)
```
-  you can see mine consumed 629,024k of memory
![](https://github.com/tnscorcoran/springboot-quarkus-serverless/blob/master/images/02-springboot-rss.png)

And of course - test your application
```
curl http://localhost:8080/greeting
```

Stop Spring Boot by clicking CTRL + c


## 2 - Quarkus JVM Mode
```
cd $REPO_HOME/quarkus-hello-world
mvn package
```

Now run it
```
java -jar target/quarkus-hello-world-1.0-SNAPSHOT-runner.jar
```

Take note of the startup time -  you can see mine took 0.667 seconds
![](https://github.com/tnscorcoran/springboot-quarkus-serverless/blob/master/images/03-quarkus-jvm-startup.png)

I created a sheet that shows
- how many times faster Quarkus is to start up than Spring Boot (in both JVM and Native mode).
- what percentage of Spring Boot's memory requirement Quarkus has.

Quarkus' 0.667 seconds startup time is more than 3 times faster than Spring Boot's 2.258 seconds - for an identical application:
![](https://github.com/tnscorcoran/springboot-quarkus-serverless/blob/master/images/04-quarkus-jvm-versus-spring-boot-startup.png)

Now take note of the memory consumption, rss:
```
ps -o pid,rss,command -p $(pgrep -f runner)
```

You can see mine consumed 158,696k of memory
![](https://github.com/tnscorcoran/springboot-quarkus-serverless/blob/master/images/05-quarkus-jvm-rss.png)



Quarkus used about a quarter of Spring Boot's memory for effectively the same application:


![](https://github.com/tnscorcoran/springboot-quarkus-serverless/blob/master/images/06-quarkus-jvm-versus-spring-boot-memory.png)

And again - verify your application
```
curl http://localhost:8080/greeting
```


Stop Quarkus by clicking CTRL + c


















## 3 - Quarkus Native Mode

Stay in directory $REPO_HOME/quarkus-hello-world
Now compile the application down to a native image using GraalVM (for instructions on how to set this up - go to https://quarkus.io/guides/building-native-image)

```
mvn package -Pnative -Dquarkus.native.container-build=true
```

Now after *mvn package* run it:
```
./target/quarkus-hello-world-1.0-SNAPSHOT-runner
```

Again take note of the startup time -  you can see mine took 0.012 seconds
![](https://github.com/tnscorcoran/springboot-quarkus-serverless/blob/master/images/07-quarkus-native-startup.png)

That's nearly 200 times faster than Spring Boot - for an identical application:
![](https://github.com/tnscorcoran/springboot-quarkus-serverless/blob/master/images/08-quarkus-native-versus-spring-boot-startup.png)

Now take note of the memory consumption, rss:
```
ps -o pid,rss,command -p $(pgrep -f runner)
```

You can see mine consumed 18,258k of memory
![](https://github.com/tnscorcoran/springboot-quarkus-serverless/blob/master/images/09-quarkus-native-memory.png)



Quarkus in Native mode uses about 3% of Spring Boot's memory for this hello world application:

![](https://github.com/tnscorcoran/springboot-quarkus-serverless/blob/master/images/10-quarkus-native-versus-spring-boot-memory.png)

And again - test your application
```
curl http://localhost:8080/greeting
```

Stop Quarkus by clicking CTRL + c

ps -o pid,rss,command -p $(pgrep -f runner)

# Bonus - Live code update
This demo focuses on the startup-time and memory advantages of Quarkus over a traditional cloud native stack like Spring Boot.
Quarkus has several another benefits - like the ability to combine imperative and reactive programming in the same application, user friendly error reporting, and a superb extension network. But a massive benefit of Quarkus is *live code updates*. When running in *dev* mode, to see code, package, maven dependency changes, all you need to do is save the file - no need to rebuild.

Let's test this out. Execute the following to startup in *dev* mode
```
cd $REPO_HOME/quarkus-hello-world
 ./mvnw compile quarkus:dev
``` 
Test out the app:
```
curl http://localhost:8080/greeting
```

Now, keep the app running but make a change to a source file - say
this file's default message:
![](https://github.com/tnscorcoran/springboot-quarkus-serverless/blob/master/images/11-modified-Greeting.png)

Save the file. Test it again:
```
curl http://localhost:8080/greeting
```
Your application now reflects the change you did without a rebuild!


## Part 2 - Quarkus in Native mode - as a serverless workload on Kubernetes/OpenShift

## Steps
```
cd $REPO_HOME/quarkus-hello-world
```
First we need to package our native application into a container image using the provided Dockerfile (/quarkus-hello-world/Dockerfile.native) and push it to our remote container registry. First login to your remote container registry, using _podman login_ or _docker login_ then execute the following or your equivalent according how you want to tag and name your repo:
```
docker build -f ./Dockerfile.native -t <registry-username>/<repo-name>:latest .
docker tag <registry-username>/<repo-name>:latest <registry>/<registry-username>/<repo-name>:latest
docker push <registry>/<registry-username>/<repo-name>:latest
```
or in my case:
```
docker build -f ./Dockerfile.native -t tnscorcoran/quarkus-serverless:latest .
docker tag tnscorcoran/quarkus-serverless:latest quay.io/tnscorcoran/quarkus-serverless:latest
docker push quay.io/tnscorcoran/quarkus-serverless:latest
```


On [http://quay.io](http://quay.io), I label my new repo _quarkus-serverless_ with _latest_ 

![](https://github.com/tnscorcoran/springboot-quarkus-serverless/blob/master/images/12-tag-image-latest.png)


When I then make this new repo public, as shown, it will be available to pull into my cluster.

![](https://github.com/tnscorcoran/springboot-quarkus-serverless/blob/master/images/13-make-repo-public.png)

Next login to your OpenShift cluster as an adminstrator. 

Now we're going to provision our Serverless Operator, which will allow us to create a new _KNative_ serverless runtime for our application. 

Go to Operators -> Operator Hub -> search for _serverless_ and choose the OpenShift Serverless Operator:

![](https://github.com/tnscorcoran/springboot-quarkus-serverless/blob/master/images/15-OpenShift-Serverless-Operator.png)

Click Install

![](https://github.com/tnscorcoran/springboot-quarkus-serverless/blob/master/images/16-install-OpenShift-Serverless-Operator.png)

Click Update Channel 4.4 and Subscribe.

![](https://github.com/tnscorcoran/springboot-quarkus-serverless/blob/master/images/17-subscribe.png)

We need a project / namespace called _knative-serving_ (it needs that name). Create as follows

![](https://github.com/tnscorcoran/springboot-quarkus-serverless/blob/master/images/18-new-project-knative-serving.png)

With your new project _knative-serving_ selected, we will deploy the knative serving _API_. As follows

![](https://github.com/tnscorcoran/springboot-quarkus-serverless/blob/master/images/19-new-knative-serving.png)

Move to Workloads -> Pods and wait until all are ready and running:

![](https://github.com/tnscorcoran/springboot-quarkus-serverless/blob/master/images/20-knative-serving-pods-ready.png)


Next we're going to pull in our Quarkus container image and run it in _Serverlesss_ mode. To house our new Quarkus Serverless application, create a new namespace/project, in my case I call it _quarkus-serverless_:

![](https://github.com/tnscorcoran/springboot-quarkus-serverless/blob/master/images/14-new-project.png)

Now it's time to pull in our Quarkus image in Serverless mode. Change to the Developer perpective, choose Topology and create an application from a container image as shown:

![](https://github.com/tnscorcoran/springboot-quarkus-serverless/blob/master/images/21-developer-perspective-create-from-container-image.png)

Choose the registry/repository you created earlier, in my case _quay.io/tnscorcoran/quarkus-serverless_. Choose _KNative Service_ and accept the other defaults:

![](https://github.com/tnscorcoran/springboot-quarkus-serverless/blob/master/images/22-knative-app-create.png)

*Note if you want you can modify the scaling defaults using the _Scaling_ link at the bottom of the screen. By default it scales to zero after some seconds, which is great for saving cloud costs and we'll experience below.*

After a few seconds the application is available - as indicated by the sold blue circle. Click on the URL link
![](https://github.com/tnscorcoran/springboot-quarkus-serverless/blob/master/images/23-app-deployed.png)

Then append *_/greeting_* and you can see the JSON payload returned from this simple API endpoint:

![](https://github.com/tnscorcoran/springboot-quarkus-serverless/blob/master/images/24-json-payload.png)

Finally after about 30 seconds, the application scales to zero as shown. Any request will waken it very quickly thanks to Quarkus Native's rapid startup times you saw earlier.
![](https://github.com/tnscorcoran/springboot-quarkus-serverless/blob/master/images/25-scale-to-zero.png)


## Summary

Quarkus is a new Open Source Red Hat sponsored Java framework. It's designed with cloud native development and Kubernetes in mind.

It's got radically lower memory and faster startup than traditional cloud native Java (we used Spring Boot to represent that). The advantages are most pronounced in Native mode - making it an ideal candidate for Serverless workloads. Serverless and scale to zero can dramatically reduce your cloud infrastructre costs as machines can be powered down to zero in quiet periods.

Quarkus in JVM mode, also advantages are still significant - making it the best choice for longer lived applications where JVM capabilities, in particular Garbage Collection, are needed.

We demonstrated the following 
- 2 applications, both identical in functionality; one Quarkus and one Spring Boot
- the compartive startup times and memory consumption in a) Spring Boot, b) Quarkus in JVM mode and c) Quarkus in Native mode
- a really cool feature of Quarkus - live code update.
- an outstanding use case of Quarkus Native mode - serverless on Kubernetes. We used the industry leading Kubernetes distribution _Red Hat OpenShift_.

To learn more about Quarkus visit [Red Hat build of Quarkus](https://developers.redhat.com/products/quarkus/getting-started) and [https://quarkus.io](https://quarkus.io)

This demo can be viewed on You Tube at [https://youtu.be/bMkcVYTW8Z8](https://youtu.be/bMkcVYTW8Z8).
