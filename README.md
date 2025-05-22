# ollama-intel-npu

This repo illustrates the use of Intel(R) Core(TM) Ultra NPU against Ollama through OpenVINO GenAI backend with Docker. This is a compilation of several projects.  I'm no AI expert, just a hobbiest that wanted to make some use of the NPU to see if it performed any better than the GPU. I would love for someone to take this and run with it.

# Results
Intel(R) Core(TM) Ultra 7 265K

|Metric             |GPU                    |NPU                    |
|-------------------|:---------------------:|:---------------------:|
|Load time          | 7034.00               |29956.00               |
|Generate time      | 166356.30 ± 0.00 ms   |88097.87 ± 0.00 ms     |
|Tokenization time  | 0.64 ± 0.00 ms        |1.14 ± 0.00 ms         |
|Detokenization time| 0.10 ± 0.00 ms        |0.41 ± 0.00 ms         |
|TTFT               | 332.98 ± 0.00 ms      |6897.71 ± 0.00 ms      |
|TPOT               | 81.12 ± 2.70 ms/token |145.78 ± 0.54 ms/token |
|Throughput         | 12.33 ± 0.41 tokens/s |6.86 ± 0.03 tokens/s   |


# References
|   |   |
|---|---|
| ollama_ov | https://github.com/zhaohb/ollama_ov |
| openvinotoolkit | https://github.com/openvinotoolkit/docker_ci/tree/master/dockerfiles |
| Intel NPU Drivers | https://github.com/intel/linux-npu-driver/releases |
| OpenVINO Blog | https://blog.openvino.ai/blog-posts/ollama-integrated-with-openvino-accelerating-deepseek-inference|

# Prerequisites
In addition to requiring the drivers in the container itself, you'll need to ensure you have the device drivers installed on your host system. To make things easier, I matched the versions of Ubuntu and the drivers between my host systems and container. Not sure if this is required but it was my best chance at success.

## Option 1
Follow the instructions for the Intel DL Streamer prerequisites here. https://dlstreamer.github.io/get_started/install/install_guide_ubuntu.html#id1

## Option 2
Install manually using the instructions from the Intel's GitHub.
|   |   |
|---|---|
| NPU Drivers | https://github.com/intel/linux-npu-driver/releases |
| GPU Drivers | https://github.com/intel/compute-runtime/releases |

# Usage

## Build
First we want to clone and build the container. You may want to update some of these packages as they're not dynamically pulled.
```bash
$ git clone https://github.com/dgshue/ollama-intel-npu
$ cd ollama-intel-npu
$ docker build -t ollama-intel-npu:latest -f Dockerfile_intel_npu .
```
## Run
You need to create a folder for downloading models. Long term I would recommend mounting to the filesystem rather than a volume.
```bash
$ mkdir /home/user/somepath
$ docker run -it --rm \
    --device=/dev/dri \
    --device=/dev/accel/accel0 \
    -v /home/user/somepath:/root/.ollama \
    ollama-intel-npu:latest
```
You should see GenAI find 3 inference devices.

```
level=INFO source=routes.go:1478 msg="GenAI inference device" Device=CPU "Build Number"=2025.1.0-18503-6fec06580ab-releases/2025/1 description=openvino_intel_cpu_plugin
level=INFO source=routes.go:1478 msg="GenAI inference device" Device=GPU "Build Number"=2025.1.0-18503-6fec06580ab-releases/2025/1 description="Intel GPU plugin"
level=INFO source=routes.go:1478 msg="GenAI inference device" Device=NPU "Build Number"=2025.1.0-18503-6fec06580ab-releases/2025/1 description=openvino_intel_npu_plugin
```

## Compose with OpenWebUI
Additionally, you can stand up the newly built ollama container and OpenWebUI with the provided Docker compose or you can define your own deployment.
```bash
$ docker compose up
```
From here, you can access OpenWebUI via port http://*your_ip_address*:3000

## Import from openVINO IR
How to create an Ollama model based on Openvino IR. modelscope and huggingface-cli should already be installed.

### Example
Let's take [deepseek-ai/DeepSeek-R1-Distill-Qwen-7B](https://hf-mirror.com/deepseek-ai/DeepSeek-R1-Distill-Qwen-7B) as an example.

1. Shell into the container
    You can get your container ID with 'docker ps'
    ```bash
    $ docker exec -it CONTAINER_ID sh
    ```

