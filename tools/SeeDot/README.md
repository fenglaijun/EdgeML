# SeeDot

SeeDot is an automatic quantization tool that generates efficient machine learning (ML) inference code for IoT devices.

### **Overview**

ML models are usually expressed in floating-point, and IoT devices typically lack hardware support for floating-point arithmetic. Hence, running such ML models on IoT devices involves simulating floating-point arithmetic in software, which is very inefficient. SeeDot addresses this issue by generating fixed-point code with only integer operations. To enable this, SeeDot takes as input trained floating-point models (like [Bonsai](https://github.com/microsoft/EdgeML/blob/master/docs/publications/Bonsai.pdf) or [ProtoNN](https://github.com/microsoft/EdgeML/blob/master/docs/publications/ProtoNN.pdf)) and generates efficient fixed-point code that can run on microcontrollers. The SeeDot compiler uses novel compilation techniques like automatically inferring certain parameters used in the fixed-point code, optimized exponentiation computation, etc. With these techniques, the generated fixed-point code has comparable classification accuracy and performs significantly faster than the floating-point code.

To know more about SeeDot, please refer to our publication [here](https://www.microsoft.com/en-us/research/publication/compiling-kb-sized-machine-learning-models-to-constrained-hardware/).

This document describes the tool usage with an example.

### **Software requirements**

1. [**Python 3**](https://www.python.org/) with following packages:
   - **[Antrl4](http://www.antlr.org/)** (antlr4-python3-runtime; tested with version 4.7.2)
   - **[Numpy](http://www.numpy.org/)** (tested with version 1.16.2)
   - **[Scikit-learn](https://scikit-learn.org/)** (tested with version 0.20.3)
2. Linux packages:
   - **[gcc](https://www.gnu.org/software/gcc/)** (tested with version 7.3.0)
   - **[make](https://www.gnu.org/software/make/)** (tested with version 4.1)

### **Usage**

SeeDot can be invoked using **`SeeDot.py`** file. The arguments for the script are supplied as follows:

```
usage: SeeDot.py [-h] [-a] --train  --test  --model  [--tempdir] [-o]

optional arguments:
  -h, --help      show this help message and exit
  -a , --algo     Algorithm to run ('bonsai' or 'protonn')
  --train         Training set file
  --test          Testing set file
  --model         Directory containing trained model (output from
                  Bonsai/ProtoNN trainer)
  --tempdir       Scratch directory for intermediate files
  -o , --outdir   Directory to output the generated Arduino sketch
```

An example invocation is as follows:
```
python SeeDot.py -a bonsai --train path/to/train.npy --test path/to/test.npy --model path/to/Bonsai/model
```

SeeDot expects the `train` and the `test` data files in a specific format. Each data file should be of the shape `[numberOfDataPoints, numberOfFeatures + 1]`, where the class label is in the first column. The tool currently support the following file formats for the data files: numpy arrays (.npy), tab-separated values (.tsv), comma-separated values (.csv), and libsvm (.txt).

The path to the trained Bonsai/ProtoNN model is specified in the `--model` argument. After training, the learned parameters are stored in this directory in a specific format. For Bonsai, the learned parameters are `Z`, `W`, `V`, `T`, `Sigma`, `Mean`, and `Std`. For ProtoNN, the learned parameters are `W`, `B`, and `Z`. These parameters can be either numpy arrays (.npy) or plaintext files.

The `tempdir` directory is used to store the intermediate files generated by the compiler. The device-specific fixed-point code is stored in the `outdir` directory.


## Getting started: Quantizing ProtoNN on usps10

To help get started with SeeDot, we provide 1) a pre-loaded fixed-point model, and 2) instructions to generate fixed-point code for the ProtoNN predictor on the **[usps10](https://www.csie.ntu.edu.tw/~cjlin/libsvmtools/datasets/multiclass/)** dataset. The process for generating fixed-point code for the Bonsai predictor is similar.

### Pre-loaded model

To make it easy to test the SeeDot-generated code, a ready-to-upload Arduino sketch is provided that can be run on an Arduino device without any changes. The sketch is located at `Tools/SeeDot/seedot/arduino/arduino.ino` and contains pre-loaded ProtoNN model on the usps10 dataset. To upload the sketch to the device, skip steps 1-3 in the below guide and follow the [step 4: Prediction on the device](https://github.com/microsoft/EdgeML/tree/Feature/SeeDot/Tools/SeeDot#step-4-prediction-on-the-device).

### Generating fixed-point code

This process consists of four steps: 1) installing EdgeML TensorFlow library, 2) training ProtoNN on usps10, 3) quantizing the trained model with SeeDot, and 4) performing prediction on the device.

#### **Step 1: Installing EdgeML TensorFlow library**

1. Clone the EdgeML repository and navigate to the right directory.
     ```
     git clone https://github.com/Microsoft/EdgeML
     cd EdgeML/tf/
     ```

2. Install the EdgeML library.
     ```
     pip install -r requirements-cpu.txt
     pip install -e .
     ```

#### **Step 2: Training ProtoNN on usps10** 

1. Navigate to the ProtoNN examples directory.
     ```
     cd examples/ProtoNN
     ```
     
2. Fetch usps10 data and create output directory.
     ```
     python fetch_usps.py
     python process_usps.py
     mkdir usps10/output
      ```

3. Invoke ProtoNN trainer using the following command.
      ```
      python protoNN_example.py --data-dir ./usps10 --projection-dim 25 --num-prototypes 55 --epochs 100 -sW 0.3 -o usps10/output
      ```
  This would give around 90.035% classification accuracy. The trained model is stored in the `output` directory.

More information on using the ProtoNN trainer can be found [here](https://github.com/Microsoft/EdgeML/tree/master/tf/examples/ProtoNN).

#### **Step 3: Quantizing with SeeDot**

1. Navigate to the SeeDot directory and create the output directory.
      ```
      cd ../../../Tools/SeeDot
      mkdir arduino
      ```

2. Invoke SeeDot using the following command.
      ```
      python SeeDot.py -a protonn --train ../../tf/examples/ProtoNN/usps10/train.npy --test ../../tf/examples/ProtoNN/usps10/test.npy --model ../../tf/examples/ProtoNN/usps10/output -o arduino
      ```

   The SeeDot-generated code would give around 89.985% classification accuracy. The difference in classification accuracy is 0.05% compared to the floating-point code. The generated code is stored in the `arduino` folder which contains the sketch along with two files: model.h and predict.cpp. `model.h` contains the quantized model and `predict.cpp` contains the inference code.

#### **Step 4: Prediction on the device**

Follow the below steps to perform prediction on the device, where the SeeDot-generated code is run on a single data-point stored on the device's flash memory.

1. Open the Arduino sketch file located at `arduino/arduino.ino` in the [Arduino IDE](https://www.arduino.cc/en/main/software).
2. Connect the Arduino microcontroller to the computer and choose the correct board configuration.
3. Upload the sketch to the device.
4. Open the Serial Monitor and select baud rate specified in the sketch (default is 115200) to monitor the output.
5. The average prediction time is computed every 100 iterations. On an Arduino Uno, the average prediction time is 35991 micro seconds.

More device-specific details on extending the Arduino sketch for other use cases can be found in [`arduino/README.md`](https://github.com/microsoft/EdgeML/blob/Feature/SeeDot/Tools/SeeDot/seedot/arduino/README.md).


The above workflow has been tested on Arduino Uno and Arduino MKR1000. It is expected to work on other Arduino devices as well.


Copyright (c) Microsoft Corporation. All rights reserved. Licensed under the MIT license.
