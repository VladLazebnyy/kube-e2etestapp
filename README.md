# Kubernetes End to End Test App [![Build Status](https://travis-ci.org/ocadotechnology/kube-e2etestapp.svg?branch=master)](https://travis-ci.org/ocadotechnology/kube-e2etestapp)

This application tests the features of Kubernetes periodically in order to collect data and highlight symptoms of underlying issues.

Performance metrics are captured by each test using statsd to measure the effectiveness of the cluster and help highlight problems with it. See [Kubernetes Manifests](#kubernetes-manifests) for detail on how to deploy this. [Contributing](./CONTRIBUTING.md) includes a diagram of how this works under the hood and how people can contribute to this project.

# Contents

  * [Configuration](#configuration)
    + [Kubernetes manifests](#kubernetes-manifests)
  * [Metrics and alerts](#metrics-and-alerts)
    + [Time-based metrics](#time-based-metrics)
    + [HTTP metrics](#http-metrics)
    + [Error metrics](#error-metrics)
  * [Health check dashboard](#health-check-dashboard)
  * [Test List](#test-list)
    + [Namespace tests](#namespace-tests)
    + [Service tests](#service-tests)
    + [Deployment tests](#deployment-tests)
      - [Deployment update tests](#deployment-update-tests)
      - [Deployment scaling tests](#deployment-scaling-tests)
    + [HTTP request tests](#http-request-tests)
    
## Configuration
The following environment variables can be used to configure the e2etest container.

Name | description | default 
--- | --- | --- 
TIME_TO_REPORT_PROBLEM | Used on healthcheck page. Time in minutes to wait before displaying an error indicating how long it's been since the last test was ran. | 20 
SECONDS_BETWEEN_RUNS | Time to sleep between runs of the test suite. | 0.0
TEST_NAMESPACE | Namespace to use during testing. All other resources will reside in this namespace. | kubee2etests
TEST_DEPLOYMENT | Deployment name to create during testing. | kubee2etestapp
TEST_SERVICE | Service name to create during testing | kubee2etests
FLASK_PORT | Port on which to run flask app | 8081
STATSD_PORT | Port on which `statsd` is running | 8125
LOG_LEVEL | log level for test runner | INFO
DOCKER_REGISTRY_HOST | Host from which to pull nginx pod for deployment based tests. We allow only configuration of the host, not the image because service and get requests expect that we'll be able to get something from an nginx web server. | `` 
CUSTOM_TEST_DEPLOYMENT_LABELS | Dictionary of key-value pairs to add to the labels applied to every test deployment. Will already be labelled with `app: hello-minikube` | `'{}'`

### Kubernetes manifests
Manifests required to run this app completely headless are in `./manifests`. In addition to this the `./contrib` directory contains some manifests you may find useful:

- `./contrib/monitoring-config.yaml` contains configuration for monitoring. There are comments to indicate what each entry does.
- `./contrib/frontend.yaml` contains configuration for running the frontend status dashboard. Optionally, it includes a certificate resource for use with `kube-cert-manager`. The ingress host name (and certificate domain) needs changing to match what you use on your cluster.

It's recommended you download the releases zip and apply the manifests from there, rather than the repo directly, as the e2etest tag is set using semantic versioning git attributes.

## Metrics and alerts
All metrics created by this application are prefixed by `e2etest.` and are measured using the Statsd client library. For more information on Statsd metric types please see the [statsd project repo](https://github.com/etsy/statsd). We use a Statsd -> Prometheus bridge when deploying this because:

1. There are multiple tests running in multiple containers, so we can't use the prometheus client library which has a server running on a separate thread
1. We need to dedupe any metrics which are collected by multiple containers


### Time-based metrics
Time based metrics are bucketed into the statsd metric `e2etest.action.<namespace>.<resource>.<action>`.
Actions:
- create
- update
- scale
- delete
- http_get

Resources are any of those mentioned in the test list below. Note that http_get only applies to Services and Pods, and scale only applies to deployments.

### HTTP metrics
Request timings are sent to the time based metric bucket `e2etest.action.<namespace>.<resource>.http_get`.
Results of HTTP requests are sent to the counter metric `e2etest.http.<namespace>.<resource>.<result>`.
Resource can be:
- service
- pod

Result can be:
- any HTTP status code
- "wrong_response" meaning the response text didn't match what was expected

### Error metrics
Errors are counted using the statsd metric `e2etest.errors.<namespace>.<resource>.<area>.<error>`.

Given that there could be multiple categories of error generated by this application, see the [troubleshooting doc](troubleshooting.md) for more detail on the different [labels](https://prometheus.io/docs/concepts/data_model/#metric-names-and-labels) associated with `e2etest.errors`.

## Health check dashboard
The application provides a dashboard to quickly view the results of the last run of tests. The display shows the namespace in which the test is running, test name, status (passing or failing), info if the test is failing, and a timestamp of when the test was last run. 

This is intended to be a first port of call for identifying problems with the platform, supplemented by metrics from Prometheus, and has an ingress so you can view it at `status.<your domain here>`

To enable this view you need the manifests in `contrib/frontend.yaml` - example below

![Healthcheck Dashboard](frontend.png)

## Test List
### Namespace tests
1. create a namespace (name set by environment variable, or defaulted to kubee2etests)
1. check namespace exists
1. check namespace is empty
1. Delete the namespace
1. check namespace is deleted

### Service tests
1. create service in namespace
1. check service exists and has 0 endpoints
1. check service has N endpoints following deployment creation
1. check service has 0 endpoints following deployment scaling
1. delete service
1. check service is deleted

### Deployment tests
1. create deployment (some simple http container), N (default: 3) instances, DC anti affinity enabled
1. check pods have been scheduled
1. check pods have been started
1. check each pod is on a different node
1. check each pod's node is in a different data centre
1. delete deployment

#### Deployment update tests
1. update deployment to a different image
1. Check old pods are deleted
1. validate that all pods update to new image

#### Deployment scaling tests
1. scale deployment to 0
1. check pods are marked for deletion
1. check pods are deleted

### HTTP request tests
1. make http request to each pod individually
1. check each pod log output non empty
1. check pod log output includes http request to service/pod
1. make http request to service
1. check http request to each pod gets expected response
1. check http request to service gets expected response
1. make http request to service address N*2 times, get expected responses
1. make http request to each pod individually after update, get expected response
1. make http request to service address N*2 times after update, get expected responses


### DNS test
The DNS test attempts to resolve a well known name within the cluster from the test container itself - we added this because we found that DNS pods could be up but due to various issues DNS was not actually working at all.

The main script, `kubee2etests/scripts/test_runner.py` runs a different selection of the above tests based on a command line argument `suite`. It can be ran locally using `python3 kubee2etests/scripts/test_runner.py <suite>` using any of the test names below.

Test suite name | tasks
--- | --- 
namespace |  Create and check the namespace
deployment | Deployment creation and pod check tests
deployment_update | Create a deployment if it's not there, deployment update tests
deployment_scale | Create a deployment if it's not there, deployment scaling tests
service | Namespace, service creation tests
deployment_service | Create a service if it's not there, create a deployment if it's not there, service endpoints correct tests
deployment_scale_service | Create a service if it's not there, create a deployment if it's not there, scale the deployment, service endpoints correct tests
http |Create a service if it's not there, create a deployment if it's not there,  HTTP request tests
http_update | Create a service if it's not there, create a deployment if it's not there, deployment update tests, HTTP request tests
dns | Attempt to resolve name, report healthy if passed, failed if failed.

Excluding the DNS test (which has no namespace) and the namespace test, all tests assume the namespace defined in the environment variable `TEST_NAMESPACE` will exist when they start - if the namespace does not exist, the test will quit. 

