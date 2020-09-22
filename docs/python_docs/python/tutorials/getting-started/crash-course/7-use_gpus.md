<!--- Licensed to the Apache Software Foundation (ASF) under one -->
<!--- or more contributor license agreements.  See the NOTICE file -->
<!--- distributed with this work for additional information -->
<!--- regarding copyright ownership.  The ASF licenses this file -->
<!--- to you under the Apache License, Version 2.0 (the -->
<!--- "License"); you may not use this file except in compliance -->
<!--- with the License.  You may obtain a copy of the License at -->

<!---   http://www.apache.org/licenses/LICENSE-2.0 -->

<!--- Unless required by applicable law or agreed to in writing, -->
<!--- software distributed under the License is distributed on an -->
<!--- "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY -->
<!--- KIND, either express or implied.  See the License for the -->
<!--- specific language governing permissions and limitations -->
<!--- under the License. -->

# Step 7: Load and Run a NN using GPU

In this step, you learn how to use graphics processing units (GPUs) with MXNet. If you use GPUs to train and deploy neural networks, you get significantly more computational power and speed when compared to central processing units (CPUs).

## Prerequisites

Before you start the other steps here, make sure you have at least one Nvidia GPU in your machine and CUDA properly installed. GPUs from AMD and Intel are not supported. Install the GPU-enabled version of MXNet. You can find information about how to install the GPU version of MXNet for your setup, [here](https://mxnet.apache.org/versions/1.7/get_started/?).

Use the following commands to check the number GPUs that are available.

```{.python .input  n=2}
from mxnet import np, npx, gluon, autograd
from mxnet.gluon import nn
import time
npx.set_np()

npx.num_gpus()
```

## Allocate data to a GPU

MXNet's ndarray is very similar to NumPy. One major difference is MXNet's ndarray has a `context` attribute that specifies which device an array is on. By default, it is on `npx.cpu()`. Change it to the first GPU with the following code. Use `npx.gpu()` or `npx.gpu(0)` to indicate the first GPU.

```{.python .input  n=10}
gpu = npx.gpu() if npx.num_gpus() > 0 else npx.cpu()
x = np.ones((3,4), ctx=gpu)
x
```

If you're using a CPU, MXNet allocates data on main memory and tries to use as many CPU cores as possible.  This is true even if there is more than one CPU socket. If there are multiple GPUs, MXNet specifies which GPUs the ndarray is allocated.

Assume there is a least one more GPU. Create another ndarray and assign it there. If you only have one GPU, then you get an error. In the example code here, you copy `x` to the second GPU, `npx.gpu(1)`:

```{.python .input  n=11}
gpu_1 = npx.gpu(1) if npx.num_gpus() > 1 else npx.cpu()
x.copyto(gpu_1)
```

MXNet requries that users explicitly move data between devices. But several operators such as `print`, and `asnumpy`, will implicitly move data to main memory.

## Choosing GPU Ids
If you have multiple GPUs on your machine, MXNet can access each of them through 0-indexing. As you saw before, the first GPU was accessed using `npx.gpu(1)`, and the second using `npx.gpu(1)`. This extends to however many GPUs your machine has. So if your machine has 8 GPUs, the last GPU is accessed using `npx.gpu(7)`. This allows you to select what GPUs to use for operations and training. You might find it particularly useful when you want to training multiple networks on separate GPUs.  

## Run an operation on a GPU

To perform an operation on a particular GPU, you only need to guarantee that the input of an operation is already on that GPU. The output is allocated on the same GPU as well. Almost all operators in the `np` and `npx` module support running on a GPU.

```{.python .input  n=21}
y = np.random.uniform(size=(3,4), ctx=gpu)
x + y
```

Remember that if the inputs are not on the same GPU, you get an error.

## Run a neural network on a GPU

To run a neural network on a GPU, you only need to copy and move the input data and parameters to the GPU. Reuse the previously defined LeafNetwork in [Training Neural Networks](trainNN.md). The following code example shows this.

```{.python .input  n=16}
# The convolutional block has a convolution layer, a max pool layer and a batch normalization layer
def conv_block(filters, kernel_size=2, stride=2, batch_norm=True):
    conv_block = nn.HybridSequential()
    conv_block.add(nn.Conv2D(channels=filters, kernel_size=kernel_size, activation='relu'),
              nn.MaxPool2D(pool_size=4, strides=stride))
    if batch_norm:
        conv_block.add(nn.BatchNorm())
    return conv_block

# The dense block consists of a dense layer and a dropout layer
def dense_block(neurons, activation='relu', dropout=0.2):
    dense_block = nn.HybridSequential()
    dense_block.add(nn.Dense(neurons, activation=activation))
    if dropout:
        dense_block.add(nn.Dropout(dropout))
    return dense_block

# Create neural network blueprint using the blocks
class LeafNetwork(nn.HybridBlock):
    def __init__(self):
        super(LeafNetwork, self).__init__()
        self.conv1 = conv_block(32)
        self.conv2 = conv_block(64)
        self.conv3 = conv_block(128)
        self.flatten = nn.Flatten()
        self.dense1 = dense_block(100)
        self.dense2 = dense_block(10)
        self.dense3 = nn.Dense(2)
        
    def forward(self, batch):
        batch = self.conv1(batch)
        batch = self.conv2(batch)
        batch = self.conv3(batch)
        batch = self.flatten(batch)
        batch = self.dense1(batch)
        batch = self.dense2(batch)
        batch = self.dense3(batch)
        
        return batch
```

Load the saved parameters into GPU 0 directly as shown here, or use `net.collect_params().reset_ctx(gpu)` to change the device.

```{.python .input  n=20}
net.load_parameters('leaf_models.params', ctx=gpu)
```

Use the following command to create input data on GPU 0. The forward function will then run on GPU 0.

```{.python .input  n=22}
x = np.random.uniform(size=(1, 3, 128, 128), ctx=gpu)
net(x)
```

## Training with multiple GPUs

Finally, you can see how to use multiple GPUs to jointly train a neural network through data parallelism. Assume there are *n* GPUs. Split each data batch into *n* parts, and then each GPU will run the forward and backward passes using one part of the data.

First copy the data definitions with the following commands, and the transform functions from the tutorial [Training Neural Networks](trainNN.md).

```{.python .input}
# Import transforms as compose a series of transformations to the images
from mxnet.gluon.data.vision import transforms

jitter_param = 0.05

# mean and std for normalizing image value in range (0,1)
mean = [0.485, 0.456, 0.406]
std = [0.229, 0.224, 0.225]

training_transformer = transforms.Compose([
    transforms.Resize(size=224, keep_ratio=True),
    transforms.CenterCrop(128),
    transforms.RandomFlipLeftRight(),
    transforms.RandomColorJitter(contrast=jitter_param),
    transforms.ToTensor(),
    transforms.Normalize(mean, std)
])

validation_transformer = transforms.Compose([
    transforms.Resize(size=224, keep_ratio=True),
    transforms.CenterCrop(128),
    transforms.ToTensor(),
    transforms.Normalize(mean, std)
])

# Create data loaders
batch_size = 4
train_loader = gluon.data.DataLoader(train_dataset.transform_first(training_transformer),batch_size=batch_size, shuffle=True, try_nopython=True)
validation_loader = gluon.data.DataLoader(val_dataset.transform_first(validation_transformer), batch_size=batch_size, try_nopython=True)
test_loader = gluon.data.DataLoader(test_dataset.transform_first(validation_transformer), batch_size=batch_size, try_nopython=True)
```

### Define a helper function 
This is the same test function defined previously in the **Step 6**.
```{.python .input}
# Function to return the accuracy for the validation and test set
def test(val_data):
    acc = gluon.metric.Accuracy()
    for batch in val_data:
        data = batch[0]
        labels = batch[1]
        outputs = model(data)
        acc.update([labels], [outputs])
        
    _, accuracy = acc.get()
    return accuracy
```

The training loop is quite similar to that shown earlier. The major differences are highlighted in the following code.

```{.python .input}
# Diff 1: Use two GPUs for training.
available_gpus = [npx.gpu(i) for i in range(npx.num_gpus())]
num_gpus = 2
devices = available_gpus[:num_gpus]
print('Using {} GPUs'.format(len(devices)))

# Diff 2: reinitialize the parameters and place them on multiple GPUs
net.initialize(force_reinit=True, ctx=devices)

# Loss and trainer are the same as before
loss_fn = gluon.loss.SoftmaxCrossEntropyLoss()
optimizer = 'sgd'
optimizer_params = {'learning_rate': 0.001}
trainer = gluon.Trainer(net.collect_params(), optimizer, optimizer_params)

epochs = 2
accuracy = gluon.metric.Accuracy()
log_interval = 5

for epoch in range(10):
    train_loss = 0.
    tic = time.time()
    btic = time.time()
    accuracy.reset()
    for idx, batch in enumerate(train_loader):
        data, label = batch[0], batch[1]

        # Diff 3: split batch and load into corresponding devices
        data_list = gluon.utils.split_and_load(data, devices)
        label_list = gluon.utils.split_and_load(label, devices)

        # Diff 4: run forward and backward on each devices.
        # MXNet will automatically run them in parallel
        with autograd.record():
            outputs = [net(X)
                      for X in data_list]
            losses = [loss_fn(output, label)
                      for output, label in zip(outputs, label_list)]
        for l in losses:
            l.backward()
        trainer.step(batch_size)

        # Diff 5: sum losses over all devices. Here, the float 
        # function will copy data into CPU.
        train_loss += sum([float(l.sum()) for l in losses])
        accuracy.update(label_list, outputs)
        if log_interval and (idx + 1) % log_interval == 0:
            _, acc = accuracy.get()
     
            print(f"""Epoch[{epoch + 1}] Batch[{idx + 1}] Speed: {batch_size / (time.time() - btic)} samples/sec \
                  batch loss = {train_loss} | accuracy = {acc}""")
            btic = time.time()

    _, acc = accuracy.get()
    
    acc_val = test(validation_loader)
    print(f"[Epoch {epoch + 1}] training: accuracy={acc}")
    print(f"[Epoch {epoch + 1}] time cost: {time.time() - tic}")
    print(f"[Epoch {epoch + 1}] validation: validation accuracy={acc_val}")
```

## Next steps

Now you have completed training and predicting with a neural network by using NP on MXNet and
Gluon. You can check the guides to these two front ends: [What is NP on MXNet](../deepnumpy/index.html) and [gluon](../gluon_from_experiment_to_deployment.md).