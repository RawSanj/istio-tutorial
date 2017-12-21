# Java (Spring Boot) + Istio on Kubernetes/OpenShift

Istio capabilities explored:
* Jaeger tracing
* Prometheus+Grafana monitoring
* Smart routing between Services (Canary Deployments) including user-agent based routing


There are three different and super simple microservices in this system and they are chained together in the following sequence:

customer -> preferences -> recommendations

For now, they have a simple exception handling solution for dealing with 
a missing dependent service, it just returns the error message to the end-user.

## Setup minishift
Assumes minishift, tested with minshift v1.10.0+10461c6

My creation script
```bash
#!/bin/bash

# add the location of minishift execuatable to PATH
# I also keep other handy tools like kubectl and kubetail.sh 
# in that directory

export PATH=/Users/burr/minishift_1.10.0/:$PATH

minishift profile set istio2-demo
minishift config set memory 8GB
minishift config set cpus 3
minishift config set vm-driver virtualbox
minishift addon enable admin-user
minishift config set openshift-version v3.7.0

MINISHIFT_ENABLE_EXPERIMENTAL=y minishift start --metrics
```
## Setup environment

```
eval $(minishift oc-env)
eval $(minishift docker-env)
```

## Istio installation script

```bash
#!/bin/bash

curl -LO https://github.com/istio/istio/releases/download/0.4.0/istio-0.4.0-osx.tar.gz

gunzip istio-0.4.0-osx.tar.gz 

tar -xvzf istio-0.4.0-osx.tar

cd istio-0.4.0

oc login $(minishift ip):8443 -u admin -p admin

oc adm policy add-scc-to-user anyuid -z istio-ingress-service-account -n istio-system

oc adm policy add-scc-to-user anyuid -z istio-egress-service-account -n istio-system

oc adm policy add-scc-to-user anyuid -z default -n istio-system

oc create -f install/kubernetes/istio.yaml

oc project istio-system 

oc expose svc istio-ingress 

oc apply -f install/kubernetes/addons/prometheus.yaml

oc apply -f install/kubernetes/addons/grafana.yaml

oc apply -f install/kubernetes/addons/servicegraph.yaml

oc expose svc servicegraph

oc expose svc grafana

oc process -f https://raw.githubusercontent.com/jaegertracing/jaeger-openshift/master/all-in-one/jaeger-all-in-one-template.yml | oc create -f -

oc get pods -w

```
## Deploy Customer

Make sure you have are logged in
```
oc status
oc whoami
```
and you have setup the project/namespace

```
oc new-project springistio
oc adm policy add-scc-to-user privileged -z default -n springistio

```

```
cd customer
mvn clean package
docker build -t example/customer .
docker images | grep customer
```

Currently using the "manual" way of injecting the Envoy sidecar
Add istioctl to your $PATH

```
istioctl version

oc apply -f <(istioctl kube-inject -f src/main/kubernetes/Deployment.yml) -n springistio

oc create -f src/main/kubernetes/Service.yml

oc expose service customer

oc get route

curl customer-springistio.$(minishift ip).nip.io

cd ..
```

## Deploy preferences
```
cd preferences

mvn clean package

docker build -t example/preferences .

docker images | grep preferences

oc apply -f <(istioctl kube-inject -f src/main/kubernetes/Deployment.yml) -n springistio

oc create -f src/main/kubernetes/Service.yml

curl customer-springistio.$(minishift ip).nip.io

cd ..
```

## Deploy recommendations
```
cd recommendations

mvn clean package

docker build -t example/recommendations:v1 .

docker images | grep recommendations

oc apply -f <(istioctl kube-inject -f src/main/kubernetes/Deployment.yml) -n springistio

oc create -f src/main/kubernetes/Service.yml

curl customer-springistio.$(minishift ip).nip.io
```

it returns

```
C100 *{"P1":"Red", "P2":"Big"} && Clifford v1 * 

```
## Note: Updating & Redeploying Code
When you wish to change code (e.g. editing the .java files) and wish to "redeploy", simply:
```
cd {servicename}

vi src/main/java/com/example/{servicename}/{Servicename}

Controller.java
mvn clean package
docker build examples/{servicename} .
oc get pod -l app=recommendations | grep recommendations | awk '{print $1}'

oc delete pod -l app=recommendations,version=v1
```
Based on the Deployment configuration, Kubernetes/OpenShift will recreate the pod, based on the new docker image

