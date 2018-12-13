# Istio Circuit Breaker Mission for Node.js

## Purpose

Showcase Circuit Breaking in Istio in Node.js applications

## Prerequisites

* Openshift 3.10 cluster with Istio. For local development, download the latest release from [Maistra](https://github.com/Maistra/origin/releases) and run:

```
# Set oc to be the Maistra one
oc cluster up --enable="*,istio"
oc login -u system:admin
oc apply -f https://raw.githubusercontent.com/Maistra/openshift-ansible/maistra-0.6/istio/cr-minimal.yaml -n istio-operator
oc get pods -n istio-system -w
```
Wait until the `openshift-ansible-istio-installer-job-xxxx` job has completed. It can take several minutes. The OpenShift console is available on https://127.0.0.1:8443.

* Create a new project/namespace on the cluster. This is where your application will be deployed.

```
oc login -u system:admin
oc adm policy add-cluster-role-to-user admin developer --as=system:admin
oc adm policy add-scc-to-user anyuid -z default -n myproject
oc adm policy add-scc-to-user privileged -z default -n myproject
oc login -u developer -p developer
oc new-project <whatever valid project name you want> # not required
```

### Build and Deploy the Application

#### With Source to Image build (S2I)

Run the following commands to apply and execute the OpenShift templates that will configure and deploy the applications:

```bash
find . | grep openshiftio | grep application | xargs -n 1 oc apply -f

oc new-app --template=nodejs-istio-circuit-breaker-greeting-service -p SOURCE_REPOSITORY_URL=https://github.com/nodeshift-starters/nodejs-istio-circuit-breaker -p SOURCE_REPOSITORY_REF=master -p SOURCE_REPOSITORY_DIR=greeting-service
oc new-app --template=nodejs-istio-circuit-breaker-name-service -p SOURCE_REPOSITORY_URL=https://github.com/nodeshift-starters/nodejs-istio-circuit-breaker -p SOURCE_REPOSITORY_REF=master -p SOURCE_REPOSITORY_DIR=name-service
```

## Use Cases

Any steps issuing `oc` commands require the user to have run `oc login` first and switched to the appropriate project with `oc project <project name>`.

### Without Istio Configuration

1. Create a Gateway and Virtual Service in Istio so that we can access the service within the Mesh:
    ```
    oc apply -f istio-config/gateway.yaml
    ```
2. Retrieve the URL for the Istio Ingress Gateway route, with the below command, and open it in a web browser.
    ```
    echo http://$(oc get route istio-ingressgateway -o jsonpath='{.spec.host}{"\n"}' -n istio-system)/nodejs-istio-circuit-breaker/
    ```
3. The user will be presented with the web page of the Booster
4. Click "Start" to issue repeating concurrent requests in batches of 10 to the greeting service
5. Click "Stop" to cease issuing more requests
6. The number of concurrent requests can be set to anything between 1 and 20
7. There should be no failures and all calls are ok

### With Istio Configuration

#### Initial Setup

1. Run `oc project <project>` to connect to the project created by the Launcher, or one you created in a previous step
2. Create a `VirtualService` for the name service, which is required to use `DestinationRule` later
    ````
    oc create -f istio-config/virtual-service.yml -n $(oc project -q)
    ````
3. Trying the application again you will notice no change from the current behavior

#### Istio Circuit Breaker Configuration

1. Apply a `DestinationRule` that activates Istio's Circuit Breaker on the name service,
configuring it to allow a maximum of 100 concurrent connections
    ````
    oc create -f istio-config/initial-destination-rule.yml -n $(oc project -q)
    ````
2. Trying the application again you should see no change,
as we're only able to make up to 20 concurrent connections which is not enough to trigger the circuit breaker.
3. Remove the initial destination rule
    ````
    oc delete -f istio-config/initial-destination-rule.yml
    ````
4. Apply a more restrictive destination rule
    ````
    oc create -f istio-config/restrictive-destination-rule.yml -n $(oc project -q)
    ````
5. Trying the application again you can see about a third of all requests are triggering the fallback response because the circuit is open
6. If we check "Simulate load", which adds a delay into how quickly the name service responds, and click "Start".
We now see that the majority of our calls trigger the fallback as our name service takes too long to respond.
