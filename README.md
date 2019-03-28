# torchparse: PyTorch cfg Model Parser

Simple (and for now, sequential) PyTorch model parser. Allowes to define a model in a cfg file for easier iteration.

**Features**

- Don't have to worry about I/O dimensions
- Easily define dimension reshapes between layers
- Repeat block syntax for less typing
- Get a neat `mm.ModuleDict` back with the desired `nn.Sequentials`

## Contents
- [Installation](#installation)
- [Simple-Usage](#simple-usage)
- [Supported modules](#supported-modules)
- [Detailed Usage](#detailed-usage)

### Installation

```bash
pip install git+https://github.com/ksanjeevan/torchparse.git
```



### Simple Usage

Define model in a `.cfg` file (or string), e.g.:

```bash
[convs_module]
    REPEATx3
        [conv2d]
            out_channels=32
            kernel_size=3
            stride=1
            padding=valid
        [batchnorm2d]
        [elu]
        [maxpool2d]
            kernel_size=3
            stride=3
    END

[moddims]
    collapse=[0,1,2]

[dense_module]
    [linear]
        out_features = 500
    [relu]
    [linear]
        out_features = 10
```
Then, calling **`parse_cfg(cfg_fname, shape=[3,200,300])`** returns:

```python
ModuleDict(
  (convs): Sequential(
    (conv2d_0): Conv2d(3, 32, kernel_size=(3, 3), stride=(1, 1))
    (batchnorm2d_0): BatchNorm2d(32, eps=1e-05, momentum=0.1, affine=True, track_running_stats=True)
    (elu_0): ELU(alpha=1.0)
    (maxpool2d_0): MaxPool2d(kernel_size=3, stride=3, padding=0, dilation=1, ceil_mode=False)
    (conv2d_1): Conv2d(32, 32, kernel_size=(3, 3), stride=(1, 1))
    (batchnorm2d_1): BatchNorm2d(32, eps=1e-05, momentum=0.1, affine=True, track_running_stats=True)
    (elu_1): ELU(alpha=1.0)
    (maxpool2d_1): MaxPool2d(kernel_size=3, stride=3, padding=0, dilation=1, ceil_mode=False)
    (conv2d_2): Conv2d(32, 32, kernel_size=(3, 3), stride=(1, 1))
    (batchnorm2d_2): BatchNorm2d(32, eps=1e-05, momentum=0.1, affine=True, track_running_stats=True)
    (elu_2): ELU(alpha=1.0)
    (maxpool2d_2): MaxPool2d(kernel_size=3, stride=3, padding=0, dilation=1, ceil_mode=False)
  )
  (dense): Sequential(
    (linear_0): Linear(in_features=1920, out_features=500, bias=True)
    (relu_0): ReLU()
    (linear_1): Linear(in_features=500, out_features=10, bias=True)
  )
)

```




### Supported modules

Implemented layers (`nn.Module`):

- [x] `Linear`
- [x] `LSTM`, `GRU`, `RNN`
- [ ] `Conv1d`
- [ ] `MaxPool1d`
- [x] `MaxPool2d`
- [x] `Conv2d`
- [x] `BatchNorm1d`, `BatchNorm2d`
- [ ] `AvgPool2d`, `AvgPool1d`
- [x] `ReLU`, `ELU`, `LeakyReLU`, `Sigmoid`
- [x] `Dropout`
- [ ] `AdaptiveMaxPool2d`, `AdaptiveMaxPool1d`
- [ ] `ConvTranspose2d`, `ConvTranspose1d`


### Detailed Usage


#### *[moddims]*: account for Tensor manipulations

Allows to incorporate in the cfg file any tranpose or reshape that will occur in the `forward` call, since this will affect the  intermmediate shapes.

##### transpose
---

For example if in `forward()`:
```
...
# (batch, height, width, channel) -> (batch, channel, height, width)
x = x.permute([0,3,1,2])
...
```

then in `.cfg` add:

```
...
[moddims]
	transpose=[2,0,1]
...
```


##### collapse
---

For example if in `forward()`:
```
...
# (batch, time, freq, channel) -> (batch, time, freq*channel)
batch, time = x.size()[:2]
x = x.view(batch, time, -1)
...
```

then in `.cfg` add:

```
...
[moddims]
	collapse=[1,2]
...
```

##### drop
---

For example if in `forward()`:
```
...
# (batch, time, feature) -> (batch, feature)
x = x[:,-1]
...
```

then in `.cfg` add:

```
...
[moddims]
	drop=[0]
...
```


#### *[_module]*: sub-module Sequential blocks
Even for a sequential model there might be transformations applied in the `forward` call that aren't defined in the `nn.Module` (e.g.: example above where the `conv_module` will be seperatley defined from the `recur_module` since the `foraward` call will deal with the reshapes, packing sequences, etc.). 

`torchparse.parse_cfg` will return an `nn.ModuleDict`. If no submodules are explicitly defined, the `nn.ModuleDict` will only have one key (`main`) mapping to the defined `nn.Sequential`.

#### *REPEATx*: for repeating blocks of layers

If the model has blocks of layers that repeat with the same paramater values (like in the example above or in `example.py`), they can be written in the `.cfg` as:

```bash
...
REPEATx3
    [conv2d]
        out_channels=64
        kernel_size=3
        stride=1
        padding=valid
    [batchnorm2d]
    [elu]
    [maxpool2d]
        kernel_size=4
        stride=4
END
...
```


### TODO

- [x] Non _module cfg handling
- [x] Block repetitions
- [ ] Skip connections