# Asynchronous Bind with the Automation Broker (formerly Ansible Service Broker)

## Introduction

In February 2017, John Matthews introduced PnT to the Ansible Playbook Bundles and the Automation Broker. Back then, they were known by different names, AnsibleApps and Ansible Service Broker respectively.

A year and a name change later, the work done with the Ansible Playbook Bundles helped us codify the contract between the Broker and the Bundles which yielded the Service Bundle. The Service Bundle contract allows you to create a container that uses whatever you want inside: shell scripts, Helm Chart, etc. Ansible Playbook Bundles are an implementation of the Service Bundle contract.

Our user base has grown as well. We have quite a few projects that have chosen to use Ansible Playbook Bundles as the means for deploying their applications/services in cloud native environments: Keycloak, Elasticsearch, Kibana, Prometheus, RocketChat and many others.

## Actions
The Automation Broker implements the Open Service Broker API, which defines a set of actions used to provision, deprovision, update, bind to and unbind from services in cloud native environments. In the case of the Automation Broker, we’ve targeted OpenShift and Kubernetes.


| Action | Description | Common Example |
| ------ | ----------- | -------------- |
| Provision | deploy a service or set of services to a cluster or other location | install and configure Keycloak service in an OpenShift cluster |
| Bind | creates a set of credentials or coordinates to tie two or more services together | create a Keycloak auth token, create a client in a realm, and encode the token |
| Unbind | revoke the credentials created by a bind | revoke the Keycloak token used to connect to the service, delete the client in the realm |
| Deprovision | remove the service previously provisioned | uninstall the Keycloak service from the OpenShift cluster and remove any Realms created |
| Update | migrate from a development plan to a GOLD level plan with storage | given a deployed PostgreSQL deployed as a developer instance, you want to switch it to a GOLD level production plan to have the service backed by storage. Migrating your data so that you can move from dev to production more easily |

## Bind Action
Most authors will likely use provision and deprovision to deploy their services to the cluster. But there is also a class of applications that have the need for a binding. What a binding represents may vary by service. In general creation of a binding generates credentials necessary for accessing the resource.

## Problems with synchronous bind
In OpenShift 3.7, the Automation Broker supported the 2.13 version of the Open Service Broker API which only defined a synchronous bind. Before we created the Automation Broker, many brokers were tied to a single resource. For example, there would be a PostgreSQL broker handling the resources for a PostgreSQL database. In these simple brokers having a synchronous bind was not a huge problem since they would more than likely return a simple set of credentials that were pre-determined. Not a lot of work was being put into the bind methods.

But for a more powerful and general purpose broker, like the Automation Broker, the managing of multiple and differing services became possible. With these differing services, came diverse use cases. Let’s use PostgreSQL as an example. Let’s say you want to create a hosted PostgreSQL service where each binding creates a new set of user credentials and their own database. Because the Open Service Broker API spec has a performance expectation, this limits what you can do in a synchronous call. Launching a meta-container to perform actions or attempting to run any other long-running action is not feasible in such a time.

Because of this limitation, the Automation Broker was only able to allow bind creation tasks in the provision action. Our synchronous bind would simply return the credentials created by the previous provision call. It limited what one could accomplish, making the hosted scenario described above almost impossible. What would be great is if we had an asynchronous version of bind and unbind to allow for more flexibility in what broker authors could do.

## Introducing Asynchronous bind
The other actions defined in the Open Service Broker API spec have asynchronous variants:  provision, deprovision, and update, can all be run asynchronously. These actions can perform complex tasks like launching several pods, creating storage, etc. The tasks can be monitored using the last_operation API.

Adding asynchronous bind to the Automation Broker was an easy task from a perspective. The biggest hurdle is getting these changes into the upstream Open Service Broker API specification. Working with other community members from Google, IBM, Pivotal and Red Hat we were able to get the asynchronous bind spec to a place where we could start to implement it. 

Having an asynchronous bind method allows a broker, especially the Automation Broker, perform some complex tasks that were previously only available during the other phases. With this new flexibility comes new opportunities from the trivial case of returning a pre-populated set of credentials or retrieving credentials from a deployed vault to generating certificates signed by a self-signed CA certificate created in the provision action. Maybe we want to support two-factor authentication with our services, where the bind operation waits for some external action, like a query on your phone, to confirm that the bind should happen.

None of the above would be possible with a synchronous bind, at least not without possibly timing out the call. 

## Enabling Asynchronous Bind in the Broker
In OpenShift 3.9, the asynchronous bind feature can be enabled in the Service Catalog and the Automation Broker for testing. First, the `AsyncBindingOperations` feature gate needs to be enabled in the Service Catalog:

```
# oc edit daemonset controller-manager -n kube-service-catalog
      containers:
      - args:
        - controller-manager
        - -v
        - "5"
        - --leader-election-namespace
        - kube-service-catalog
        - --broker-relist-interval
        - 5m
        - --feature-gates
        - OriginatingIdentity=true
        - --feature-gates
        - AsyncBindingOperations=true
```

Then the `launch_apb_on_bind` feature needs to be enabled on the Automation Broker. This setting tells the broker that it can launch an Ansible Playbook Bundle container during an asynchronous bind operation. To enable the feature edit the broker’s configmap:

```
# oc get configmap broker-config -o yaml
apiVersion: v1
data:
  broker-config: |
...
    broker:
      dev_broker: True
      bootstrap_on_startup: true
      refresh_interval: "600s"
      launch_apb_on_bind: True
...
```
After updating the configmap, rollout a new broker:

```
# oc rollout latest dc/asb
deploymentconfig “asb” rolled out
```

Once the features are enabled on both the Service Catalog and the Automation Broker, all binds will now be performed asynchronously.

## Let’s help each other
We can help each other by using the Ansible Playbook Bundle to containerize your applications which will help test our Automation Broker. We can help by guiding teams to create their Ansible Playbook Bundles as well as add feature requests that come from the testing or unique use cases.

To keep up with developments or to learn more:

* Website:
  * http://automationbroker.io/
* Trello:
https://trello.com/b/50JhiC5v/ansible-service-broker
* Github:
  * Automation Broker
    * https://github.com/openshift/ansible-service-broker
  * Ansible Playbook Bundle tooling and examples
    * https://github.com/ansibleplaybookbundle
  * Service Bundle library
    * https://github.com/automationbroker/bundle-lib
* IRC:
  * Freenode #asbroker
* Mailing list:
  * https://www.redhat.com/mailman/listinfo/ansible-service-broker
* YouTube:
  * https://www.youtube.com/channel/UC04eOMIMiV06_RSZPb4OOBw

