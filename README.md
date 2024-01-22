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
python export.py --weights runs/train/exp/weights/best.pt --include onnx --opset 11 --imgsz 256 320 --export-to-snpe --batch-size 2
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
```

## Convert dlc to quantized dlc
* Create a set of example inputs. Change this code to include a set of real inputs
```
import torch
import numpy as np
data = torch.rand(1, 3, 256, 320)
# Convert the tensor to a NumPy array and save it to a binary file
with open(r"float_data.bin", 'wb') as file:
    np.array(data.numpy(), dtype=np.float32).tofile(file)
```
* create a txt file with the path to all the example inputs see `examples/target_raw_list.txt`
* Run the quantization script
  * The script will create a quantized dlc file
```bash
snpe_2_10_0_4541/bin/x86_64-linux-clang/snpe-dlc-quantize \
      --input_dlc=runs/train/exp/weights/best.dlc \
      --output_dlc=runs/train/exp/weights/best_quantized.dlc \
      --input_list=examples/target_raw_list.txt \
      --debug1
```

## Run the model on the DSP
* update the C++ following the instructions in this [yolov5 issue](https://github.com/ultralytics/yolov5/issues/4790#issuecomment-1148899676)
```bash
Dnn::BBoxesSortedByConfidence DecodeOutput(tcb::span<Math::Scalar> dnn_output, const YoloV5Config &config)
{
  SCOPED_TRACE_ALGO("YoloV5::DecodeOutput");
  Dnn::BBoxesSortedByConfidence bboxes;
  // modified from https://github.com/ultralytics/yolov5/issues/4790
  for (size_t c = 4; c < dnn_output.size(); c += 6) {
    if (dnn_output[c] >= config.confidence_threshold_before_nms) {
      float cx = dnn_output[c - 4];
      float cy = dnn_output[c - 3];
      float w = dnn_output[c - 2];
      float h = dnn_output[c - 1];
      int gridX, gridY;
      int anchor_gridX, anchor_gridY;
      const auto &anchor_x = config.anchor_x;
      const auto &anchor_y = config.anchor_y;
      const auto &filter_size_y = config.filter_size_y;
      const auto &filter_size_x = config.filter_size_x;
      const auto &num_filters = config.num_filters;

      int stride;
      int ci = (int)(c / 6);
      if (ci < num_filters[0]) {
        gridX = (ci % (filter_size_x[0] * filter_size_y[0])) % filter_size_x[0];
        gridY = (int)((ci % (filter_size_x[0] * filter_size_y[0])) / filter_size_x[0]);
        anchor_gridX = anchor_x[((int)(ci / (filter_size_x[0] * filter_size_y[0])))];
        anchor_gridY = anchor_y[((int)(ci / (filter_size_x[0] * filter_size_y[0])))];
        stride = 8;
      } else if (ci >= num_filters[0] && ci < num_filters[0] + num_filters[1]) {
        gridX = ((ci - num_filters[0]) % (filter_size_x[1] * filter_size_y[1])) % filter_size_x[1];
        gridY = (int)(((ci - num_filters[0]) % (filter_size_x[1] * filter_size_y[1])) / filter_size_x[1]);
        anchor_gridX = anchor_x[(int)((ci - num_filters[0]) / (filter_size_x[1] * filter_size_y[1])) + 3];
        anchor_gridY = anchor_y[(int)((ci - num_filters[0]) / (filter_size_x[1] * filter_size_y[1])) + 3];
        stride = 16;
      } else {
        gridX = ((ci - num_filters[1]) % (filter_size_x[2] * filter_size_y[2])) % filter_size_x[2];
        gridY = (int)(((ci - num_filters[1]) % (filter_size_x[2] * filter_size_y[2])) / filter_size_x[2]);
        anchor_gridX = anchor_x[(int)((ci - num_filters[1] - num_filters[0]) / (filter_size_x[2] * filter_size_y[2])) + 6];
        anchor_gridY = anchor_y[(int)((ci - num_filters[1] - num_filters[0]) / (filter_size_x[2] * filter_size_y[2])) + 6];
        stride = 32;
      }
      cx = (float)(cx * 2 - 0.5 + gridX) * stride;
      cy = (float)(cy * 2 - 0.5 + gridY) * stride;
      w = w * 2 * w * 2 * anchor_gridX;
      h = h * 2 * h * 2 * anchor_gridY;
      float left = cx - w / 2;
      float top = cy - h / 2;
      float right = cx + w / 2;
      float bottom = cy + h / 2;
      float confidence = dnn_output[c];
      cv::Rect2f bbox{ left, top, right - left, bottom - top };

      Dnn::DetectionResult result{ config.keyboard_class, confidence, bbox };
      bboxes.emplace(confidence, result);
    }
  }
  return bboxes;
}
```
* Build the C++ code
```bash
/data/local/tmp/dnn-test/snpe2.10 # LD_LIBRARY_PATH=arm64-v8a ./run-dnn --native-library-dir arm64-v8a --dnn ../best_quantized.dlc   --input-layer images --width 320 --height 256 --channels 3 --iterations 100 --batch-size 1
```
* The runtime should be around 13ms for a single image with size 256x320
