# Traccc as-a-Service

Main objective: run [traccc](https://github.com/acts-project/traccc/tree/main) as-a-Service. Getting this working includes creating three main components:

1. a shared library of `traccc` and writing a standalone version with the essential pieces of the code included
2. a custom backend using the standalone version above to launch the Triton server
3. a client to send data to the server

A minimal description of how to build a working version is detailed below. For more in-depth descriptions of each step in the process, consult the [git repo](https://github.com/milescb/traccc-aaS) of this project. 

## Running out of the box

### Get the code

Simply clone the repository with 

```
git clone --recurse-submodules git@github.com:milescb/traccc-aaS.git
```

### Docker

A docker built for the triton server can be found at `docker.io/milescb/triton-server:latest`. To run this do

```
shifter --module=gpu --image=milescb/tritonserver:latest
```

or use your favorite docker application and mount the appropriate directories. 

```{note}
An image has been built with the custom backend pre-installed at `docker.io/milescb/traccc-aas:v1.1`. To run this, open the image, then run the server with

`tritonserver --model-repository=$MODEL_REPO`

To see how this was built, consult the `Dockerfile` in the [traccc-aaS repository](https://github.com/milescb/traccc-aaS). 
```

### Shared Library on NERSC

To run out of the box on [NERSC](https://www.nersc.gov), an installation of `traccc` and the backend can be found at `/global/cfs/projectdirs/m3443/data/traccc-aaS/software/prod/ver_09152024/install`. To set up the environment, run the docker then set the following environment variables

```
export DATADIR=/global/cfs/projectdirs/m3443/data/traccc-aaS/data
export INSTALLDIR=/global/cfs/projectdirs/m3443/data/traccc-aaS/software/prod/ver_09152024/install
export PATH=$INSTALLDIR/bin:$PATH
export LD_LIBRARY_PATH=$INSTALLDIR/lib:$LD_LIBRARY_PATH
```

Then the server can be launched with 

```
tritonserver --model-repository=$INSTALLDIR/models
```

```{note}

If you do not have access to NERSC computing resources, then you will need to build `traccc` from source. Follow the instructions in the [traccc repo](https://github.com/acts-project/traccc/tree/main) then update `INSTALLDIR` and `DATADIR` appropriately. 

To avoid having to build from source, deploy the server with the docker image `docker.io/milescb/tritonserver:latest` as directed above. 

```

## Building the backend

First, enter the docker and set environment variables as documented above. Then run

```
cd backend/traccc-gpu && mkdir build install && cd build
cmake -B . -S ../ \
    -DCMAKE_INSTALL_PREFIX=../install/

cmake --build . --target install -- -j20
```

Then, the server can be launched as above:

```
tritonserver --model-repository=../../models
```

## Run a simple client 

Once the server is launched, run the model via:

```
cd client && python TracccTritonClient.py 
```

This launches a simple `gRPC` client which makes one inference request to the deployed server. To test the scalability of the model and obtain performance metrics, the `perf_analyzer` tool can be used. For more information on how to obtain metrics and make plots, consult the [traccc-aaS-performance repository](https://github.com/milescb/traccc-aaS-performance).