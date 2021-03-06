# PyTorch Network Slimming

This repo implements the following paper in [PyTorch](https://pytorch.org):  

[**Learning Efficient Convolutional Networks Through Network Slimming**](http://openaccess.thecvf.com/content_iccv_2017/html/Liu_Learning_Efficient_Convolutional_ICCV_2017_paper.html)

Different with most other popular network slimming repos, this implementation enables training & pruning models in hand with a few lines of new codes. Writing codes specifically for pruning is not required. Moreover, the pruned model can be further improved and deployed with toolkits supporting ONNX. An example of further acceleration with OpenVINO is included.

BN layers are automatically identified by 1) parse the traced graph by [TorchScript](https://pytorch.org/docs/stable/jit.html), and 2) identify **prunable BN layers** which only have Convolution(groups=1)/Linear in preceding & succeeding layers. Example of a **prunable BN**:

            conv/linear --> ... --> BN --> ... --> conv/linear
                                     |
                                    ...
                                     | --> relu --> ... --> conv/linear
                                             |
                                             | --> ... --> maxpool --> ... --> conv/linear
Note that without coding for patches, such as [channel_selection](https://github.com/Eric-mingjie/network-slimming/blob/master/models/channel_selection.py), the prunable BNs for some network architectures (ResNet/DenseNet/...) will be less than the [official implementation](https://github.com/Eric-mingjie/network-slimming). For more details please refer to source code. 

It is supposed to support user defined models with Convolution(groups=1)/Linear and BN layers. The package is tested with the [CIFAR-100](https://www.cs.toronto.edu/~kriz/cifar.html) examples in this repo, and an in-house [Conv3d](https://pytorch.org/docs/stable/nn.html#conv3d) based model for video classification. 

<font size=2> \* ***DataParalell*** is not supported </font>

## Known Issue
### Node without an explicit name in traced graph
This code depends on traced graph by [TorchScript](https://pytorch.org/docs/stable/jit.html), so any graph without an explicit module name will fail. For example:

   ```python
   def foward(self, x)
       ...
       for name, child in self.backbone.named_children():
           x = child(x)
       ...
   ```
### shortcut from BN to BN
will be fixed later ...

## Results on CIFAR-100

### Accuracy (%)

|                           | Original | L1 on BN | PR=0.3 | PR=0.5 | PR=0.7 | PR=0.3 (TFS) | PR=0.5 (TFS) | PR=0.7 (TFS) |
| :------------------------ | :------: | :------: | :----: | :----: | :----: | :----------: | :----------: | :----------: |
| ResNet-18                 |  78.90   |  78.21   | 78.60  | 78.02  | 74.97  |    78.31     |    76.84     |    73.93     |
| ResNet-18 (L1 on all BNs) |  78.90   |  78.51   | 79.07  | 77.58  | 75.30  |    78.29     |    76.77     |    74.26     |
| VGG-11                    |  71.04   |  70.66   | 71.69  | 69.16  | 58.95  |    70.74     |    68.50     |    61.13     |
| simplified VGG-11         |  71.72   |  71.62   | 71.55  | 68.80  |   F    |    70.58     |    67.37     |    52.87     |
| DenseNet-63               |  78.34   |  78.06   | 78.11  | 77.76  | 76.33  |    78.12     |    77.65     |    76.01     |

### Params (M)

|                           | Original | PR=0.3 | PR=0.5 | PR=0.7 |
| :------------------------ | :------: | :----: | :----: | :----: |
| ResNet-18                 |  10.70   |  7.91  |  6.44  |  4.56  |
| ResNet-18 (L1 on all BNs) |  10.70   |  7.84  |  6.39  |  4.52  |
| VGG-11                    |  27.20   | 21.75  | 19.35  | 18.00  |
| simplified VGG-11         |   8.85   |  4.63  |  2.09  |  0.54  |
| DenseNet-63               |   2.19   |  1.63  |  1.32  |  0.93  |

### GFLOPS

|                           | Original | PR=0.3 | PR=0.5 | PR=0.7 |
| :------------------------ | :------: | :----: | :----: | :----: |
| ResNet-18                 |   0.52   |  0.32  |  0.21  |  0.12  |
| ResNet-18 (L1 on all BNs) |   0.52   |  0.33  |  0.22  |  0.12  |
| VGG-11                    |   0.19   |  0.14  |  0.09  |  0.06  |
| simplified VGG-11         |   0.16   |  0.06  |  0.03  |  0.01  |
| DenseNet-63               |   0.30   |  0.18  |  0.12  |  0.07  |

<font size=2> \* **F**: Failed to converge </font>

<font size=2> \* **PR**: Prune Ratio </font>

<font size=2>\* **TFS**: Train-from-scratch as proposed in Liu's later paper on ICLR 2019 [**Rethinking the Value of Network Pruning**](https://openreview.net/forum?id=rJlnB3C5Ym). </font>

## Requirements

Python >= 3.6  
torch >= 1.1.0  
torchvision >= 0.3.0  

## Usage

1. Import from [netslim](./netslim) in your training script
   
     ```python
   from netslim import update_bn
   ```
   
2. Insert *updat_bn* between *loss.backward()* and *optimizer.step()*. The following is an example:

   *before*

   ```python
   ...
   loss.backward()
   optimizer.step()
   ...
   ```

   *after*

      ```python
   ...
   loss.backward()
   update_bn(model)  # or update_bn(model, s), specify s to control sparsity on BN
   optimizer.step()
   ...
      ```

   <font size=2> \* ***update_bn*** puts L1 regularization on all BNs. Sparsity on prunable BNs only is also supported for networks with complex connections, such as ResNet. Check examples for more details. </font>

3. Prune the model after training

   ```python
   from netslim import prune
   # For example, input_shape for CIFAR is (3, 32, 32)
   pruned_model = prune(model, input_shape, prune_ratio=0.5)
   ```

4. Fine-tune & export model

5. Load the pruned model and have fun

   ```
   from netslim import load_pruned_model
   model = MyModel()
   weights = torch.load("/path/to/pruned-weights.pth")
   pruned_model = load_pruned_model(model, weights)
   ...
   ```

## Run CIFAR-100 examples

### ResNet-18

   ```shell
sh experiment-resnet.sh
   ```

### VGG-11

   ```shell
sh experiment-vgg11.sh
sh experiment-vgg11s.sh % simplified VGG-11 by replacing classifier with a Linear
   ```

### DenseNet-63

   ```shell
sh experiment-densenet.sh
   ```

### Train from scratch

Run shell scripts end with "_tfs". For example, train pruned ResNet-18 after running L1 regularized training:

```shell
sh experiment_resnet_tfs.sh
```

### Load & test pruned model (ResNet-18 example)

   ```shell
python test.py --arch resnet18 --resume_path output-resnet18-bn-pr05/ckpt_best.pth
   ```

<font size=2> ***-all-bn*** refers to L1 sparsity on all BN layers </font>

## Inference with [OpenVINO](https://software.intel.com/en-us/openvino-toolkit)

The efficiency of pruned model can be further improved by using OpenVINO, if you are working with Intel processors/accelerators.  

1. Download OpenVINO from the official website [OpenVINO](https://software.intel.com/en-us/openvino-toolkit).

2. After installation, initialize the environment:

   ```shell
   source /opt/intel/openvino/bin/setupvars.sh
   ```

3. Take simplified VGG-11 as an example, after finished running *experiment-vgg11s.sh*, convert pruned model to OpenVINO IR:

   ```shell
   python export2openvino.py --arch vgg11s --weights output-vgg11s-bn-pr05/ckpt_best.pth --outname vgg11s-pr05
   python export2openvino.py --arch vgg11s --weights output-vgg11s-bn-pr05/ckpt_best.pth --outname vgg11s-pr05 --fp16
   ```

4. Test with CIFAR-100

   ```shell
   python test-openvino.py vgg11s-FP32
   python test-openvino.py vgg11s-FP16
   ```

5. Compare runtime

   ```shell
   python cpu-runtime-benchmark-vgg11s.py 
   ```

On a desktop with Xeon 8160 CPU, the FPS results are:

| PyTorch Full Model | PyTorch PR=0.5 | OpenVINO PR=0.5 FP32 | OpenVINO PR=0.5 FP16 |
| :----------------: | :------------: | :------------------: | :------------------: |
|       89.50        |     146.21     |        960.49        |        973.29        |

## Acknowledgement

The implementation of ***udpate_bn*** is referred to [pytorch-slimming](https://github.com/foolwood/pytorch-slimming).
