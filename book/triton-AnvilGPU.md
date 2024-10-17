## Simple Example to run on Anvil-GPU at Purdue

A quick and short tutorial to set up an example backend on Anvil Notebook Service with GPU to understand the basics of the Tirton backend and client

This is based on the Nivida example tutorial for [Pytorch backend](https://github.com/triton-inference-server/tutorials/tree/main/Quick_Deploy/PyTorch) and adopted to get it to work on Anvil Notebook Service 

Follow the following link to login to [Anvil Notebook Service](https://notebook.anvilcloud.rcac.purdue.edu/hub/login)


### Step 1: Setup the Notebook
 - Choose `Apptainer Notebook - Run apptainer inside of this notebook`
 - Choose the `Launcher` and open two `terminal` tabs. We will need one to set up the server and one for the client.  


### Get the Pytorch resnet50 model


This step tries to get the resnet50 model in the pytorch `.pt` files extension.


```{note}

You can copy it from `/shared-storage/pytorch/model.pt`
 to the folder where you plan to store the PyTorch model.

cp /shared-storage/pytorch/model.pt .

```

### Prepare the model configs 

For each model, you will need a model configuration file. Depending on the backend you are using; there is a fixed rule for how the files need to be structured.

For the PyTorch model, what you need is the following.

```Json
name: "resnet50"
platform: "pytorch_libtorch"
max_batch_size : 0
input [
 {
    name: "input__0"
    data_type: TYPE_FP32
    dims: [ 3, 224, 224 ]
    reshape { shape: [ 1, 3, 224, 224 ] }
 }
]
output [
 {
    name: "output__0"
    data_type: TYPE_FP32
    dims: [ 1, 1000 ,1, 1]
    reshape { shape: [ 1, 1000 ] }
 }
]

```


### Prepare model folder and structure 

The model repository must fulfill specific structures and names, as shown in the following. More detailed information for other backends can be found in the [official documentation](https://github.com/triton-inference-server/server/blob/main/docs/user_guide/model_repository.md). 


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






### Step.2: Set Up the Triton Inference Server

Open a new terminal tab in the notebook


```bash 
# Set model folder 
export YOUR_MODEL_FOLDER="{YOUR_MODEL_FOLDER}"

# Run the container with the Triton server 
apptainer run --nv --unsquash -B /proc:/proc -B /shared-storage:/mnt/shared-storage /images/tritonserver:24.09-py3

# Spin up a triton server
tritonserver --model-repository=${YOUR_MODEL_FOLDER}

```

You should see the following printout on the terminal. 

```bash
...
+----------+---------+--------+
| Model | Version | Status |
+----------+---------+--------+
| resnet50 | 1 | READY |
+----------+---------+--------+
...

```



### Setup Client 

Open another terminal to run the client script and send the inference request. 

We are kind of cheating here, given the ternail are on the same machine. In real life scenario, you will need the IP of the server to forward the request to a remote server and deal with authentication. 

```bash 
# Luanch the server docker images 
apptainer run --nv --unsquash -B /proc:/proc -B /shared-storage:/mnt/shared-storage /images/tritonserver-tutorial:24.08-py3

# Download the input images
wget  -O img1.jpg "https://www.hakaimagazine.com/wp-content/uploads/header-gulf-birds.jpg"

#Get the dummy client
cp /shared-storage/pytorch/client.py .

# Check if the connection is ok 
curl -v localhost:8000/v2/health/ready
```

You should see the following message if the connection and the server are in a good state.

```
...
< HTTP/1.1 200 OK
< Content-Length: 0
< Content-Type: text/plain
...
```

Now, we are ready to run the client script!


```bash
# Straightforward script to send an image to server
python client.py 
```

It will take some time, depending on the GPU utilization. But if everything goes well, you will be able to see the following printout.

```bash 

[b'12.474469:90' b'11.525709:92' b'9.660509:14' b'8.406358:136'
 b'8.220254:11']
```


The output is `<confidence_score>:<classification_index>`

Now, you get a triton client and server talking to each other!