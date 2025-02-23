## Build/Use IPEX-LLM-serving xpu image

### Build Image
```bash
docker build \
  --build-arg http_proxy=.. \
  --build-arg https_proxy=.. \
  --build-arg no_proxy=.. \
  --rm --no-cache -t intelanalytics/ipex-llm-serving-xpu:2.2.0-SNAPSHOT .
```


### Use the image for doing xpu serving


To map the `xpu` into the container, you need to specify `--device=/dev/dri` when booting the container.

An example could be:
```bash
#/bin/bash
export DOCKER_IMAGE=intelanalytics/ipex-llm-serving-xpu:2.2.0-SNAPSHOT

sudo docker run -itd \
        --net=host \
        --device=/dev/dri \
        --name=CONTAINER_NAME \
        --shm-size="16g" \
        $DOCKER_IMAGE
```


After the container is booted, you could get into the container through `docker exec`.

To verify the device is successfully mapped into the container, run `sycl-ls` to check the result. In a machine with Arc A770, the sampled output is:

```bash
root@arda-arc12:/# sycl-ls
[opencl:acc:0] Intel(R) FPGA Emulation Platform for OpenCL(TM), Intel(R) FPGA Emulation Device 1.2 [2023.16.7.0.21_160000]
[opencl:cpu:1] Intel(R) OpenCL, 13th Gen Intel(R) Core(TM) i9-13900K 3.0 [2023.16.7.0.21_160000]
[opencl:gpu:2] Intel(R) OpenCL Graphics, Intel(R) Arc(TM) A770 Graphics 3.0 [23.17.26241.33]
[ext_oneapi_level_zero:gpu:0] Intel(R) Level-Zero, Intel(R) Arc(TM) A770 Graphics 1.3 [1.3.26241]
```
After the container is booted, you could get into the container through `docker exec`.

Currently, we provide two different serving engines in the image, which are FastChat serving engine and vLLM serving engine.


#### Lightweight serving engine

