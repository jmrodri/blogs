# DRAFT
# Up and running with the OpenShift Ansible Broker

## Overview

In OpenShift 3.7 we introduced two new concepts, Service Catalog and Brokers.
The Service Catalog gives a place for various service providers to list their
services. It also helps users find and acquire these services for their needs.

For more detailed information about Service Catalog checkout Paul Morie's
[Service Catalog deep dive](https://blog.openshift.com/openshift-commons-briefing-106-service-catalog-on-openshift-3-7-deep-dive/)

To be useful, the Service Catalog needs content.  What drives the Service
Catalog's content is the second concept introduced in OpenShift
3.7: brokers. Service Brokers implement the [Open Service Broker
API](https://github.com/openservicebrokerapi/servicebroker) (OSB API). *GIVE A
BLURBL ABOUT OSB API*

Brokers are responsible for a set of services. You can have brokers that
only do one service, for example a MySQL broker. You can have
a broker that offers a specific type of facility, like the OpenShift Template
broker which can support a collection of services based on templates. Or you can
have a more powerful, generic broker like the OpenShift Ansible Broker which supplies a
set of services based on Ansible Playbook Bundle (APB) application definitions.

APBs provide a new method for defining and distributing container applications
in OpenShift, consisting of a bundle of Ansible playbooks build into a container
image with an Ansible runtime.  APBs leverage Ansible to create a standard
mechanism for automating complex deployments.

For more detailed information about Ansible Playbook Bundles checkout the
[OpenShift Commons](https://blog.openshift.com/openshift-commons-briefing-74-deploying-multi-container-applications-ansible-service-broker/)
briefing by Todd Sanders and John Matthews.

Now that you know what the OpenShift Ansible Broker is, let's get one up and
running.

## Setup
As with most applications, there are a variety of ways of setting up the broker
from templates to Makefile targets to the OpenShift installer. For the purpose
of this blog post we will focus on using a simple OpenShift template to launch the broker.

You will need an OpenShift 3.7 cluster running with the service catalog enabled.
[INSERT LINK TO HOW TO DO THAT].  I typically just start a cluster with the
following command:

```bash
oc cluster up --service-catalog=true
```

**NOTE:** if you are using Fedora 26 ensure you are using docker-1.13.1-44 or newer:
https://bugzilla.redhat.com/show_bug.cgi?id=1504709

Once the cluster is running, we can install the OpenShift Ansible Broker into the
cluster and register it with the service catalog. First, we will need a new
project to run the broker in. Using the CLI let's create the
`ansible-service-broker` project.

```bash
oc login -u system:admin
oc new-project ansible-service-broker
```

After a successful run, you'll see something like this:

```bash
Now using project "ansible-service-broker" on server "https://127.0.0.1:8443".

You can add applications to this project with the 'new-app' command. For example, try:

    oc new-app centos/ruby-22-centos7~https://github.com/openshift/ruby-ex.git

to build a new example application in Ruby.
```

With the project now created, we can deploy the broker. We've assembled an
OpenShift [template](https://raw.githubusercontent.com/jmrodri/simple-asb/up-and-running/deploy-ansible-service-broker.template.yaml)
that can be used. Now let's download the template, process the variables, and create the broker.

```bash
curl -s https://raw.githubusercontent.com/jmrodri/simple-asb/up-and-running/deploy-ansible-service-broker.template.yaml | oc process -n "ansible-service-broker" -f - | oc create -f -
```

A successful deployment will look like the following.

```bash
service "asb" created
service "asb-etcd" created
serviceaccount "asb" created
clusterrolebinding "asb" created
clusterrole "asb-auth" created
clusterrolebinding "asb-auth-bind" created
clusterrole "access-asb-role" created
persistentvolumeclaim "etcd" created
deploymentconfig "asb" created
deploymentconfig "asb-etcd" created
secret "asb-auth-secret" created
secret "registry-auth-secret" created
configmap "broker-config" created
serviceaccount "ansibleservicebroker-client" created
clusterrolebinding "ansibleservicebroker-client" created
secret "ansibleservicebroker-client" created
route "asb-1338" created
clusterservicebroker "ansible-service-broker" created
```

We now have an OpenShift cluster with a Service Catalog and Ansible Broker
running.  You can communicate with the broker through the service catalog
using the oc command line. Here is an example of listing out the available
APB service classes:

```bash
oc get clusterserviceclasses --all-namespaces -o custom-columns=NAME:.metadata.name,DISPLAYNAME:spec.externalMetadata.displayName | grep APB
```

It may take some time for the broker to sync the APBs into the catalog. If you
get no APBs at first, run it again in a few seconds. Once they are available we
can provision one.

## Provision an APB

As we mentioned above, the Broker implements the OSB API. This API contains some
key verbs: provision, bind, and others. Provision will typically deploy a
service in your cluster. In the case of the Ansible Broker, it will provision
your service using the Ansible Playbook Bundle meta-container invoking the
provision playbook. We will provision a MediaWiki application that is backed by
a PostgreSQL DB. We will accomplish that by provisioning a PostgreSQL APB and a
MediaWiki APB.

Once the two APBs have been provisioned, we will create a binding. Bind is
another one of the OSB API verbs used to provide credentials / coordinates for
specific services. We will create a binding for the PostgreSQL service. Like the
provision, the Broker will use the PostgreSQL APB meta-container to create the bind
by invoking the bind playbook.

Let's recap, first we will provision two APBs: PostgreSQL and MediaWiki. Then
create and consume the binding to the PostgreSQL database. Finally we will
verify the MediaWiki service is up and running.

## Provision PostgreSQL APB
Here we will provision the PostgreSQL APB

1. We need to visit the console UI at https://127.0.0.1:8443, after accepting the
certificate, you should see the login screen:

![screenshot of login screen](up-and-running-login-screen.png)

2. Login with `admin:admin`. You should see a list of services, some marked 
with APB:

![screenshot of apbs](up-and-running-apb-list-ui.png)

3. From the list of services, select the PostgreSQL APB. You will be prompted for some
information as you provision the service.

![screenshot of provisioning postgresql](up-and-running-psql-1-prov.png)

4. Next step is to choose a plan, select the Development plan.

![screenshot of plan selection](up-and-running-psql-2-plan.png)

5. The configuration screen will ask you for some information.  Create a new project named
*blog-project* and Description of *Blog Project*.  Then enter a password, it is fine to
keep the other values as defaults.

![screenshot of config selection](up-and-running-psql-3-config.png)

Above we selected an APB to provision. We created a project to put the service
in to. We supplied some configuration parameters to the service. The parameters
are actually supplied by the APB. That is part of the services metadata which is
exposed to OpenShift. This allows the UI to be catered to the particular
service. We will see a different set of parameters when we provision the
MediaWiki APB later.

One concept you may have noticed was that of plans.  Plans are another OSB
API concept that are akin to tiers or pricing plans. For example, you could
have a development plan that has minimal resources, lower cost and little to
no persistence storage. This would let users use a service for development purposes.
Or you could have for example, a production plan, that has high-availability, a good
bit of persistence storage, and more resources. The PostgreSQL APB exposes
two plans: development and production.

## Create the Binding
A bindings is a link between a service instance and an application. To save time,
we will create a binding while provisioning the PostgreSQL APB. This will save
the credentials for the PostgreSQL DB into a secret that can be shared with
another applications.

![screenshot of create binding selection](up-and-running-psql-4-binding.png)

While the PostgreSQL APB provisions, we can move on to provision the next
application.

![screenshot of results selection](up-and-running-psql-5-results.png)

## Provision MediaWiki APB
While the PostgreSQL APB is provisioning, we can provision the next APB.

1. Let's provision the MediaWiki APB.

![screenshot of mediawiki provision](up-and-running-mediawiki-1-prov.png)

2. Configure MediaWiki, select the *Blog Project* we created earlier and enter
   in passwords.

![screenshot of mediawiki config](up-and-running-mediawiki-2-config.png)

3. We can see it deploying, and that PostgreSQL has already been deployed.

![screenshot of mediawiki deploying](up-and-running-mediawiki-deploying.png)

4. Once MediaWiki has been provisioned. We see the default startup page.

![screenshot of default mediawiki startpage](up-and-running-mediawiki-startpage.png)


## Consume Binding
![screenshot of mediawiki env secrets](up-and-running-mediawiki-secret-env.png)
### List the services from the CLI
The UI isn't the only way to interact with the broker. We can list the
provisioned services using the CLI..

## Verify MediaWiki
```bash
$ oc get serviceinstances --all-namespaces
NAMESPACE      NAME                      AGE
blog-project   dh-mediawiki-apb-rhzcs    1m
blog-project   dh-postgresql-apb-t84wc   7m
```

Let's checkout the secrets in the blog-project
```bash
$ oc get secrets -n blog-project | awk -F, 'BEGIN{IGNORECASE=1}; NR==1 {print $1}; /^dh/ {print $1}'
NAME                                        TYPE                                  DATA      AGE
dh-mediawiki-apb-parametersch7a5            Opaque                                1         22m
dh-postgresql-apb-parameters43rfr           Opaque                                1         28m
dh-postgresql-apb-t84wc-credentials-x9xd8   Opaque                                6         27m
```

So what have we done? We brought up a 3.7 cluster, deployed the Ansible broker,
listed and provisioned a couple of APBs.

## Come check out Ansible Broker
If you would like to know more about the OpenShift Ansible Broker I encourage
you to checkout the project at: [https://github.com/openshift/ansible-service-broker/](https://github.com/openshift/ansible-service-broker/)

Also consider subscribing to:

* IRC: Freenode #asbroker
* Mailing list: [https://www.redhat.com/mailman/listinfo/ansible-service-broker](https://www.redhat.com/mailman/listinfo/ansible-service-broker)
* Github: [https://github.com/openshift/ansible-service-broker/](https://github.com/openshift/ansible-service-broker/)
* YouTube: [ansible-service-broker](https://www.youtube.com/channel/UC04eOMIMiV06_RSZPb4OOBw) 
