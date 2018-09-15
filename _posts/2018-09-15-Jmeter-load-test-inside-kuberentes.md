---

title: "[En] Jmeter Load test inside kuberentes"
date: '2018-09-15T09:53:00.000+02:00'
tags:
- Jmeter
- Kubernetes
- Helm
- Load tests
---
I'm not a QA guy but sometimes I need to do a few load tests / checks for different
projects. I know that Jmeter is not a perfect tool but it can do the job.

Because I do not find any interesting project that easily ship Apache Jmeter inside Kubernetes.
Yes, there are a few concepts but each required to run 5-10 scripts to run test.
I decided to create home project that ship jmeter via helm chart in to kubernetes.
All required code was published on github [kuberentes-jmeter](https://github.com/kaarolch/kubernetes-jmeter).

Currently there are multiple scenarios of using jmeter with kubernetes:
*   Master and slaves are located on vms or bare metal machines. There are connected
to your kubernetes cluster. Unfortunately this case would be difficult when you would
like to test some internal components not exposed outside cluster (possible but
  additional configuration of ingress or port forwarding required).
*   Master on your local station / vm. Slaves run inside kubernetes. This would
allow you to easily use Jmeter master UI but still this is not enough for long
testing or testing automation.
*   Master and slaves ship to kubernetes and run on demand (kubernetes-jmeter)
is deployed this way.

# Requirements

*   Kubernetes cluster > 1.10 or minikube instance
*   Helm initialized on client and k8s cluster
*   jmeter test cases' files

**Optionally:**
*   Grafana and InfluxDB instances, there is possibility to use own metric aggregation
stack.

When everything is deployed from the
[kubernetes-jmeter](https://github.com/kaarolch/kubernetes-jmeter/tree/master/charts/jmeter)
chart by default Grafana and InfluxDB pods are also deployed in the same namespace as jmeter.

![kubernetes-jmeter architecture](https://github.com/kaarolch/kubernetes-jmeter/raw/master/images/kubernetes-jmeter_architecture.png){:class="img-responsive"}

## Installation
Below there is a example run inside minikube. That deploy the whole stack with
Grafana and InfluxDB instance inside `default` namespace.

```
git clone git@github.com:kaarolch/kubernetes-jmeter.git
cd kubernetes-jmeter/charts/jmeter
helm dep update
helm install -n test ./
```
There is possibility to provide custom value yaml with `-f` flag.
```
helm install install -n test ./ -f my_values.yaml
```
More about charts' values could be found here:
*   [kubernetes-jmeter](https://github.com/kaarolch/kubernetes-jmeter/blob/master/README.md#configuration)
*   [Grafana](https://github.com/helm/charts/blob/master/stable/grafana/README.md#configuration)
*   [InfluDB](https://github.com/helm/charts/blob/master/stable/influxdb/values.yaml)

The command deploys Jmeter on the Kubernetes cluster in the [default](https://github.com/kaarolch/kubernetes-jmeter/blob/master/charts/jmeter/values.yaml) value yaml.

> **Tip**: List all releases using `helm list`

If you change deployment name (`-n test`) please update Grafana datasource influx
`url` inside your custom values.yaml files.

Kuberentes-jmeter could be deployed without Grafana and InfluxDB. Useful when the
metric collection stack is already deployed.
```
helm install install -n test ./ --set grafana.enabled=false,influxdb.enabled=false
```

### Results

```
✘ karol@albert > helm status test
LAST DEPLOYED: Sat Sep 15 11:29:38 2018
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1/Secret
NAME          TYPE    DATA  AGE
test-grafana  Opaque  3     1m

==> v1/Service
NAME               TYPE       CLUSTER-IP    EXTERNAL-IP  PORT(S)             AGE
test-grafana       ClusterIP  10.98.160.66  <none>       80/TCP              1m
test-influxdb      ClusterIP  10.96.254.63  <none>       8086/TCP,2003/TCP   1m
test-jmeter-slave  ClusterIP  None          <none>       1099/TCP,60001/TCP  1m

==> v1beta2/Deployment
NAME          DESIRED  CURRENT  UP-TO-DATE  AVAILABLE  AGE
test-grafana  1        1        1           0          1m

==> v1beta1/PodSecurityPolicy
NAME          DATA   CAPS      SELINUX   RUNASUSER  FSGROUP   SUPGROUP  READONLYROOTFS  VOLUMES
test-grafana  false  RunAsAny  RunAsAny  RunAsAny   RunAsAny  false     configMap,emptyDir,projected,secret,downwardAPI,persistentVolumeClaim

==> v1/Pod(related)
NAME                                 READY  STATUS             RESTARTS  AGE
test-grafana-8698cfff94-gc576        0/1    Running            0         1m
test-influxdb-6dbb46d956-2c754       0/1    ContainerCreating  0         1m
test-jmeter-master-84b7c6d545-8t5rz  0/1    ContainerCreating  0         1m
test-jmeter-slave-b6675b7b7-2llh4    0/1    ContainerCreating  0         1m
test-jmeter-slave-b6675b7b7-9d89p    0/1    ContainerCreating  0         1m

==> v1/Deployment
NAME                DESIRED  CURRENT  UP-TO-DATE  AVAILABLE  AGE
test-jmeter-master  1        1        1           0          1m
test-jmeter-slave   2        2        2           0          1m

==> v1/ConfigMap
NAME           DATA  AGE
test-grafana   2     1m
test-influxdb  1     1m

==> v1/ServiceAccount
NAME          SECRETS  AGE
test-grafana  1        1m

==> v1/ClusterRole
NAME                      AGE
test-grafana-clusterrole  1m

==> v1/ClusterRoleBinding
NAME                             AGE
test-grafana-clusterrolebinding  1m

==> v1beta1/Role
NAME          AGE
test-grafana  1m

==> v1beta1/RoleBinding
NAME          AGE
test-grafana  1m

==> v1beta1/Deployment
NAME           DESIRED  CURRENT  UP-TO-DATE  AVAILABLE  AGE
test-influxdb  1        1        1           0          1m


NOTES:
KUBERNETES JMETER

Services deployed to default namespace.
Jmester master deployment scheduled test-jmeter-master but tests need to run manually.

InfluxDB

Api url:http://test-influxdb.default.svc.cluster.local:8086
Graphite for jmeter config:
-Host: test-influxdb.default.svc.cluster.local
-Port: 2003
-RootMetrixPrefix [influx database]: jmeter
Grafana
Grafan is available inside Kuberentes cluster via: http://test-grafana.default.svc.cluster.local:3000
User: admin
Password: `kubectl -n default get secrets test-grafana -o 'go-template={ {index .data "admin-password"} }' | base64 -d`
```

## Run sample test

### Manual run
Copy [example](https://github.com/kaarolch/kubernetes-jmeter/blob/master/examples/simple_test.jmx) test to jmeter master pod

```
kubectl cp examples/simple_test.jmx $(kubectl get pod -l "app=jmeter-master" -o jsonpath='{.items[0].metadata.name}'):/test/

```
Run tests

```
kubectl exec  -it $(kubectl get pod -l "app=jmeter-master" -o jsonpath='{.items[0].metadata.name}') -- sh -c 'ONE_SHOT=true; /run-test.sh'
```

By default jmeter run in manual mode, all test need to be trigger by operator
or additional service like jenkins job.
If tests are loaded from config map, the jmeter master could run them automatically.
In this case chart attribute `oneShotTest` need to be set to `true`.

**Remember that each restart of jmeter-master would restart test when oneShotTest==true**

### Results
```
karol@albert > kubectl exec  -it $(kubectl get pod -l "app=jmeter-master" -o jsonpath='{.items[0].metadata.name}') -- sh -c 'ONE_SHOT=true; /run-test.sh'
Sep 15, 2018 9:47:16 AM java.util.prefs.FileSystemPreferences$1 run
INFO: Created user preferences directory.
Creating summariser <summary>
Created the tree successfully using /test/simple_test.jmx
Configuring remote engine: 172.17.0.15
Configuring remote engine: 172.17.0.18
Starting remote engines
Starting the test @ Sat Sep 15 09:47:17 GMT 2018 (1537004837319)
Remote engines have been started
Waiting for possible Shutdown/StopTestNow/Heapdump message on port 4445
summary +    403 in 00:00:12 =   32.8/s Avg:   284 Min:   156 Max:  1643 Err:     0 (0.00%) Active: 16 Started: 16 Finished: 0
summary +   1197 in 00:00:18 =   65.8/s Avg:   248 Min:   150 Max:  1307 Err:     0 (0.00%) Active: 0 Started: 16 Finished: 16
summary =   1600 in 00:00:30 =   52.5/s Avg:   257 Min:   150 Max:  1643 Err:     0 (0.00%)
Tidying up remote @ Sat Sep 15 09:47:50 GMT 2018 (1537004870310)
... end of run
```

## Summary
This project is done during my spare time so each new feature, bugs fixing is done
when I have a one or two hours of spare time. If you see that something could be
fixed or changed create an issue or pull request.

I'm still working on a few new features, they could be found in [github](https://github.com/kaarolch/kubernetes-jmeter#to-do)
