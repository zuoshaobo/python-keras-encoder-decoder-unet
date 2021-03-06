# Vehicle Recognition Using Encoder-Decoder Networks

This presents two mechanisms for performing segmentation 
of image data with respect to cars, a classic encoder-decoder
and the new `U-Net` encoder-decoder.  

The data source is [Udacity's annotated driving dataset](https://github.com/udacity/self-driving-car/tree/master/annotations).
This data is used as a source of images and result masks
for training networks to directly map an input image to a 
car segmentation mask.  

The annotations from the datasets have some aspects that are
inconsistent and incorrectly labeled.  The two datasets have
different formats for the `.csv` files.  I added a header to 
the Autti dataset, replaced the space delimiters with commas
and removed the string delimiters from the label.  This dataset
also has sub-labels for traffic light color, so an extra header
column was added for that.  Both of the datasets had incorrect
identification of the bounds, with `xmax` and `ymin` swapped.
This is easy to identify by plotting bounding boxes on the 
original image.  If you intend to use this code with these 
datasets, you will need to udpate the `.csv` files for both
to meet these requirements. 

The approach I took to the masking was to generate the masks 
and resize the images in a completely separate step from training
the network.  The reason for this is I would prefer to simply have a 
collection of segmentation training data and not have to decode it 
each time it is run.  I also would like this training set for 
use in the future without having to remember that the bounding boxes
required correction.  As a bonus, it loads much faster as small images
from SSD.  Due to GPU memory constraints I decided to 
resize to 240x160.  This is a slightly different
aspect ratio than the original image, but the actual scaled dimension
of 240x150 has problems with the max-pooling and up-sampling in the model
due to divisibility.  For three-channel `unit8` feature images and single-channel 
`float32` point label segmentation masks, this yields a total consumption 
for all 22,065 images of just under 6GB.

For training, the resized images are loaded.  The feature images are 
left as RGB, whereas the label images are grayscale.  OpenCV loads our 
grayscale images as color, so we deliberately have to transform back
to grayscale.  The label is forced to an appropriate shape as normalized
`float32`. Note pre-initializing `numpy` arrays for loading, as this 
conserves memory for large datasets.  

Further transformations are defined for the following operations, 
which are applied by the Keras generator:

 * Luminance to simulate different lighting conditions
 * Translation both horizontal and vertical to simulate different car positions
 * Expansion with unconstrained aspect ratio to simulate different car geometries

All of these are applied to the feature images, but only the geometric
transformations are applied to the label images.  The transformations
are applied in a Keras generator for augmentation.  The generator 
supports batches because batching allows us to train faster by not
making backpropagation steps for each feature-label pair.  One has to be
particularly careful with the dimension of tensors in the translation
and expansion.  The reason for this is that the masks are pre-normalized,
and when OpenCV performs operations on a single channel, the assumption is
made to not make the single-dimensional data abstract to multiple dimensions.
This is not how TensorFlow sees the world, so we have to reshape it.

The loss function used is the intersection over union measure.  Well, the 
negative of the intersection over union, as the function itself is a goodness
of fit function. This simply measures the similarity of the two images by computing 
the relative overlap.  The intersection over union function itself is actually 
somewhat of a challenge to compute in Keras because conditional counting is
difficult.  You can easily use the `K.map_fn` function on a flattened label,
but creating a mapping function that returns the right Tensor (and not a boolean
Tensor) is difficult.  `K.switch`, for example, does not work without a constructor
for a Tensor, and TensorFlow constants are not valid.  In any case, the standard
trick is to use something like enough to the intersection over union metric
in kind, but tractable and preferably fast.  The most obvious simplification is to
just wave our hands on the specifics of the intersection calculation and just
use the product of the Tensors instead of the count of the common support.
To simplify this even further, we can replace the union calculation with
an estimator that is biased away from zero.  These two intersection over
union estimators perform differently, and affect hyperparameters differently.

There are two candidate models, both encoder-decoders. The first model is simple 
and is able to be specified using a Keras `Sequential` object.  The second model is what is called 
a `U-Net` because in certain diagrams the model looks somewhat like a `U`.  It is
quite similar to the first model, but between similar convolutional layers 
there is a merge of the layers.  This has the effect of allowing deep convolution
layers to be merged with more shallow convolution layers, and has been found to
help increase performance in segmentation problems ([U-Net: Convolutional Networks for Biomedical Image Segmentation](https://arxiv.org/pdf/1505.04597.pdf)).  This model cannot be
implemented using a Keras `Sequential` object because the merging is not 
sequential. 

You will note that the border mode is `same`.  Unlike a network where the top end
feeds fully connected layers, with both of these networks we are most interested in
preserving the layer size.  That is to say, the output prediction needs to be the 
same size as the input feature image in order for us to be able to compute our
intersection over union properly.  The simplest way to do this is to use `same` to
make the size under max-pooling and up-sampling invariant when using the same stride.

Given the time it takes to train the network, it is prudent to use Keras' 
`ModelCheckpoint` functionality.  This saves after each epoch of the loss is 
better than all previous losses.  This ensures that we get the best solution,
even when further training fails to reduce loss further, or even increases it.

A test script was added that takes images and applies the segmentation as an
overlay.
