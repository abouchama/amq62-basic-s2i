# Example of JBoss A-MQ (ActiveMQ) on OpenShift using S2I

Configuration of the JBoss A-MQ image can also be modified using the S2I (Source-to-image) feature.

Custom A-MQ broker configuration can be specified by creating an openshift-activemq.xml file inside 
the git directory of your application’s Git project root. On each commit, 
the file will be copied to the conf directory in the A-MQ root and its contents used to configure the broker.

##S2I Build OpenShift:

### Create first a new project (namespace) called Broker:

```
oc new-project broker
```

Now let's s2i our conf to the jboss-amq image

```
oc new-build registry.access.redhat.com/jboss-amq-6/amq62-openshift:1.3~https://github.com/abouchama/amq62-basic-s2i.git
--> Found Docker image 884d69b (6 weeks old) from registry.access.redhat.com for "registry.access.redhat.com/jboss-amq-6/amq62-openshift:1.3"

    JBoss A-MQ 6.2 
    -------------- 
    A reliable messaging platform that supports standard messaging paradigms for a real-time enterprise.

    Tags: messaging, amq, java, jboss, xpaas

    * An image stream will be created as "amq62-openshift:1.3" that will track the source image
    * A source build using source code from https://github.com/abouchama/amq62-basic-s2i.git will be created
      * The resulting image will be pushed to image stream "amq62-basic-s2i:latest"
      * Every time "amq62-openshift:1.3" changes a new build will be triggered

--> Creating resources with label build=amq62-basic-s2i ...
    imagestream "amq62-openshift" created
    imagestream "amq62-basic-s2i" created
    buildconfig "amq62-basic-s2i" created
--> Success
    Build configuration "amq62-basic-s2i" created and build triggered.
    Run 'oc logs -f bc/amq62-basic-s2i' to stream the build progress.
```
To stream the build progress, run 'oc logs -f bc/amq62-basic-s2i',
You can see here that our conf openshift-activemq.xml has been copied to the image stream:

```
$ oc logs -f bc/amq62-basic-s2i
Cloning "https://github.com/abouchama/amq62-basic-s2i.git" ...
	Commit:	b4f9e885a2126a0ab91f1356fae104c4c490de6a (add configuration)
	Author:	Gogs <gogs@fake.local>
	Date:	Fri Mar 10 11:22:30 2017 +0100

Copying config files from project...
'/tmp/src/configuration/openshift-activemq.xml' -> '/opt/amq/conf/openshift-activemq.xml'


Pushing image 172.30.34.183:5000/amq/amq62-basic-s2i:latest ...
Pushed 0/7 layers, 3% complete
Pushed 1/7 layers, 18% complete
Pushed 2/7 layers, 35% complete
Pushed 3/7 layers, 47% complete
Pushed 4/7 layers, 82% complete
Pushed 5/7 layers, 87% complete
Pushed 6/7 layers, 94% complete
Pushed 7/7 layers, 100% complete
Push successful
```
Let's get now, our image stream URL:

```
$ oc get is
NAME              DOCKER REPO                                 TAGS      UPDATED
amq62-basic-s2i   172.30.34.183:5000/broker/amq62-basic-s2i   latest    15 minutes ago
amq62-openshift   172.30.34.183:5000/broker/amq62-openshift   1.3       15 minutes ago
```

Now, you have to change the image steam on the template "template-amq62-basic-s2i.json", like following:

```
"image": "172.30.34.183:5000/broker/amq62-basic-s2i"
```

###create the template in the namespace
```
$ oc create -n broker -f template-amq62-basic-s2i.json 
template "amq62-basic-s2i" created
```
###create the service account "amq-service-account"
```
oc create -f https://raw.githubusercontent.com/abouchama/amq62-basic-s2i/master/amq-service-account.json
serviceaccount "amq-service-account" created
```

###ensure the service account is added to the namespace for view permissions... (for pod scaling)
```
oc policy add-role-to-user view system:serviceaccount:broker:amq-service-account
```

###use the template in the namespace then to create your Broker:
```
$ oc new-app --template="broker/amq62-basic-s2i"
--> Deploying template amq62-basic-s2i for "broker/amq62-basic-s2i"

     amq62-basic-s2i
     ---------
     Application template for JBoss A-MQ brokers. These can be deployed as standalone or in a mesh. This template supports SSL and requires usage of OpenShift secrets.

     * With parameters:
        * APPLICATION_NAME=broker
        * MQ_PROTOCOL=openwire
        * MQ_QUEUES=
        * MQ_TOPICS=
        * MQ_USERNAME=admin # generated
        * MQ_PASSWORD=admin # generated
        * AMQ_MESH_DISCOVERY_TYPE=kube
        * AMQ_STORAGE_USAGE_LIMIT=1 gb
        * IMAGE_STREAM_NAMESPACE=openshift

--> Creating resources with label app=amq62-basic-s2i ...
    service "broker-amq-tcp" created
    deploymentconfig "broker-amq-a" created
--> Success
    Run 'oc status' to view your app.
```
###Update of openshift-activemq.xml

You should setup the GitHub webhook URL in order to trigger a new build after each update.

You can also do it manually, like following:
```
$ git commit -am "changing my activemq conf"

$ git push -u origin master

$ oc start-build amq62-basic-s2i -n broker
build "amq62-basic-s2i-2" started

$ oc deploy broker-amq-a --latest -n broker
Started deployment #2
Use 'oc logs -f dc/broker-amq-a' to track its progress.
```