# GNN4ITk as a Service

## The server
Last update: 2024-09-04.

Reference: [repo](https://github.com/xju2/tracking-as-a-service)

THe following are instructions of running a Trition server for GNN4ITk using Perlmutter at NERSC.

1. Clone the tracking as a service repo and download the model.
```bash
git clone https://github.com/xju2/tracking-as-a-service
cd tracking-as-a-service
python scripts/download_model.py
```

2. Request a GPU node and write down the server ID, like `nid00888`, which will the server URL for the client.
```bash
srun -C "gpu&hbm80g" -q interactive -N 1 -G 1 -c 32 -t 4:00:00 -A m3443 --pty /bin/bash -l
```

3. Launch the server.

```bash
./scripts/start-tritonserver.sh
```

## The client

Last update: 2024-09-04.

We developed a Trition tool in Athena to facilitate the use of Trition. 
The tool is called `TritonTool` [link to code](https://gitlab.cern.ch/xju/athena/-/blob/gnn_aas/Control/AthOnnx/AthTritonComps/src/TritonTool.h?ref_type=heads), 
which implements the inference interface, `IAthInferenceTool`. 

We created an example to show how to use the `TritonTool` in Athena. 
The example is located in `InnerDetector/InDetGNNTracking/src/GNNTrackFinderTritonTool.h` 
[link to code](https://gitlab.cern.ch/xju/athena/-/blob/gnn_aas/InnerDetector/InDetGNNTracking/src/GNNTrackFinderTritonTool.h?ref_type=heads). 
Following are the steps to compile the code, setup the environment, and run the example at **Perlmutter**. 
Please adjust the steps for other platforms.

Because the TritonClient is not yet installed in the Athena release 
([see the MR](https://gitlab.cern.ch/atlas/atlasexternals/-/merge_requests/1105)), 
we created a container that contains the TritonClient: `docexoty/alma9-atlasos:triton-client-grpc1p62p3`. 
For the same reason, The example is only available at the `gnn_aas` branch of my fork, [https://gitlab.cern.ch/xju/athena.git](https://gitlab.cern.ch/xju/athena.git).


1. Clone the code and Launch the container.
See the ATLAS [gittutorial](https://atlassoftwaredocs.web.cern.ch/gittutorial/gitlab-fork/) 
documentation on how to use `git`. If not limited to disk space, I recommend to use full checkout.

```bash
git clone -b gnn_aas https://gitlab.cern.ch/xju/athena.git

shifter --image=docexoty/alma9-atlasos:triton-client-grpc1p62p3 --module=cvmfs bash
ATHENA_PATH="path-to-Athena"
cd $ATHENA_PATH
source /global/cfs/cdirs/atlas/scripts/setupATLAS.sh
setupATLAS

asetup Athena,main,here,latest
which athena
```
The command `which athena` should output the path to the athena executable.

2. Set up the Triton dependency environment (this can be skipped once the Trition Client is installed.)

```bash
LCG_VERSION=$(echo $ROOTSYS | awk -F / '{print $10}')
LCG_ROOT=${LCG_RELEASE_BASE}/${LCG_VERSION}
LCG_PATH=/cvmfs/sft.cern.ch/lcg/releases/${LCG_VERSION}


grpc_Install_Dir=${LCG_PATH}/grpc/1.48.0/${LCG_PLATFORM}
c_ares_ROOT=${LCG_PATH}/c_ares/1.17.1/${LCG_PLATFORM}
absl_ROOT=${LCG_PATH}/absl/20230802.1/${LCG_PLATFORM}
re2_ROOT=${LCG_PATH}/re2/2023.11.01/${LCG_PLATFORM}
tritonclient_ROOT=/home/atlas/install/tritonclient

export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:${tritonclient_ROOT}/lib64:$grpc_Install_Dir/lib:$c_ares_ROOT/lib64:$absl_ROOT/lib64:$re2_ROOT/lib64
```

3. Compile the relevant packages. 
Create a file named `package_filters.txt` with the following content:
```
+ InnerDetector/InDetGNNTracking
+ Control/AthOnnx/AthOnnxInterfaces
+ Control/AthOnnx/AthTritonComps
+ Tracking/TrkConfig
- .*
```
Then compile the packages in a build directory `mkdir ../run_athena/build && cd ../run_athena/build`:
```bash
cmake -DATLAS_PACKAGE_FILTER_FILE=../../athena/package_filters.txt \
  -DCMAKE_PREFIX_PATH="${tritonclient_ROOT}/lib64/cmake;${grpc_Install_Dir}/lib/cmake;${c_ares_ROOT}/lib64/cmake;${absl_ROOT}/lib64/cmake;${re2_ROOT}/lib64/cmake" ../../athena/Projects/WorkDir

make -j20

source x86_64-el9-gcc13-opt/setup.sh
```

4. Run the example in the `run_athena/run` directory. `mkdir ../run && cd ../run`.
You may have to use rucio to download the RDO file.
```bash
RDO_FILENAME="inputData/RDO.37737772._000213.pool.root.1"

function clean_up() {
    rm InDetIdDict.xml PoolFileCatalog.xml hostnamelookup.tmp eventLoopHeartBeat.txt
}

function gnn4pixel() {
    clean_up
    export ATHENA_CORE_NUMBER=1
    if [ -z "$1" ]; then
        echo "Please provide the Triton server ID."
        return
    fi
    TritionServer=$1

    Reco_tf.py \
        --CA 'all:True' --autoConfiguration 'everything' \
        --conditionsTag 'all:OFLCOND-MC15c-SDR-14-05' \
        --geometryVersion 'all:ATLAS-P2-RUN4-03-00-00' \
        --multithreaded 'False' \
        --steering 'doRAWtoALL' \
        --digiSteeringConf 'StandardInTimeOnlyTruth' \
        --postInclude 'all:PyJobTransforms.UseFrontier' \
        --preInclude 'all:Campaigns.PhaseIIPileUp200' 'InDetConfig.ConfigurationHelpers.OnlyTrackingPreInclude' 'InDetGNNTracking.InDetGNNTrackingFlags.gnnTritonValidation' \
        --preExec 'flags.Tracking.GNN.usePixelHitsOnly = True; flags.Tracking.ITkGNNPass.doAmbiguityResolutionForGNN = False; flags.Tracking.GNN.Triton.model = "GNN4Pixel"' "flags.Tracking.GNN.Triton.url = \"${TritionServer}\"" \
        --inputRDOFile "${RDO_FILENAME}" \
        --outputAODFile 'test.aod.gnnTriton.root'  \
        --jobNumber '1' \
		--athenaopts='--loglevel=INFO' \
        --maxEvents -1 2>&1 | tee log.gnnTrition.txt
}

gnn4pixel nid00888
```

5. To evaluate the performance, we can run the `IDPVM` package [see the link for more info](https://gitlab.cern.ch/atlas/athena/-/tree/main/InnerDetector/InDetValidation/InDetPhysValMonitoring?ref_type=heads).
```bash
runIDPVM.py --filesInput test.aod.gnnreader.debug.root --outputFile physval.root --doTightPrimary 
```
The output file `physval.root` contains the performance of the GNN tracking.