```
oc describe deployment recommendations | grep Replicas
```

## Tracing

## Monitoring

## Istio RouteRule Changes
### recommendations:v2 
So we can experiment with Istio routing rules by making a change to RecommendationsController.java like

```
System.out.println("Big Red Dog v2");

return "Clifford v2";
```

The "v2" tag during the docker build is significant.

There is also a 2nd deployment.yml file to label things correctly

```
cd recommendations

mvn clean compile package

docker build -t example/recommendations:v2 .

docker images | grep recommendations
example/recommendations                  v2                  c31e399a9628        5 seconds ago       438MB
example/recommendations                  latest              f072978d9cf6        17 minutes ago      438MB

cd ..

oc apply -f <(istioctl kube-inject -f kubernetesfiles/recommendations_v2_deployment.yml) -n springistio

oc apply -f kubernetesfiles/recommendations_v2_service.yml

oc get pods -w

curl customer-springistio.$(minishift ip).nip.io
```

you likely see "Clifford v1"

```
curl customer-springistio.$(minishift ip).nip.io
```

you should see "Clifford v2" as by default you get random load-balancing when there is more than one Pod behind a Service


Make sure you have established "springistio" as the namespace/project that you will be working in, allowing you to skip the -n springistio in subsequent commands

```
oc project springistio 
```

## Istio Route Rules

#### Set all users to recommendations:v2
```
oc create -f istiofiles/route-rule-recommendations-v2.yml 

curl customer-springistio.$(minishift ip).nip.io
```

you should only see v2 being returned

#### All users to recommendations:v1
Note: "replace" instead of "create" since we are overlaying the previous rule

```
oc replace -f istiofiles/route-rule-recommendations-v1.yml 

oc get routerules

oc get routerules/recommendations-default -o yaml 
```
#### All users to recommendations v1 and v2 
By simply removing the rule

```
oc delete routerules/recommendations-default
```

### Retry
Instead of failing immediately, retry the Service N more times

You could update the RecommendationsController.java to throw out some
 503's or you could leverage Istio's fault injection capability.   We will use Istio and return 503's about 50% of the time

```
oc create -f istiofiles/route-rule-recommendations-v2_503.yml 
```

Now, if you hit the customer endpoint several times, you should see some 503's

```
curl customer-springistio.$(minishift ip).nip.io
C100 *{"P1":"Red", "P2":"Big"} && 503 Service Unavailable * 
```

Now add the retry rule
```
oc create -f istiofiles/route-rule-recommendations-v2_retry.yml 
```
and you will see it work every time
```
curl customer-springistio.$(minishift ip).nip.io
C100 *{"P1":"Red", "P2":"Big"} && Clifford v2 * 
```
Now, delete the retry rule and see the old behavior, some random 503s
```
oc delete routerule recommendations-v2-retry
curl customer-springistio.$(minishift ip).nip.io
```
Now, delete the 503 rule and back to random load-balancing between v1 and v2
```
oc delete routerule recommendations-v2-503
curl customer-springistio.$(minishift ip).nip.io
```

### Timeout
Wait only N seconds before giving up and failing.  At this point, no other route rules should be in effect.  oc get routerules and oc delete routerule rulename if there are some.

First, introduce some wait time in recommendations v2. Update RecommendationsController.java to include a Thread.sleep, making it a slow perfomer

```
        System.out.println("Big Red Dog v2");

        // begin circuit-breaker example
        try {
			Thread.sleep(3000);
		} catch (InterruptedException e) {			
			e.printStackTrace();
		}
        System.out.println("recommendations ready to return");
        // end circuit-breaker example
        return "Clifford v2";
```
Rebuild, redeploy
```
cd recommendations

mvn clean compile package

docker build -t example/recommendations:v2 .

docker images | grep recommendations

oc delete pod -l app=recommendations,version=v2

cd ..
```
Hit the customer endpoint a few times, to see the load-balancing between v1 and v2 but with v2 taking a bit of time to respond

