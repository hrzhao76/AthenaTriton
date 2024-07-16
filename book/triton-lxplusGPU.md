Quick an short tutorial to setup example backend on lxplus

This is based on the Nivida example tutorial for Pytorch backend develment



### Get the Pytorch model

```bash
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
```
model_repository
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

Open another terminal to run the

Using the LXPLUS-GPU, we need to make sure using the same machine as the one with Server to avoid the need to deal with authentication.  

```bash 
# Longin to same LXPLUS-GPU

ssh 


export IMAGE_FOLDER="/eos/user/{initial}/{whoami}/TritonDemo/"

singularity --dir $IMAGE_FOLDER  pull docker:/nvcr.io/nvidia/tritonserver:22.04-py3-sdk


singularity run --nv -e  -B /cvmfs:/cvmfs -B /afs/cern.ch/user/{initial}:/home -B /afs/cern.ch/user/{initial}/{whoami}:/srv -B /afs:/afs -B /eos:/eos tritonserver_22.04-py3-sdk.sif
# Need to get the correct version of torch and torchvision
python -m pip install torchvision=0.17

# Check if the connection is ok 
curl -v localhost:8000/v2/health/ready

You should see the following message if the connect and ther server is in good state

< HTTP/1.1 200 OK
< Content-Length: 0
< Content-Type: text/plain

```


Now are are ready to run the client script!

```bash
cd {TritonDemo}/tutorials/Quick_Deploy/PyTorch/

python client.py 

```

It will take sometime depend on the GPU utilzation. But you shall be able to see the following printout if wverything goes well

```bash 

[b'12.474469:90' b'11.525709:92' b'9.660509:14' b'8.406358:136'
 b'8.220254:11']

```






