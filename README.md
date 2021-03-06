## Notes

**The notebooks are not specifically written for speed, instead they aim to create an easy comparison between the frameworks. However, any suggestions on improving the training-time are welcome!**

**Notebooks are run on Nvidia K80 GPU (and in another branch on the M60), on [Microsoft Azure Data Science Virtual Machine for Linux (Ubuntu)](https://azuremarketplace.microsoft.com/en-us/marketplace/apps/microsoft-ads.linux-data-science-vm-ubuntu?tab=Overview), where frameworks have been updated to the latest version**

## Goal

Create a Rosetta Stone of deep-learning frameworks to allow data-scientists to easily leverage their expertise from one framework to another (by translating, rather than learning from scratch). Also, to make the models more transparent to comparisons in terms of training-time and default-options.

A lot of online tutorials use very-low level APIs, which are very verbose, and don't make much sense (given higher-level helpers being available) for most use-cases unless one plans to create new layers. Here we try to apply the highest-level API possible, conditional on being to override conflicting defaults, to allow an easier comparison between frameworks. It will demonstrated that the code structure becomes very similar once higher-level APIs are used and can be roughly represented as:

- Load data; x_train, x_test, y_train, y_test = cifar_for_library(channel_first=?, one_hot=?)
- Generate CNN/RNN symbol (usually no activation on final dense-layer)
- Specify loss (cross-entropy comes bundles with softmax), optimiser and initialise weights + sessions
- Train on mini-batches from train-set using custom iterator (common data-source for all frameworks)
- Predict on fresh mini-batches from test-set
- Evaluate accuracy

Since we are essentially comparing a series of deterministic mathematical operations (albeit with a random initialization), it does not make sense to me to compare the accuracy across frameworks and instead they are reported as **checks we want to match**, to make sure we are comparing the same model architecture. 

## Results

### CNN (VGG-style) on CIFAR-10 - Image Recognition

| DL Library                               | Test Accuracy (%) | Training Time (s) |
| ---------------------------------------- | ----------------- | ----------------- |
| [Caffe2](Caffe2_CNN.ipynb)               | 79                | 149               |
| [MXNet](MXNet_CNN.ipynb)                 | 77                | 149               |
| [Gluon](Gluon_CNN.ipynb)                 | 77                | 157               |
| [CNTK](CNTK_CNN.ipynb)                   | 78                | 166               |
| [PyTorch](PyTorch_CNN.ipynb)             | 78                | 168               |
| [Tensorflow](Tensorflow_CNN.ipynb)       | 78                | 173               |
| [Keras(CNTK)](Keras_CNTK_CNN.ipynb)      | 78                | 200               |
| [Chainer](Chainer_CNN.ipynb)             | 79                | 240               |
| [Keras(TF)](Keras_TF_CNN.ipynb)          | 77                | 252               |
| [Lasagne(Theano)](Theano_Lasagne_CNN.ipynb) | 77                | 253               |
| [Keras(Theano)](Keras_Theano_CNN.ipynb)  | 78                | 269               |

Input for this model is the standard [CIFAR-10 dataset](http://www.cs.toronto.edu/~kriz/cifar.html) containing 50k training images and 10k test images, uniformly split across 10 classes. Each 32 by 32 px image is supplied as a tensor of shape (3, 32, 32) with pixel intensity re-scaled from 0-255 to 0-1. For example: ![automobile](common/automobile10.PNG) with corresponding y=(0, 1, 0, 0, 0, 0, 0, 0, 0, 0) where labels=[airplane, automobile, bird, cat, deer, dog, frog, horse, ship, truck]

### RNN (GRU) on IMDB - Natural Language Processing (Sentiment Analysis)

| DL Library                          | Test Accuracy (%) | Training Time (s) |
| ----------------------------------- | ----------------- | ----------------- |
| [CNTK](CNTK_RNN.ipynb)              | 86                | 29                |
| [Pytorch](PyTorch_RNN.ipynb)        | 85                | 32                |
| [Tensorflow](Tensorflow_RNN.ipynb)  | 85                | 77                |
| [MXNet](MXNet_RNN.ipynb)            | 86                | 87                |
| [Keras(TF)](Keras_TF_RNN.ipynb)     | 85                | 205               |
| [Keras(CNTK)](Keras_CNTK_RNN.ipynb) | 86                | 215               |

Input for this model is the standard [IMDB movie review dataset](http://ai.stanford.edu/~amaas/data/sentiment/) containing 25k training reviews and 25k test reviews, uniformly split across 2 classes (positive/negative). Reviews are already downloaded as a tensor of word indexes e.g. (If you like adult comedy cartoons, like South Park) is received as (1 2 3 4 5 6 3 7 8). Processing follows [Keras](https://github.com/fchollet/keras/blob/master/keras/datasets/imdb.py) approach where start-character is set as 1, out-of-vocab (vocab size of 30k is used) represented as 2 and thus word-index starts from 3. Zero-padded / truncated to fixed axis of 150 words per review.

*Note: For CNTK I use optimized_rnnstack instead of Recurrence(LSTM()), since we have a vanilla RNN that can be easily reduced to CuDNN level. For more complicated variants (e.g. Layer Normalisation) one cannot use this. CNTK also supports [dynamic axes](https://cntk.ai/pythondocs/sequence.html) which means we don't need to pad the input to 150 words and can consume as-is, however since I could not find a way to do this with other frameworks I have fallen back to padding - which is a bit unfair on CNTK and understates its capabilities*

The classification model creates an embedding matrix of size (150x125) and then applies 100 gated recurrent units and takes as output the final output (not sequence of outputs and not hidden state). Any suggestions on alterations to this are welcome.

### Lessons Learned

The below offers some insights I gained after trying to match test-accuracy across frameworks and from all the GitHub issues/PRs raised.

1. The above examples (except for Keras), for ease of comparison, try to use the same level of API and so all use the same generator-function. For MXNet and CNTK I have experimented with a higher-level API, where I use the framework's training generator function. The speed improvement is neglible in this example because the whole dataset is loaded as NumPy array in RAM and the only processing done each epoch is a shuffle. I suspect the framework's generators perform the shuffle asynchronously. Curiously, it seems that the frameworks shuffle on a batch-level, rather than on an observation level, and thus ever so slightly decreases the test-accuracy (at least after 10 epochs). For scenarios where we have IO activity and perhaps pre-processing and data-augmentation on the fly, custom generators would have a much bigger impact on performance.


| DL Library                               | Test Accuracy (%) | Training Time (s) |
| ---------------------------------------- | ----------------- | ----------------- |
| [MXNet w/Generator](MXNet_CNN_highAPI.ipynb) | 77                | 147               |
| [CNTK w/Generator](CNTK_CNN_highAPI.ipynb) | 77                | 153               |

2. Enabling CuDNN's auto-tune/exhaustive search paramater (which selects the most efficient CNN algorithm for images of fixed-size) has a huge performance boost. This had to be manually enabled for Caffe2, PyTorch and Theano. It appears CNTK, MXNet and Tensorflow have this enabled by default. I'm not sure about Chainer. Yangqing mentions that the performance boost between cudnnGet (default) and cudnnFind is, however, much smaller on the Titan X GPU; it seems that the K80 + new cudnn makes the problem more promininet in this case. Running cudnnFind for every combination of size in object detection has serious performance regressions, however, so exhaustive_search should be disabled for object detection

3. When using Keras it's important to choose the [NCHW] ordering that matches the back-end framework. CNTK operates with channels first and by mistake I had Keras configured to expect channels last. It then must have changed the order at each batch which degraded performance severely.

4. Tensorflow, PyTorch, Caffe2 and Theano required a boolean supplied to the pooling-layer indicating whether we were training or not (this had a huge impact on test-accuracy, 72 vs 77%). Dropout should not be applied to test in this case.

5. Tensorflow was a bit annoying and required two more changes: speed was improved a lot by enabling TF_ENABLE_WINOGRAD_NONFUSED and also changing the dimensions supplied to channel first rather than last (data_format='channels_first'). Enabling the WINOGRAD for convolutions also, naturally, improved Keras with TF as a backend

6. Softmax is usually bundled with cross_entropy_loss() for most functions and it's worth checking if you need an activation on your final fully-connected layer to save time applying it twice

7. Kernel initializer for different frameworks can vary (I've found this to have +/- 1% effect on accuracy) and I try to specify xavier/glorot uniform whenever possible/not too verbose

8. Type of momentum implemented for SGD-momentum; I had to turn off unit_gain (which was on by default in CNTK) to match other frameworks' implementations

9. Caffe2 has an extra optimisation for the first layer of a network (no_gradient_to_input=1) that produces a small speed-boost by not computing gradients for input. It's possible that Tensorflow and MXNet already enable this by default. Computing this gradient could be useful for research purposes and for networks like deep-dream

10. Applying the ReLU activation after max-pooling (insteaad of before) means you perform a calculation after dimensionality-reduction and thus shave off a few seconds. This helped reduce MXNet time by 3 seconds

11. Some **further checks** which may be useful: 
   * specifying kernel as (3) becomes a symmetric tuple (3, 3) or 1D convolution (3, 1)?
   * strides (for max-pooling) are (1, 1) by default or equal to kernel (Keras does this)? 
   * default padding is usually off (0, 0)/valid but useful to check it's not on/'same'
   * is the default activation on a convolutional layer 'None' or 'ReLu' (Lasagne)
   * the bias initializer may vary (sometimes no bias is included)
   * gradient clipping and treatment of inifinty/NaNs may differ across frameworks
   * some frameworks support sparse labels instead of one-hot (which I use if available, e.g. Tensorflow has f.nn.sparse_softmax_cross_entropy_with_logits)
   * data-type assumptions may be different - I try to use float32 and int32 for X and y but, for example, torch needs double for y (to be coerced into torch.LongTensor(y).cuda)
   * if the framework has a slightly lower-level API make sure during testing you don't compute the gradient by setting something like training=False

12. Installing Caffe2 for python 3.5 proved a bit difficult so I wanted to share the process:
   ```
   # build as root
   sudo -s
   cd /opt/caffe2
   make clean
   git pull
   git checkout v0.8.1
   git submodule update
   export CPLUS_INCLUDE_PATH=/anaconda/envs/py35/include/python3.5m
   mkdir build
   cd build
   echo $PATH
   # CONFIRM that Anaconda is not in the path
   cmake .. -DBLAS=MKL -DPYTHON_INCLUDE_DIR=/anaconda/envs/py35/include/python3.5m -DPYTHON_LIBRARY=/anaconda/envs/py35/lib/libpython3.5m.so -DPYTHON_EXECUTABLE=/anaconda/envs/py35/bin/python
   make -j$(nproc)
   make install
   ```