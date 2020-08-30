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
Test it out in your favourite SOAP API testing tool. I use postman:
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

### Create Backend
Click NEW BACKEND. Populate the form with the following details:
```
Name:               Stores SOAP Policy Backend
System Name:        stores-soap-policy-backend
Description:        Stores SOAP Policy Backend
Private Endpoint:   Populate with the output of the following:
                    echo -en "\n\nhttp://stores-soap.soap-rest.svc.cluster.local:8080\n"

```
In my case the _Private Endpoint_ was
```
http://stores-soap.soap-rest.svc.cluster.local:8080
```
This is a security benefit associated with backends on OpenShift. This _Service_ (svc.cluster.local) URL is not accessible outside OpenShift. So you need never expose your backend API to the outside world - only the secured route to it via 3scale.


### Update Product
For simplicity, we're going to use our existing Product, called API. From the drop down menu on the top left hand side of the screen, choose _API_ which will cause the _Product: API_ to be selected.
![](https://github.com/tnscorcoran/3scale-soap-2-rest/blob/master/_images/11-api-product.png)

Choose _Integration->Backends_ then delete the API Backend that's there then _Add Backend_
![](https://github.com/tnscorcoran/3scale-soap-2-rest/blob/master/_images/12-delete-add-backends.png)

Add our SOAP Backend to our _API_ product with a path of _/_as follows:
![](https://github.com/tnscorcoran/3scale-soap-2-rest/blob/master/_images/13-add-backend-to-product.png)

No you need to create a _*3scale method*_, so as to be able to record and control, rate limit etc, these SOAP requests. 

From the top panel, navigate to: API -> Integration -> Methods & Metrics.

Click New Method and populate with the following values:
```
Friendly name:  StoresWS
System name:    stores/storesws
Description:    Stores SOAP Web Service
```
A method will be used to track the number of hits on the SOAP API. You’ll see later in this lab how this method is not granular enough to track the number of hits on each SOAP operation.

Next we need to map incoming request to this new method. From the top panel, navigate to: API -> Integration -> Mapping Rules. Click _Add Mapping Rule_ and create one as follows:
```
Verb:               POST
Pattern:            /StoresWS
Metric or Method:   StoresWS
Increment:          1
```

Now move to the _Configuration_ screen and promote this change to our Nginx based _APICast_ gateway:
![](https://github.com/tnscorcoran/3scale-soap-2-rest/blob/master/_images/14-promote-apicast.png)

Immediately, a sample CURL comand will appear. Copy this for later use:
![](https://github.com/tnscorcoran/3scale-soap-2-rest/blob/master/_images/15-apicast-curl.png)

Before we test our newly managed SOAP API, we need to change a setting on our Staging APIcast gateway. In order for it to perdiodically check the manager configuration every 5 seconds, and reconfigure itself if its configuration has changed. DO the following.
- In OpenShift, select the 3scale project.
- Navigate to Workloads-> Deployment Configs -> apicast-staging -> Environment tab. Make the following change:
```
APICAST_CONFIGURATION_LOADER:           boot
APICAST_CONFIGURATION_CACHE:            5
```
![](https://github.com/tnscorcoran/3scale-soap-2-rest/blob/master/_images/16-dc-apicast.png)

Save it - and it will recreate the pod.

Now go to your favourite SOAP API testing tool, Postman in my case and paste in the curl command just copied. Append the path _/StoresWS_ and test out the API. All going well you now have an authenticated/authorised API!
![](https://github.com/tnscorcoran/3scale-soap-2-rest/blob/master/_images/17-postman-apicast.png)


### Manage that SOAP service in 3scale - fine grained
If we want to track, control access, rate limit etc, individual operations, we need a further step.

To do this we use the SOAPAction header on the client side and use extra method(s) and the SOAP APICast policy on the manager (which gets propagaed to APICast as configured earlier with our 5 second interval).

I'm going to use the _Get All Stores_ operation as an example.

### Create new method
From the top panel, navigate to: Stores SOAP Policy API -> Integration -> Methods & Metrics.
Click New Metric and populate as follows:
```
Friendly name:  GetAllStores
System Name:    Stores/getAllStores
Unit:           hits
```

Now we configure the SOAP Policy. Navigate to: Stores SOAP Policy API → Integration → Configuration → edit APIcast configuration. Expand the Policies section. In the Policy Chain section, click Add Policy and select SOAP.
In the Policy Chain section, use the up or down arrows to reorder the policies so that the SOAP policy is first, followed by the APIcast policy. Enter the following mapping rule:
```
Delta:              1
metric_system_name: Stores/getAllStores
Pattern:            /Stores/getAllStores
```
Now on the same 3scale screen, update your Policy, then update your Policy Chain.
Next on the menu on the left, go to Integration -> Configuration and _Promote to Staging_ as you did above. 
5 seconds later, your APICast gateway will have pulled down your configuration of fine grained SOAP API Management.

Now go to your favourite SOAP API testing tool, Postman in my case. Create new request, a copy of the previous coarse grained APICast request, this time passing in the SOAPAction header _http://www.rhmart.com/Stores/getAllStores_. Make the request as shown:
![](https://github.com/tnscorcoran/3scale-soap-2-rest/blob/master/_images/18-postman-apicast-soapaction.png)

Now to verify this call has been managed and is reporting to the new metric, on the 3scale web console select the API Product on top and navigate to Analytics -> Traffic and select the metric _GetAllStores_. Each request will increment the counter: 
![](https://github.com/tnscorcoran/3scale-soap-2-rest/blob/master/_images/19-analytics-get-all-stores.png)

(incidentally, you can also rate limit and monetise on this metric)

## Setup a RESTful interface to our SOAP service
Now we're going to deploy a transformation service that exposes the SOAP Service as a RESTful API.
Execute the following:
```
oc create -f $REPO_HOME/templates/stores-fis.json
oc new-app --template=stores-fis --param ROUTE_HOST=stores-fis.$OCP_WILDCARD_DOMAIN
```
This takes a few minutes as it's a Maven build that downloads many dependencies
Execute this periodically until _stores-fis-1-xxxxx_ is Ready with 1/1.
```
oc get pods
```
When that's ready, copy the OpenShift route to this _Stores-FIS_ (with the project soap-rest selected as shown)
![](https://github.com/tnscorcoran/3scale-soap-2-rest/blob/master/_images/20-stores-fis-route.png)

Now in Postman, or your equivalent, make the API call. Simply append _/allstores_ to the route URL yoiu just copied:
![](https://github.com/tnscorcoran/3scale-soap-2-rest/blob/master/_images/21-stores-fis-postman.png)

### Setup the Fuse Console to view the Camel routes used to do the transformation.

Create a cert and key as described on page 
[generate-certificate-openshift4](https://access.redhat.com/documentation/en-us/red_hat_fuse/7.7/html-single/fuse_on_openshift_guide/index#fuse-console-generate-certificate-openshift4)

Then create a secret using them:
```
oc create secret tls fuse75-console-tls-proxying --cert server.crt --key server.key
```

Execute the following
```
BASEURL=https://raw.githubusercontent.com/jboss-fuse/application-templates/application-templates-2.1.0.fuse-770012-redhat-00004
oc replace -n openshift -f ${BASEURL}/fis-image-streams.json
oc new-app -f ${BASEURL}/fuse-console-namespace-os4.json
```
Again this may take a few minutes. Run this periodically until _fuse77-console-1-xxxxx_ is Ready with 1/1:
```
oc get pods
```

Create a secret using your [https://access.redhat.com](https://access.redhat.com/) credentials in the _soap-rest_ project
```
oc project soap-rest
oc create secret docker-registry my-registry-auth \
--docker-server=registry.redhat.io \
--docker-username="yourusername" \
--docker-password="yourpassword"
oc secrets link default my-registry-auth --for=pull
oc secrets link builder my-registry-auth --for=pull
```

Set these variables on the _stores-fis_  deployment config to allow insecure connection from the Fuse console to the _Stores-Fis_ transformation service;
```
oc set env dc/stores-fis \
    AB_JOLOKIA_AUTH_OPENSHIFT=false \
    AB_JOLOKIA_PASSWORD_RANDOM=false \
    AB_JOLOKIA_OPTS=useSslClientAuthentication=false,protocol=https
```

Now manually add this port to the _stores-fis_ Service yaml:
```
    - name: jolokia
      protocol: TCP
      port: 8778
      targetPort: 8778
```
![](https://github.com/tnscorcoran/3scale-soap-2-rest/blob/master/_images/22-stores-fis-service.png)

Now go to Networking-> Routes and open the Fuse Console route. Use your OpenShift admin credentials and accept the permissions confirmation. You'll see this screen. Click _Connect_ on the _stores-fis-xxxx_ row
![](https://github.com/tnscorcoran/3scale-soap-2-rest/blob/master/_images/24-fuse-console-1.png)

You can now see get a visual representation of the Camel routes used to do the SOAP / REST transformation, e.g. Get All Stores:
![](https://github.com/tnscorcoran/3scale-soap-2-rest/blob/master/_images/25-fuse-console-2-get-all-stores.png)



----------------------------------------------------------------------------------------------------



----------------------------------------------------------------------------------------------------
