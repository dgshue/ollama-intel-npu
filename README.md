# ollama-intel-npu

This repo illustrates the use of Intel(R) Core(TM) Ultra NPU against Ollama through OpenVINO GenAI backend with Docker. This is a compilation of several projects.

# References
| ollama_ov | https://github.com/zhaohb/ollama_ov |
| openvinotoolkit | https://github.com/openvinotoolkit/docker_ci/tree/master/dockerfiles |
| Intel NPU Drivers | https://github.com/intel/linux-npu-driver/releases |

# Prerequisites
In addition to requiring the drivers in the container itself, you'll need to ensure you have the device drivers installed on your host system. To make things easier, I matched the versions of Ubuntu and the drivers between my host systems and container. Not sure if this is required but it was my best chance at success.

## Option 1
Follow the instructions for the Intel DL Streamer prerequisites here. https://dlstreamer.github.io/get_started/install/install_guide_ubuntu.html#id1

## Option 2
Install manually using the instructions from the Intel's GitHub. 
| NPU Drivers | https://github.com/intel/linux-npu-driver/releases |
| GPU Drivers | https://github.com/intel/compute-runtime/releases |

# Usage

```bash
$ git clone https://github.com/dgshue/ollama-intel-npu
$ cd ollama-intel-npu
$ docker build -t ollama-intel-npu:latest -f Dockerfile_intel_npu .
```

```bash
$ docker compose up
```