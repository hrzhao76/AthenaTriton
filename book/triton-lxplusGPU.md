
## Simple Example to run on LXPLUS-GPU

Quick and short tutorial to setup example backend on lxplus-GPU to understand the basics of the Tirton backend and client

This is based on the Nivida example tutorial for [Pytorch backend](https://github.com/triton-inference-server/tutorials/tree/main/Quick_Deploy/PyTorch) and adopted to get it to work on LXPLUS-GPU node 


More information for LXPLUS-GPU can be found [here](https://clouddocs.web.cern.ch/gpu/index.html)


### Get the Pytorch resnet50 model

```bash
# Connect to lxplus GPU mode
ssh {user_name}@lxplus.gpu.cern.ch

#Create work directory 
cd ~
mkdir TritonDemo

#  Clone the Official tutorial 
git clone https://github.com/triton-inference-server/tutorials.git .

# Cache directory
SINGULARITY_CACHEDIR="/eos/user/{INITIAL}/{YOUR_ACCOUNT}/singularity/"

# image folder, beeter to store in EOS
export IMAGE_FOLDER="/eos/user/{INITIAL}/{YOUR_ACCOUNT}/TritonDemo/"

#Pull the image
singularity pull --dir $IMAGE_FOLDER docker://nvcr.io/nvidia/pytorch:22.04-py3

# Run the image
singularity run --nv -B /afs -B /eos -B /cvmfs pytorch_22.04-py3.sif

# Get the model.pt
python export.py

```


### Prepare the model configs and structure 

The model repository needs to fulfill certain structures and names. As shown in the following. More detail information for other backend can be found in the [official documentiton](https://github.com/triton-inference-server/server/blob/main/docs/user_guide/model_repository.md). 


```
models
|
+-- resnet50
    |
    +-- config.pbtxt
    +-- 1
        |
        +-- model.pt
```



### Set Up Triton Inference Server

```bash 
singularity run --nv -e --no-home -B {YOUR_MODEL_FOLDER}:/models tritonserver_22.04-py3.sif

tritonserver --model-repository=/models

```

### Setup Client 

Open another terminal to run the client script and send the inference request. 

Using the LXPLUS-GPU, we need to make sure to use the same machine as the one with the Server to avoid the need to deal with authentication. 
```bash 
# Longin to same LXPLUS-GPU
# You need to replace the XXX with the same node number as the server above.
ssh {user_name}@lxplusXXX.cern.ch

export IMAGE_FOLDER="/eos/user/{initial}/{whoami}/TritonDemo/"

singularity --dir $IMAGE_FOLDER  pull docker:/nvcr.io/nvidia/tritonserver:22.04-py3-sdk


singularity run --nv -e  -B /cvmfs:/cvmfs -B /afs/cern.ch/user/{initial}:/home -B /afs/cern.ch/user/{initial}/{whoami}:/srv -B /afs:/afs -B /eos:/eos tritonserver_22.04-py3-sdk.sif
# Need to get the correct version of torch and torchvision
python -m pip install torchvision=0.17

# Check if the connection is ok 
curl -v localhost:8000/v2/health/ready
```

You should see the following message if the connect and ther server is in good state

```
< HTTP/1.1 200 OK
< Content-Length: 0
< Content-Type: text/plain

```

Now, we are ready to run the client script!


```bash
# Move to folder storing the script
cd {TritonDemo}/tutorials/Quick_Deploy/PyTorch/

# Very simple script to send a images to server
python client.py 

```

It will take some time, depending on the GPU utilization. But you shall be able to see the following printout if everything goes well

```bash 

[b'12.474469:90' b'11.525709:92' b'9.660509:14' b'8.406358:136'
 b'8.220254:11']
```

You get a triton client and server talking to each other!




