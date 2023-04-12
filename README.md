# KServe-Fluid-Demo

Demo to use Fluid to accelerate KServe on downloading large model

## Prerequisite

* [KServe](https://kserve.github.io/website/master/admin/serverless/serverless/)
  * Please follow this [guideline](https://kserve.github.io/website/master/developer/developer/#deploy-kserve-from-master-branch) to deploy KServe from `master` branch
  * Enable node selector
    ```sh

    kubectl patch configmap/config-features \
      --namespace knative-serving \
      --type merge \
      --patch '{"data":{"kubernetes.podspec-nodeselector":"enabled"}}'
    ```
* [Fluid](https://github.com/fluid-cloudnative/fluid/blob/master/docs/en/userguide/get_started.md)
  * Deploy Fluid from `master` branch
    ```sh
    # Deploy Fluid from master branch
    git clone https://github.com/fluid-cloudnative/fluid

    # if you are using alluxio runtime
    helm upgrade --install \
        --set runtime.alluxio.enabled=true \
        --set webhook.reinvocationPolicy=IfNeeded \
        fluid --create-namespace --namespace=fluid-system fluid/charts/fluid/fluid

    # if you are using jindo runtime
    helm upgrade --install \
        --set runtime.jindo.enabled=true \
        --set webhook.reinvocationPolicy=IfNeeded \
        fluid --create-namespace --namespace=fluid-system fluid/charts/fluid/fluid
    ```

## Prepare Application

I will build a demo application using KServe to serve a Large Language Model (LLM)

### Build and Push Application Image

```sh
export DOCKER_REPO='docker.io/<username>'

# build and push application image
cd docker
docker build -f Dockerfile -t ${DOCKER_REPO}/kserve-fluid:bloom-gpu-v1 .
docker push ${DOCKER_REPO}/kserve-fluid:bloom-gpu-v1
cd ..
```

### Prepare Model Artifact

Download model from HuggingFace, Bloom models are using in this tutorial

```sh
python -m venv .venv
source .venv/bin/activate
pip install --upgrade pip
pip install 'transformers[torch]'

# create models folder
mkdir -p models

# please check https://huggingface.co/docs/transformers/model_doc/bloom for other bloom models and update the script accordingly
python download_model.py --model_name="bigscience/bloom-560m" --model_dir="models"
# output_dir: models/models--bigscience--bloom-560m/snapshots/e985a63cdc139290c5f700ff1929f0b5942cced2
```

Upload the model to S3 bucket
```sh
# update the path accordingly
aws s3 cp --recursive ${output_dir} s3://${bucket}/models/bloom-560m
```

### Run The Application Locally

```sh
docker run -it --rm -p 8080:8080 \
  -v $(pwd)/models:/mnt/models \
  -e MODEL_URL=/mnt/${output_dir} \
  -e MODEL_NAME=bloom \
  ${DOCKER_REPO}/kserve-fluid:bloom-gpu-v1
```

### Test The Application

```sh
curl -i -X POST -H "Content-Type: application/json" "localhost:8080/v1/models/bloom:predict" -d '{"prompt": "It was a dark and stormy night", "result_length": 50}'
```

## Demo

### Enable Direct VolumeMount for PVC In KServe

```sh
# edit inferenceservice-config and update enableDirectPvcVolumeMount to true
kubectl edit configmap inferenceservice-config  -n kserve

#   storageInitializer: |-
#     {
#         "image" : "kserve/storage-initializer:latest",
#         "memoryRequest": "100Mi",
#         "memoryLimit": "1Gi",
#         "cpuRequest": "100m",
#         "cpuLimit": "1",
#         "storageSpecSecretName": "storage-config",
#         "enableDirectPvcVolumeMount": false        # change to true
#     }
```

### Create Demo Namespace

```sh
kubectl create ns kserve-fluid-demo

# add label to inject fluid sidecar
kubectl label namespace kserve-fluid-demo fluid.io/enable-injection=true
```

### Create S3 Credentials Secret And Service Account

```sh
# please update the s3 credentials to yours
# NOTE: for Jindo runtime, https is not supported in current version
kubectl create -f s3creds.yaml -n kserve-fluid-demo
```

### Create Dataset And Runtime

Create Dataset and Runtime (In this tutorial I used `JindoFS` as the example, there are other runtimes like JuiceFS and Alluxio)

```sh
# please update the `mountPoint` and `options`
kubectl create -f jindo.yaml -n kserve-fluid-demo

dataset.data.fluid.io/s3-data created
jindoruntime.data.fluid.io/s3-data created
```


Check the status

```sh
kubectl get po -n kserve-fluid-demo

NAME                       READY   STATUS    RESTARTS   AGE
s3-data-jindofs-master-0   1/1     Running   0          28s
s3-data-jindofs-worker-0   1/1     Running   0          5s
s3-data-jindofs-worker-1   1/1     Running   0          5s


kubectl get jindoruntime,dataset -n kserve-fluid-demo

NAME                                 MASTER PHASE   WORKER PHASE   FUSE PHASE   AGE
jindoruntime.data.fluid.io/s3-data   Ready          Ready          Ready        70s

NAME                            UFS TOTAL SIZE   CACHED   CACHE CAPACITY   CACHED PERCENTAGE   PHASE   AGE
dataset.data.fluid.io/s3-data   3.14GiB          0.00B    100.00GiB        0.0%                Bound   70s
```

A PVC is created to mount the dataset into the application

```
kubectl get pvc -n kserve-fluid-demo

NAME      STATUS   VOLUME                      CAPACITY   ACCESS MODES   STORAGECLASS   AGE
s3-data   Bound    kserve-fluid-demo-s3-data   100Pi      ROX            fluid          65s
```

### Preload The Data

Preload the data into workers to improve the performance

```sh
# please update the `path` under `target`
kubectl create -f dataload.yaml -n kserve-fluid-demo
```

Check the dataload status

```sh
kubectl get dataload -n kserve-fluid-demo

NAME          DATASET   PHASE      AGE     DURATION
s3-dataload   s3-data   Complete   4m50s   2m22s

kubectl get dataset -n kserve-fluid-demo

NAME      UFS TOTAL SIZE   CACHED    CACHE CAPACITY   CACHED PERCENTAGE   PHASE   AGE
s3-data   3.14GiB          3.02GiB   100.00GiB        96.2%               Bound   13m
```

Data are cached successfully

### Deploy The Demo Application

Create the InferenceService with Fluid

```sh
# please update the `STORAGE_URI`
kubectl create -f fluid-isvc.yaml -n kserve-fluid-demo

# run this command if you want to use the KServe Storage Initializer
kubectl create -f kserve-isvc.yaml -n kserve-fluid-demo
```

### Test The Demo Application

```sh
export MODEL_NAME=fluid-bloom
# export MODEL_NAME=kserve-bloom
export SERVICE_HOSTNAME=$(kubectl get inferenceservice ${MODEL_NAME} -n kserve-fluid-demo -o jsonpath='{.status.url}' | cut -d "/" -f 3)
export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].port}')

curl -v -H "Content-Type: application/json" -H "Host: ${SERVICE_HOSTNAME}" "http://${INGRESS_HOST}:${INGRESS_PORT}/v1/models/bloom:predict" -d '{"prompt": "It was a dark and stormy night", "result_length": 50}'
```

### Performance Benchmark

To conduct a performance benchmark, we will compare the scaling time of the inference service with Storage initializer and Fluid across different model artifacts, measuring the time it takes for the service to scale from 0 to 1.

We will use the following machine types for our tests:

* Data: m5.xlarge
* Workload: m5.2xlarge, m5.4xlarge

We will use the following Fluid runtimes for our tests:

* JindoFS

For the purposes of our benchmark, we will assume that nodes are readily available and that images are cached in the nodes, so we will not be factoring in node provisioning time or image pulling time.

Other assumptions:

* S3 bucket and Cluster are in the same region (eu-central-1)


Command to measure the total time:

```sh
curl --connect-timeout 3600 --max-time 3600 -o /dev/null -s -w 'Total: %{time_total}s\n' -H "Content-Type: application/json" -H "Host: ${SERVICE_HOSTNAME}" "http://${INGRESS_HOST}:${INGRESS_PORT}/v1/models/bloom:predict" -d '{"prompt": "It was a dark and stormy night", "result_length": 50}'
# Total time: 53.037210s, status code: 200
```

| Model Name            | Model Size | Machine Type | KServe + Storage Initializer                         | KServe + Fluid(JindoFS)                    |
|-----------------------|------------|--------------|------------------------------------------------------|--------------------------------------------|
| bigscience/bloom-560m | 3.14GB     | m5.2xlarge   | total: 52.725s (download: 25.091s, load: 3.549s)     | total: 23.286s (load: 4.763s) (2 workers)  |
| bigscience/bloom-7b1  | 26.35GB    | m5.4xlarge   | total: 365.479s (download: 219.844s, load: 102.299s) | total: 53.037s (load: 24.137s) (3 workers) |


### Clean Up

```sh
kubectl delete -f fluid-isvc.yaml -n kserve-fluid-demo
kubectl delete -f kserve-isvc.yaml -n kserve-fluid-demo

kubectl delete -f dataload.yaml -n kserve-fluid-demo
kubectl delete -f jindo.yaml -n kserve-fluid-demo

kubectl delete -f s3creds.yaml -n kserve-fluid-demo

# uninstall Fluid
helm delete fluid -n fluid-system
kubectl get crd | grep data.fluid.io | awk '{ print $1 }' | xargs kubectl delete crd
kubectl delete ns fluid-system
```

## Reference
* [KNative with Fluid](https://github.com/fluid-cloudnative/fluid/blob/master/docs/en/samples/knative.md)
* [JindoFS](https://www.alibabacloud.com/blog/introducing-jindofs-a-high-performance-data-lake-storage-solution_595600)
* [KServe with S3](https://kserve.github.io/website/0.8/modelserving/storage/s3/s3/)
* [Bloom](https://huggingface.co/docs/transformers/model_doc/bloom)