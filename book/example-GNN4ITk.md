# GNN4ITk as a Service

## The server
Reference:  [repo](https://github.com/hrzhao76/GNN4ITk-aaS)

## The client

Last update: 2021-07-01.

We developed a Trition tool in Athena to facilitate the use of Trition. The tool is called `TritonTool` [link to code](https://gitlab.cern.ch/xju/athena/-/blob/triton_client/Control/AthOnnx/AthTritonComps/src/TritonTool.h?ref_type=heads), which implements the inference interface, `IAthInferenceTool`. 

We created an example to show how to use the `TritonTool` in Athena. The example is located in `InnerDetector/InDetGNNTracking/src/GNNTrackFinderTritonTool.h` [link to code](https://gitlab.cern.ch/xju/athena/-/blob/triton_client/InnerDetector/InDetGNNTracking/src/GNNTrackFinderTritonTool.h?ref_type=heads). Following are the steps to compile the code, setup the envirionemtn, and run the example at **Perlmutter**. Please adjust the steps for other platforms.

Because the TritonClient is not yet installed in the Athena release ([see the MR](https://gitlab.cern.ch/atlas/atlasexternals/-/merge_requests/1105)), we created a container that contains the TritonClient: `docexoty/alma9-atlasos:with-triton-client`. For the same reason, The example is only available at the `triton_client` branch of my fork, [https://gitlab.cern.ch/xju/athena.git](https://gitlab.cern.ch/xju/athena.git).


1. Clone the code and Launch the container.
See ATLAS [gittutorial](https://atlassoftwaredocs.web.cern.ch/gittutorial/gitlab-fork/) documentation for how to use `git`. If not limited to disk space, I recomment to use full checkout.

```bash
git clone -b triton_client https://gitlab.cern.ch/xju/athena.git

shifter --image=docexoty/alma9-atlasos:with-triton-client --module=cvmfs bash
ATHENA_PATH="path-to-athena"
cd $ATHENA_PATH
source /global/cfs/cdirs/atlas/scripts/setupATLAS.sh
setupATLAS

asetup Athena,main,here,latest
which athena
```
The command `which athena` should output the path to the athena executable.

2. Setup the Triton dependency environment (this can be skipped once the Trition Client is installed.)

```bash
LCG_VERSION=$(echo $ROOTSYS | awk -F / '{print $10}')
LCG_ROOT=${LCG_RELEASE_BASE}/${LCG_VERSION}
LCG_PATH=/cvmfs/sft.cern.ch/lcg/releases/${LCG_VERSION}


grpc_Install_Dir=${LCG_PATH}/grpc/1.48.0/${LCG_PLATFORM}
c_ares_ROOT=${LCG_PATH}/c_ares/1.17.1/${LCG_PLATFORM}
absl_ROOT=${LCG_PATH}/absl/20230802.1/${LCG_PLATFORM}
re2_ROOT=${LCG_PATH}/re2/2023.11.01/${LCG_PLATFORM}

export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/home/atlas/install/lib64:$grpc_Install_Dir/lib:$c_ares_ROOT/lib64:$absl_ROOT/lib64:$re2_ROOT/lib64
```

3. Compile the relevant packages. 
Create a file named as `package_filters.txt` with the following content:
```
+ InnerDetector/InDetGNNTracking
+ Control/AthOnnx/AthOnnxInterfaces
+ Control/AthOnnx/AthTritonComps
+ Tracking/TrkConfig
- .*
```
Then compile the packages in a build directory `mkdir ../run_athena/build && cd ../run_athena/build`:
```bash
cmake -DATLAS_PACKAGE_FILTER_FILE=../../athena/package_filters.txt -DCMAKE_PREFIX_PATH="/home/atlas/install/lib64/cmake;${grpc_Install_Dir}/lib/cmake;${c_ares_ROOT}/lib64/cmake;${absl_ROOT}/lib64/cmake;${re2_ROOT}/lib64/cmake" ../../athena/Projects/WorkDir

make -j20

source x86_64-el9-gcc13-opt/setup.sh
```

```{note}
The server URL is hard-coded in the configuration [`InDetGNNTrackingConfig.py`](https://gitlab.cern.ch/xju/athena/-/blob/triton_client/InnerDetector/InDetGNNTracking/python/InDetGNNTrackingConfig.py#L66-77). Please change the URL to the server you are using!
```

4. Run the example in the `run_athena/run` directory. `mkdir ../run && cd ../run`.
You may have to use rucio to download the RDO file.
```bash
RDO_FILENAME="inputData/RDO.37737772._000213.pool.root.1"

function gnn_tracking() {
    rm InDetIdDict.xml PoolFileCatalog.xml hostnamelookup.tmp eventLoopHeartBeat.txt
    export ATHENA_CORE_NUMBER=1

    Reco_tf.py \
        --CA 'all:True' --autoConfiguration 'everything' \
        --conditionsTag 'all:OFLCOND-MC15c-SDR-14-05' \
        --geometryVersion 'all:ATLAS-P2-RUN4-03-00-00' \
        --perfmon 'fullmonmt' \
        --multithreaded 'True' \
        --steering 'doRAWtoALL' \
        --digiSteeringConf 'StandardInTimeOnlyTruth' \
        --postInclude 'all:PyJobTransforms.UseFrontier' \
        --preInclude 'all:Campaigns.PhaseIIPileUp200' 'InDetConfig.ConfigurationHelpers.OnlyTrackingPreInclude' 'InDetGNNTracking.InDetGNNTrackingFlags.gnnTritonValidation' \
        --preExec 'flags.Tracking.GNN.usePixelHitsOnly = True' \
        --postExec 'all:cfg.getService("AlgResourcePool").CountAlgorithmInstanceMisses = True' \
        --inputRDOFile "${RDO_FILENAME}" \
        --outputAODFile 'test.aod.gnnreader.debug.root'  \
        --jobNumber '1' \
        --maxEvents 5 2>&1 | tee log.gnnreader_debug.txt
}

gnn_tracking
```

5. To evaluate the performance, we can run the `IDPVM` package [see the link for more info](https://gitlab.cern.ch/atlas/athena/-/tree/main/InnerDetector/InDetValidation/InDetPhysValMonitoring?ref_type=heads).
```bash
runIDPVM.py --filesInput test.aod.gnnreader.debug.root --outputFile physval.root --doTightPrimary 
```
The output file `physval.root` contains the performance of the GNN tracking.
