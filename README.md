# Apache HTTP Server (httpd) S2I Sample Application

This is a very basic sample application repository that can be built and deployed
on [OpenShift](https://www.openshift.com) using the [Apache HTTP Server builder image](https://github.com/sclorg/httpd-container).

The application serves a single static html page via httpd.

To build and run the application:

```
$ s2i build https://github.com/sclorg/httpd-ex centos/httpd-24-centos7 myhttpdimage
$ docker run -p 8080:8080 myhttpdimage
$ # browse to http://localhost:8080
```

You can also build and deploy the application on OpenShift, assuming you have a
working `oc` command line environment connected to your cluster already:

`$ oc new-app centos/httpd-24-centos7~https://github.com/sclorg/httpd-ex`

You can also deploy the sample template for the application:

`$ oc new-app -f https://raw.githubusercontent.com/sclorg/httpd-ex/master/openshift/templates/httpd.json`


## Apache HTTP Server with secure routes

To deploy the application with secure routes use the following steps

### Update/refresh Custom Self Signed Certs
If needed (or necessary) regenerate the certificates using the following steps. This step is not required since default certificate and key are provided but in case they expire this might be needed.

1. Clone the repository container the self signed certificate playbook from https://github.com/cadjai/generate-custom-certificates.git
2. Change into the cloned custom certificates playbook directory 
3. Update the variables to reflect your desired common name,  service ...
4. Run the playbook to generate the self signed CA and certificates using `ansible-playbook create-self-signed-certs.yml`
5. Copy the appropriates certificates into the httpd-ssl directory of the httpd-ex application (copy CA, cert and key)


### Build and run the application
To build the updated container image containing the pki certificate and key using S2I use the following steps

1. Login to the cluster
2. Create a new project using `oc new-project <project-name>` or change into an existing project using `oc project <project-name>`
3. Ensure you are back into the cloned httpd application directory
4. Deploy the application using the following command `oc new-app --code="." --name=httpd --strategy=docker --loglevel=7` . Feel free to change the name of the application if desired
5. Build the container image with the SSL certificate overlaid inside using `oc start-build httpd --from-dir=. --follow`

Once the application is successfully build you should be able to review the various k8 objects like the deployment, service ...

### Create the Routes

Expose the various routes using the following commands

#### Create unsecure route

Create the default unsecure route using 
```
oc expose service httpd
```  
Once the route is create you can retrieve the route using 
```
oc get route httpd -ojsonpath='{"http://"}{.spec.host}{"\n"}'
```   
You can now use your browser to navigate to the route or use curl to query the route. 

#### Create edge route

Create the edge route using 
```
oc create route edge httpd-edge --service=httpd --port=8080
```  
Once the route is create you can retrieve the route using 
```
oc get route httpd-edge -ojsonpath='{"https://"}{.spec.host}{"\n"}'
```   
You can now use your browser to navigate to the route or use curl to query the route. 

#### Create passthrough route 

Create the passthrough route using 
```
oc create route passthrough httpd-passthrough --service=httpd --port=8443
```  
Once the route is create you can retrieve the route using 
```
oc get route httpd-passthrough -ojsonpath='{"https://"}{.spec.host}{"\n"}'
```   
You can now use your browser to navigate to the route or use curl to query the route. 

#### Create reencrypt route

Create the reencrypt route using 
```
oc create route reencrypt httpd-reencrypt --service=httpd --port=8443 --dest-ca-cert=ca/ca.crt
```  
Once the route is create you can retrieve the route using 
```
oc get route httpd-reencrypt -ojsonpath='{"https://"}{.spec.host}{"\n"}'
```  
You can now use your browser to navigate to the route or use curl to query the route.   

> : Warning **Note that to create the reencrypt route the only certificate attribute needed is the dest-ca-cert. If you set both the dest-ca-cert and the ca-cert you get the 'application not available' error.**

For the secure route generated above you can also add path and/or hostname to the command if needed. 
> : Warning If using path with reencrypt route on an OpenShift 4.10.x, ensure that the pod has TLS1.3 enabled. We have seen an issue where with path enabled the route stopped working and it took a lot of troubleshooting to get to the bottom of this. 
