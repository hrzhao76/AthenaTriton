# Deployment on the NRP Nautilus Kubernetes (k8) cluster with `SuperSONIC`

For server-side large-scale deployment we are using the [SuperSONIC](https://fastmachinelearning.org/SuperSONIC/index.html) 
framework. To begin, clone the repository

```
git clone git@github.com:fastmachinelearning/SuperSONIC.git
```

### Deploying the Server

Deploying the server on Nautilus is as easy as sourcing the setup script:

```bash
source deploy-nautilus-atlas.sh
```

The settings are defined in `values/values-nautilus-atlas.yaml` files. To select which GPUs to run on, uncomment the appropriate lines of code corresponding to the following possible selections:

```
    - NVIDIA-A10
    - NVIDIA-A40
    - NVIDIA-A100-SXM4-80GB
    - NVIDIA-L40
    - NVIDIA-A100-80GB-PCIe
    - NVIDIA-A100-80GB-PCIe-MIG-1g.10gb
    - NVIDIA-L4
    - NVIDIA-A100-PCIE-40GB
    - NVIDIA-GH200-480GB
```

```{note}
To run on A100s at NRP, you have to reserve a time. This can be accomplished at [this link](https://portal.nrp.ai/reservations/)
```

To load multiple models per GPU, edit the `triton.args` string to:

```
  args:
    - |
      /opt/tritonserver/bin/tritonserver \
      --model-repository=/traccc-aaS/traccc-aaS/backend/nmodels_<NUMBER_OF_MODELS> \
      --log-verbose=1 \
      --exit-on-error=true
```
and be sure to replace `<NUMBER_OF_MODELS>` with the number of Triton model instances you'd like to load. By default, this loads one model instance. 

Finally, to run across multiple GPUs, increase or decrease the `replicas: n` where `n` is the number of GPUs to request. This loads one Triton server per GPU with the request number of models you've selected. 

For more information on configuring the server, consult the [SuperSONIC Configuration Guide](https://fastmachinelearning.org/SuperSONIC/configuration-guide.html). 

### Running the client

In order for the client to interface with the server, the location of the server needs to be specified. First, ensure the server is running

```bash
kubectl get pods -n atlas-sonic
```
which has output something like:

```
NAME                            READY   STATUS    RESTARTS   AGE
envoy-atlas-7f6d99df88-667jd    1/1     Running   0          86m
triton-atlas-594f595dbf-n4sk7   1/1     Running   0          86m
```

or use the [k9s](https://k9scli.io) tool to manage your pods. You can then check everything is healthy with

```bash
curl -kv https://atlas.nrp-nautilus.io/v2/health/ready
```

which should produce somewhere in the output the lines:

```
< HTTP/1.1 200 OK
< Content-Length: 0
< Content-Type: text/plain
```

Then, the client can be run with, for instance:

```bash
python TracccTritonClient.py -u atlas.nrp-nautilus.io --ssl
```

To see what's going on from the server side, run

```bash
kubectl logs triton-atlas-594f595dbf-n4sk7
```

where `triton-atlas-594f595dbf-n4sk7` is the name of the server found when running the `get pods` command above. 

To run with [`perf_analyzer`](https://docs.nvidia.com/deeplearning/triton-inference-server/archives/triton-inference-server-2280/user-guide/docs/user_guide/perf_analyzer.html) and make plots, consult the [performance repo](https://github.com/milescb/traccc-aaS-performance/tree/main). 

### !!! Important !!!

Make sure to `uninstall` once the server is not needed anymore. 

```bash
helm uninstall atlas-sonic -n atlas-sonic
```

Make sure to read the [Policies](https://docs.nationalresearchplatform.org/userdocs/start/policies/) before using Nautilus. 