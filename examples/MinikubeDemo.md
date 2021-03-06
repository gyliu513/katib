# Simple Minikube Demo
You can deploy katib components and try a simple mnist demo on your laptop!

## Requirement
* VirtualBox
* Minikube
* kubectl

## deploy
Start Katib on Minikube with [deploy.sh](./MinikubeDemo/deploy.sh).
A Minikube cluster and Katib components will be deployed!

You can check them with `kubectl -n kubeflow get pods`.
Don't worry if the `vizier-core` get an error. 
It will be recovered after DB will be prepared.
Wait until all components will be Running status.

Then, start port-forward for katib services `6789 -> manager` and `8000 -> UI`.

kubectl v1.10~
```
$ kubectl -n kubeflow port-forward svc/vizier-core 6789:6789 &
$ kubectl -n kubeflow port-forward svc/katib-ui 8000:80 &
```

kubectl ~v1.9

```
& kubectl -n kubeflow port-forward $(kubectl -n kubeflow get pod -o=name | grep vizier-core | sed -e "s@pods\/@@") 6789:6789 &
& kubectl -n kubeflow port-forward $(kubectl -n kubeflow get pod -o=name | grep katib-ui | sed -e "s@pods\/@@") 8000:80 &
```

## Create Study
### Random Suggestion Demo
```
$ kubectl apply -f random-example.yaml
```
Only this command, a study will start, generate hyper-parameters and save the results.
The configurations for the study(hyper-parameter feasible space, optimization parameter, optimization goal, suggestion algorithm, and so on) are defined in `random-example.yaml`,
In this demo, hyper-parameters are embedded as args.
You can embed hyper-parameters in another way(e.g. environment values) by using template.
It is defined in `WorkerSpec.GoTemplate.RawTemplate`.
It is written in [go template](https://golang.org/pkg/text/template/) format.

In this demo, 3 hyper parameters 
* Learning Rate (--lr) - type: double
* Number of NN Layer (--num-layers) - type: int
* optimizer (--optimizer) - type: categorical
are randomly generated.

```
$ kubectl -n kubeflow get studyjob
NAME             AGE
random-example   2m
```

Check the study status.

```
$ kubectl -n kubeflow describe studyjobs random-example
Name:         random-example
Namespace:    kubeflow
Labels:       controller-tools.k8s.io=1.0
Annotations:  kubectl.kubernetes.io/last-applied-configuration={"apiVersion":"kubeflow.org/v1alpha1","kind":"StudyJob","metadata":{"annotations":{},"labels":{"controller-tools.k8s.io":"1.0"},"name":"random-example"...
API Version:  kubeflow.org/v1alpha1
Kind:         StudyJob
Metadata:
  Cluster Name:
  Creation Timestamp:  2018-08-15T01:29:13Z
  Generation:          0
  Resource Version:    173289
  Self Link:           /apis/kubeflow.org/v1alpha1/namespaces/kubeflow/studyjobs/random-example
  UID:                 9e136400-a02a-11e8-b88c-42010af0008b
Spec:
  Study Spec:
    Metricsnames:
      accuracy
    Name:                random-example
    Objectivevaluename:  Validation-accuracy
    Optimizationgoal:    0.98
    Optimizationtype:    maximize
    Owner:               crd
    Parameterconfigs:
      Feasible:
        Max:          0.03
        Min:          0.01
      Name:           --lr
      Parametertype:  double
      Feasible:
        Max:          3
        Min:          2
      Name:           --num-layers
      Parametertype:  int
      Feasible:
        List:
          sgd
          adam
          ftrl
      Name:           --optimizer
      Parametertype:  categorical
  Suggestion Spec:
    Request Number:         3
    Suggestion Algorithm:   random
    Suggestion Parameters:  <nil>
  Worker Spec:
    Command:
      python
      /mxnet/example/image-classification/train_mnist.py
      --batch-size=64
    Image:        katib/mxnet-mnist-example
    Worker Type:  Default
Status:
  Best Objective Value:         <nil>
  Conditon:                     Running
  Early Stopping Parameter Id:
  Studyid:                      qb397cc06d1f8302
  Suggestion Parameter Id:
  Trials:
    Trialid:  p18ee16163b85678
    Workeridlist:
      Objective Value: <nil>
      Conditon:        Running
      Workerid:        td08f74b9939350d
    Trialid:           pb1be3dbe53a5cb0
    Workeridlist:
      Objective Value: <nil>
      Conditon:        Running
      Workerid:        p2b23e25cce4092c
    Trialid:           m64209fe0867e91a
    Workeridlist:
      Objective Value: <nil>
      Conditon:        Running
      Workerid:        q6258c1ac98a00a5
Events:                <none>
```

When the Spec.Status.State becomes `Completed`, the study is completed.
You can look the result on `http://127.0.0.1:8000/katib`.

### Use ConfigMap for Worker Template
In Random example, the template for workers is defined in StudyJob manifest.
A ConfigMap is also used for worker template.
Let's use [this](./workerConfigMap.yaml) template.
```
kubectl apply -f workerConfigMap.yaml
```
This template will be shared among the three demos below(Grid, Hyperband, and GPU).

### Grid Demo
Almost same as random suggestion.

In this demo, Katib will make 4 grids for learning rate (--lr) Min 0.03 and Max 0.07.
```
kubectl apply -f grid-example.yaml
```

### Hyperband Demo
In this demo, the eta is 3 and the R is 9.
```
kubectl apply -f hypb-example.yaml
```

## UI
You can check your study results with Web UI.
Acsess to `http://127.0.0.1:8000/katib`
The Results will be saved automatically.

### Using GPU demo
You can set any configuration for your worker pods.
Here, try to set config for GPU.
The manifest of the worker pods are generated from a template.
The templates are defined in [ConfigMap](./workerConfigMap.yaml).
There are two templates: defaultWorkerTemplate.yaml and gpuWorkerTemplate.yaml.
You can add your template for worker.
Then you should specify the template in your studyjob spec.
[This example](/examples/gpu-example.yaml) uses `gpuWorkerTemplate.yaml`.
Set "/worker-template/gpuWorkerTemplate.yaml" at `workerTemplatePath` field and specify gpu number at `workerParameters/Gpu`.
You can apply it same as other examples.
```
$ kubectl apply -f gpu-example.yaml
$ kubectl -n kubeflow get studyjob

NAME             AGE
gpu-example      1m
random-example   17m

$ kubectl -n kubeflow describe studyjob gpu-example

Name:         gpu-example
Namespace:    kubeflow
Labels:       controller-tools.k8s.io=1.0
Annotations:  kubectl.kubernetes.io/last-applied-configuration={"apiVersion":"kubeflow.org/v1alpha1","kind":"StudyJob","metadata":{"annotations":{},"labels":{"controller-tools.k8s.io":"1.0"},"name":"gpu-example","n...
API Version:  kubeflow.org/v1alpha1
Kind:         StudyJob
Metadata:
  Cluster Name:
  Creation Timestamp:  2018-08-15T01:48:12Z
  Generation:          0
  Resource Version:    175002
  Self Link:           /apis/kubeflow.org/v1alpha1/namespaces/kubeflow/studyjobs/gpu-example
  UID:                 44afac4c-a02d-11e8-b88c-42010af0008b
Spec:
  Study Spec:
    Metricsnames:
      accuracy
    Name:                gpu-example

	...

  Worker Spec:
    Command:
      python
      /mxnet/example/image-classification/train_mnist.py
      --batch-size=64
    Image:  katib/mxnet-mnist-example
    Worker Parameters:
      Gpu:                 1
    Worker Template Path:  /worker-template/gpuWorkerTemplate.yaml
    Worker Type:           Default
Status:
  Best Objective Value:         <nil>
  Conditon:                     Running
  Early Stopping Parameter Id:
  Studyid:                      k549e927046f2136
  Suggestion Parameter Id:
  Trials:
    Trialid:  t721857cd426b68b
    Workeridlist:
      Objective Value: <nil>
      Conditon:        Running
      Workerid:        g07cba174ada521e
    Trialid:           f27c0ac1c6664533
    Workeridlist:
      Objective Value: <nil>
      Conditon:        Running
      Workerid:        h8d5062f2f1b8633
    Trialid:           v129109d1331a98e
    Workeridlist:
      Objective Value: <nil>
      Conditon:        Running
      Workerid:        x8f172a64645690e
```

Check if the GPU configuration works correctly.

```
$ kubectl -n kubeflow describe pod g07cba174ada521e-88wpn
Name:           g07cba174ada521e-88wpn
Namespace:      kubeflow
Node:           <none>
Labels:         controller-uid=44bfb99f-a02d-11e8-b88c-42010af0008b
                job-name=g07cba174ada521e
Annotations:    <none>
Status:         Pending
IP:
Controlled By:  Job/g07cba174ada521e
Containers:
  g07cba174ada521e:
    Image:  katib/mxnet-mnist-example
    Port:   <none>
    Command:
      python
      /mxnet/example/image-classification/train_mnist.py
      --batch-size=64
      --lr=0.0175
      --num-layers=2
      --optimizer=adam
    Limits:
      nvidia.com/gpu:  1
    Requests:
      nvidia.com/gpu:  1
    Environment:       <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-knffp (ro)
Conditions:
  Type           Status
  PodScheduled   False
Volumes:
  default-token-knffp:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-knffp
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
                 nvidia.com/gpu:NoSchedule
Events:
  Type     Reason            Age               From               Message
  ----     ------            ----              ----               -------
  Warning  FailedScheduling  6s (x21 over 4m)  default-scheduler  0/3 nodes are available: 3 Insufficient nvidia.com/gpu.
```

## Metrics Collection

### Design of Metrics Collector
![metricscollectordesign](https://user-images.githubusercontent.com/10014831/47256754-e32cb480-d4bf-11e8-98e9-4bbec562ad75.png)

### Default Metrics Collector

The default metrics collector collects metrics from the StdOut of workers.
It is deployed as a cronjob. It will collect and report metrics periodically.
It collects metrics through k8s pod log API.
You should print logs in {metrics name}={value} style.
In the above demo, the objective value name is *Validation-accuracy* and the metrics are [*accuracy*], so your training code should print like below.
```
epoch 1:
batch1 accuracy=0.3
batch2 accuracy=0.5

Validation-accuracy=0.4

epoch 2:
batch1 accuracy=0.7
batch2 accuracy=0.8

Validation-accuracy=0.75
```
The metrics collector will collect all logs of metrics.
The manifest of metrics collector is also generated from template and defined [here](/manifests/studyjobcontroller/metricsControllerConfigMap.yaml).
You can add your template and specify `spec.metricsCollectorSpec.metricsCollectorTemplatePath` in a studyjob manifest.

### TF Event File Metrics Collector

The TF Event file metrics collector will collect metrics from tf.event files.
It is also deployed as a cronjob.
When you use TF Event File Metrics Collector, you need to share files between the metrics collector and the worker by PVC.
There is an example for TF Event file metrics collector.
First, please create PV and PVC to share event file.
```
$ kubectl apply -f tfevent-volume/
```
Then, create a studyjob that uses TF Event file metrics collector.
```
$ kubectl apply -f tf-event_test.yaml
```

It will create a tensorflow worker from whose eventfile metrics are collected.
The code of tensorflow is [the official tutorial for mnist with summary](https://github.com/tensorflow/tensorflow/blob/master/tensorflow/examples/tutorials/mnist/mnist_with_summaries.py).
It will save event file to `/log/train` and `/log/test` directory.
They have same named metrics ('accuracy' and 'cross_entropy').
The accuracy in training and test will be saved in *train/* directory and *test/* directory respectively.
In a studyjob, please add directry name to the name of metrics as a prefix e.g. `train/accuracy`, `test/accuracy`.

## ModelManagement

You can export model data to yaml file with CLI.

```
katib-cli -s {{server-cli}} pull study {{study ID or name}}  -o {{filename}}
```

And you can push your existing models to Katib with CLI.
`mnist-models.yaml` trains 22 models using random suggestion with the Parameter Config below.

```
configs:
    - name: --lr
      parametertype: 1
      feasible:
        max: "0.07"
        min: "0.03"
        list: []
    - name: --lr-factor
      parametertype: 1
      feasible:
        max: "0.05"
        min: "0.005"
        list: []
    - name: --lr-step
      parametertype: 2
      feasible:
        max: "20"
        min: "5"
        list: []
    - name: --optimizer
      parametertype: 4
      feasible:
        max: ""
        min: ""
        list:
        - sgd
        - adam
        - ftrl
```
You can explore the model easily on KatibUI.

```
katib-cli push md -f mnist-models.yaml
```

## Clean
Clean up with `./destroy.sh` script.
It will stop port-forward process and delete minikube cluster.