To run Lightweight serving on one intel gpu using `IPEX-LLM` as backend, you can refer to this [readme](https://github.com/intel-analytics/ipex-llm/tree/main/python/llm/example/GPU/Lightweight-Serving).

For convenience, we have included a file `/llm/start-lightweight_serving-service` in the image.


#### Pipeline parallel serving engine

To run Pipeline parallel serving using `IPEX-LLM` as backend, you can refer to this [readme](https://github.com/intel-analytics/ipex-llm/tree/main/python/llm/example/GPU/Pipeline-Parallel-FastAPI).

For convenience, we have included a file `/llm/start-pp_serving-service.sh` in the image.


#### FastChat serving engine

To set up model serving using `IPEX-LLM` as backend using FastChat, you can refer to this [quickstart](https://ipex-llm.readthedocs.io/en/latest/doc/LLM/Quickstart/fastchat_quickstart.html#) or follow these quick steps to deploy a demo.

##### Quick Setup for FastChat with IPEX-LLM

1. **Start the Docker Container** 
   
    Run the following command to launch a Docker container with device access:

    ```bash
    #/bin/bash
    export DOCKER_IMAGE=intelanalytics/ipex-llm-serving-xpu:latest

    sudo docker run -itd \
            --net=host \
            --device=/dev/dri \
            --name=demo-container \
            # Example: map host model directory to container
            -v /LLM_MODELS/:/llm/models/ \  
            --shm-size="16g" \
            # Optional: set proxy if needed
            -e http_proxy=... \ 
            -e https_proxy=... \
            -e no_proxy="127.0.0.1,localhost" \
            $DOCKER_IMAGE
    ```

2. **Start the FastChat Service**
   
    Enter the container and start the FastChat service:
    ```bash
    #/bin/bash

    # This command assumes that you have mapped the host model directory to the container
    # and the model directory is /llm/models/
    # we take Yi-1.5-34B as an example, and you can replace it with your own model

    ps -ef | grep "fastchat" | awk '{print $2}' | xargs kill -9
    pip install -U gradio==4.43.0
    
    # start controller
    python -m fastchat.serve.controller &

    export USE_XETLA=OFF
    export SYCL_PI_LEVEL_ZERO_USE_IMMEDIATE_COMMANDLISTS=2
    
    export TORCH_LLM_ALLREDUCE=0
    export CCL_DG2_ALLREDUCE=1
    # CCL needed environment variables
    export CCL_WORKER_COUNT=4
    # pin ccl worker to cores
    # export CCL_WORKER_AFFINITY=32,33,34,35
    export FI_PROVIDER=shm
    export CCL_ATL_TRANSPORT=ofi
    export CCL_ZE_IPC_EXCHANGE=sockets
    export CCL_ATL_SHM=1
    
    source /opt/intel/1ccl-wks/setvars.sh
    
    python -m ipex_llm.serving.fastchat.vllm_worker \
    --model-path /llm/models/Yi-1.5-34B \
    --device xpu \
    --enforce-eager \
    --disable-async-output-proc \
    --distributed-executor-backend ray \
    --dtype float16 \
    --load-in-low-bit fp8 \
    --tensor-parallel-size 4 \
    --gpu-memory-utilization 0.9 \
    --max-model-len 4096 \
    --max-num-batched-tokens 8000 &
    
    sleep 120
    
    python -m fastchat.serve.gradio_web_server &
    ```

This quick setup allows you to deploy FastChat with IPEX-LLM efficiently.

#### vLLM serving engine

To run vLLM engine using `IPEX-LLM` as backend, you can refer to this [document](https://github.com/intel-analytics/ipex-llm/blob/main/docs/mddocs/DockerGuides/vllm_docker_quickstart.md).

We have included multiple example files in `/llm/`:
1. `vllm_offline_inference.py`: Used for vLLM offline inference example
2. `benchmark_vllm_throughput.py`: Used for benchmarking throughput
3. `payload-1024.lua`: Used for testing request per second using 1k-128 request
4. `start-vllm-service.sh`: Used for template for starting vLLM service
5. `vllm_offline_inference_vision_language.py`: Used for vLLM offline inference vision example

##### Online benchmark throurgh api_server

We can benchmark the api_server to get an estimation about TPS (transactions per second).  To do so, you need to start the service first according to the instructions in this [section](https://github.com/intel-analytics/ipex-llm/blob/main/docs/mddocs/DockerGuides/vllm_docker_quickstart.md#Serving).

###### Online benchmark through benchmark_util

After starting vllm service, Sending reqs through `vllm_online_benchmark.py`
```bash
python vllm_online_benchmark.py $model_name $max_seqs $input_length $output_length
```
If `input_length` and `output_length` are not provided, the script will use the default values of 1024 and 512, respectively.

And it will output like this:
```bash
model_name: Qwen1.5-14B-Chat
max_seq: 12
Warm Up: 100%|█████████████████████████████████████████████████████| 24/24 [01:36<00:00,  4.03s/req]
Benchmarking: 100%|████████████████████████████████████████████████| 60/60 [04:03<00:00,  4.05s/req]
Total time for 60 requests with 12 concurrent requests: xxx seconds.
Average responce time: xxx
Token throughput: xxx

Average first token latency: xxx milliseconds.
P90 first token latency: xxx milliseconds.
P95 first token latency: xxx milliseconds.

Average next token latency: xxx milliseconds.
P90 next token latency: xxx milliseconds.
P95 next token latency: xxx milliseconds.
```

###### Online benchmark with multimodal through benchmark_util


After starting vllm service, Sending reqs through `vllm_online_benchmark_multimodal.py`
```bash
export image_url="http://farm6.staticflickr.com/5268/5602445367_3504763978_z.jpg"
python vllm_online_benchmark_multimodal.py --model-name $model_name --image-url $image_url --prompt "What is in the image?" --port 8000
```

`image_url` can be `/llm/xxx.jpg` or `"http://xxx.jpg`.

And it will output like this:
```bash
model_name: MiniCPM-V-2_6
Warm Up: 100%|███████████████████████████████████████████████████████| 2/2 [00:03<00:00,  1.68s/req]
Warm Up: 100%|███████████████████████████████████████████████████████| 1/1 [00:10<00:00, 10.42s/req]
Benchmarking: 100%|██████████████████████████████████████████████████| 3/3 [00:31<00:00, 10.43s/req]
Total time for 3 requests with 1 concurrent requests: xxx seconds.
Average responce time: xxx
Token throughput: xxx

Average first token latency: xxx milliseconds.
P90 first token latency: xxx milliseconds.
P95 first token latency: xxx milliseconds.

Average next token latency: xxx milliseconds.
P90 next token latency: xxx milliseconds.
P95 next token latency: xxx milliseconds.
```

###### Online benchmark through wrk
In container, do the following:
1. modify the `/llm/payload-1024.lua` so that the "model" attribute is correct.  By default, we use a prompt that is roughly 1024 token long, you can change it if needed.
2. Start the benchmark using `wrk` using the script below:

```bash
cd /llm
# You can change -t and -c to control the concurrency.
# By default, we use 12 connections to benchmark the service.
wrk -t12 -c12 -d15m -s payload-1024.lua http://localhost:8000/v1/completions --timeout 1h
```

#### Offline benchmark through benchmark_vllm_throughput.py

We have included the benchmark_throughput script provied by `vllm` in our image as `/llm/benchmark_vllm_throughput.py`.  To use the benchmark_throughput script, you will need to download the test dataset through:

```bash
wget https://huggingface.co/datasets/anon8231489123/ShareGPT_Vicuna_unfiltered/resolve/main/ShareGPT_V3_unfiltered_cleaned_split.json
```

The full example looks like this:
```bash
cd /llm/

wget https://huggingface.co/datasets/anon8231489123/ShareGPT_Vicuna_unfiltered/resolve/main/ShareGPT_V3_unfiltered_cleaned_split.json

export MODEL="YOUR_MODEL"

# You can change load-in-low-bit from values in [sym_int4, fp8, fp16]

python3 /llm/benchmark_vllm_throughput.py \
    --backend vllm \
    --dataset /llm/ShareGPT_V3_unfiltered_cleaned_split.json \
    --model $MODEL \
    --num-prompts 1000 \
    --seed 42 \
    --trust-remote-code \
    --enforce-eager \
    --dtype float16 \
    --device xpu \
    --load-in-low-bit sym_int4 \
    --gpu-memory-utilization 0.85
```

> Note: you can adjust --load-in-low-bit to use other formats of low-bit quantization.


You can also adjust `--gpu-memory-utilization` rate using the below script to find the best performance using the following script:

```bash
#!/bin/bash

# Define the log directory
LOG_DIR="YOUR_LOG_DIR"
# Check if the log directory exists, if not, create it
if [ ! -d "$LOG_DIR" ]; then
    mkdir -p "$LOG_DIR"
fi

# Define an array of model paths
MODELS=(
    "YOUR TESTED MODELS"
)

# Define an array of utilization rates
UTIL_RATES=(0.85 0.90 0.95)

# Loop over each model
for MODEL in "${MODELS[@]}"; do
    # Loop over each utilization rate
    for RATE in "${UTIL_RATES[@]}"; do
        # Extract a simple model name from the path for easier identification
        MODEL_NAME=$(basename "$MODEL")

        # Define the log file name based on the model and rate
        LOG_FILE="$LOG_DIR/${MODEL_NAME}_utilization_${RATE}.log"

        # Execute the command and redirect output to the log file
        # Sometimes you might need to set --max-model-len if memory is not enough
        # load-in-low-bit accepts inputs [sym_int4, fp8, fp16]
        python3 /llm/benchmark_vllm_throughput.py \
            --backend vllm \
            --dataset /llm/ShareGPT_V3_unfiltered_cleaned_split.json \
            --model $MODEL \
            --num-prompts 1000 \
            --seed 42 \
            --trust-remote-code \
            --enforce-eager \
            --dtype float16 \
            --load-in-low-bit sym_int4 \
            --device xpu \
            --gpu-memory-utilization $RATE &> "$LOG_FILE"
    done
done

# Inform the user that the script has completed its execution
echo "All benchmarks have been executed and logged."
```
