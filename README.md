# OV_SD_CPP
C++ pipeline with OpenVINO native API for Stable Diffusion v1.5 with LMS Discrete Scheduler

## Step 1: Prepare environment

C++ Packages:
* OpenVINO: intall with `conda install -c conda-forge openvino=2023.1.0`
* Boost: Install with `sudo apt-get install libboost-all-dev` for LMSDiscreteScheduler's integration

Notice: 

SD Preparation in two steps above could be auto implemented with build_dependencies.sh.

This script provides 2 ways to install OpenVINO 2023.1.0: [conda-forge](https://anaconda.org/conda-forge/openvino) and [Download archives](https://storage.openvinotoolkit.org/repositories/openvino/packages/2023.1/windows/). 

Use Intel sample [writeOutputBmp function](https://github.com/openvinotoolkit/openvino/blob/539b5a83ba7fcbbd348e4dc308e4a0f2dee8343c/samples/cpp/common/utils/include/samples/common.hpp#L155) instead of OpenCV for image saving.
```shell
cd scripts
chmod +x build_dependencies.sh
./build_dependencies.sh
```

## Step 2: Prepare SD model and Tokenizer model
* SD v1.5 model:

Refer [this link](https://github.com/intel-innersource/frameworks.ai.openvino.llm.bench/blob/main/public/convert.py#L124-L184) to generate SD v1.5 model, reshape to static model (1,3,512,512) for best performance or dynamic model. 

With downloaded models, the model conversion from PyTorch model to OpenVINO IR could be done with script convert_model.py in the scripts directory. Please convert the model into `FP16_static` or `FP16_dyn` 

```shell
python -m convert_model.py -b 1 -t <INT8|FP16|FP32> -sd Path_to_your_SD_model
```
Notice: Now the pipeline support batch size = 1 only

* Lora enabling with safetensors

Refer [this blog for python pipeline](https://blog.openvino.ai/blog-posts/enable-lora-weights-with-stable-diffusion-controlnet-pipeline), here the C++ implementation includes: 
the safetensor model is loaded with file [src/safetensors.h](https://github.com/hsnyder/safetensors.h), layer name and weight are modified with `Eigen Lib` and inserted into the SD model with `ov::pass::MatcherPass` in the file `src/lora_cpp.hpp` 

SD model [dreamlike-anime-1.0](https://huggingface.co/dreamlike-art/dreamlike-anime-1.0) and Lora [soulcard](https://civitai.com/models/67927?modelVersionId=72591) are tested in this pipeline

* Tokenizer model:
  
1. The script convert_sd_tokenizer.py in the scripts dir could serialize the tokenizer model IR

2. Build OV extension:
 
```git clone https://github.com/apaniukov/openvino_contrib/  -b tokenizer-fix-decode```

Refer to PR OpenVINO [custom extension](https://github.com/openvinotoolkit/openvino_contrib/pull/687) ( new feature still in experiments )

3. Read model with extension in the SD pipeline 

Notice:

Tokenizer Model IR and built extension file are provided in this repo

![image](https://github.com/yangsu2022/OV_SD_CPP/assets/102195992/bac14f96-69c9-4ec4-b694-21a62a3176f4)

## Step 3: Build Pipeline

```shell
conda activate SD-CPP
mkdir build && cd build
cmake -DCMAKE_BUILD_TYPE=Release ..
make
```
Notice:

Use the same OpenVINO environment as the tokenizer extension

## Step 4: Run Pipeline
```shell
 ./SD-generate -t <text> -n <negPrompt> -s <seed> --height <output image> --width <output image> -d <debugLogger> -e <useOVExtension> -r <readNPLatent> -m <modelPath> -p <precision> -l <lora.safetensors> -a <alpha> -h <help>
```

Usage:
  OV_SD_CPP [OPTION...]

* `-p, --posPrompt arg` Initial positive prompt for SD  (default: cyberpunk cityscape like Tokyo New York  with tall buildings at dusk golden hour cinematic lighting)
* `-n, --negPrompt arg` Default is empty with space (default: )
* `-d, --device arg`    AUTO, CPU, or GPU (default: CPU)
* `--step arg`          Number of diffusion step ( default: 20)
* `-s, --seed arg`      Number of random seed to generate latent (default: 42)
* `--num arg`           Number of image output(default: 1)
* `--height arg`        Height of output image (default: 512)
* `--width arg`         Width of output image (default: 512)
* `--log arg`           Generate logging into log.txt for debug
* `-c, --useCache`      Use model caching
* `-e, --useOVExtension`Use OpenVINO extension for tokenizer
* `-r, --readNPLatent`  Read numpy generated latents from file
* `-m, --modelPath arg` Specify path of SD model IR (default: ../models/dreamlike-anime-1.0)
* `-t, --type arg`      Specify the type of SD model IR (FP16_static or FP16_dyn) (default: FP16_static)
* `-l, --loraPath arg`  Specify path of lora file. (*.safetensors). (default: ../models/soulcard.safetensors)
* `-a, --alpha arg`     alpha for lora (default: 0.75)
* `-h, --help`          Print usage

Example:

Positive prompt: cyberpunk cityscape like Tokyo New York  with tall buildings at dusk golden hour cinematic lighting

Negative prompt: (empty, here couldn't use OV tokenizer, check the issues for details)  

Read the numpy latent instead of C++ std lib for the alignment with Python pipeline 

* Generate image without lora ` ./SD-generate -r -l "" `

![image](https://github.com/intel-sandbox/OV_SD_CPP/assets/102195992/66047d66-08a3-4272-abdc-7999d752eea0)

* Generate image with soulcard lora ` ./SD-generate -r `

![image](https://github.com/intel-sandbox/OV_SD_CPP/assets/102195992/0f6e2e3e-74fe-4bd4-bb86-df17cb4bf3f8)

* Generate the debug logging into log.txt: ` ./SD-generate --log`
* Generate different size image with dynamic model(C++ lib generated latent): ` ./SD-generate -m Your_Own_Path/dreamlike-anime-1.0 -l '' -t FP16_dyn --height 448 --width 704 `

![image](https://github.com/yangsu2022/OV_SD_CPP/assets/102195992/9bd58b64-6688-417e-b435-c0991247b97b)

## Benchmark:
The performance and image quality of C++ pipeline are aligned with Python

To align the performance with [Python SD pipeline](https://github.com/FionaZZ92/OpenVINO_sample/tree/master/SD_controlnet), C++ pipeline will print the duration of each model inferencing only

For the diffusion part, the duration is for all the steps of Unet inferencing, which is the bottleneck

For the generation quality, be careful with the negative prompt and random latent generation

## Limitation:
* Pipeline features:
```shell
- Batch size 1
- LMS Discrete Scheduler
- Text to image
```
* Program optimization:
now parallel optimization with std::for_each only and add_compile_options(-O3 -march=native -Wall) with CMake
* The pipeline with INT8 model IR not improve the performance  
* Lora enabling only for FP16
* Random generation fails to align, C++ random with MT19937 results is differ from numpy.random.randn(). Hence, please use `-r, --readNPLatent` for the alignment with Python(this latent file is for output image 512X512 only) 
* OV extension tokenizer cannot recognize the special character, like “.”, ”,”, “”, etc. When write prompt, need to use space to split words, and cannot accept empty negative prompt.
So use default tokenizer without config `-e, --useOVExtension`, when negative prompt is empty
  
## Setup in Windows 10 with VS2019:
1. Download [Anaconda3](https://repo.anaconda.com/archive/Anaconda3-2023.09-0-Windows-x86_64.exe) and setup Conda env SD-CPP for OpenVINO with conda-forge and use the anaconda prompt terminal for CMake

2. C++ dependencies:
* OpenVINO:
To deployment without Conda: [Download archives* with OpenVINO](https://storage.openvinotoolkit.org/repositories/openvino/packages/2023.1/windows/), unzip and setup environment vars with `.\setupvars.bat`
* Boost:
```shell
1. Download from https://sourceforge.net/projects/boost/files/boost-binaries/1.83.0/
2. Unzip
3. Setup: bootstrap.bat 
4. Build: b2.exe
5. Install: b2.exe install
```
Installed boost in the path C:/Boost, add CMakeList with `SET(BOOST_ROOT"C:/Boost")`
* Eigen:
```shell
1. Download from https://gitlab.com/libeigen/eigen/-/archive/3.4.0/eigen-3.4.0.zip 
2. Unzip to path C:/Eigen3/eigen-3.4.0 
3. Run next step's build.bat will report error: not found Eigen3Config.cmake/eigen3-config.cmake
- Create build folder for Eigen and Open VS in this path C:/Eigen3/eigen-3.4.0/build
- Open VS's developer PS terminal to do "cmake .." and redo the CMake 
```
Ref:[not found Eigen3Config.cmake/eigen3-config.cmake](https://stackoverflow.com/questions/48144415/not-found-eigen3-dir-when-configuring-a-cmake-project-in-windows)

3. CMake with command lines, create a script build.bat:

```shell
rmdir /Q /S build
mkdir build
cd build
cmake -G "Visual Studio 16 2019" -A x64 ^
 -DCMAKE_BUILD_TYPE=Release ^
      ..
cmake --build . --config Release
```

4. Put safetensors and model IR into the models folder with the defaut path:
`models\dreamlike-anime-1.0\FP16_static` 
`models\soulcard.safetensors`

5. Run with Anaconda prompt:  

```shell
cd PROJECT_SOURCE_DIR\build
.\Release\SD-generate.exe -l ''  // without lora
.\Release\SD-generate.exe
```
```shell
Notice: 
* must run command line within path of build, or .exe could not find the models
* .exe is in the Release folder 
```

6. Debug within Visual Studio(open .sln file in the `build` folder)
