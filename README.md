# Extract VPR model to SNPE format
This repo contains the code to convert pre-trained yolov5 model to run on SNPE Qualcomm chip DSP.
It is based on the [YOLOv5 repo](https://github.com/ultralytics/yolov5) by Ultralytics, AGPL-3.0 license

### Steps to converting the model to SNPE format to run on the DSP:
* Train the detector using the train.py script
* Export the model to ONNX format using  the export.py script
* Convert the model to dlc using Qualcomm's SNPE converter
* Quantize the model using Qualcomm's SNPE converter
* Run the model on the DSP using Qualcomm's SNPE SDK, and the provided C++ code

## Train the detector on COCO dataset
* Follow the instructions in the [YOLOv5 repo](https://docs.ultralytics.com/yolov5/tutorials/train_custom_data/)
```bash
cd yolov5
pip install -r requirements.txt  # install
 # single class 
python train.py --data coco128.yaml --weights '' --cfg yolov5s-LeakyReLU_1class.yaml --img 320 --single-cls  
# multi class 
python train.py --data coco128.yaml --weights '' --cfg yolov5s-LeakyReLU.yaml --img 320 
```

##  Extract the onnx model from the YOLOv5 repo
```bash
python export.py --weights runs/train/exp/weights/best.pt --include onnx --opset 11 --imgsz 256 320 --export-to-snpe
```

## Make sure your model looks good
* go to https://netron.app/ and drag your onnx file to the browser
* get the input name there

## Convert the model to SNPE format.
* Download SNPE-converter from [Qualcomm's SNPE Tools](https://developer.qualcomm.com/software/qualcomm-neural-processing-sdk/tools-archive)
  * This code was tested using [Qualcomm Neural Processing SDK for Linux v2.10.0](https://developer.qualcomm.com/downloads/qualcomm-neural-processing-sdk-linux-v2100)
  * Driver Version: 460.73.01  CUDA Version: 11.3 
  * Install the SNPE converter to a conda environment using
      ```bash
      conda env create -f snpe_environment_clean.yml
      conda activate snpe_conda2
      ```
* Run SNPE env setup 
  * The script sets up the following environment variables.
    * SNPE_ROOT: root directory of the SNPE SDK installation
    * ONNX_HOME: root directory of the ONNX installation provided
    * The script also updates PATH, LD_LIBRARY_PATH, and PYTHONPATH.
    ```bash
    cd $SNPE_ROOT
    source bin/envsetup.sh -o $ONNX_DIR
    where $ONNX_DIR is the path to the ONNX installation.
    # for example
    source snpe_2_10_0_4541/bin/envsetup.sh -o /home/ubuntu/anaconda3/envs/snpe_conda3/lib/python3.6/site-packages/onnx
    ```
    * this warning is ok
      ```bash
      [WARNING] Can't find ANDROID_NDK_ROOT or ndk-build. SNPE needs android ndk to build the NativeCppExample
      ``` 
* Run SNPE extractor on example input to dlc
```bash
python3.6 snpe_2_10_0_4541/bin/x86_64-linux-clang/snpe-onnx-to-dlc -i runs/train/exp/weights/best.onnx -d 'images' 1,3,256,320
```

* If you get the following error `ImportError: libpython3.6m.so.1.0: cannot open shared object file: No such file or directory`
  * try running the following command `sudo apt-get install libpython3.6`
  `pip3 install pyyaml`
* This error is OK
```bash
* WARNING_OP_NOT_SUPPORTED_BY_ONNX: Unable to register converter supported Operation [GridSample:Version 16] with your Onnx installation. Got: No schema registered for 'GridSample'!. Converter will bail if Model contains this Op.
``

## Convert dlc to quantized dlc
* Create a set of example inputs
* Run the quantization script
  * The script will create a quantized dlc file
```bash
snpe_2_10_0_4541/bin/x86_64-linux-clang/snpe-dlc-quantize \
      --input_dlc=runs/train/exp/weights/best.dlc \
      --output_dlc=runs/train/exp/weights/best_quantized.dlc \
      --input_list=examples/target_raw_list.txt \
      --debug1
```

