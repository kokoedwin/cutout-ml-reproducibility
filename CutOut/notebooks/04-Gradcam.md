
::: {.cell .markdown}
# 04. Grad-CAM
:::

 

::: {.cell .markdown}
###### What is Grad-CAM?
Grad-CAM (Gradient-weighted Class Activation Mapping) is a technique that provides visual explanations for decisions made by Convolutional Neural Network (CNN) models. It uses the gradients of any target concept, flowing into the final convolutional layer to produce a coarse localization map highlighting the important regions in the image for predicting the concept.

Grad-CAM is not limited to a specific architecture, it can be applied to a wide range of CNN models without any changes to their existing structure or requiring re-training. It's also class-discriminative, allowing it to effectively manage multi-label scenarios.

By visualizing the model's focus areas with Grad-CAM, we can assess how effectively Cutout is encouraging the model to use a broader range of features. For example, if a model trained with Cutout still primarily focuses on a single region, that might suggest the Cutout squares are too small, or not numerous enough. Conversely, if the focus areas are well spread across the image, it would confirm that Cutout is indeed pushing the model to generalize better.

If you want to understand more about Grad-CAM? Check this paper (https://arxiv.org/abs/1610.02391)
:::


::: {.cell .markdown}

## Import Library

:::


::: {.cell .code}
```python
import torch
import torch.nn as nn
import torch.nn.functional as F
from torch.autograd import Variable
import torch.backends.cudnn as cudnn
from torch.optim.lr_scheduler import MultiStepLR
from torchvision import datasets, transforms
import numpy as np
import os
from tqdm import tqdm
import math
import cv2
import matplotlib.pyplot as plt
from PIL import Image
```
:::


::: {.cell .markdown}
Check Cuda GPU availability and set seed number
:::
::: {.cell .code}
``` python
cuda = torch.cuda.is_available()
print(cuda)
cudnn.benchmark = True  # Should make training should go faster for large models

seed = 1
torch.manual_seed(seed)
np.random.seed(seed)
```
:::

::: {.cell .markdown}
If you are using Google Colab, here's a step-by-step how to connect with your google drive:

1. On the left sidebar of the Colab notebook interface, you will see a folder icon with the Google Drive logo. Click on this folder icon to open the file explorer.

2. If you haven't connected your Google Drive to Colab yet, it will prompt you to do so. Click the "Mount Drive" button to connect your Google Drive to Colab.

3. Once your Google Drive is mounted, you can use the file explorer to navigate to the file you want to open. Click on the folders to explore the contents of your Google Drive.

4. When you find the file you want to open, click the three dots next to the name of the file in the file explorer. From the options that appear, choose "Copy path." This action will copy the full path of the file to your clipboard. Paste the copy path into the 'current_path' below.
:::

::: {.cell .code}
``` python
current_path ="./"
```
:::

::: {.cell .markdown}
Note: This section of the notebook requires specific model checkpoints to be available in the "checkpoints" directory. These checkpoints are files where the trained models were saved during previous sections. You don't necessarily need to run every part of this notebook to use GradCam for a specific model. If you intend to use GradCam for a particular model (e.g., ResNet18 or WideResNet), please make sure that you have previously run and saved the model checkpoints for that architecture. These checkpoints should be named appropriately (e.g., "resnet18_cifar100_cutout.pt" or "wideresnet_cifar10.pt") and be located in the "checkpoints" directory.
:::

::: {.cell .markdown}
## 4.2 Implementation Grad-CAM for ResNet Model
:::

::: {.cell .markdown}
This is the code of the ResNet18 Architecture and it is the similar to the code we introduced earlier in the section about ResNet18, but we added hook layers. Hook layers are inserted at specific layers of the model to capture the intermediate feature maps. These feature maps represent the learned activations of the model as it processes the input image. GRAD-CAM uses these activations to understand which parts of the image are contributing the most to the final prediction.
:::

::: {.cell .code}
``` python

# ResNet
# From https://github.com/uoguelph-mlrg/Cutout/blob/master/model/resnet.py

def conv3x3(in_planes, out_planes, stride=1):
    return nn.Conv2d(in_planes, out_planes, kernel_size=3, stride=stride, padding=1, bias=False)


class BasicBlock(nn.Module):
    expansion = 1

    def __init__(self, in_planes, planes, stride=1):
        super(BasicBlock, self).__init__()
        self.conv1 = conv3x3(in_planes, planes, stride)
        self.bn1 = nn.BatchNorm2d(planes)
        self.conv2 = conv3x3(planes, planes)
        self.bn2 = nn.BatchNorm2d(planes)

        self.shortcut = nn.Sequential()
        if stride != 1 or in_planes != self.expansion*planes:
            self.shortcut = nn.Sequential(
                nn.Conv2d(in_planes, self.expansion*planes, kernel_size=1, stride=stride, bias=False),
                nn.BatchNorm2d(self.expansion*planes)
            )

    def forward(self, x):
        out = F.relu(self.bn1(self.conv1(x)))
        out = self.bn2(self.conv2(out))
        out += self.shortcut(x)
        out = F.relu(out)
        return out


class Bottleneck(nn.Module):
    expansion = 4

    def __init__(self, in_planes, planes, stride=1):
        super(Bottleneck, self).__init__()
        self.conv1 = nn.Conv2d(in_planes, planes, kernel_size=1, bias=False)
        self.bn1 = nn.BatchNorm2d(planes)
        self.conv2 = nn.Conv2d(planes, planes, kernel_size=3, stride=stride, padding=1, bias=False)
        self.bn2 = nn.BatchNorm2d(planes)
        self.conv3 = nn.Conv2d(planes, self.expansion*planes, kernel_size=1, bias=False)
        self.bn3 = nn.BatchNorm2d(self.expansion*planes)

        self.shortcut = nn.Sequential()
        if stride != 1 or in_planes != self.expansion*planes:
            self.shortcut = nn.Sequential(
                nn.Conv2d(in_planes, self.expansion*planes, kernel_size=1, stride=stride, bias=False),
                nn.BatchNorm2d(self.expansion*planes)
            )

    def forward(self, x):
        out = F.relu(self.bn1(self.conv1(x)))
        out = F.relu(self.bn2(self.conv2(out)))
        out = self.bn3(self.conv3(out))
        out += self.shortcut(x)
        out = F.relu(out)
        return out


class ResNet(nn.Module):
    def __init__(self, block, num_blocks, num_classes=10):
        super(ResNet, self).__init__()
        self.in_planes = 64

        self.conv1 = conv3x3(3,64)
        self.bn1 = nn.BatchNorm2d(64)
        self.layer1 = self._make_layer(block, 64, num_blocks[0], stride=1)
        self.layer2 = self._make_layer(block, 128, num_blocks[1], stride=2)
        self.layer3 = self._make_layer(block, 256, num_blocks[2], stride=2)
        self.layer4 = self._make_layer(block, 512, num_blocks[3], stride=2)
        self.linear = nn.Linear(512*block.expansion, num_classes)

        # Register hooks for Grad-CAM
        self.gradients = None
        self.activations = None
        self.layer4.register_forward_hook(self._store_activations_hook)
        self.layer4.register_backward_hook(self._store_gradients_hook)

    def _store_activations_hook(self, module, input, output):
        self.activations = output

    def _store_gradients_hook(self, module, grad_input, grad_output):
        self.gradients = grad_output[0]

    def _make_layer(self, block, planes, num_blocks, stride):
        strides = [stride] + [1]*(num_blocks-1)
        layers = []
        for stride in strides:
            layers.append(block(self.in_planes, planes, stride))
            self.in_planes = planes * block.expansion
        return nn.Sequential(*layers)

    def forward(self, x):
        out = F.relu(self.bn1(self.conv1(x)))
        out = self.layer1(out)
        out = self.layer2(out)
        out = self.layer3(out)
        out = self.layer4(out) 
        out = F.avg_pool2d(out, 4)
        out = out.view(out.size(0), -1)
        out = self.linear(out)
        return out

def ResNet18(num_classes=10):
    return ResNet(BasicBlock, [2,2,2,2], num_classes)
```
:::

::: {.cell .markdown}
## 4.2.1 Implementation Grad-CAM for ResNet18 Model for CIFAR-10
:::

::: {.cell .code}
``` python

resnet18_gradcam_cifar10 = ResNet18(num_classes=10)
resnet18_gradcam_cifar10.load_state_dict(torch.load(current_path + "checkpoints/resnet18_cifar10.pt"))
resnet18_gradcam_cifar10.eval()

resnet18_gradcam_cifar10_cutout = ResNet18(num_classes=10)
resnet18_gradcam_cifar10_cutout.load_state_dict(torch.load(current_path + "checkpoints/resnet18_cifar10_cutout.pt"))
resnet18_gradcam_cifar10_cutout.eval()

resnet18_gradcam_cifar10_da = ResNet18(num_classes=10)
resnet18_gradcam_cifar10_da.load_state_dict(torch.load(current_path + "checkpoints/resnet18_cifar10_da.pt"))
resnet18_gradcam_cifar10_da.eval()

resnet18_gradcam_cifar10_da_cutout = ResNet18(num_classes=10)
resnet18_gradcam_cifar10_da_cutout.load_state_dict(torch.load(current_path + "checkpoints/resnet18_cifar10_da_cutout.pt"))
resnet18_gradcam_cifar10_da_cutout.eval()
```
:::

::: {.cell .markdown}
Let's try to see the result from the testloader of CIFAR-10 dataset
:::

::: {.cell .code}
``` python
transform_cifar10 = transforms.Compose([
    transforms.Resize((32, 32)),
    transforms.ToTensor(),
    transforms.Normalize(mean=[0.485, 0.456, 0.406], std=[0.229, 0.224, 0.225])
])

testset_cifar10 = datasets.CIFAR10(root='./data', train=False, download=True, transform=transform_cifar10)
testloader_cifar10 = torch.utils.data.DataLoader(testset_cifar10, batch_size=1, shuffle=True, num_workers=2)

```
:::

::: {.cell .code}
``` python
cifar10_classes = [
    "Airplane", "Automobile", "Bird", "Cat", "Deer",
    "Dog", "Frog", "Horse", "Ship", "Truck"
]
```
:::

::: {.cell .code}
``` python
# Get a batch from the testloader
images, labels = next(iter(testloader_cifar10))
input_tensor = images  # As your batch_size is 1, you will have a single image here

# Forward pass
resnet18_gradcam_cifar10.zero_grad()
output_resnet18_gradcam_cifar10 = resnet18_gradcam_cifar10(input_tensor)

resnet18_gradcam_cifar10_cutout.zero_grad()
output_resnet18_gradcam_cifar10_cutout = resnet18_gradcam_cifar10_cutout(input_tensor)

resnet18_gradcam_cifar10_da.zero_grad()
output_resnet18_gradcam_cifar10_da = resnet18_gradcam_cifar10_da(input_tensor)

resnet18_gradcam_cifar10_da_cutout.zero_grad()
output_resnet18_gradcam_cifar10_da_cutout = resnet18_gradcam_cifar10_da_cutout(input_tensor)

# Get the index of the max log-probability
target_resnet18_gradcam_cifar10 = output_resnet18_gradcam_cifar10.argmax(1)
output_resnet18_gradcam_cifar10.max().backward()

target_resnet18_gradcam_cifar10_cutout = output_resnet18_gradcam_cifar10_cutout.argmax(1)
output_resnet18_gradcam_cifar10_cutout.max().backward()

target_resnet18_gradcam_cifar10_da = output_resnet18_gradcam_cifar10_da.argmax(1)
output_resnet18_gradcam_cifar10_da.max().backward()

target_resnet18_gradcam_cifar10_da_cutout = output_resnet18_gradcam_cifar10_da_cutout.argmax(1)
output_resnet18_gradcam_cifar10_da_cutout.max().backward()

# Map the predicted class indices to the class labels
predicted_class_resnet18_gradcam_cifar10 = cifar10_classes[target_resnet18_gradcam_cifar10.item()]
predicted_class_resnet18_gradcam_cifar10 = cifar10_classes[target_resnet18_gradcam_cifar10_cutout.item()]
predicted_class_resnet18_gradcam_cifar10_da = cifar10_classes[target_resnet18_gradcam_cifar10_da.item()]
predicted_class_resnet18_gradcam_cifar10_da_cutout = cifar10_classes[target_resnet18_gradcam_cifar10_da_cutout.item()]


# Get the gradients and activations
gradients_resnet18_gradcam_cifar10 = resnet18_gradcam_cifar10.gradients.detach().cpu()
activations_resnet18_gradcam_cifar10 = resnet18_gradcam_cifar10.activations.detach().cpu()

gradients_resnet18_gradcam_cifar10_cutout = resnet18_gradcam_cifar10_cutout.gradients.detach().cpu()
activations_resnet18_gradcam_cifar10_cutout = resnet18_gradcam_cifar10_cutout.activations.detach().cpu()

gradients_resnet18_gradcam_cifar10_da = resnet18_gradcam_cifar10_da.gradients.detach().cpu()
activations_resnet18_gradcam_cifar10_da = resnet18_gradcam_cifar10_da.activations.detach().cpu()

gradients_resnet18_gradcam_cifar10_da_cutout = resnet18_gradcam_cifar10_da_cutout.gradients.detach().cpu()
activations_resnet18_gradcam_cifar10_da_cutout = resnet18_gradcam_cifar10_da_cutout.activations.detach().cpu()


# Calculate the weights
weights_resnet18_gradcam_cifar10 = gradients_resnet18_gradcam_cifar10.mean(dim=(2, 3), keepdim=True)

weights_resnet18_gradcam_cifar10_cutout = gradients_resnet18_gradcam_cifar10_cutout.mean(dim=(2, 3), keepdim=True)

weights_resnet18_gradcam_cifar10_da = gradients_resnet18_gradcam_cifar10_da.mean(dim=(2, 3), keepdim=True)

weights_resnet18_gradcam_cifar10_da_cutout = gradients_resnet18_gradcam_cifar10_da_cutout.mean(dim=(2, 3), keepdim=True)

# Calculate the weighted sum of activations (Grad-CAM)
cam_resnet18_gradcam_cifar10 = (weights_resnet18_gradcam_cifar10 * activations_resnet18_gradcam_cifar10).sum(dim=1, keepdim=True)
cam_resnet18_gradcam_cifar10 = F.relu(cam_resnet18_gradcam_cifar10)  # apply ReLU to the heatmap
cam_resnet18_gradcam_cifar10 = F.interpolate(cam_resnet18_gradcam_cifar10, size=(32, 32), mode='bilinear', align_corners=False)
cam_resnet18_gradcam_cifar10 = cam_resnet18_gradcam_cifar10.squeeze().numpy()

cam_resnet18_gradcam_cifar10_cutout = (weights_resnet18_gradcam_cifar10_cutout * activations_resnet18_gradcam_cifar10_cutout).sum(dim=1, keepdim=True)
cam_resnet18_gradcam_cifar10_cutout = F.relu(cam_resnet18_gradcam_cifar10_cutout)  # apply ReLU to the heatmap
cam_resnet18_gradcam_cifar10_cutout = F.interpolate(cam_resnet18_gradcam_cifar10_cutout, size=(32, 32), mode='bilinear', align_corners=False)
cam_resnet18_gradcam_cifar10_cutout = cam_resnet18_gradcam_cifar10_cutout.squeeze().numpy()

cam_resnet18_gradcam_cifar10_da = (weights_resnet18_gradcam_cifar10_da * activations_resnet18_gradcam_cifar10_da).sum(dim=1, keepdim=True)
cam_resnet18_gradcam_cifar10_da = F.relu(cam_resnet18_gradcam_cifar10_da)  # apply ReLU to the heatmap
cam_resnet18_gradcam_cifar10_da = F.interpolate(cam_resnet18_gradcam_cifar10_da, size=(32, 32), mode='bilinear', align_corners=False)
cam_resnet18_gradcam_cifar10_da = cam_resnet18_gradcam_cifar10_da.squeeze().numpy()

cam_resnet18_gradcam_cifar10_da_cutout = (weights_resnet18_gradcam_cifar10_da_cutout * activations_resnet18_gradcam_cifar10_da_cutout).sum(dim=1, keepdim=True)
cam_resnet18_gradcam_cifar10_da_cutout = F.relu(cam_resnet18_gradcam_cifar10_da_cutout)  # apply ReLU to the heatmap
cam_resnet18_gradcam_cifar10_da_cutout = F.interpolate(cam_resnet18_gradcam_cifar10_da_cutout, size=(32, 32), mode='bilinear', align_corners=False)
cam_resnet18_gradcam_cifar10_da_cutout = cam_resnet18_gradcam_cifar10_da_cutout.squeeze().numpy()


# Normalize the heatmap
cam_resnet18_gradcam_cifar10 -= cam_resnet18_gradcam_cifar10.min()
cam_resnet18_gradcam_cifar10 /= cam_resnet18_gradcam_cifar10.max()

cam_resnet18_gradcam_cifar10_cutout -= cam_resnet18_gradcam_cifar10_cutout.min()
cam_resnet18_gradcam_cifar10_cutout /= cam_resnet18_gradcam_cifar10_cutout.max()

cam_resnet18_gradcam_cifar10_da -= cam_resnet18_gradcam_cifar10_da.min()
cam_resnet18_gradcam_cifar10_da /= cam_resnet18_gradcam_cifar10_da.max()

cam_resnet18_gradcam_cifar10_da_cutout -= cam_resnet18_gradcam_cifar10_da_cutout.min()
cam_resnet18_gradcam_cifar10_da_cutout /= cam_resnet18_gradcam_cifar10_da_cutout.max()

# Since the images from the dataloader are normalized, you have to denormalize them before plotting
mean = torch.tensor([0.485, 0.456, 0.406])
std = torch.tensor([0.229, 0.224, 0.225])
img = images.squeeze().detach().cpu() * std[..., None, None] + mean[..., None, None]
img = img.permute(1, 2, 0).numpy()

# Superimpose the heatmap onto the original image
heatmap_resnet18_gradcam_cifar10 = cv2.applyColorMap(np.uint8(255 * cam_resnet18_gradcam_cifar10), cv2.COLORMAP_JET)
heatmap_resnet18_gradcam_cifar10 = cv2.cvtColor(heatmap_resnet18_gradcam_cifar10, cv2.COLOR_BGR2RGB)
superimposed_img_resnet18_gradcam_cifar10 = heatmap_resnet18_gradcam_cifar10 * 0.4 + img * 255

heatmap_resnet18_gradcam_cifar10_cutout = cv2.applyColorMap(np.uint8(255 * cam_resnet18_gradcam_cifar10_cutout), cv2.COLORMAP_JET)
heatmap_resnet18_gradcam_cifar10_cutout = cv2.cvtColor(heatmap_resnet18_gradcam_cifar10_cutout, cv2.COLOR_BGR2RGB)
superimposed_img_resnet18_gradcam_cifar10_cutout = heatmap_resnet18_gradcam_cifar10_cutout * 0.4 + img * 255

heatmap_resnet18_gradcam_cifar10_da = cv2.applyColorMap(np.uint8(255 * cam_resnet18_gradcam_cifar10_da), cv2.COLORMAP_JET)
heatmap_resnet18_gradcam_cifar10_da = cv2.cvtColor(heatmap_resnet18_gradcam_cifar10_da, cv2.COLOR_BGR2RGB)
superimposed_img_resnet18_gradcam_cifar10_da = heatmap_resnet18_gradcam_cifar10_da * 0.4 + img * 255

heatmap_resnet18_gradcam_cifar10_da_cutout = cv2.applyColorMap(np.uint8(255 * cam_resnet18_gradcam_cifar10_da_cutout), cv2.COLORMAP_JET)
heatmap_resnet18_gradcam_cifar10_da_cutout = cv2.cvtColor(heatmap_resnet18_gradcam_cifar10_da_cutout, cv2.COLOR_BGR2RGB)
superimposed_img_resnet18_gradcam_cifar10_da_cutout = heatmap_resnet18_gradcam_cifar10_da_cutout * 0.4 + img * 255

class_label = str(labels.item())

# Display the original image and the Grad-CAM
fig, ax = plt.subplots(nrows=1, ncols=5, figsize=(20, 4))

ax[0].imshow(img)
ax[0].set_title('(Class: ' + cifar10_classes[int(class_label)] + ')')
ax[0].axis('off')
ax[1].imshow(superimposed_img_resnet18_gradcam_cifar10 / 255)
ax[1].set_title('Pred: ' +predicted_class_resnet18_gradcam_cifar10)
ax[1].axis('off')
ax[2].imshow(superimposed_img_resnet18_gradcam_cifar10_cutout / 255)
ax[2].set_title('Pred: ' +predicted_class_resnet18_gradcam_cifar10)
ax[2].axis('off')
ax[3].imshow(superimposed_img_resnet18_gradcam_cifar10_da / 255)
ax[3].set_title'Pred: ' + (predicted_class_resnet18_gradcam_cifar10_da)
ax[3].axis('off')
ax[4].imshow(superimposed_img_resnet18_gradcam_cifar10_da_cutout / 255)
ax[4].set_title('Pred: ' + predicted_class_resnet18_gradcam_cifar10_da_cutout)
ax[4].axis('off')

# Adjust the vertical position of the suptitle to bring it closer to the subplots.
fig.suptitle("Original Image - Grad-CAM -  Grad-CAM with Cutout - Grad-CAM with Data Augmentation - Grad-CAM with Cutout & Data Augmentation", y=1.1) 

# Automatically adjust subplot params for better layout.
plt.tight_layout()

plt.show()


```
:::

::: {.cell .markdown}
Now you can try to load your image, preprocess it and convert it into a PyTorch tensor. Choose an image that is in the CIFAR-10 classes (airplanes, cars, birds, cats, deer, dogs, frogs, horses, ships, and trucks). The preprocessing steps should be the same as the ones you used for training your model. Let’s say you have an image `image.jpeg`:
:::

::: {.cell .code}
```python
# Load the image
image_path = "image.jpeg"
image = Image.open(current_path + image_path)

# Define the transformations: resize, to tensor, normalize (replace the mean and std with values you used for training)
preprocess = transforms.Compose([
    transforms.Resize((32, 32)),
    transforms.ToTensor(),
    transforms.Normalize(mean=[0.485, 0.456, 0.406], std=[0.229, 0.224, 0.225])
])

# Preprocess the image
input_tensor = preprocess(image)
input_tensor = input_tensor.unsqueeze(0)  # add batch dimension.  C,H,W => B,C,H,W
```
:::


::: {.cell .markdown}
Apply Grad-CAM
:::

::: {.cell .code}
```python
# Forward pass
resnet18_gradcam_cifar10.zero_grad()
output_resnet18_gradcam_cifar10 = resnet18_gradcam_cifar10(input_tensor)

resnet18_gradcam_cifar10_cutout.zero_grad()
output_resnet18_gradcam_cifar10_cutout = resnet18_gradcam_cifar10_cutout(input_tensor)

resnet18_gradcam_cifar10_da.zero_grad()
output_resnet18_gradcam_cifar10_da = resnet18_gradcam_cifar10_da(input_tensor)

resnet18_gradcam_cifar10_da_cutout.zero_grad()
output_resnet18_gradcam_cifar10_da_cutout = resnet18_gradcam_cifar10_da_cutout(input_tensor)

# Get the index of the max log-probability
target_resnet18_gradcam_cifar10 = output_resnet18_gradcam_cifar10.argmax(1)
output_resnet18_gradcam_cifar10.max().backward()

target_resnet18_gradcam_cifar10_cutout = output_resnet18_gradcam_cifar10_cutout.argmax(1)
output_resnet18_gradcam_cifar10_cutout.max().backward()

target_resnet18_gradcam_cifar10_da = output_resnet18_gradcam_cifar10_da.argmax(1)
output_resnet18_gradcam_cifar10_da.max().backward()

target_resnet18_gradcam_cifar10_da_cutout = output_resnet18_gradcam_cifar10_da_cutout.argmax(1)
output_resnet18_gradcam_cifar10_da_cutout.max().backward()

# Map the predicted class indices to the class labels
predicted_class_resnet18_gradcam_cifar10 = cifar10_classes[target_resnet18_gradcam_cifar10.item()]
predicted_class_resnet18_gradcam_cifar10 = cifar10_classes[target_resnet18_gradcam_cifar10_cutout.item()]
predicted_class_resnet18_gradcam_cifar10_da = cifar10_classes[target_resnet18_gradcam_cifar10_da.item()]
predicted_class_resnet18_gradcam_cifar10_da_cutout = cifar10_classes[target_resnet18_gradcam_cifar10_da_cutout.item()]


# Get the gradients and activations
gradients_resnet18_gradcam_cifar10 = resnet18_gradcam_cifar10.gradients.detach().cpu()
activations_resnet18_gradcam_cifar10 = resnet18_gradcam_cifar10.activations.detach().cpu()

gradients_resnet18_gradcam_cifar10_cutout = resnet18_gradcam_cifar10_cutout.gradients.detach().cpu()
activations_resnet18_gradcam_cifar10_cutout = resnet18_gradcam_cifar10_cutout.activations.detach().cpu()

gradients_resnet18_gradcam_cifar10_da = resnet18_gradcam_cifar10_da.gradients.detach().cpu()
activations_resnet18_gradcam_cifar10_da = resnet18_gradcam_cifar10_da.activations.detach().cpu()

gradients_resnet18_gradcam_cifar10_da_cutout = resnet18_gradcam_cifar10_da_cutout.gradients.detach().cpu()
activations_resnet18_gradcam_cifar10_da_cutout = resnet18_gradcam_cifar10_da_cutout.activations.detach().cpu()


# Calculate the weights
weights_resnet18_gradcam_cifar10 = gradients_resnet18_gradcam_cifar10.mean(dim=(2, 3), keepdim=True)

weights_resnet18_gradcam_cifar10_cutout = gradients_resnet18_gradcam_cifar10_cutout.mean(dim=(2, 3), keepdim=True)

weights_resnet18_gradcam_cifar10_da = gradients_resnet18_gradcam_cifar10_da.mean(dim=(2, 3), keepdim=True)

weights_resnet18_gradcam_cifar10_da_cutout = gradients_resnet18_gradcam_cifar10_da_cutout.mean(dim=(2, 3), keepdim=True)

# Calculate the weighted sum of activations (Grad-CAM)
cam_resnet18_gradcam_cifar10 = (weights_resnet18_gradcam_cifar10 * activations_resnet18_gradcam_cifar10).sum(dim=1, keepdim=True)
cam_resnet18_gradcam_cifar10 = F.relu(cam_resnet18_gradcam_cifar10)  # apply ReLU to the heatmap
cam_resnet18_gradcam_cifar10 = F.interpolate(cam_resnet18_gradcam_cifar10, size=(32, 32), mode='bilinear', align_corners=False)
cam_resnet18_gradcam_cifar10 = cam_resnet18_gradcam_cifar10.squeeze().numpy()

cam_resnet18_gradcam_cifar10_cutout = (weights_resnet18_gradcam_cifar10_cutout * activations_resnet18_gradcam_cifar10_cutout).sum(dim=1, keepdim=True)
cam_resnet18_gradcam_cifar10_cutout = F.relu(cam_resnet18_gradcam_cifar10_cutout)  # apply ReLU to the heatmap
cam_resnet18_gradcam_cifar10_cutout = F.interpolate(cam_resnet18_gradcam_cifar10_cutout, size=(32, 32), mode='bilinear', align_corners=False)
cam_resnet18_gradcam_cifar10_cutout = cam_resnet18_gradcam_cifar10_cutout.squeeze().numpy()

cam_resnet18_gradcam_cifar10_da = (weights_resnet18_gradcam_cifar10_da * activations_resnet18_gradcam_cifar10_da).sum(dim=1, keepdim=True)
cam_resnet18_gradcam_cifar10_da = F.relu(cam_resnet18_gradcam_cifar10_da)  # apply ReLU to the heatmap
cam_resnet18_gradcam_cifar10_da = F.interpolate(cam_resnet18_gradcam_cifar10_da, size=(32, 32), mode='bilinear', align_corners=False)
cam_resnet18_gradcam_cifar10_da = cam_resnet18_gradcam_cifar10_da.squeeze().numpy()

cam_resnet18_gradcam_cifar10_da_cutout = (weights_resnet18_gradcam_cifar10_da_cutout * activations_resnet18_gradcam_cifar10_da_cutout).sum(dim=1, keepdim=True)
cam_resnet18_gradcam_cifar10_da_cutout = F.relu(cam_resnet18_gradcam_cifar10_da_cutout)  # apply ReLU to the heatmap
cam_resnet18_gradcam_cifar10_da_cutout = F.interpolate(cam_resnet18_gradcam_cifar10_da_cutout, size=(32, 32), mode='bilinear', align_corners=False)
cam_resnet18_gradcam_cifar10_da_cutout = cam_resnet18_gradcam_cifar10_da_cutout.squeeze().numpy()


# Normalize the heatmap
cam_resnet18_gradcam_cifar10 -= cam_resnet18_gradcam_cifar10.min()
cam_resnet18_gradcam_cifar10 /= cam_resnet18_gradcam_cifar10.max()

cam_resnet18_gradcam_cifar10_cutout -= cam_resnet18_gradcam_cifar10_cutout.min()
cam_resnet18_gradcam_cifar10_cutout /= cam_resnet18_gradcam_cifar10_cutout.max()

cam_resnet18_gradcam_cifar10_da -= cam_resnet18_gradcam_cifar10_da.min()
cam_resnet18_gradcam_cifar10_da /= cam_resnet18_gradcam_cifar10_da.max()

cam_resnet18_gradcam_cifar10_da_cutout -= cam_resnet18_gradcam_cifar10_da_cutout.min()
cam_resnet18_gradcam_cifar10_da_cutout /= cam_resnet18_gradcam_cifar10_da_cutout.max()

# Since the images from the dataloader are normalized, you have to denormalize them before plotting
img = cv2.imread(current_path + image_path)
img = cv2.resize(img, (32, 32))
img = cv2.cvtColor(img, cv2.COLOR_BGR2RGB)


# Superimpose the heatmap onto the original image
heatmap_resnet18_gradcam_cifar10 = cv2.applyColorMap(np.uint8(255 * cam_resnet18_gradcam_cifar10), cv2.COLORMAP_JET)
heatmap_resnet18_gradcam_cifar10 = cv2.cvtColor(heatmap_resnet18_gradcam_cifar10, cv2.COLOR_BGR2RGB)
superimposed_img_resnet18_gradcam_cifar10 = heatmap_resnet18_gradcam_cifar10 * 0.4 + img * 255

heatmap_resnet18_gradcam_cifar10_cutout = cv2.applyColorMap(np.uint8(255 * cam_resnet18_gradcam_cifar10_cutout), cv2.COLORMAP_JET)
heatmap_resnet18_gradcam_cifar10_cutout = cv2.cvtColor(heatmap_resnet18_gradcam_cifar10_cutout, cv2.COLOR_BGR2RGB)
superimposed_img_resnet18_gradcam_cifar10_cutout = heatmap_resnet18_gradcam_cifar10_cutout * 0.4 + img * 255

heatmap_resnet18_gradcam_cifar10_da = cv2.applyColorMap(np.uint8(255 * cam_resnet18_gradcam_cifar10_da), cv2.COLORMAP_JET)
heatmap_resnet18_gradcam_cifar10_da = cv2.cvtColor(heatmap_resnet18_gradcam_cifar10_da, cv2.COLOR_BGR2RGB)
superimposed_img_resnet18_gradcam_cifar10_da = heatmap_resnet18_gradcam_cifar10_da * 0.4 + img * 255

heatmap_resnet18_gradcam_cifar10_da_cutout = cv2.applyColorMap(np.uint8(255 * cam_resnet18_gradcam_cifar10_da_cutout), cv2.COLORMAP_JET)
heatmap_resnet18_gradcam_cifar10_da_cutout = cv2.cvtColor(heatmap_resnet18_gradcam_cifar10_da_cutout, cv2.COLOR_BGR2RGB)
superimposed_img_resnet18_gradcam_cifar10_da_cutout = heatmap_resnet18_gradcam_cifar10_da_cutout * 0.4 + img * 255

class_label = str(labels.item())
```
:::

::: {.cell .markdown}
Visualize the image and the Grad-CAM heatmap
:::

::: {.cell .code}
```python
# Display the original image and the Grad-CAM
fig, ax = plt.subplots(nrows=1, ncols=5, figsize=(20, 4))

ax[0].imshow(img)
ax[0].set_title('(Class: ' + cifar10_classes[int(class_label)] + ')')
ax[0].axis('off')
ax[1].imshow(superimposed_img_resnet18_gradcam_cifar10 / 255)
ax[1].set_title(predicted_class_resnet18_gradcam_cifar10)
ax[1].axis('off')
ax[2].imshow(superimposed_img_resnet18_gradcam_cifar10_cutout / 255)
ax[2].set_title(predicted_class_resnet18_gradcam_cifar10)
ax[2].axis('off')
ax[3].imshow(superimposed_img_resnet18_gradcam_cifar10_da / 255)
ax[3].set_title(predicted_class_resnet18_gradcam_cifar10_da)
ax[3].axis('off')
ax[4].imshow(superimposed_img_resnet18_gradcam_cifar10_da_cutout / 255)
ax[4].set_title(predicted_class_resnet18_gradcam_cifar10_da_cutout)
ax[4].axis('off')

# Adjust the vertical position of the suptitle to bring it closer to the subplots.
fig.suptitle("Original Image - Grad-CAM -  Grad-CAM with Cutout - Grad-CAM with Data Augmentation - Grad-CAM with Cutout & Data Augmentation", y=1.1) 

# Automatically adjust subplot params for better layout.
plt.tight_layout()

plt.show()
```
:::


::: {.cell .markdown}
## 4.2.2 Implementation Grad-CAM for ResNet18 Model for CIFAR-100
:::

::: {.cell .code}
``` python

resnet18_gradcam_cifar100 = ResNet18(num_classes=100)
resnet18_gradcam_cifar100.load_state_dict(torch.load(current_path + "checkpoints/resnet18_cifar100.pt"))
resnet18_gradcam_cifar100.eval()

resnet18_gradcam_cifar100_cutout = ResNet18(num_classes=100)
resnet18_gradcam_cifar100_cutout.load_state_dict(torch.load(current_path + "checkpoints/resnet18_cifar100_cutout.pt"))
resnet18_gradcam_cifar100_cutout.eval()

resnet18_gradcam_cifar100_da = ResNet18(num_classes=100)
resnet18_gradcam_cifar100_da.load_state_dict(torch.load(current_path + "checkpoints/resnet18_cifar100_da.pt"))
resnet18_gradcam_cifar100_da.eval()

resnet18_gradcam_cifar100_da_cutout = ResNet18(num_classes=100)
resnet18_gradcam_cifar100_da_cutout.load_state_dict(torch.load(current_path + "checkpoints/resnet18_cifar100_da_cutout.pt"))
resnet18_gradcam_cifar100_da_cutout.eval()
```
:::

::: {.cell .markdown}
Let's try to see the result from the testloader of CIFAR-10 dataset
:::

::: {.cell .code}
``` python
transform_cifar100 = transforms.Compose([
    transforms.Resize((32, 32)),
    transforms.ToTensor(),
    transforms.Normalize(mean=[0.485, 0.456, 0.406], std=[0.229, 0.224, 0.225])
])

testset_cifar100 = datasets.CIFAR100(root='./data', train=False, download=True, transform=transform_cifar100)
testloader_cifar100 = torch.utils.data.DataLoader(testset_cifar100, batch_size=1, shuffle=True, num_workers=2)

```
:::

::: {.cell .code}
``` python
cifar100_classes = [
    "Apple", "Aquarium fish", "Baby", "Bear", "Beaver", "Bed", "Bee", "Beetle", 
    "Bicycle", "Bottle", "Bowl", "Boy", "Bridge", "Bus", "Butterfly", "Camel", 
    "Can", "Castle", "Caterpillar", "Cattle", "Chair", "Chimpanzee", "Clock", 
    "Cloud", "Cockroach", "Couch", "Crab", "Crocodile", "Cup", "Dinosaur", 
    "Dolphin", "Elephant", "Flatfish", "Forest", "Fox", "Girl", "Hamster", 
    "House", "Kangaroo", "Computer keyboard", "Lamp", "Lawn-mower", "Leopard", "Lion",
    "Lizard", "Lobster", "Man", "Maple tree", "Motorcycle", "Mountain", "Mouse",
    "Mushrooms", "Oak tree", "Oranges", "Orchids", "Otter", "Palm tree", "Pears",
    "Pickup truck", "Pine tree", "Plain", "Plates", "Poppies", "Porcupine",
    "Possum", "Rabbit", "Raccoon", "Ray", "Road", "Rocket", "Roses", "Sea", "Seal",
    "Shark", "Shrew", "Skunk", "Skyscraper", "Snail", "Snake", "Spider", "Squirrel",
    "Streetcar", "Sunflowers", "Sweet peppers", "Table", "Tank", "Telephone", "Television", 
    "Tiger", "Tractor", "Train", "Trout", "Tulips", "Turtle", "Wardrobe", "Whale", 
    "Willow tree", "Wolf", "Woman", "Worm"
]

```
:::

::: {.cell .code}
``` python
# Get a batch from the testloader
images, labels = next(iter(testloader_cifar100))
input_tensor = images  # As your batch_size is 1, you will have a single image here

# Forward pass
resnet18_gradcam_cifar100.zero_grad()
output_resnet18_gradcam_cifar100 = resnet18_gradcam_cifar100(input_tensor)

resnet18_gradcam_cifar100_cutout.zero_grad()
output_resnet18_gradcam_cifar100_cutout = resnet18_gradcam_cifar100_cutout(input_tensor)

resnet18_gradcam_cifar100_da.zero_grad()
output_resnet18_gradcam_cifar100_da = resnet18_gradcam_cifar100_da(input_tensor)

resnet18_gradcam_cifar100_da_cutout.zero_grad()
output_resnet18_gradcam_cifar100_da_cutout = resnet18_gradcam_cifar100_da_cutout(input_tensor)

# Get the index of the max log-probability
target_resnet18_gradcam_cifar100 = output_resnet18_gradcam_cifar100.argmax(1)
output_resnet18_gradcam_cifar100.max().backward()

target_resnet18_gradcam_cifar100_cutout = output_resnet18_gradcam_cifar100_cutout.argmax(1)
output_resnet18_gradcam_cifar100_cutout.max().backward()

target_resnet18_gradcam_cifar100_da = output_resnet18_gradcam_cifar100_da.argmax(1)
output_resnet18_gradcam_cifar100_da.max().backward()

target_resnet18_gradcam_cifar100_da_cutout = output_resnet18_gradcam_cifar100_da_cutout.argmax(1)
output_resnet18_gradcam_cifar100_da_cutout.max().backward()

# Map the predicted class indices to the class labels
predicted_class_resnet18_gradcam_cifar100 = cifar100_classes[target_resnet18_gradcam_cifar100.item()]
predicted_class_resnet18_gradcam_cifar100 = cifar100_classes[target_resnet18_gradcam_cifar100_cutout.item()]
predicted_class_resnet18_gradcam_cifar100_da = cifar100_classes[target_resnet18_gradcam_cifar100_da.item()]
predicted_class_resnet18_gradcam_cifar100_da_cutout = cifar100_classes[target_resnet18_gradcam_cifar100_da_cutout.item()]


# Get the gradients and activations
gradients_resnet18_gradcam_cifar100 = resnet18_gradcam_cifar100.gradients.detach().cpu()
activations_resnet18_gradcam_cifar100 = resnet18_gradcam_cifar100.activations.detach().cpu()

gradients_resnet18_gradcam_cifar100_cutout = resnet18_gradcam_cifar100_cutout.gradients.detach().cpu()
activations_resnet18_gradcam_cifar100_cutout = resnet18_gradcam_cifar100_cutout.activations.detach().cpu()

gradients_resnet18_gradcam_cifar100_da = resnet18_gradcam_cifar100_da.gradients.detach().cpu()
activations_resnet18_gradcam_cifar100_da = resnet18_gradcam_cifar100_da.activations.detach().cpu()

gradients_resnet18_gradcam_cifar100_da_cutout = resnet18_gradcam_cifar100_da_cutout.gradients.detach().cpu()
activations_resnet18_gradcam_cifar100_da_cutout = resnet18_gradcam_cifar100_da_cutout.activations.detach().cpu()


# Calculate the weights
weights_resnet18_gradcam_cifar100 = gradients_resnet18_gradcam_cifar100.mean(dim=(2, 3), keepdim=True)

weights_resnet18_gradcam_cifar100_cutout = gradients_resnet18_gradcam_cifar100_cutout.mean(dim=(2, 3), keepdim=True)

weights_resnet18_gradcam_cifar100_da = gradients_resnet18_gradcam_cifar100_da.mean(dim=(2, 3), keepdim=True)

weights_resnet18_gradcam_cifar100_da_cutout = gradients_resnet18_gradcam_cifar100_da_cutout.mean(dim=(2, 3), keepdim=True)

# Calculate the weighted sum of activations (Grad-CAM)
cam_resnet18_gradcam_cifar100 = (weights_resnet18_gradcam_cifar100 * activations_resnet18_gradcam_cifar100).sum(dim=1, keepdim=True)
cam_resnet18_gradcam_cifar100 = F.relu(cam_resnet18_gradcam_cifar100)  # apply ReLU to the heatmap
cam_resnet18_gradcam_cifar100 = F.interpolate(cam_resnet18_gradcam_cifar100, size=(32, 32), mode='bilinear', align_corners=False)
cam_resnet18_gradcam_cifar100 = cam_resnet18_gradcam_cifar100.squeeze().numpy()

cam_resnet18_gradcam_cifar100_cutout = (weights_resnet18_gradcam_cifar100_cutout * activations_resnet18_gradcam_cifar100_cutout).sum(dim=1, keepdim=True)
cam_resnet18_gradcam_cifar100_cutout = F.relu(cam_resnet18_gradcam_cifar100_cutout)  # apply ReLU to the heatmap
cam_resnet18_gradcam_cifar100_cutout = F.interpolate(cam_resnet18_gradcam_cifar100_cutout, size=(32, 32), mode='bilinear', align_corners=False)
cam_resnet18_gradcam_cifar100_cutout = cam_resnet18_gradcam_cifar100_cutout.squeeze().numpy()

cam_resnet18_gradcam_cifar100_da = (weights_resnet18_gradcam_cifar100_da * activations_resnet18_gradcam_cifar100_da).sum(dim=1, keepdim=True)
cam_resnet18_gradcam_cifar100_da = F.relu(cam_resnet18_gradcam_cifar100_da)  # apply ReLU to the heatmap
cam_resnet18_gradcam_cifar100_da = F.interpolate(cam_resnet18_gradcam_cifar100_da, size=(32, 32), mode='bilinear', align_corners=False)
cam_resnet18_gradcam_cifar100_da = cam_resnet18_gradcam_cifar100_da.squeeze().numpy()

cam_resnet18_gradcam_cifar100_da_cutout = (weights_resnet18_gradcam_cifar100_da_cutout * activations_resnet18_gradcam_cifar100_da_cutout).sum(dim=1, keepdim=True)
cam_resnet18_gradcam_cifar100_da_cutout = F.relu(cam_resnet18_gradcam_cifar100_da_cutout)  # apply ReLU to the heatmap
cam_resnet18_gradcam_cifar100_da_cutout = F.interpolate(cam_resnet18_gradcam_cifar100_da_cutout, size=(32, 32), mode='bilinear', align_corners=False)
cam_resnet18_gradcam_cifar100_da_cutout = cam_resnet18_gradcam_cifar100_da_cutout.squeeze().numpy()


# Normalize the heatmap
cam_resnet18_gradcam_cifar100 -= cam_resnet18_gradcam_cifar100.min()
cam_resnet18_gradcam_cifar100 /= cam_resnet18_gradcam_cifar100.max()

cam_resnet18_gradcam_cifar100_cutout -= cam_resnet18_gradcam_cifar100_cutout.min()
cam_resnet18_gradcam_cifar100_cutout /= cam_resnet18_gradcam_cifar100_cutout.max()

cam_resnet18_gradcam_cifar100_da -= cam_resnet18_gradcam_cifar100_da.min()
cam_resnet18_gradcam_cifar100_da /= cam_resnet18_gradcam_cifar100_da.max()

cam_resnet18_gradcam_cifar100_da_cutout -= cam_resnet18_gradcam_cifar100_da_cutout.min()
cam_resnet18_gradcam_cifar100_da_cutout /= cam_resnet18_gradcam_cifar100_da_cutout.max()

# Since the images from the dataloader are normalized, you have to denormalize them before plotting
mean = torch.tensor([0.485, 0.456, 0.406])
std = torch.tensor([0.229, 0.224, 0.225])
img = images.squeeze().detach().cpu() * std[..., None, None] + mean[..., None, None]
img = img.permute(1, 2, 0).numpy()

# Superimpose the heatmap onto the original image
heatmap_resnet18_gradcam_cifar100 = cv2.applyColorMap(np.uint8(255 * cam_resnet18_gradcam_cifar100), cv2.COLORMAP_JET)
heatmap_resnet18_gradcam_cifar100 = cv2.cvtColor(heatmap_resnet18_gradcam_cifar100, cv2.COLOR_BGR2RGB)
superimposed_img_resnet18_gradcam_cifar100 = heatmap_resnet18_gradcam_cifar100 * 0.4 + img * 255

heatmap_resnet18_gradcam_cifar100_cutout = cv2.applyColorMap(np.uint8(255 * cam_resnet18_gradcam_cifar100_cutout), cv2.COLORMAP_JET)
heatmap_resnet18_gradcam_cifar100_cutout = cv2.cvtColor(heatmap_resnet18_gradcam_cifar100_cutout, cv2.COLOR_BGR2RGB)
superimposed_img_resnet18_gradcam_cifar100_cutout = heatmap_resnet18_gradcam_cifar100_cutout * 0.4 + img * 255

heatmap_resnet18_gradcam_cifar100_da = cv2.applyColorMap(np.uint8(255 * cam_resnet18_gradcam_cifar100_da), cv2.COLORMAP_JET)
heatmap_resnet18_gradcam_cifar100_da = cv2.cvtColor(heatmap_resnet18_gradcam_cifar100_da, cv2.COLOR_BGR2RGB)
superimposed_img_resnet18_gradcam_cifar100_da = heatmap_resnet18_gradcam_cifar100_da * 0.4 + img * 255

heatmap_resnet18_gradcam_cifar100_da_cutout = cv2.applyColorMap(np.uint8(255 * cam_resnet18_gradcam_cifar100_da_cutout), cv2.COLORMAP_JET)
heatmap_resnet18_gradcam_cifar100_da_cutout = cv2.cvtColor(heatmap_resnet18_gradcam_cifar100_da_cutout, cv2.COLOR_BGR2RGB)
superimposed_img_resnet18_gradcam_cifar100_da_cutout = heatmap_resnet18_gradcam_cifar100_da_cutout * 0.4 + img * 255

class_label = str(labels.item())

# Display the original image and the Grad-CAM
fig, ax = plt.subplots(nrows=1, ncols=5, figsize=(20, 4))

ax[0].imshow(img)
ax[0].set_title('(Class: ' + cifar100_classes[int(class_label)] + ')')
ax[0].axis('off')
ax[1].imshow(superimposed_img_resnet18_gradcam_cifar100 / 255)
ax[1].set_title('Pred: ' +predicted_class_resnet18_gradcam_cifar100)
ax[1].axis('off')
ax[2].imshow(superimposed_img_resnet18_gradcam_cifar100_cutout / 255)
ax[2].set_title('Pred: ' +predicted_class_resnet18_gradcam_cifar100)
ax[2].axis('off')
ax[3].imshow(superimposed_img_resnet18_gradcam_cifar100_da / 255)
ax[3].set_title('Pred: ' +predicted_class_resnet18_gradcam_cifar100_da)
ax[3].axis('off')
ax[4].imshow(superimposed_img_resnet18_gradcam_cifar100_da_cutout / 255)
ax[4].set_title('Pred: ' +predicted_class_resnet18_gradcam_cifar100_da_cutout)
ax[4].axis('off')

# Adjust the vertical position of the suptitle to bring it closer to the subplots.
fig.suptitle("Original Image - Grad-CAM -  Grad-CAM with Cutout - Grad-CAM with Data Augmentation - Grad-CAM with Cutout & Data Augmentation", y=1.1) 

# Automatically adjust subplot params for better layout.
plt.tight_layout()

plt.show()


```
:::

::: {.cell .markdown}
Now you can try to load your image, preprocess it and convert it into a PyTorch tensor. Choose an image that is in the CIFAR-10 classes (airplanes, cars, birds, cats, deer, dogs, frogs, horses, ships, and trucks). The preprocessing steps should be the same as the ones you used for training your model. Let’s say you have an image `image.jpeg`:
:::

::: {.cell .code}
```python
# Load the image
image_path = "image.jpeg"
image = Image.open(current_path + image_path)

# Define the transformations: resize, to tensor, normalize (replace the mean and std with values you used for training)
preprocess = transforms.Compose([
    transforms.Resize((32, 32)),
    transforms.ToTensor(),
    transforms.Normalize(mean=[0.485, 0.456, 0.406], std=[0.229, 0.224, 0.225])
])

# Preprocess the image
input_tensor = preprocess(image)
input_tensor = input_tensor.unsqueeze(0)  # add batch dimension.  C,H,W => B,C,H,W
```
:::


::: {.cell .markdown}
Apply Grad-CAM
:::

::: {.cell .code}
```python
# Forward pass
resnet18_gradcam_cifar100.zero_grad()
output_resnet18_gradcam_cifar100 = resnet18_gradcam_cifar100(input_tensor)

resnet18_gradcam_cifar100_cutout.zero_grad()
output_resnet18_gradcam_cifar100_cutout = resnet18_gradcam_cifar100_cutout(input_tensor)

resnet18_gradcam_cifar100_da.zero_grad()
output_resnet18_gradcam_cifar100_da = resnet18_gradcam_cifar100_da(input_tensor)

resnet18_gradcam_cifar100_da_cutout.zero_grad()
output_resnet18_gradcam_cifar100_da_cutout = resnet18_gradcam_cifar100_da_cutout(input_tensor)

# Get the index of the max log-probability
target_resnet18_gradcam_cifar100 = output_resnet18_gradcam_cifar100.argmax(1)
output_resnet18_gradcam_cifar100.max().backward()

target_resnet18_gradcam_cifar100_cutout = output_resnet18_gradcam_cifar100_cutout.argmax(1)
output_resnet18_gradcam_cifar100_cutout.max().backward()

target_resnet18_gradcam_cifar100_da = output_resnet18_gradcam_cifar100_da.argmax(1)
output_resnet18_gradcam_cifar100_da.max().backward()

target_resnet18_gradcam_cifar100_da_cutout = output_resnet18_gradcam_cifar100_da_cutout.argmax(1)
output_resnet18_gradcam_cifar100_da_cutout.max().backward()

# Map the predicted class indices to the class labels
predicted_class_resnet18_gradcam_cifar100 = cifar100_classes[target_resnet18_gradcam_cifar100.item()]
predicted_class_resnet18_gradcam_cifar100 = cifar100_classes[target_resnet18_gradcam_cifar100_cutout.item()]
predicted_class_resnet18_gradcam_cifar100_da = cifar100_classes[target_resnet18_gradcam_cifar100_da.item()]
predicted_class_resnet18_gradcam_cifar100_da_cutout = cifar100_classes[target_resnet18_gradcam_cifar100_da_cutout.item()]


# Get the gradients and activations
gradients_resnet18_gradcam_cifar100 = resnet18_gradcam_cifar100.gradients.detach().cpu()
activations_resnet18_gradcam_cifar100 = resnet18_gradcam_cifar100.activations.detach().cpu()

gradients_resnet18_gradcam_cifar100_cutout = resnet18_gradcam_cifar100_cutout.gradients.detach().cpu()
activations_resnet18_gradcam_cifar100_cutout = resnet18_gradcam_cifar100_cutout.activations.detach().cpu()

gradients_resnet18_gradcam_cifar100_da = resnet18_gradcam_cifar100_da.gradients.detach().cpu()
activations_resnet18_gradcam_cifar100_da = resnet18_gradcam_cifar100_da.activations.detach().cpu()

gradients_resnet18_gradcam_cifar100_da_cutout = resnet18_gradcam_cifar100_da_cutout.gradients.detach().cpu()
activations_resnet18_gradcam_cifar100_da_cutout = resnet18_gradcam_cifar100_da_cutout.activations.detach().cpu()


# Calculate the weights
weights_resnet18_gradcam_cifar100 = gradients_resnet18_gradcam_cifar100.mean(dim=(2, 3), keepdim=True)

weights_resnet18_gradcam_cifar100_cutout = gradients_resnet18_gradcam_cifar100_cutout.mean(dim=(2, 3), keepdim=True)

weights_resnet18_gradcam_cifar100_da = gradients_resnet18_gradcam_cifar100_da.mean(dim=(2, 3), keepdim=True)

weights_resnet18_gradcam_cifar100_da_cutout = gradients_resnet18_gradcam_cifar100_da_cutout.mean(dim=(2, 3), keepdim=True)

# Calculate the weighted sum of activations (Grad-CAM)
cam_resnet18_gradcam_cifar100 = (weights_resnet18_gradcam_cifar100 * activations_resnet18_gradcam_cifar100).sum(dim=1, keepdim=True)
cam_resnet18_gradcam_cifar100 = F.relu(cam_resnet18_gradcam_cifar100)  # apply ReLU to the heatmap
cam_resnet18_gradcam_cifar100 = F.interpolate(cam_resnet18_gradcam_cifar100, size=(32, 32), mode='bilinear', align_corners=False)
cam_resnet18_gradcam_cifar100 = cam_resnet18_gradcam_cifar100.squeeze().numpy()

cam_resnet18_gradcam_cifar100_cutout = (weights_resnet18_gradcam_cifar100_cutout * activations_resnet18_gradcam_cifar100_cutout).sum(dim=1, keepdim=True)
cam_resnet18_gradcam_cifar100_cutout = F.relu(cam_resnet18_gradcam_cifar100_cutout)  # apply ReLU to the heatmap
cam_resnet18_gradcam_cifar100_cutout = F.interpolate(cam_resnet18_gradcam_cifar100_cutout, size=(32, 32), mode='bilinear', align_corners=False)
cam_resnet18_gradcam_cifar100_cutout = cam_resnet18_gradcam_cifar100_cutout.squeeze().numpy()

cam_resnet18_gradcam_cifar100_da = (weights_resnet18_gradcam_cifar100_da * activations_resnet18_gradcam_cifar100_da).sum(dim=1, keepdim=True)
cam_resnet18_gradcam_cifar100_da = F.relu(cam_resnet18_gradcam_cifar100_da)  # apply ReLU to the heatmap
cam_resnet18_gradcam_cifar100_da = F.interpolate(cam_resnet18_gradcam_cifar100_da, size=(32, 32), mode='bilinear', align_corners=False)
cam_resnet18_gradcam_cifar100_da = cam_resnet18_gradcam_cifar100_da.squeeze().numpy()

cam_resnet18_gradcam_cifar100_da_cutout = (weights_resnet18_gradcam_cifar100_da_cutout * activations_resnet18_gradcam_cifar100_da_cutout).sum(dim=1, keepdim=True)
cam_resnet18_gradcam_cifar100_da_cutout = F.relu(cam_resnet18_gradcam_cifar100_da_cutout)  # apply ReLU to the heatmap
cam_resnet18_gradcam_cifar100_da_cutout = F.interpolate(cam_resnet18_gradcam_cifar100_da_cutout, size=(32, 32), mode='bilinear', align_corners=False)
cam_resnet18_gradcam_cifar100_da_cutout = cam_resnet18_gradcam_cifar100_da_cutout.squeeze().numpy()


# Normalize the heatmap
cam_resnet18_gradcam_cifar100 -= cam_resnet18_gradcam_cifar100.min()
cam_resnet18_gradcam_cifar100 /= cam_resnet18_gradcam_cifar100.max()

cam_resnet18_gradcam_cifar100_cutout -= cam_resnet18_gradcam_cifar100_cutout.min()
cam_resnet18_gradcam_cifar100_cutout /= cam_resnet18_gradcam_cifar100_cutout.max()

cam_resnet18_gradcam_cifar100_da -= cam_resnet18_gradcam_cifar100_da.min()
cam_resnet18_gradcam_cifar100_da /= cam_resnet18_gradcam_cifar100_da.max()

cam_resnet18_gradcam_cifar100_da_cutout -= cam_resnet18_gradcam_cifar100_da_cutout.min()
cam_resnet18_gradcam_cifar100_da_cutout /= cam_resnet18_gradcam_cifar100_da_cutout.max()

# Since the images from the dataloader are normalized, you have to denormalize them before plotting
img = cv2.imread(current_path + image_path)
img = cv2.resize(img, (32, 32))
img = cv2.cvtColor(img, cv2.COLOR_BGR2RGB)


# Superimpose the heatmap onto the original image
heatmap_resnet18_gradcam_cifar100 = cv2.applyColorMap(np.uint8(255 * cam_resnet18_gradcam_cifar100), cv2.COLORMAP_JET)
heatmap_resnet18_gradcam_cifar100 = cv2.cvtColor(heatmap_resnet18_gradcam_cifar100, cv2.COLOR_BGR2RGB)
superimposed_img_resnet18_gradcam_cifar100 = heatmap_resnet18_gradcam_cifar100 * 0.4 + img * 255

heatmap_resnet18_gradcam_cifar100_cutout = cv2.applyColorMap(np.uint8(255 * cam_resnet18_gradcam_cifar100_cutout), cv2.COLORMAP_JET)
heatmap_resnet18_gradcam_cifar100_cutout = cv2.cvtColor(heatmap_resnet18_gradcam_cifar100_cutout, cv2.COLOR_BGR2RGB)
superimposed_img_resnet18_gradcam_cifar100_cutout = heatmap_resnet18_gradcam_cifar100_cutout * 0.4 + img * 255

heatmap_resnet18_gradcam_cifar100_da = cv2.applyColorMap(np.uint8(255 * cam_resnet18_gradcam_cifar100_da), cv2.COLORMAP_JET)
heatmap_resnet18_gradcam_cifar100_da = cv2.cvtColor(heatmap_resnet18_gradcam_cifar100_da, cv2.COLOR_BGR2RGB)
superimposed_img_resnet18_gradcam_cifar100_da = heatmap_resnet18_gradcam_cifar100_da * 0.4 + img * 255

heatmap_resnet18_gradcam_cifar100_da_cutout = cv2.applyColorMap(np.uint8(255 * cam_resnet18_gradcam_cifar100_da_cutout), cv2.COLORMAP_JET)
heatmap_resnet18_gradcam_cifar100_da_cutout = cv2.cvtColor(heatmap_resnet18_gradcam_cifar100_da_cutout, cv2.COLOR_BGR2RGB)
superimposed_img_resnet18_gradcam_cifar100_da_cutout = heatmap_resnet18_gradcam_cifar100_da_cutout * 0.4 + img * 255

class_label = str(labels.item())
```
:::

::: {.cell .markdown}
Visualize the image and the Grad-CAM heatmap
:::

::: {.cell .code}
```python
# Display the original image and the Grad-CAM
fig, ax = plt.subplots(nrows=1, ncols=5, figsize=(20, 4))

ax[0].imshow(img)
ax[0].set_title('(Class: ' + cifar100_classes[int(class_label)] + ')')
ax[0].axis('off')
ax[1].imshow(superimposed_img_resnet18_gradcam_cifar100 / 255)
ax[1].set_title('Pred: ' +predicted_class_resnet18_gradcam_cifar100)
ax[1].axis('off')
ax[2].imshow(superimposed_img_resnet18_gradcam_cifar100_cutout / 255)
ax[2].set_title('Pred: ' +predicted_class_resnet18_gradcam_cifar100)
ax[2].axis('off')
ax[3].imshow(superimposed_img_resnet18_gradcam_cifar100_da / 255)
ax[3].set_title('Pred: ' +predicted_class_resnet18_gradcam_cifar100_da)
ax[3].axis('off')
ax[4].imshow(superimposed_img_resnet18_gradcam_cifar100_da_cutout / 255)
ax[4].set_title('Pred: ' +predicted_class_resnet18_gradcam_cifar100_da_cutout)
ax[4].axis('off')

# Adjust the vertical position of the suptitle to bring it closer to the subplots.
fig.suptitle("Original Image - Grad-CAM -  Grad-CAM with Cutout - Grad-CAM with Data Augmentation - Grad-CAM with Cutout & Data Augmentation", y=1.1) 

# Automatically adjust subplot params for better layout.
plt.tight_layout()
plt.show()
```
:::


::: {.cell .code}
``` python

```
:::
::: {.cell .markdown}
## 4.3 Implementation Grad-CAM for WideResNet Model
:::


::: {.cell .markdown}
This is the code of the WideResNet Architecture and it is the similar to the code we introduced earlier in the section about WideResNet, but we added hook layers. Hook layers are inserted at specific layers of the model to capture the intermediate feature maps. These feature maps represent the learned activations of the model as it processes the input image. GRAD-CAM uses these activations to understand which parts of the image are contributing the most to the final prediction.
:::

::: {.cell .markdown}
### WideResNet Code
:::

::: {.cell .code}
``` python
# WideResNet

# From https://github.com/uoguelph-mlrg/Cutout/blob/master/model/wide_resnet.py

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

        # Register hooks for Grad-CAM
        self.gradients = None
        self.activations = None
        self.block3.register_forward_hook(self._store_activations_hook)
        self.block3.register_backward_hook(self._store_gradients_hook)

    def _store_activations_hook(self, module, input, output):
        self.activations = output

    def _store_gradients_hook(self, module, grad_input, grad_output):
        self.gradients = grad_output[0]

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


::: {.cell .markdown}
## 4.3.1 Implementation Grad-CAM for WideResNet Model for CIFAR-10
:::

::: {.cell .code}
``` python

wideresnet_gradcam_cifar10 = WideResNet(depth=28, num_classes=10, widen_factor=10, dropRate=0.3)
wideresnet_gradcam_cifar10.load_state_dict(torch.load(current_path + "checkpoints/wideresnet_cifar10.pt"))
wideresnet_gradcam_cifar10.eval()

wideresnet_gradcam_cifar10_cutout = WideResNet(depth=28, num_classes=10, widen_factor=10, dropRate=0.3)
wideresnet_gradcam_cifar10_cutout.load_state_dict(torch.load(current_path + "checkpoints/wideresnet_cifar10_cutout.pt"))
wideresnet_gradcam_cifar10_cutout.eval()

wideresnet_gradcam_cifar10_da = WideResNet(depth=28, num_classes=10, widen_factor=10, dropRate=0.3)
wideresnet_gradcam_cifar10_da.load_state_dict(torch.load(current_path + "checkpoints/wideresnet_cifar10_da.pt"))
wideresnet_gradcam_cifar10_da.eval()

wideresnet_gradcam_cifar10_da_cutout = WideResNet(depth=28, num_classes=10, widen_factor=10, dropRate=0.3)
wideresnet_gradcam_cifar10_da_cutout.load_state_dict(torch.load(current_path + "checkpoints/wideresnet_cifar10_da_cutout.pt"))
wideresnet_gradcam_cifar10_da_cutout.eval()
```
:::

::: {.cell .markdown}
Let's try to see the result from the testloader of CIFAR-10 dataset
:::

::: {.cell .code}
``` python
transform_cifar10 = transforms.Compose([
    transforms.Resize((32, 32)),
    transforms.ToTensor(),
    transforms.Normalize(mean=[0.485, 0.456, 0.406], std=[0.229, 0.224, 0.225])
])

testset_cifar10 = datasets.CIFAR10(root='./data', train=False, download=True, transform=transform_cifar10)
testloader_cifar10 = torch.utils.data.DataLoader(testset_cifar10, batch_size=1, shuffle=True, num_workers=2)

```
:::

::: {.cell .code}
``` python
cifar10_classes = [
    "Airplane", "Automobile", "Bird", "Cat", "Deer",
    "Dog", "Frog", "Horse", "Ship", "Truck"
]
```
:::

::: {.cell .code}
``` python
# Get a batch from the testloader
images, labels = next(iter(testloader_cifar10))
input_tensor = images  # As your batch_size is 1, you will have a single image here

# Forward pass
wideresnet_gradcam_cifar10.zero_grad()
output_wideresnet_gradcam_cifar10 = wideresnet_gradcam_cifar10(input_tensor)

wideresnet_gradcam_cifar10_cutout.zero_grad()
output_wideresnet_gradcam_cifar10_cutout = wideresnet_gradcam_cifar10_cutout(input_tensor)

wideresnet_gradcam_cifar10_da.zero_grad()
output_wideresnet_gradcam_cifar10_da = wideresnet_gradcam_cifar10_da(input_tensor)

wideresnet_gradcam_cifar10_da_cutout.zero_grad()
output_wideresnet_gradcam_cifar10_da_cutout = wideresnet_gradcam_cifar10_da_cutout(input_tensor)

# Get the index of the max log-probability
target_wideresnet_gradcam_cifar10 = output_wideresnet_gradcam_cifar10.argmax(1)
output_wideresnet_gradcam_cifar10.max().backward()

target_wideresnet_gradcam_cifar10_cutout = output_wideresnet_gradcam_cifar10_cutout.argmax(1)
output_wideresnet_gradcam_cifar10_cutout.max().backward()

target_wideresnet_gradcam_cifar10_da = output_wideresnet_gradcam_cifar10_da.argmax(1)
output_wideresnet_gradcam_cifar10_da.max().backward()

target_wideresnet_gradcam_cifar10_da_cutout = output_wideresnet_gradcam_cifar10_da_cutout.argmax(1)
output_wideresnet_gradcam_cifar10_da_cutout.max().backward()

# Map the predicted class indices to the class labels
predicted_class_wideresnet_gradcam_cifar10 = cifar10_classes[target_wideresnet_gradcam_cifar10.item()]
predicted_class_wideresnet_gradcam_cifar10 = cifar10_classes[target_wideresnet_gradcam_cifar10_cutout.item()]
predicted_class_wideresnet_gradcam_cifar10_da = cifar10_classes[target_wideresnet_gradcam_cifar10_da.item()]
predicted_class_wideresnet_gradcam_cifar10_da_cutout = cifar10_classes[target_wideresnet_gradcam_cifar10_da_cutout.item()]


# Get the gradients and activations
gradients_wideresnet_gradcam_cifar10 = wideresnet_gradcam_cifar10.gradients.detach().cpu()
activations_wideresnet_gradcam_cifar10 = wideresnet_gradcam_cifar10.activations.detach().cpu()

gradients_wideresnet_gradcam_cifar10_cutout = wideresnet_gradcam_cifar10_cutout.gradients.detach().cpu()
activations_wideresnet_gradcam_cifar10_cutout = wideresnet_gradcam_cifar10_cutout.activations.detach().cpu()

gradients_wideresnet_gradcam_cifar10_da = wideresnet_gradcam_cifar10_da.gradients.detach().cpu()
activations_wideresnet_gradcam_cifar10_da = wideresnet_gradcam_cifar10_da.activations.detach().cpu()

gradients_wideresnet_gradcam_cifar10_da_cutout = wideresnet_gradcam_cifar10_da_cutout.gradients.detach().cpu()
activations_wideresnet_gradcam_cifar10_da_cutout = wideresnet_gradcam_cifar10_da_cutout.activations.detach().cpu()


# Calculate the weights
weights_wideresnet_gradcam_cifar10 = gradients_wideresnet_gradcam_cifar10.mean(dim=(2, 3), keepdim=True)

weights_wideresnet_gradcam_cifar10_cutout = gradients_wideresnet_gradcam_cifar10_cutout.mean(dim=(2, 3), keepdim=True)

weights_wideresnet_gradcam_cifar10_da = gradients_wideresnet_gradcam_cifar10_da.mean(dim=(2, 3), keepdim=True)

weights_wideresnet_gradcam_cifar10_da_cutout = gradients_wideresnet_gradcam_cifar10_da_cutout.mean(dim=(2, 3), keepdim=True)

# Calculate the weighted sum of activations (Grad-CAM)
cam_wideresnet_gradcam_cifar10 = (weights_wideresnet_gradcam_cifar10 * activations_wideresnet_gradcam_cifar10).sum(dim=1, keepdim=True)
cam_wideresnet_gradcam_cifar10 = F.relu(cam_wideresnet_gradcam_cifar10)  # apply ReLU to the heatmap
cam_wideresnet_gradcam_cifar10 = F.interpolate(cam_wideresnet_gradcam_cifar10, size=(32, 32), mode='bilinear', align_corners=False)
cam_wideresnet_gradcam_cifar10 = cam_wideresnet_gradcam_cifar10.squeeze().numpy()

cam_wideresnet_gradcam_cifar10_cutout = (weights_wideresnet_gradcam_cifar10_cutout * activations_wideresnet_gradcam_cifar10_cutout).sum(dim=1, keepdim=True)
cam_wideresnet_gradcam_cifar10_cutout = F.relu(cam_wideresnet_gradcam_cifar10_cutout)  # apply ReLU to the heatmap
cam_wideresnet_gradcam_cifar10_cutout = F.interpolate(cam_wideresnet_gradcam_cifar10_cutout, size=(32, 32), mode='bilinear', align_corners=False)
cam_wideresnet_gradcam_cifar10_cutout = cam_wideresnet_gradcam_cifar10_cutout.squeeze().numpy()

cam_wideresnet_gradcam_cifar10_da = (weights_wideresnet_gradcam_cifar10_da * activations_wideresnet_gradcam_cifar10_da).sum(dim=1, keepdim=True)
cam_wideresnet_gradcam_cifar10_da = F.relu(cam_wideresnet_gradcam_cifar10_da)  # apply ReLU to the heatmap
cam_wideresnet_gradcam_cifar10_da = F.interpolate(cam_wideresnet_gradcam_cifar10_da, size=(32, 32), mode='bilinear', align_corners=False)
cam_wideresnet_gradcam_cifar10_da = cam_wideresnet_gradcam_cifar10_da.squeeze().numpy()

cam_wideresnet_gradcam_cifar10_da_cutout = (weights_wideresnet_gradcam_cifar10_da_cutout * activations_wideresnet_gradcam_cifar10_da_cutout).sum(dim=1, keepdim=True)
cam_wideresnet_gradcam_cifar10_da_cutout = F.relu(cam_wideresnet_gradcam_cifar10_da_cutout)  # apply ReLU to the heatmap
cam_wideresnet_gradcam_cifar10_da_cutout = F.interpolate(cam_wideresnet_gradcam_cifar10_da_cutout, size=(32, 32), mode='bilinear', align_corners=False)
cam_wideresnet_gradcam_cifar10_da_cutout = cam_wideresnet_gradcam_cifar10_da_cutout.squeeze().numpy()


# Normalize the heatmap
cam_wideresnet_gradcam_cifar10 -= cam_wideresnet_gradcam_cifar10.min()
cam_wideresnet_gradcam_cifar10 /= cam_wideresnet_gradcam_cifar10.max()

cam_wideresnet_gradcam_cifar10_cutout -= cam_wideresnet_gradcam_cifar10_cutout.min()
cam_wideresnet_gradcam_cifar10_cutout /= cam_wideresnet_gradcam_cifar10_cutout.max()

cam_wideresnet_gradcam_cifar10_da -= cam_wideresnet_gradcam_cifar10_da.min()
cam_wideresnet_gradcam_cifar10_da /= cam_wideresnet_gradcam_cifar10_da.max()

cam_wideresnet_gradcam_cifar10_da_cutout -= cam_wideresnet_gradcam_cifar10_da_cutout.min()
cam_wideresnet_gradcam_cifar10_da_cutout /= cam_wideresnet_gradcam_cifar10_da_cutout.max()

# Since the images from the dataloader are normalized, you have to denormalize them before plotting
mean = torch.tensor([0.485, 0.456, 0.406])
std = torch.tensor([0.229, 0.224, 0.225])
img = images.squeeze().detach().cpu() * std[..., None, None] + mean[..., None, None]
img = img.permute(1, 2, 0).numpy()

# Superimpose the heatmap onto the original image
heatmap_wideresnet_gradcam_cifar10 = cv2.applyColorMap(np.uint8(255 * cam_wideresnet_gradcam_cifar10), cv2.COLORMAP_JET)
heatmap_wideresnet_gradcam_cifar10 = cv2.cvtColor(heatmap_wideresnet_gradcam_cifar10, cv2.COLOR_BGR2RGB)
superimposed_img_wideresnet_gradcam_cifar10 = heatmap_wideresnet_gradcam_cifar10 * 0.4 + img * 255

heatmap_wideresnet_gradcam_cifar10_cutout = cv2.applyColorMap(np.uint8(255 * cam_wideresnet_gradcam_cifar10_cutout), cv2.COLORMAP_JET)
heatmap_wideresnet_gradcam_cifar10_cutout = cv2.cvtColor(heatmap_wideresnet_gradcam_cifar10_cutout, cv2.COLOR_BGR2RGB)
superimposed_img_wideresnet_gradcam_cifar10_cutout = heatmap_wideresnet_gradcam_cifar10_cutout * 0.4 + img * 255

heatmap_wideresnet_gradcam_cifar10_da = cv2.applyColorMap(np.uint8(255 * cam_wideresnet_gradcam_cifar10_da), cv2.COLORMAP_JET)
heatmap_wideresnet_gradcam_cifar10_da = cv2.cvtColor(heatmap_wideresnet_gradcam_cifar10_da, cv2.COLOR_BGR2RGB)
superimposed_img_wideresnet_gradcam_cifar10_da = heatmap_wideresnet_gradcam_cifar10_da * 0.4 + img * 255

heatmap_wideresnet_gradcam_cifar10_da_cutout = cv2.applyColorMap(np.uint8(255 * cam_wideresnet_gradcam_cifar10_da_cutout), cv2.COLORMAP_JET)
heatmap_wideresnet_gradcam_cifar10_da_cutout = cv2.cvtColor(heatmap_wideresnet_gradcam_cifar10_da_cutout, cv2.COLOR_BGR2RGB)
superimposed_img_wideresnet_gradcam_cifar10_da_cutout = heatmap_wideresnet_gradcam_cifar10_da_cutout * 0.4 + img * 255

class_label = str(labels.item())

# Display the original image and the Grad-CAM
fig, ax = plt.subplots(nrows=1, ncols=5, figsize=(20, 4))

ax[0].imshow(img)
ax[0].set_title('(Class: ' + cifar10_classes[int(class_label)] + ')')
ax[0].axis('off')
ax[1].imshow(superimposed_img_wideresnet_gradcam_cifar10 / 255)
ax[1].set_title('Pred: ' + predicted_class_wideresnet_gradcam_cifar10)
ax[1].axis('off')
ax[2].imshow(superimposed_img_wideresnet_gradcam_cifar10_cutout / 255)
ax[2].set_title('Pred: ' +predicted_class_wideresnet_gradcam_cifar10)
ax[2].axis('off')
ax[3].imshow(superimposed_img_wideresnet_gradcam_cifar10_da / 255)
ax[3].set_title('Pred: ' +predicted_class_wideresnet_gradcam_cifar10_da)
ax[3].axis('off')
ax[4].imshow(superimposed_img_wideresnet_gradcam_cifar10_da_cutout / 255)
ax[4].set_title('Pred: ' +predicted_class_wideresnet_gradcam_cifar10_da_cutout)
ax[4].axis('off')

# Adjust the vertical position of the suptitle to bring it closer to the subplots.
fig.suptitle("Original Image - Grad-CAM -  Grad-CAM with Cutout - Grad-CAM with Data Augmentation - Grad-CAM with Cutout & Data Augmentation", y=1.1) 

# Automatically adjust subplot params for better layout.
plt.tight_layout()
plt.show()
```
:::


::: {.cell .markdown}
Now you can try to load your image, preprocess it and convert it into a PyTorch tensor. Choose an image that is in the CIFAR-10 classes (airplanes, cars, birds, cats, deer, dogs, frogs, horses, ships, and trucks). The preprocessing steps should be the same as the ones you used for training your model. Let’s say you have an image `image.jpeg`:
:::

::: {.cell .code}
```python
# Load the image
image_path = "image.jpeg"
image = Image.open(current_path + image_path)

# Define the transformations: resize, to tensor, normalize (replace the mean and std with values you used for training)
preprocess = transforms.Compose([
    transforms.Resize((32, 32)),
    transforms.ToTensor(),
    transforms.Normalize(mean=[0.485, 0.456, 0.406], std=[0.229, 0.224, 0.225])
])

# Preprocess the image
input_tensor = preprocess(image)
input_tensor = input_tensor.unsqueeze(0)  # add batch dimension.  C,H,W => B,C,H,W
```
:::


::: {.cell .markdown}
Apply Grad-CAM
:::

::: {.cell .code}
```python
# Forward pass
wideresnet_gradcam_cifar10.zero_grad()
output_wideresnet_gradcam_cifar10 = wideresnet_gradcam_cifar10(input_tensor)

wideresnet_gradcam_cifar10_cutout.zero_grad()
output_wideresnet_gradcam_cifar10_cutout = wideresnet_gradcam_cifar10_cutout(input_tensor)

wideresnet_gradcam_cifar10_da.zero_grad()
output_wideresnet_gradcam_cifar10_da = wideresnet_gradcam_cifar10_da(input_tensor)

wideresnet_gradcam_cifar10_da_cutout.zero_grad()
output_wideresnet_gradcam_cifar10_da_cutout = wideresnet_gradcam_cifar10_da_cutout(input_tensor)

# Get the index of the max log-probability
target_wideresnet_gradcam_cifar10 = output_wideresnet_gradcam_cifar10.argmax(1)
output_wideresnet_gradcam_cifar10.max().backward()

target_wideresnet_gradcam_cifar10_cutout = output_wideresnet_gradcam_cifar10_cutout.argmax(1)
output_wideresnet_gradcam_cifar10_cutout.max().backward()

target_wideresnet_gradcam_cifar10_da = output_wideresnet_gradcam_cifar10_da.argmax(1)
output_wideresnet_gradcam_cifar10_da.max().backward()

target_wideresnet_gradcam_cifar10_da_cutout = output_wideresnet_gradcam_cifar10_da_cutout.argmax(1)
output_wideresnet_gradcam_cifar10_da_cutout.max().backward()

# Map the predicted class indices to the class labels
predicted_class_wideresnet_gradcam_cifar10 = cifar10_classes[target_wideresnet_gradcam_cifar10.item()]
predicted_class_wideresnet_gradcam_cifar10 = cifar10_classes[target_wideresnet_gradcam_cifar10_cutout.item()]
predicted_class_wideresnet_gradcam_cifar10_da = cifar10_classes[target_wideresnet_gradcam_cifar10_da.item()]
predicted_class_wideresnet_gradcam_cifar10_da_cutout = cifar10_classes[target_wideresnet_gradcam_cifar10_da_cutout.item()]


# Get the gradients and activations
gradients_wideresnet_gradcam_cifar10 = wideresnet_gradcam_cifar10.gradients.detach().cpu()
activations_wideresnet_gradcam_cifar10 = wideresnet_gradcam_cifar10.activations.detach().cpu()

gradients_wideresnet_gradcam_cifar10_cutout = wideresnet_gradcam_cifar10_cutout.gradients.detach().cpu()
activations_wideresnet_gradcam_cifar10_cutout = wideresnet_gradcam_cifar10_cutout.activations.detach().cpu()

gradients_wideresnet_gradcam_cifar10_da = wideresnet_gradcam_cifar10_da.gradients.detach().cpu()
activations_wideresnet_gradcam_cifar10_da = wideresnet_gradcam_cifar10_da.activations.detach().cpu()

gradients_wideresnet_gradcam_cifar10_da_cutout = wideresnet_gradcam_cifar10_da_cutout.gradients.detach().cpu()
activations_wideresnet_gradcam_cifar10_da_cutout = wideresnet_gradcam_cifar10_da_cutout.activations.detach().cpu()


# Calculate the weights
weights_wideresnet_gradcam_cifar10 = gradients_wideresnet_gradcam_cifar10.mean(dim=(2, 3), keepdim=True)

weights_wideresnet_gradcam_cifar10_cutout = gradients_wideresnet_gradcam_cifar10_cutout.mean(dim=(2, 3), keepdim=True)

weights_wideresnet_gradcam_cifar10_da = gradients_wideresnet_gradcam_cifar10_da.mean(dim=(2, 3), keepdim=True)

weights_wideresnet_gradcam_cifar10_da_cutout = gradients_wideresnet_gradcam_cifar10_da_cutout.mean(dim=(2, 3), keepdim=True)

# Calculate the weighted sum of activations (Grad-CAM)
cam_wideresnet_gradcam_cifar10 = (weights_wideresnet_gradcam_cifar10 * activations_wideresnet_gradcam_cifar10).sum(dim=1, keepdim=True)
cam_wideresnet_gradcam_cifar10 = F.relu(cam_wideresnet_gradcam_cifar10)  # apply ReLU to the heatmap
cam_wideresnet_gradcam_cifar10 = F.interpolate(cam_wideresnet_gradcam_cifar10, size=(32, 32), mode='bilinear', align_corners=False)
cam_wideresnet_gradcam_cifar10 = cam_wideresnet_gradcam_cifar10.squeeze().numpy()

cam_wideresnet_gradcam_cifar10_cutout = (weights_wideresnet_gradcam_cifar10_cutout * activations_wideresnet_gradcam_cifar10_cutout).sum(dim=1, keepdim=True)
cam_wideresnet_gradcam_cifar10_cutout = F.relu(cam_wideresnet_gradcam_cifar10_cutout)  # apply ReLU to the heatmap
cam_wideresnet_gradcam_cifar10_cutout = F.interpolate(cam_wideresnet_gradcam_cifar10_cutout, size=(32, 32), mode='bilinear', align_corners=False)
cam_wideresnet_gradcam_cifar10_cutout = cam_wideresnet_gradcam_cifar10_cutout.squeeze().numpy()

cam_wideresnet_gradcam_cifar10_da = (weights_wideresnet_gradcam_cifar10_da * activations_wideresnet_gradcam_cifar10_da).sum(dim=1, keepdim=True)
cam_wideresnet_gradcam_cifar10_da = F.relu(cam_wideresnet_gradcam_cifar10_da)  # apply ReLU to the heatmap
cam_wideresnet_gradcam_cifar10_da = F.interpolate(cam_wideresnet_gradcam_cifar10_da, size=(32, 32), mode='bilinear', align_corners=False)
cam_wideresnet_gradcam_cifar10_da = cam_wideresnet_gradcam_cifar10_da.squeeze().numpy()

cam_wideresnet_gradcam_cifar10_da_cutout = (weights_wideresnet_gradcam_cifar10_da_cutout * activations_wideresnet_gradcam_cifar10_da_cutout).sum(dim=1, keepdim=True)
cam_wideresnet_gradcam_cifar10_da_cutout = F.relu(cam_wideresnet_gradcam_cifar10_da_cutout)  # apply ReLU to the heatmap
cam_wideresnet_gradcam_cifar10_da_cutout = F.interpolate(cam_wideresnet_gradcam_cifar10_da_cutout, size=(32, 32), mode='bilinear', align_corners=False)
cam_wideresnet_gradcam_cifar10_da_cutout = cam_wideresnet_gradcam_cifar10_da_cutout.squeeze().numpy()


# Normalize the heatmap
cam_wideresnet_gradcam_cifar10 -= cam_wideresnet_gradcam_cifar10.min()
cam_wideresnet_gradcam_cifar10 /= cam_wideresnet_gradcam_cifar10.max()

cam_wideresnet_gradcam_cifar10_cutout -= cam_wideresnet_gradcam_cifar10_cutout.min()
cam_wideresnet_gradcam_cifar10_cutout /= cam_wideresnet_gradcam_cifar10_cutout.max()

cam_wideresnet_gradcam_cifar10_da -= cam_wideresnet_gradcam_cifar10_da.min()
cam_wideresnet_gradcam_cifar10_da /= cam_wideresnet_gradcam_cifar10_da.max()

cam_wideresnet_gradcam_cifar10_da_cutout -= cam_wideresnet_gradcam_cifar10_da_cutout.min()
cam_wideresnet_gradcam_cifar10_da_cutout /= cam_wideresnet_gradcam_cifar10_da_cutout.max()

# Load the original image
img = cv2.imread(current_path + image_path)
img = cv2.resize(img, (32, 32))
img = cv2.cvtColor(img, cv2.COLOR_BGR2RGB)

# Superimpose the heatmap onto the original image
heatmap_wideresnet_gradcam_cifar10 = cv2.applyColorMap(np.uint8(255 * cam_wideresnet_gradcam_cifar10), cv2.COLORMAP_JET)
heatmap_wideresnet_gradcam_cifar10 = cv2.cvtColor(heatmap_wideresnet_gradcam_cifar10, cv2.COLOR_BGR2RGB)
superimposed_img_wideresnet_gradcam_cifar10 = heatmap_wideresnet_gradcam_cifar10 * 0.4 + img * 255

heatmap_wideresnet_gradcam_cifar10_cutout = cv2.applyColorMap(np.uint8(255 * cam_wideresnet_gradcam_cifar10_cutout), cv2.COLORMAP_JET)
heatmap_wideresnet_gradcam_cifar10_cutout = cv2.cvtColor(heatmap_wideresnet_gradcam_cifar10_cutout, cv2.COLOR_BGR2RGB)
superimposed_img_wideresnet_gradcam_cifar10_cutout = heatmap_wideresnet_gradcam_cifar10_cutout * 0.4 + img * 255

heatmap_wideresnet_gradcam_cifar10_da = cv2.applyColorMap(np.uint8(255 * cam_wideresnet_gradcam_cifar10_da), cv2.COLORMAP_JET)
heatmap_wideresnet_gradcam_cifar10_da = cv2.cvtColor(heatmap_wideresnet_gradcam_cifar10_da, cv2.COLOR_BGR2RGB)
superimposed_img_wideresnet_gradcam_cifar10_da = heatmap_wideresnet_gradcam_cifar10_da * 0.4 + img * 255

heatmap_wideresnet_gradcam_cifar10_da_cutout = cv2.applyColorMap(np.uint8(255 * cam_wideresnet_gradcam_cifar10_da_cutout), cv2.COLORMAP_JET)
heatmap_wideresnet_gradcam_cifar10_da_cutout = cv2.cvtColor(heatmap_wideresnet_gradcam_cifar10_da_cutout, cv2.COLOR_BGR2RGB)
superimposed_img_wideresnet_gradcam_cifar10_da_cutout = heatmap_wideresnet_gradcam_cifar10_da_cutout * 0.4 + img * 255

class_label = str(labels.item())

# Display the original image and the Grad-CAM
fig, ax = plt.subplots(nrows=1, ncols=5, figsize=(20, 4))

ax[0].imshow(img)
ax[0].set_title('(Class: ' + cifar10_classes[int(class_label)] + ')')
ax[0].axis('off')
ax[1].imshow(superimposed_img_wideresnet_gradcam_cifar10 / 255)
ax[1].set_title('Pred: ' + predicted_class_wideresnet_gradcam_cifar10)
ax[1].axis('off')
ax[2].imshow(superimposed_img_wideresnet_gradcam_cifar10_cutout / 255)
ax[2].set_title('Pred: ' +predicted_class_wideresnet_gradcam_cifar10)
ax[2].axis('off')
ax[3].imshow(superimposed_img_wideresnet_gradcam_cifar10_da / 255)
ax[3].set_title('Pred: ' +predicted_class_wideresnet_gradcam_cifar10_da)
ax[3].axis('off')
ax[4].imshow(superimposed_img_wideresnet_gradcam_cifar10_da_cutout / 255)
ax[4].set_title('Pred: ' +predicted_class_wideresnet_gradcam_cifar10_da_cutout)
ax[4].axis('off')

# Adjust the vertical position of the suptitle to bring it closer to the subplots.
fig.suptitle("Original Image - Grad-CAM -  Grad-CAM with Cutout - Grad-CAM with Data Augmentation - Grad-CAM with Cutout & Data Augmentation", y=1.1) 

# Automatically adjust subplot params for better layout.
plt.tight_layout()

plt.show()
```
:::

::: {.cell .markdown}
## 4.3.2 Implementation Grad-CAM for WideResNet Model for CIFAR-100
:::

::: {.cell .code}
``` python

wideresnet_gradcam_cifar100 = WideResNet(num_classes=100)
wideresnet_gradcam_cifar100.load_state_dict(torch.load(current_path + "checkpoints/wideresnet_cifar100.pt"))
wideresnet_gradcam_cifar100.eval()

wideresnet_gradcam_cifar100_cutout = WideResNet(num_classes=100)
wideresnet_gradcam_cifar100_cutout.load_state_dict(torch.load(current_path + "checkpoints/wideresnet_cifar100_cutout.pt"))
wideresnet_gradcam_cifar100_cutout.eval()

wideresnet_gradcam_cifar100_da = WideResNet(num_classes=100)
wideresnet_gradcam_cifar100_da.load_state_dict(torch.load(current_path + "checkpoints/wideresnet_cifar100_da.pt"))
wideresnet_gradcam_cifar100_da.eval()

wideresnet_gradcam_cifar100_da_cutout = WideResNet(num_classes=100)
wideresnet_gradcam_cifar100_da_cutout.load_state_dict(torch.load(current_path + "checkpoints/wideresnet_cifar100_da_cutout.pt"))
wideresnet_gradcam_cifar100_da_cutout.eval()
```
:::

::: {.cell .markdown}
Let's try to see the result from the testloader of CIFAR-10 dataset
:::

::: {.cell .code}
``` python
transform_cifar100 = transforms.Compose([
    transforms.Resize((32, 32)),
    transforms.ToTensor(),
    transforms.Normalize(mean=[0.485, 0.456, 0.406], std=[0.229, 0.224, 0.225])
])

testset_cifar100 = datasets.CIFAR100(root='./data', train=False, download=True, transform=transform_cifar100)
testloader_cifar100 = torch.utils.data.DataLoader(testset_cifar100, batch_size=1, shuffle=True, num_workers=2)

```
:::

::: {.cell .code}
``` python
cifar100_classes = [
    "Apple", "Aquarium fish", "Baby", "Bear", "Beaver", "Bed", "Bee", "Beetle", 
    "Bicycle", "Bottle", "Bowl", "Boy", "Bridge", "Bus", "Butterfly", "Camel", 
    "Can", "Castle", "Caterpillar", "Cattle", "Chair", "Chimpanzee", "Clock", 
    "Cloud", "Cockroach", "Couch", "Crab", "Crocodile", "Cup", "Dinosaur", 
    "Dolphin", "Elephant", "Flatfish", "Forest", "Fox", "Girl", "Hamster", 
    "House", "Kangaroo", "Computer keyboard", "Lamp", "Lawn-mower", "Leopard", "Lion",
    "Lizard", "Lobster", "Man", "Maple tree", "Motorcycle", "Mountain", "Mouse",
    "Mushrooms", "Oak tree", "Oranges", "Orchids", "Otter", "Palm tree", "Pears",
    "Pickup truck", "Pine tree", "Plain", "Plates", "Poppies", "Porcupine",
    "Possum", "Rabbit", "Raccoon", "Ray", "Road", "Rocket", "Roses", "Sea", "Seal",
    "Shark", "Shrew", "Skunk", "Skyscraper", "Snail", "Snake", "Spider", "Squirrel",
    "Streetcar", "Sunflowers", "Sweet peppers", "Table", "Tank", "Telephone", "Television", 
    "Tiger", "Tractor", "Train", "Trout", "Tulips", "Turtle", "Wardrobe", "Whale", 
    "Willow tree", "Wolf", "Woman", "Worm"
]

```
:::

::: {.cell .code}
``` python
# Get a batch from the testloader
images, labels = next(iter(testloader_cifar100))
input_tensor = images  # As your batch_size is 1, you will have a single image here

# Forward pass
wideresnet_gradcam_cifar100.zero_grad()
output_wideresnet_gradcam_cifar100 = wideresnet_gradcam_cifar100(input_tensor)

wideresnet_gradcam_cifar100_cutout.zero_grad()
output_wideresnet_gradcam_cifar100_cutout = wideresnet_gradcam_cifar100_cutout(input_tensor)

wideresnet_gradcam_cifar100_da.zero_grad()
output_wideresnet_gradcam_cifar100_da = wideresnet_gradcam_cifar100_da(input_tensor)

wideresnet_gradcam_cifar100_da_cutout.zero_grad()
output_wideresnet_gradcam_cifar100_da_cutout = wideresnet_gradcam_cifar100_da_cutout(input_tensor)

# Get the index of the max log-probability
target_wideresnet_gradcam_cifar100 = output_wideresnet_gradcam_cifar100.argmax(1)
output_wideresnet_gradcam_cifar100.max().backward()

target_wideresnet_gradcam_cifar100_cutout = output_wideresnet_gradcam_cifar100_cutout.argmax(1)
output_wideresnet_gradcam_cifar100_cutout.max().backward()

target_wideresnet_gradcam_cifar100_da = output_wideresnet_gradcam_cifar100_da.argmax(1)
output_wideresnet_gradcam_cifar100_da.max().backward()

target_wideresnet_gradcam_cifar100_da_cutout = output_wideresnet_gradcam_cifar100_da_cutout.argmax(1)
output_wideresnet_gradcam_cifar100_da_cutout.max().backward()

# Map the predicted class indices to the class labels
predicted_class_wideresnet_gradcam_cifar100 = cifar100_classes[target_wideresnet_gradcam_cifar100.item()]
predicted_class_wideresnet_gradcam_cifar100 = cifar100_classes[target_wideresnet_gradcam_cifar100_cutout.item()]
predicted_class_wideresnet_gradcam_cifar100_da = cifar100_classes[target_wideresnet_gradcam_cifar100_da.item()]
predicted_class_wideresnet_gradcam_cifar100_da_cutout = cifar100_classes[target_wideresnet_gradcam_cifar100_da_cutout.item()]


# Get the gradients and activations
gradients_wideresnet_gradcam_cifar100 = wideresnet_gradcam_cifar100.gradients.detach().cpu()
activations_wideresnet_gradcam_cifar100 = wideresnet_gradcam_cifar100.activations.detach().cpu()

gradients_wideresnet_gradcam_cifar100_cutout = wideresnet_gradcam_cifar100_cutout.gradients.detach().cpu()
activations_wideresnet_gradcam_cifar100_cutout = wideresnet_gradcam_cifar100_cutout.activations.detach().cpu()

gradients_wideresnet_gradcam_cifar100_da = wideresnet_gradcam_cifar100_da.gradients.detach().cpu()
activations_wideresnet_gradcam_cifar100_da = wideresnet_gradcam_cifar100_da.activations.detach().cpu()

gradients_wideresnet_gradcam_cifar100_da_cutout = wideresnet_gradcam_cifar100_da_cutout.gradients.detach().cpu()
activations_wideresnet_gradcam_cifar100_da_cutout = wideresnet_gradcam_cifar100_da_cutout.activations.detach().cpu()


# Calculate the weights
weights_wideresnet_gradcam_cifar100 = gradients_wideresnet_gradcam_cifar100.mean(dim=(2, 3), keepdim=True)

weights_wideresnet_gradcam_cifar100_cutout = gradients_wideresnet_gradcam_cifar100_cutout.mean(dim=(2, 3), keepdim=True)

weights_wideresnet_gradcam_cifar100_da = gradients_wideresnet_gradcam_cifar100_da.mean(dim=(2, 3), keepdim=True)

weights_wideresnet_gradcam_cifar100_da_cutout = gradients_wideresnet_gradcam_cifar100_da_cutout.mean(dim=(2, 3), keepdim=True)

# Calculate the weighted sum of activations (Grad-CAM)
cam_wideresnet_gradcam_cifar100 = (weights_wideresnet_gradcam_cifar100 * activations_wideresnet_gradcam_cifar100).sum(dim=1, keepdim=True)
cam_wideresnet_gradcam_cifar100 = F.relu(cam_wideresnet_gradcam_cifar100)  # apply ReLU to the heatmap
cam_wideresnet_gradcam_cifar100 = F.interpolate(cam_wideresnet_gradcam_cifar100, size=(32, 32), mode='bilinear', align_corners=False)
cam_wideresnet_gradcam_cifar100 = cam_wideresnet_gradcam_cifar100.squeeze().numpy()

cam_wideresnet_gradcam_cifar100_cutout = (weights_wideresnet_gradcam_cifar100_cutout * activations_wideresnet_gradcam_cifar100_cutout).sum(dim=1, keepdim=True)
cam_wideresnet_gradcam_cifar100_cutout = F.relu(cam_wideresnet_gradcam_cifar100_cutout)  # apply ReLU to the heatmap
cam_wideresnet_gradcam_cifar100_cutout = F.interpolate(cam_wideresnet_gradcam_cifar100_cutout, size=(32, 32), mode='bilinear', align_corners=False)
cam_wideresnet_gradcam_cifar100_cutout = cam_wideresnet_gradcam_cifar100_cutout.squeeze().numpy()

cam_wideresnet_gradcam_cifar100_da = (weights_wideresnet_gradcam_cifar100_da * activations_wideresnet_gradcam_cifar100_da).sum(dim=1, keepdim=True)
cam_wideresnet_gradcam_cifar100_da = F.relu(cam_wideresnet_gradcam_cifar100_da)  # apply ReLU to the heatmap
cam_wideresnet_gradcam_cifar100_da = F.interpolate(cam_wideresnet_gradcam_cifar100_da, size=(32, 32), mode='bilinear', align_corners=False)
cam_wideresnet_gradcam_cifar100_da = cam_wideresnet_gradcam_cifar100_da.squeeze().numpy()

cam_wideresnet_gradcam_cifar100_da_cutout = (weights_wideresnet_gradcam_cifar100_da_cutout * activations_wideresnet_gradcam_cifar100_da_cutout).sum(dim=1, keepdim=True)
cam_wideresnet_gradcam_cifar100_da_cutout = F.relu(cam_wideresnet_gradcam_cifar100_da_cutout)  # apply ReLU to the heatmap
cam_wideresnet_gradcam_cifar100_da_cutout = F.interpolate(cam_wideresnet_gradcam_cifar100_da_cutout, size=(32, 32), mode='bilinear', align_corners=False)
cam_wideresnet_gradcam_cifar100_da_cutout = cam_wideresnet_gradcam_cifar100_da_cutout.squeeze().numpy()


# Normalize the heatmap
cam_wideresnet_gradcam_cifar100 -= cam_wideresnet_gradcam_cifar100.min()
cam_wideresnet_gradcam_cifar100 /= cam_wideresnet_gradcam_cifar100.max()

cam_wideresnet_gradcam_cifar100_cutout -= cam_wideresnet_gradcam_cifar100_cutout.min()
cam_wideresnet_gradcam_cifar100_cutout /= cam_wideresnet_gradcam_cifar100_cutout.max()

cam_wideresnet_gradcam_cifar100_da -= cam_wideresnet_gradcam_cifar100_da.min()
cam_wideresnet_gradcam_cifar100_da /= cam_wideresnet_gradcam_cifar100_da.max()

cam_wideresnet_gradcam_cifar100_da_cutout -= cam_wideresnet_gradcam_cifar100_da_cutout.min()
cam_wideresnet_gradcam_cifar100_da_cutout /= cam_wideresnet_gradcam_cifar100_da_cutout.max()

# Since the images from the dataloader are normalized, you have to denormalize them before plotting
mean = torch.tensor([0.485, 0.456, 0.406])
std = torch.tensor([0.229, 0.224, 0.225])
img = images.squeeze().detach().cpu() * std[..., None, None] + mean[..., None, None]
img = img.permute(1, 2, 0).numpy()

# Superimpose the heatmap onto the original image
heatmap_wideresnet_gradcam_cifar100 = cv2.applyColorMap(np.uint8(255 * cam_wideresnet_gradcam_cifar100), cv2.COLORMAP_JET)
heatmap_wideresnet_gradcam_cifar100 = cv2.cvtColor(heatmap_wideresnet_gradcam_cifar100, cv2.COLOR_BGR2RGB)
superimposed_img_wideresnet_gradcam_cifar100 = heatmap_wideresnet_gradcam_cifar100 * 0.4 + img * 255

heatmap_wideresnet_gradcam_cifar100_cutout = cv2.applyColorMap(np.uint8(255 * cam_wideresnet_gradcam_cifar100_cutout), cv2.COLORMAP_JET)
heatmap_wideresnet_gradcam_cifar100_cutout = cv2.cvtColor(heatmap_wideresnet_gradcam_cifar100_cutout, cv2.COLOR_BGR2RGB)
superimposed_img_wideresnet_gradcam_cifar100_cutout = heatmap_wideresnet_gradcam_cifar100_cutout * 0.4 + img * 255

heatmap_wideresnet_gradcam_cifar100_da = cv2.applyColorMap(np.uint8(255 * cam_wideresnet_gradcam_cifar100_da), cv2.COLORMAP_JET)
heatmap_wideresnet_gradcam_cifar100_da = cv2.cvtColor(heatmap_wideresnet_gradcam_cifar100_da, cv2.COLOR_BGR2RGB)
superimposed_img_wideresnet_gradcam_cifar100_da = heatmap_wideresnet_gradcam_cifar100_da * 0.4 + img * 255

heatmap_wideresnet_gradcam_cifar100_da_cutout = cv2.applyColorMap(np.uint8(255 * cam_wideresnet_gradcam_cifar100_da_cutout), cv2.COLORMAP_JET)
heatmap_wideresnet_gradcam_cifar100_da_cutout = cv2.cvtColor(heatmap_wideresnet_gradcam_cifar100_da_cutout, cv2.COLOR_BGR2RGB)
superimposed_img_wideresnet_gradcam_cifar100_da_cutout = heatmap_wideresnet_gradcam_cifar100_da_cutout * 0.4 + img * 255

class_label = str(labels.item())

# Display the original image and the Grad-CAM
fig, ax = plt.subplots(nrows=1, ncols=5, figsize=(20, 4))

ax[0].imshow(img)
ax[0].set_title('(Class: ' + cifar100_classes[int(class_label)] + ')')
ax[0].axis('off')
ax[1].imshow(superimposed_img_wideresnet_gradcam_cifar100 / 255)
ax[1].set_title('Pred: ' +predicted_class_wideresnet_gradcam_cifar100)
ax[1].axis('off')
ax[2].imshow(superimposed_img_wideresnet_gradcam_cifar100_cutout / 255)
ax[2].set_title('Pred: ' +predicted_class_wideresnet_gradcam_cifar100)
ax[2].axis('off')
ax[3].imshow(superimposed_img_wideresnet_gradcam_cifar100_da / 255)
ax[3].set_title('Pred: ' +predicted_class_wideresnet_gradcam_cifar100_da)
ax[3].axis('off')
ax[4].imshow(superimposed_img_wideresnet_gradcam_cifar100_da_cutout / 255)
ax[4].set_title('Pred: ' +predicted_class_wideresnet_gradcam_cifar100_da_cutout)
ax[4].axis('off')

# Adjust the vertical position of the suptitle to bring it closer to the subplots.
fig.suptitle("Original Image - Grad-CAM -  Grad-CAM with Cutout - Grad-CAM with Data Augmentation - Grad-CAM with Cutout & Data Augmentation", y=1.1) 

# Automatically adjust subplot params for better layout.
plt.tight_layout()

plt.show()


```
:::

::: {.cell .markdown}
Now you can try to load your image, preprocess it and convert it into a PyTorch tensor. Choose an image that is in the CIFAR-10 classes (airplanes, cars, birds, cats, deer, dogs, frogs, horses, ships, and trucks). The preprocessing steps should be the same as the ones you used for training your model. Let’s say you have an image `image.jpeg`:
:::

::: {.cell .code}
```python
# Load the image
image_path = "image.jpeg"
image = Image.open(current_path + image_path)

# Define the transformations: resize, to tensor, normalize (replace the mean and std with values you used for training)
preprocess = transforms.Compose([
    transforms.Resize((32, 32)),
    transforms.ToTensor(),
    transforms.Normalize(mean=[0.485, 0.456, 0.406], std=[0.229, 0.224, 0.225])
])

# Preprocess the image
input_tensor = preprocess(image)
input_tensor = input_tensor.unsqueeze(0)  # add batch dimension.  C,H,W => B,C,H,W
```
:::


::: {.cell .markdown}
Apply Grad-CAM
:::

::: {.cell .code}
```python
# Forward pass
wideresnet_gradcam_cifar100.zero_grad()
output_wideresnet_gradcam_cifar100 = wideresnet_gradcam_cifar100(input_tensor)

wideresnet_gradcam_cifar100_cutout.zero_grad()
output_wideresnet_gradcam_cifar100_cutout = wideresnet_gradcam_cifar100_cutout(input_tensor)

wideresnet_gradcam_cifar100_da.zero_grad()
output_wideresnet_gradcam_cifar100_da = wideresnet_gradcam_cifar100_da(input_tensor)

wideresnet_gradcam_cifar100_da_cutout.zero_grad()
output_wideresnet_gradcam_cifar100_da_cutout = wideresnet_gradcam_cifar100_da_cutout(input_tensor)

# Get the index of the max log-probability
target_wideresnet_gradcam_cifar100 = output_wideresnet_gradcam_cifar100.argmax(1)
output_wideresnet_gradcam_cifar100.max().backward()

target_wideresnet_gradcam_cifar100_cutout = output_wideresnet_gradcam_cifar100_cutout.argmax(1)
output_wideresnet_gradcam_cifar100_cutout.max().backward()

target_wideresnet_gradcam_cifar100_da = output_wideresnet_gradcam_cifar100_da.argmax(1)
output_wideresnet_gradcam_cifar100_da.max().backward()

target_wideresnet_gradcam_cifar100_da_cutout = output_wideresnet_gradcam_cifar100_da_cutout.argmax(1)
output_wideresnet_gradcam_cifar100_da_cutout.max().backward()

# Map the predicted class indices to the class labels
predicted_class_wideresnet_gradcam_cifar100 = cifar100_classes[target_wideresnet_gradcam_cifar100.item()]
predicted_class_wideresnet_gradcam_cifar100 = cifar100_classes[target_wideresnet_gradcam_cifar100_cutout.item()]
predicted_class_wideresnet_gradcam_cifar100_da = cifar100_classes[target_wideresnet_gradcam_cifar100_da.item()]
predicted_class_wideresnet_gradcam_cifar100_da_cutout = cifar100_classes[target_wideresnet_gradcam_cifar100_da_cutout.item()]


# Get the gradients and activations
gradients_wideresnet_gradcam_cifar100 = wideresnet_gradcam_cifar100.gradients.detach().cpu()
activations_wideresnet_gradcam_cifar100 = wideresnet_gradcam_cifar100.activations.detach().cpu()

gradients_wideresnet_gradcam_cifar100_cutout = wideresnet_gradcam_cifar100_cutout.gradients.detach().cpu()
activations_wideresnet_gradcam_cifar100_cutout = wideresnet_gradcam_cifar100_cutout.activations.detach().cpu()

gradients_wideresnet_gradcam_cifar100_da = wideresnet_gradcam_cifar100_da.gradients.detach().cpu()
activations_wideresnet_gradcam_cifar100_da = wideresnet_gradcam_cifar100_da.activations.detach().cpu()

gradients_wideresnet_gradcam_cifar100_da_cutout = wideresnet_gradcam_cifar100_da_cutout.gradients.detach().cpu()
activations_wideresnet_gradcam_cifar100_da_cutout = wideresnet_gradcam_cifar100_da_cutout.activations.detach().cpu()


# Calculate the weights
weights_wideresnet_gradcam_cifar100 = gradients_wideresnet_gradcam_cifar100.mean(dim=(2, 3), keepdim=True)

weights_wideresnet_gradcam_cifar100_cutout = gradients_wideresnet_gradcam_cifar100_cutout.mean(dim=(2, 3), keepdim=True)

weights_wideresnet_gradcam_cifar100_da = gradients_wideresnet_gradcam_cifar100_da.mean(dim=(2, 3), keepdim=True)

weights_wideresnet_gradcam_cifar100_da_cutout = gradients_wideresnet_gradcam_cifar100_da_cutout.mean(dim=(2, 3), keepdim=True)

# Calculate the weighted sum of activations (Grad-CAM)
cam_wideresnet_gradcam_cifar100 = (weights_wideresnet_gradcam_cifar100 * activations_wideresnet_gradcam_cifar100).sum(dim=1, keepdim=True)
cam_wideresnet_gradcam_cifar100 = F.relu(cam_wideresnet_gradcam_cifar100)  # apply ReLU to the heatmap
cam_wideresnet_gradcam_cifar100 = F.interpolate(cam_wideresnet_gradcam_cifar100, size=(32, 32), mode='bilinear', align_corners=False)
cam_wideresnet_gradcam_cifar100 = cam_wideresnet_gradcam_cifar100.squeeze().numpy()

cam_wideresnet_gradcam_cifar100_cutout = (weights_wideresnet_gradcam_cifar100_cutout * activations_wideresnet_gradcam_cifar100_cutout).sum(dim=1, keepdim=True)
cam_wideresnet_gradcam_cifar100_cutout = F.relu(cam_wideresnet_gradcam_cifar100_cutout)  # apply ReLU to the heatmap
cam_wideresnet_gradcam_cifar100_cutout = F.interpolate(cam_wideresnet_gradcam_cifar100_cutout, size=(32, 32), mode='bilinear', align_corners=False)
cam_wideresnet_gradcam_cifar100_cutout = cam_wideresnet_gradcam_cifar100_cutout.squeeze().numpy()

cam_wideresnet_gradcam_cifar100_da = (weights_wideresnet_gradcam_cifar100_da * activations_wideresnet_gradcam_cifar100_da).sum(dim=1, keepdim=True)
cam_wideresnet_gradcam_cifar100_da = F.relu(cam_wideresnet_gradcam_cifar100_da)  # apply ReLU to the heatmap
cam_wideresnet_gradcam_cifar100_da = F.interpolate(cam_wideresnet_gradcam_cifar100_da, size=(32, 32), mode='bilinear', align_corners=False)
cam_wideresnet_gradcam_cifar100_da = cam_wideresnet_gradcam_cifar100_da.squeeze().numpy()

cam_wideresnet_gradcam_cifar100_da_cutout = (weights_wideresnet_gradcam_cifar100_da_cutout * activations_wideresnet_gradcam_cifar100_da_cutout).sum(dim=1, keepdim=True)
cam_wideresnet_gradcam_cifar100_da_cutout = F.relu(cam_wideresnet_gradcam_cifar100_da_cutout)  # apply ReLU to the heatmap
cam_wideresnet_gradcam_cifar100_da_cutout = F.interpolate(cam_wideresnet_gradcam_cifar100_da_cutout, size=(32, 32), mode='bilinear', align_corners=False)
cam_wideresnet_gradcam_cifar100_da_cutout = cam_wideresnet_gradcam_cifar100_da_cutout.squeeze().numpy()


# Normalize the heatmap
cam_wideresnet_gradcam_cifar100 -= cam_wideresnet_gradcam_cifar100.min()
cam_wideresnet_gradcam_cifar100 /= cam_wideresnet_gradcam_cifar100.max()

cam_wideresnet_gradcam_cifar100_cutout -= cam_wideresnet_gradcam_cifar100_cutout.min()
cam_wideresnet_gradcam_cifar100_cutout /= cam_wideresnet_gradcam_cifar100_cutout.max()

cam_wideresnet_gradcam_cifar100_da -= cam_wideresnet_gradcam_cifar100_da.min()
cam_wideresnet_gradcam_cifar100_da /= cam_wideresnet_gradcam_cifar100_da.max()

cam_wideresnet_gradcam_cifar100_da_cutout -= cam_wideresnet_gradcam_cifar100_da_cutout.min()
cam_wideresnet_gradcam_cifar100_da_cutout /= cam_wideresnet_gradcam_cifar100_da_cutout.max()

# Since the images from the dataloader are normalized, you have to denormalize them before plotting
img = cv2.imread(current_path + image_path)
img = cv2.resize(img, (32, 32))
img = cv2.cvtColor(img, cv2.COLOR_BGR2RGB)


# Superimpose the heatmap onto the original image
heatmap_wideresnet_gradcam_cifar100 = cv2.applyColorMap(np.uint8(255 * cam_wideresnet_gradcam_cifar100), cv2.COLORMAP_JET)
heatmap_wideresnet_gradcam_cifar100 = cv2.cvtColor(heatmap_wideresnet_gradcam_cifar100, cv2.COLOR_BGR2RGB)
superimposed_img_wideresnet_gradcam_cifar100 = heatmap_wideresnet_gradcam_cifar100 * 0.4 + img * 255

heatmap_wideresnet_gradcam_cifar100_cutout = cv2.applyColorMap(np.uint8(255 * cam_wideresnet_gradcam_cifar100_cutout), cv2.COLORMAP_JET)
heatmap_wideresnet_gradcam_cifar100_cutout = cv2.cvtColor(heatmap_wideresnet_gradcam_cifar100_cutout, cv2.COLOR_BGR2RGB)
superimposed_img_wideresnet_gradcam_cifar100_cutout = heatmap_wideresnet_gradcam_cifar100_cutout * 0.4 + img * 255

heatmap_wideresnet_gradcam_cifar100_da = cv2.applyColorMap(np.uint8(255 * cam_wideresnet_gradcam_cifar100_da), cv2.COLORMAP_JET)
heatmap_wideresnet_gradcam_cifar100_da = cv2.cvtColor(heatmap_wideresnet_gradcam_cifar100_da, cv2.COLOR_BGR2RGB)
superimposed_img_wideresnet_gradcam_cifar100_da = heatmap_wideresnet_gradcam_cifar100_da * 0.4 + img * 255

heatmap_wideresnet_gradcam_cifar100_da_cutout = cv2.applyColorMap(np.uint8(255 * cam_wideresnet_gradcam_cifar100_da_cutout), cv2.COLORMAP_JET)
heatmap_wideresnet_gradcam_cifar100_da_cutout = cv2.cvtColor(heatmap_wideresnet_gradcam_cifar100_da_cutout, cv2.COLOR_BGR2RGB)
superimposed_img_wideresnet_gradcam_cifar100_da_cutout = heatmap_wideresnet_gradcam_cifar100_da_cutout * 0.4 + img * 255

class_label = str(labels.item())
```
:::

::: {.cell .markdown}
Visualize the image and the Grad-CAM heatmap
:::

::: {.cell .code}
```python
# Display the original image and the Grad-CAM
fig, ax = plt.subplots(nrows=1, ncols=5)

ax[0].imshow(img)
ax[0].set_title('(Class: ' + cifar100_classes[int(class_label)] + ')')
ax[0].axis('off')
ax[1].imshow(superimposed_img_wideresnet_gradcam_cifar100 / 255)
ax[1].set_title('Pred: ' +predicted_class_wideresnet_gradcam_cifar100)
ax[1].axis('off')
ax[2].imshow(superimposed_img_wideresnet_gradcam_cifar100_cutout / 255)
ax[2].set_title(predicted_class_wideresnet_gradcam_cifar100)
ax[2].axis('off')
ax[3].imshow(superimposed_img_wideresnet_gradcam_cifar100_da / 255)
ax[3].set_title('Pred: ' +predicted_class_wideresnet_gradcam_cifar100_da)
ax[3].axis('off')
ax[4].imshow(superimposed_img_wideresnet_gradcam_cifar100_da_cutout / 255)
ax[4].set_title('Pred: ' +predicted_class_wideresnet_gradcam_cifar100_da_cutout)
ax[4].axis('off')

# Adjust the vertical position of the suptitle to bring it closer to the subplots.
fig.suptitle("Original Image - Grad-CAM -  Grad-CAM with Cutout - Grad-CAM with Data Augmentation - Grad-CAM with Cutout & Data Augmentation", y=1.1) 

# Automatically adjust subplot params for better layout.
plt.tight_layout()

plt.show()
```
:::
