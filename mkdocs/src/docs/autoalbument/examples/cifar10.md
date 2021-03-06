# Image classification on the CIFAR10 dataset

The following files are also available on GitHub - [https://github.com/albumentations-team/autoalbument/tree/master/examples/cifar10](https://github.com/albumentations-team/autoalbument/tree/master/examples/cifar10)


## dataset.py

```python
import torchvision


class SearchDataset(torchvision.datasets.CIFAR10):
    def __init__(self, transform=None):
        super().__init__(root="~/data/cifar10", train=True, download=True, transform=transform)

    def __getitem__(self, index):
        image, label = self.data[index], self.targets[index]

        if self.transform is not None:
            transformed = self.transform(image=image)
            image = transformed["image"]

        return image, label
```


## search.yaml

```yaml
# @package _global_

task: classification

# Settings for Policy Model that searches augmentation policies.
policy_model:

  # Multiplier for classification loss of a model. Faster AutoAugment uses classification loss to prevent augmentations
  # from transforming images of a particular class to another class. The authors of Faster AutoAugment use 0.1 as
  # default value.
  task_factor: 0.1

  # Multiplier for the gradient penalty for WGAN-GP training. 10 is the default value that was proposed in
  # `Improved Training of Wasserstein GANs`.
  gp_factor: 10

  # Temperature for Relaxed Bernoulli distribution. The probability of applying a certain augmentation is sampled from
  # Relaxed Bernoulli distribution (because Bernoulli distribution is not differentiable). With lower values of
  # `temperature` Relaxed Bernoulli distribution behaves like Bernoulli distribution. In the paper, the authors
  # of Faster AutoAugment used 0.05 as a default value for `temperature`.
  temperature: 0.05

  # Number of augmentation sub-policies. When an image passes through an augmentation pipeline, Faster AutoAugment
  # randomly chooses one sub-policy and uses augmentations from that sub-policy to transform an input image. A larger
  # number of sub-policies leads to a more diverse set of augmentations and better performance of a model trained on
  # augmented images. However, an increase in the number of sub-policies leads to the exponential growth of a search
  # space of augmentations, so you need more training data for Policy Model to find good augmentation policies.
  num_sub_policies: 150

  # Number of chunks in a batch. Faster AutoAugment splits each batch of images into `num_chunks` chunks. Then it
  # applies the same sub-policy with the same parameters to each image in a chunk. This parameter controls the tradeoff
  # between the speed of augmentation search and diversity of augmentations. Larger `num_chunks` values will lead to
  # faster searching but less diverse set of augmentations. Note that this parameter is used only in the searching
  # phase. When you train a model with found sub-policies, Albumentations will apply a distinct set of transformations
  # to each image separately.
  num_chunks: 8

  # Number of consecutive augmentations in each sub-policy. Faster AutoAugment will sequentially apply `operation_count`
  # augmentations from a sub-policy to an image. Larger values of `operation_count` lead to better performance of
  # a model trained on augmented images. Simultaneously, larger values of `operation_count` affect the speed of search
  # and increase the searching time.
  operation_count: 4


# Settings for Classification Model that is used for two purposes:
# 1. As a model that performs classification of input images.
# 2. As a Discriminator for Policy Model.
classification_model:

  # Number of classes in the dataset. The dataset implementation should return an integer in the range
  # [0, num_classes - 1] as a class label of an image.
  num_classes: 10

  # The architecture of Classification Model. AutoAlbument uses models from
  # https://github.com/rwightman/pytorch-image-models/. Please refer to its documentation to get a list of available
  # models - https://rwightman.github.io/pytorch-image-models/#list-models-with-pretrained-weights.
  architecture: resnet18

  # Boolean flag that indicates whether the selected model architecture should load pretrained weights or use randomly
  # initialized weights.
  pretrained: False

data:
  # Class for the PyTorch Dataset and arguments to it. AutoAlbument will create an object of this class using
  # the `instantiate` method from Hydra - https://hydra.cc/docs/next/patterns/instantiate_objects/overview/.
  #
  # Note that the target class value in the `_target_` argument should be located inside PYTHONPATH so Hydra could
  # find it. The directory with the config file is automatically added to PYTHONPATH, so the default value
  # `dataset.SearchDataset` points to the class `SearchDataset` from the `dataset.py` file. This `dataset.py` file is
  # located along with the `search.yaml` file in the same directory provided by `--config-dir`.
  #
  # As an alternative, you could provide a path to a Python file with the dataset using the `dataset_file` parameter
  # instead of the `dataset` parameter. The Python file should contain the implementation of a PyTorch dataset for
  # augmentation search. The dataset class should have named `SearchDataset`. The value in `dataset_file` could either
  # be a relative or an absolute path ; in the case of a relative path, the path should be relative to this config
  # file's location.
  #
  # - Example of a relative path:
  # dataset_file: dataset.py
  #
  # - Example of an absolute path:
  # dataset_file: /projects/pytorch/dataset.py
  #
  dataset:
    _target_: dataset.SearchDataset

  # The data type of input images. Two values are supported:
  # - uint8. In that case, all input images should be NumPy arrays with the np.uint8 data type and values in the range
  #   [0, 255].
  # - float32. In that case, all input images should be NumPy arrays with the np.float32 data type and values in the
  #   range [0.0, 1.0].
  input_dtype: uint8

  # A list of preprocessing augmentations that will be applied to each image before applying augmentations from
  # a policy. A preprocessing augmentation should be defined as `key`: `value`, where `key` is the name of augmentation
  # from Albumentations, and `value` is a dictionary with augmentation parameters. The found policy will also apply
  # those preprocessing augmentations before applying the main augmentations.
  #
  # Here is an example of an augmentation pipeline that first pads an image to the size 512x512 pixels, then resizes
  # the resulting image to the size 256x256 pixels and finally crops a random patch with the size 224x224 pixels.
  #
  #  preprocessing:
  #    - PadIfNeeded:
  #        min_height: 512
  #        min_width: 512
  #    - Resize:
  #        height: 256
  #        width: 256
  #    - RandomCrop:
  #        height: 224
  #        width: 224
  #
  preprocessing: null

  # Normalization values for images. For each image, the search pipeline will subtract `mean` and divide by `std`.
  # Normalization is applied after transforms defined in `preprocessing`. Note that regardless of `input_dtype`,
  # the normalization function will always receive a `float32` input with values in the range [0.0, 1.0], so you should
  # define `mean` and `std` values accordingly. ImageNet normalization is used by default.
  normalization:
    mean: [0.485, 0.456, 0.406]
    std: [0.229, 0.224, 0.225]

  # Parameters for the PyTorch DataLoader. Please refer to the PyTorch documentation for the description of parameters -
  # https://pytorch.org/docs/stable/data.html#torch.utils.data.DataLoader.
  dataloader:
    _target_: torch.utils.data.DataLoader
    batch_size: 128
    shuffle: True
    num_workers: 4
    pin_memory: True
    drop_last: True

optim:
  # Number of epochs to search parameters of augmentations.
  epochs:  20

  # Optimizer configuration for Classification Model
  main:
    _target_: torch.optim.Adam
    lr: 1e-3
    betas: [0, 0.999]

  # Optimizer configuration for Policy Model
  policy:
    _target_: torch.optim.Adam
    lr: 1e-3
    betas: [0, 0.999]

# Device that will keep PyTorch Tensors and which will be used for training. Please refer to the PyTorch documentation
# for more information -  https://pytorch.org/docs/stable/tensor_attributes.html#torch.torch.device.
device: cuda

# Value for torch.backends.cudnn.benchmark
# https://pytorch.org/docs/stable/notes/randomness.html#cuda-convolution-benchmarking
cudnn_benchmark: True

# If set to `True` AutoAlbument will save a checkpoint that contains states of models and optimizers at the end of each
# epoch. Checkpoints will be saved to the directory `<working directory>/checkpoints`.
save_checkpoints: False

# Path to a PyTorch checkpoint that contains saved states of models and optimizers. The value should be an absolute path
# to a file. If set, AutoAlbument will resume the searching process with data from the checkpoint.
checkpoint_path: null

# Path to a directory in which AutoAlbument will save TensorBoard logs. Set the value to `null` if you want to disable
# this feature.
tensorboard_logs_dir: null

hydra:
  run:
    # Path to the directory that will contain all outputs produced by the search algorithm. `${config_dir:}` contains
    # path to the directory with the `search.yaml` config file. Please refer to the Hydra documentation for more
    # information - https://hydra.cc/docs/configure_hydra/workdir.
    dir: ${config_dir:}/outputs/${now:%Y-%m-%d}/${now:%H-%M-%S}
```