2. Download the OpenVINO model 
   1. Download from ModelScope: [DeepSeek-R1-Distill-Qwen-7B-int4-ov](https://modelscope.cn/models/zhaohb/DeepSeek-R1-Distill-Qwen-7B-int4-ov)
      ```shell
      cd /root/.ollama
      modelscope download --model zhaohb/DeepSeek-R1-Distill-Qwen-7B-int4-ov --local_dir ./DeepSeek-R1-Distill-Qwen-7B-int4-ov
      ```
   
   2. If the OpenVINO model exists in HF, we can also download it from HF. Here we take [TinyLlama-1.1B-Chat-v1.0-int4-ov](https://huggingface.co/OpenVINO/TinyLlama-1.1B-Chat-v1.0-int4-ov) as an example to introduce how to download the model from HF.

      If your network access to HuggingFace is unstable, you can try to use a proxy image to pull the model.
      ```shell
      set HF_ENDPOINT=https://hf-mirror.com
      ```
      ```
      cd /root/.ollama
      huggingface-cli download --resume-download OpenVINO/TinyLlama-1.1B-Chat-v1.0-int4-ov  --local-dir  TinyLlama-1.1B-Chat-v1.0-int4-ov --local-dir-use-symlinks False
      ```


3. Package OpenVINO IR into a tar.gz file
    ```bash
    tar -zcvf DeepSeek-R1-Distill-Qwen-7B-int4-ov.tar.gz DeepSeek-R1-Distill-Qwen-7B-int4-ov
    ```

4. Create a file named `Modelfile`, with a `FROM` instruction with the local filepath to the model you want to import.
   For convenience, we have put the model file of DeepSeek-R1-Distill-Qwen-7B-int4-ov model under example dir: [Modelfile for DeepSeek](https://github.com/openvinotoolkit/openvino_contrib/blob/master/modules/ollama_openvino/examples/modelfile/deepseek_r1_distill_qwen/Modelfile), we can use it directly.

    ```
    FROM  DeepSeek-R1-Distill-Qwen-7B-int4-ov.tar.gz
    ModelType "OpenVINO"
    InferDevice "NPU"
    PARAMETER stop ""
    PARAMETER stop "```"
    PARAMETER stop "</User|>"
    PARAMETER stop "<|end_of_sentence|>"
    PARAMETER stop "</｜"
    PARAMETER max_new_token 4096
    PARAMETER stop_id 151643
    PARAMETER stop_id 151647
    PARAMETER repeat_penalty 1.5
    PARAMETER top_p 0.95
    PARAMETER top_k 50
    PARAMETER temperature 0.8
    ```

   Note:

   1. The `ModelType "OpenVINO"` parameter is mandatory and must be explicitly set.
   2. The `InferDevice` parameter is optional. However if you want to use the NPU you'll need to set the modelfile to 'NPU' If not specified, the system will prioritize using the GPU by default. If no GPU is available, it will automatically fall back to using the CPU. If InferDevice is explicitly set, the system will strictly use the specified device. If the specified device is unavailable, the system will follow the same fallback strategy as when InferDevice is not set (i.e., GPU first, then CPU).
   3. For more information on working with a Modelfile, see the [Modelfile](./docs/modelfile.md) documentation.

5. Create the model in Ollama

   ```shell
   ollama create DeepSeek-R1-Distill-Qwen-7B-int4-ov:v1 -f Modelfile
   ```
   You will see output similar to the following:
   ```shell
      gathering model components 
      copying file sha256:77acf6474e4cafb67e57fe899264e9ca39a215ad7bb8f5e6b877dcfa0fabf919 100% 
      using existing layer sha256:77acf6474e4cafb67e57fe899264e9ca39a215ad7bb8f5e6b877dcfa0fabf919 
      creating new layer sha256:9b345e4ef9f87ebc77c918a4a0cee4a83e8ea78049c0ee2dc1ddd2a337cf7179 
      creating new layer sha256:ea49523d744c40bc900b4462c43132d1c8a8de5216fa8436cc0e8b3e89dddbe3 
      creating new layer sha256:b6bf5bcca7a15f0a06e22dcf5f41c6c0925caaab85ec837067ea98b843bf917a 
      writing manifest 
      success 
   ```

6. Run the model                     

   ```shell
   ollama run DeepSeek-R1-Distill-Qwen-7B-int4-ov:v1
   ```

## CLI Reference

### Show model information

```shell
ollama show DeepSeek-R1-Distill-Qwen-7B-int4-ov:v1 
```

### List models on your computer

```shell
ollama list
```

### List which models are currently loaded

```shell
ollama ps
```

### Stop a model which is currently running

```shell
ollama stop DeepSeek-R1-Distill-Qwen-7B-int4-ov:v1 
```