```
curl customer-springistio.$(minishift ip).nip.io
curl customer-springistio.$(minishift ip).nip.io
curl customer-springistio.$(minishift ip).nip.io

```        

Then add the timeout rule

```
oc create -f istiofiles/route-rule-recommendations-timeout.yml
curl customer-springistio.$(minishift ip).nip.io
```
You will see it return v1 OR 504 after waiting about 1 second

```
curl customer-springistio.$(minishift ip).nip.io
C100 *{"P1":"Red", "P2":"Big"} && Clifford v1 * 
curl customer-springistio.$(minishift ip).nip.io
C100 *{"P1":"Red", "P2":"Big"} && 504 Gateway Timeout * 
```

When completed, delete the timeout rule

```
oc delete routerule recommendations-timeout

```
### Smart routing based on user-agent header (Canary Deployment)

What is your user-agent?

https://www.whoishostingthis.com/tools/user-agent/

Note: the "user-agent" header being forward in the Customer and Preferences controllers in order for route rule modications around recommendations 

#### Set recommendations to all v1
```
oc create -f istiofiles/route-rule-recommendations-v1.yml 
```
#### Set Safari users to v2
```
oc create -f istiofiles/route-rule-safari-recommendations-v2.yml 

oc get routerules
```

and test with a Safari (or even Chrome on Mac since it includes Safari in the string).  Safari only sees v2 responses from recommendations

and test with a Firefox browser, it should only see v1 responses from recommendations

```
oc describe routerule recommendations-safari
```

Remove the Safari rule

```
oc delete routerule recommendations-safari
```
#### Set mobile users to v2
```
oc create -f istiofiles/route-rule-mobile-recommendations-v2.yml

curl -A "Mozilla/5.0 (iPhone; U; CPU iPhone OS 4(KHTML, like Gecko) Version/5.0.2 Mobile/8J2 Safari/6533.18.5" http://customer-springistio.192.168.99.102.nip.io/
```

### Clean up
```
oc delete routerule recommendations-safari

oc delete routerule recommendations-default
```

### Access Control

#### Whitelist

#### Blacklist

### Fault Injection

#### Delay

#### 404/503

### Rate Limiting
Nothing so far

### Mirroring Traffic (Dark Launch)
Wiretap, eavesdropping
Note: does not seem to work in 0.4.0

```
oc get pods -l app=recommendations
```
You should have 2 pods for recommendations based on the steps above

```
oc get routerules
```
You should have NO routerules
if so "oc delete routerule rulename"

Make sure you are in the main directory

```
oc create -f istiofiles/route-rule-recommendations-v1-mirror-v2.yml 

curl customer-springistio.$(minishift ip).nip.io
```

### Circuit Breaker

Update RecommendationsController.java to include a Thread.sleep, making it a slow perfomer

```
        System.out.println("Big Red Dog v2");

        // begin circuit-breaker example
        try {
			Thread.sleep(3000);
		} catch (InterruptedException e) {			
			e.printStackTrace();
		}
        System.out.println("recommendations ready to return");
        // end circuit-breaker example
        return "Clifford v2";
```
Rebuild, redeploy
```
cd recommendations

mvn clean compile package

docker build -t example/recommendations:v2 .

docker images | grep recommendations

oc delete pod -l app=recommendations,version=v2

cd ..

curl customer-springistio.$(minishift ip).nip.io

```        
Whenever you are hitting v2, you will notice the slowness in the response

Watch the logging output of recommendations

```
Term 1:
./kubetail.sh recommendations
or 
brew install stern
stern recommendations

Term 2:
curl customer-springistio.$(minishift ip).nip.io
```

Add the circuit breaker

```
istioctl create -f istiofiles/recommendations_cb_policy_version_v2.yml
istioctl get destinationpolicies
```

Add some load

```
cd gatling_test
mvn integration-test
```

If you wish to update the CB

```
istioctl get destinationpolicies recommendations-circuitbreaker -o yaml -n default
istioctl replace -f istiofiles/recommendations_cb_policy_app.yml -n default
```



