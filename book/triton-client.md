# How to write a Triton Client

## Standalone Python script

## In Athena
Last update: 2021-07-01.

We developed a Trition tool in Athena to facilitate the use of Trition. The tool is called `TritonTool` [link to code](https://gitlab.cern.ch/xju/athena/-/blob/triton_client/Control/AthOnnx/AthTritonComps/src/TritonTool.h?ref_type=heads), which implements the inference interface, `IAthInferenceTool`. 

Because the TritonClient is not yet installed in the Athena release ([see the MR](https://gitlab.cern.ch/atlas/atlasexternals/-/merge_requests/1105)), we created a container that contains the TritonClient: `docexoty/alma9-atlasos:with-triton-client`. For the same reason, The example is only available at the `triton_client` branch of my fork, [https://gitlab.cern.ch/xju/athena.git](https://gitlab.cern.ch/xju/athena.git).

We created an example to show how to use the `TritonTool` in Athena. The example is located in `InnerDetector/InDetGNNTracking/src/GNNTrackFinderTritonTool.h` [link to code](https://gitlab.cern.ch/xju/athena/-/blob/triton_client/InnerDetector/InDetGNNTracking/src/GNNTrackFinderTritonTool.h?ref_type=heads). Following are the steps to compile the code, setup the environment, and run the example at **Perlmutter**. Please adjust the steps for other platforms.
