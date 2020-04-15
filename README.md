# Worker Pod Autoscaler

[![GoDoc Widget]][GoDoc] [![CI Status](https://api.travis-ci.com/practo/k8s-worker-pod-autoscaler.svg?token=yTs54joHywqVVdshXhPm&branch=master)](https://travis-ci.com/practo/k8s-worker-pod-autoscaler)

<img src="/artifacts/images/worker-pod-autoscaler.png" width="120">

----

Scale kubernetes pods based on the Queue length of a queue in a Message Queueing Service. Worker Pod Autoscaler automatically scales the number of pods in a deployment based on observed queue length.

Currently the supported Message Queueing Services is only AWS SQS. There is a plan to integrate other commonly used message queing services.

----

# Install the WorkerPodAutoscaler

### Install
Running the below script will create the WPA [CRD](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/) and start the controller. The controller watches over all the specified queues in AWS SQS and scales the Kubernetes deployments based on the specification.

```bash
export AWS_REGIONS='ap-south-1,ap-southeast-1'
export AWS_ACCESS_KEY_ID='sample-aws-access-key-id'
export AWS_SECRET_ACCESS_KEY='sample-aws-secret-acesss-key'
./hack/install.sh
```

Note: IAM policy required is [this](artifacts/iam-policy.json).

### Verify Installation
Check the wpa resource is accessible using kubectl

```bash
kubectl get wpa
```

# Example
Do install the controller before going with the example.

- Create Deployment that needs to scale based on queue length.
```bash
kubectl create -f artifacts/examples/example-deployment.yaml
```

- Create `WPA object (example-wpa)` that will start scaling the `example-deployment` based on SQS queue length.
```bash
kubectl create -f artifacts/examples/example-wpa.yaml
```

This will start scaling `example-deployment` based on SQS queue length.

## Configuration

## WPA Resource

```yaml
apiVersion: k8s.practo.dev/v1alpha1
kind: WorkerPodAutoScaler
metadata:
  name: example-wpa
spec:
  minReplicas: 0
  maxReplicas: 10
  deploymentName: example-deployment
  queueURI: https://sqs.ap-south-1.amazonaws.com/{{aws_account_id}}/{{queue_prefix-queue_name-queue_suffix}}
  targetMessagesPerWorker: 2
  secondsToProcessOneJob: 0.03
  minAvailableReplicas: 0
```

### Flags documentation:
- **minReplicas**: minimum number of workers you want to run. (mandatory)
- **maxReplicas**: maximum number of workers you want to run. (mandatory)
- **deploymentName**: name of the kubernetes deployment in the same namespace as WPA object. (mandatory)
- **queueURI**: full URL of the queue. (mandatory)
- **targetMessagesPerWorker**: Number of jobs in the queue which have not been picked up by the workers. This also used to calculate the desired number of workers. (mandatory)
- **secondsToProcessOneJob:**: secondsToProcessOneJob specifies seconds require to process one job by one worker process. `secondsToProcessOneJob * 60` gives the number of jobs that can be processed in a minute by a single worker. We use this metric in combination with `messages sent to the queue` metric to find the desired number of minimum workers. Default of the metric is 0.0. in which case **this functionality of calculating minimum becomes disabled** for you. We recommend to specify this value as it helps in accurate calculation of desired workers. This is very useful for workers which process really fast and have messages in the queue(ApproximateNumberOfMessagesVisible in case of SQS) as zero. (optional, highly recommended)
- **minAvailableReplicas:** this specifies the worker disruption budget. Minimum number of pods that should be available at all times. Scale down can only happen to a value greater than `minAvailableReplicas`. By default workers are expected to be idempotent and can tolerate disruptions due to scale down. So the default is 0 i.e. scale down can happen when some workers are doing the processing. [MoreInfo](https://github.com/practo/k8s-worker-pod-autoscaler/issues/50). **If your workers which are doing the processing cannot tolerate the disruption due to scale down, set this value accordingly.** (optional, default=0)

## WPA Controller

```
$ bin/darwin_amd64/workerpodautoscaler run --help
Run the workerpodautoscaler

Usage:
  workerpodautoscaler run [flags]

Examples:
  workerpodautoscaler run

Flags:
      --aws-regions string            comma separated aws regions of SQS (default "ap-south-1,ap-southeast-1")
      --k8s-api-burst int              maximum burst for throttle between requests from clients(wpa) to k8s api (default 10)
      --k8s-api-qps float              qps indicates the maximum QPS to the k8s api from the clients(wpa). (default 5)
  -h, --help                          help for run
      --kube-config string            path of the kube config file, if not specified in cluster config is used
      --metrics-port string           specify where to serve the /metrics and /status endpoint. /metrics serve the prometheus metrics for WPA (default ":8787")
      --resync-period int             sync period for the worker pod autoscaler (default 20)
      --sqs-long-poll-interval int    the duration (in seconds) for which the sqs receive message call waits for a message to arrive (default 20)
      --sqs-short-poll-interval int   the duration (in seconds) after which the next sqs api call is made to fetch the queue length (default 20)
      --wpa-threads int               wpa threadiness, number of threads to process wpa resources (default 10)
```
**Note:** `k8s-api-burst` and `k8s-api-qps` flags may need to be tweaked when you are running WPA at scale.

## WPA Metrics

WPA emits the following prometheus metrics at `:8787/metrics`.
```
wpa_controller_loop_count_success{workerpodautoscaler="example-wpa", namespace="example-namespace"} 23140
wpa_controller_loop_duration_seconds{workerpodautoscaler="example-wpa", namespace="example-namespace"} 0.39
go_goroutines{endpoint="workerpodautoscaler-metrics"}	5843
```

If you have [ServiceMonitor](https://github.com/coreos/prometheus-operator/blob/master/Documentation/user-guides/getting-started.md) installed in your cluster. You can bring these metrics to Prometheus by running the following:
```
kubectl create -f artifacts/service.yaml
kubctl create -f artifacts/servicemonitor.yaml
```

# Why make a separate autoscaler CRD ?

Go through [this medium post](https://medium.com/practo-engineering/launching-worker-pod-autoscaler-3f6079728e8b) for details.

Kubernetes does support custom metric scaling using Horizontal Pod Autoscaler. Before making this we were using HPA to scale our worker pods. Below are the reasons for moving away from HPA and making a custom resource:

**TLDR;** Don't want to write and maintain custom metric exporters? Use WPA to quickly start scaling your pods based on queue length with minimum effort (few kubectl commands and you are done !)

1. **No need to write and maintain custom metric exporters**: In case of HPA with custom metrics, the users need to write and maintain the custom metric exporters. This makes sense for HPA to support all kinds of use cases. WPA comes with queue metric exporters(pollers) integrated and the whole setup can start working with 2 kubectl commands.

2. **Different Metrics for Scaling Up and Down**: Scaling up and down metric can be different based on the use case. For example in our case we want to scale up based on SQS `ApproximateNumberOfMessages` length and scale down based on `NumberOfMessagesReceived`. This is because if the worker jobs watching the queue is consuming the queue very fast, `ApproximateNumberOfMessages` would always be zero and you don't want to scale down to 0 in such cases.

3. **Fast Scaling**: We wanted to achieve super fast near real time scaling. As soon as a job comes in queue the containers should scale if needed. The concurrency, speed and interval of sync have been made configurable to keep the API calls to minimum.

4. **On-demand Workers:** min=0 is supported. It's also supported in HPA.

## Release

- Decide a `tag` and bump up the `tag` [here](https://github.com/practo/k8s-worker-pod-autoscaler/blob/master/artifacts/deployment-template.yaml#L26) and create and merge the pull request.

- Get the latest master code.
```
git clone https://github.com/practo/k8s-worker-pod-autoscaler
cd k8s-worker-pod-autoscaler
git pull origin master

```

- Build and push the image to `hub.docker.com/practodev`. Note: practodev push access is required.
```
git fetch --tags
git tag v0.2.2
make push
```

- Create a Release in Github. Refer this https://github.com/practo/k8s-worker-pod-autoscaler/releases/tag/v0.2.0 and create a release. Release should contain the Changelog information of all the issues and pull request after the last release.

-  Publish the release in Github 🎉

- For first time deployment use [this](https://github.com/practo/k8s-worker-pod-autoscaler/blob/master/artifacts/deployment-template.yaml).

- For future deployments. Edit the image in deployment with the new `tag`.
```
kubectl edit deployment -n kube-system workerpodautoscaler
```

## Contributing
It would be really helpful to add all the major message queuing service providers. This [interface](https://github.com/practo/k8s-worker-pod-autoscaler/blob/master/pkg/queue/queueing_service.go#L5-L8) implementation needs to be written down to make that possible.

- After making code changes, run the below commands to build and run locally.
```
$ make build
making bin/darwin_amd64/workerpodautoscaler

$ bin/darwin_amd64/workerpodautoscaler run --kube-config /home/user/.kube/config
```

- To add a new dependency use `go mod vendor`
- Dependency management using go modules - https://github.com/liggitt/gomodules/blob/master/README.md
- Get up to speed with go in no time - https://gobyexample.com
- Install go locally - https://golang.org/doc/install
- How to write go code - https://golang.org/doc/code.html

## Thanks

Thanks to kubernetes team for making [crds](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/) and [sample controller](https://github.com/kubernetes/sample-controller)

[latest-release]: https://github.com/practo/k8s-worker-pod-autoscaler/releases
[GoDoc]: https://godoc.org/github.com/practo/k8s-worker-pod-autoscaler
[GoDoc Widget]: https://godoc.org/github.com/practo/k8s-worker-pod-autoscaler?status.svg
