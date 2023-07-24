
::: {.cell .markdown}
### WideResNet
:::

::: {.cell .markdown}

WideResNet model implementation from https://github.com/xternalz/WideResNet-pytorch  

Note: for faster training, use Runtime > Change Runtime Type to run this notebook on a GPU.
:::


::: {.cell .markdown}

In the Cutout paper, the authors claim that:

1. Cutout improves the robustness and overall performance of convolutional neural networks.
2. Cutout can be used in conjunction with existing forms of data augmentation and other regularizers to further improve model performance.

In this section, we will evaluate these claims using ShakeShake regularization. For ShakeShake, the specific quantitative claims are given in the following table:

Test error (%, flip/translation augmentation, mean/std normalization, mean of 3 runs) and “+” indicates standard data augmentation (mirror
+ crop)

| **Network** | **CIFAR-10+** | **CIFAR-100+** |
| ----------- | ------------ | ------------- |
| Shake-shake | 2.86         | 15.58         |
| Shake-shake + cutout | 2.56 | 15.20 |

In the given table, the test errors of the Shake-shake regularization model implemented from a GitHub source on CIFAR-10+ and CIFAR-100+ datasets are presented. The "+" indicates the use of standard data augmentation techniques like mirror and crop.

On the CIFAR-10+ dataset, the use of the Shake-shake regularization model results in a test error of 2.86%. With the application of the cutout augmentation, this error decreases slightly to 2.56%. A similar pattern is noticed with the CIFAR-100+ dataset, where the Shake-shake model results in a test error of 15.58% which gets reduced to 15.20% when cutout augmentation is applied.

These results again underline the effectiveness of cutout augmentation in improving model performance, as indicated by the reduced test error rates. The Shake-shake model with both standard and cutout augmentation techniques performs exceptionally well, particularly on the CIFAR-10+ dataset.

:::



::: {.cell .markdown}
#### Implementation
:::



::: {.cell .markdown}
### 4.1.2 WideResNet Code
:::

::: {.cell .code}
``` python
# WideResNet

# From https://github.com/uoguelph-mlrg/Cutout/blob/master/model/wide_resnet.py

import math
import torch
import torch.nn as nn
import torch.nn.functional as F


class BasicBlockWide(nn.Module):
    def __init__(self, in_planes, out_planes, stride, dropRate=0.0):
        super(BasicBlockWide, self).__init__()
        self.bn1 = nn.BatchNorm2d(in_planes)
        self.relu1 = nn.ReLU(inplace=True)
        self.conv1 = nn.Conv2d(in_planes, out_planes, kernel_size=3, stride=stride,
                               padding=1, bias=False)
        self.bn2 = nn.BatchNorm2d(out_planes)
        self.relu2 = nn.ReLU(inplace=True)
        self.conv2 = nn.Conv2d(out_planes, out_planes, kernel_size=3, stride=1,
                               padding=1, bias=False)
        self.droprate = dropRate
        self.equalInOut = (in_planes == out_planes)
        self.convShortcut = (not self.equalInOut) and nn.Conv2d(in_planes, out_planes, kernel_size=1, stride=stride,
                               padding=0, bias=False) or None
    def forward(self, x):
        if not self.equalInOut:
            x = self.relu1(self.bn1(x))
        else:
            out = self.relu1(self.bn1(x))
        out = self.relu2(self.bn2(self.conv1(out if self.equalInOut else x)))
        if self.droprate > 0:
            out = F.dropout(out, p=self.droprate, training=self.training)
        out = self.conv2(out)
        return torch.add(x if self.equalInOut else self.convShortcut(x), out)

class NetworkBlock(nn.Module):
    def __init__(self, nb_layers, in_planes, out_planes, block, stride, dropRate=0.0):
        super(NetworkBlock, self).__init__()
        self.layer = self._make_layer(block, in_planes, out_planes, nb_layers, stride, dropRate)
    def _make_layer(self, block, in_planes, out_planes, nb_layers, stride, dropRate):
        layers = []
        for i in range(nb_layers):
            layers.append(block(i == 0 and in_planes or out_planes, out_planes, i == 0 and stride or 1, dropRate))
        return nn.Sequential(*layers)
    def forward(self, x):
        return self.layer(x)

class WideResNet(nn.Module):
    def __init__(self, depth, num_classes, widen_factor=1, dropRate=0.0):
        super(WideResNet, self).__init__()
        nChannels = [16, 16*widen_factor, 32*widen_factor, 64*widen_factor]
        assert((depth - 4) % 6 == 0)
        n = (depth - 4) // 6
        block = BasicBlockWide
        # 1st conv before any network block
        self.conv1 = nn.Conv2d(3, nChannels[0], kernel_size=3, stride=1,
                               padding=1, bias=False)
        # 1st block
        self.block1 = NetworkBlock(n, nChannels[0], nChannels[1], block, 1, dropRate)
        # 2nd block
        self.block2 = NetworkBlock(n, nChannels[1], nChannels[2], block, 2, dropRate)
        # 3rd block
        self.block3 = NetworkBlock(n, nChannels[2], nChannels[3], block, 2, dropRate)
        # global average pooling and classifier
        self.bn1 = nn.BatchNorm2d(nChannels[3])
        self.relu = nn.ReLU(inplace=True)
        self.fc = nn.Linear(nChannels[3], num_classes)
        self.nChannels = nChannels[3]

        for m in self.modules():
            if isinstance(m, nn.Conv2d):
                n = m.kernel_size[0] * m.kernel_size[1] * m.out_channels
                m.weight.data.normal_(0, math.sqrt(2. / n))
            elif isinstance(m, nn.BatchNorm2d):
                m.weight.data.fill_(1)
                m.bias.data.zero_()
            elif isinstance(m, nn.Linear):
                m.bias.data.zero_()
    def forward(self, x):
        out = self.conv1(x)
        out = self.block1(out)
        out = self.block2(out)
        out = self.block3(out)
        out = self.relu(self.bn1(out))

        out = F.avg_pool2d(out, 8)
        out = out.view(-1, self.nChannels)
        out = self.fc(out)
        return out
```
:::