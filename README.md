
<!-- README.md is generated from README.Rmd. Please edit that file -->

# Flood-modelling

### Authors: Jhony Sánchez & Rodrigo Malagón

<!-- badges: start -->
<!-- badges: end -->

``` r
library(keras)
library(tensorflow)
library(tfdatasets)
library(purrr)
library(ggplot2)
library(rsample)
library(stars)
#> Cargando paquete requerido: abind
#> Cargando paquete requerido: sf
#> Linking to GEOS 3.12.1, GDAL 3.8.4, PROJ 9.3.1; sf_use_s2() is TRUE
library(raster)
#> Cargando paquete requerido: sp
library(reticulate)
library(mapview)
```

# I. Pre-processing

We begin our workflow with a preprocessing stage whose main objective is
to obtain a labelled dataset consisting of tile multispectral images of
the area of study and their corresponding labelling masks. The workflow
was conducted on QGis.

After the images were collected from the open data [Xpress
platform](https://xpress.maxar.com/?lat=0&lon=0&zoom=3.0) by Maxar, we
assemble a mosaic of the study area. The mosaic was clipped into a
single regular square raster.

Afterwards, we break the area of study into 12 raster images of uniform
size to break the optimize furthcoming preprocessing stages with these
smaller inputs.

The `GenericRegionMerging` segmentation tool from the OTB toolbox is
then used per image to create uniform patches accross each image and
thus facilitate the labelling procedure. To this end, the `scale factor`
was set to $350$ and the spectral and spatial homogeneity parameters to
the default values of $0.5$.

After this, the visual identification and manual labelling of *flooded*
segments was performed. Then, we transform the labelled vectors into
binary images and perform a tiling of the whole dataset into more than
$8500$ images of size $128\times 128$.

# II. Modelling

In this main stage of the project, we follow the general workflow
exposed in the tutorial [Introduction to Deep Learning in R for the
Analysis of UAV-based Remote Sensing
Data](https://dachro.github.io/ogh_summer_school_2020/Tutorial_DL_UAV.html#from_image_to_pixel-by-pixel_classification).
As key changes to this general workflow, we adapt some processing steps
to the multispectral data structure of our project and perform a
stratification of masks to enhance the dataset split into training and
testing sets.

## II.1 Dataset preparation

In this stage, we begin defining some helping functions to process the
tile imagery in preparation to the tensor dataset creation. We stress
that though this process may be carried in general after the creation of
the tensor dataset, we perform this processing due to the lack of
straightforward reading capabilities of `.tiff` files by the `keras`
API.

``` r
# Processing functions
extract_matrix <- function(file){
  raster <- read_stars(file)
  matrix <- raster[[1]]
  matrix
}

extract_mask <- function(file){
  raster <- read_stars(file)
  matrix <- raster[,,][[1]]
  matrix
}


normalize_matrix <- function(matrix){
  matrix/max(matrix)
}
```

Due to processing limitations, we select a subset of $5000$ labelled
tiles to carry out the CNN learning stage:

``` r
path_to_imgs <- './Data/10300100FAA4CA00/tiles_image/'
path_to_masks <- './Data/10300100FAA4CA00/tiles_labels/'

files <- data.frame(
  img_path = list.files(path_to_imgs, full.names = TRUE, pattern = "*.TIF$"),
  mask_path = list.files(path_to_masks, full.names = TRUE, pattern = "*.TIF$")
)
files<-files[1:5000,]
```

We then extract the multispectral images and normalize their values to
work with features in the $[0,1]$ interval.

``` r
files$img <- lapply(files$img_path,extract_matrix)

files$mask <- lapply(files$mask_path,extract_mask)


files$img <- lapply(files$img,normalize_matrix)
```

We filter the tiles with a $128\times 128-$size to produce a uniform
dataset to be fed to the CNN. Furthermore, we enrich our dataset with an
extracted *mask weight* that ranks masks from 0 to 10 in terms of
quantity of 1’s.

``` r
# Data size filtering
files$dim_x <- lapply(files$img,function(x){dim(x)['x']})|> unlist()
files <- files[files[,'dim_x'] == 128,]
files$dim_y <- lapply(files$img,function(x){dim(x)['y']}) |> unlist()
files <- files[files[,'dim_y'] == 128,]

# Mask weight creation
files$mask_weight <- unlist(lapply(files$mask,function(x){as.integer(10*round(mean(x),1))}))
      #integer type enforced to be passed as strata parameter in the next step
```

We perform a data split with a stratification of our dataset using the
*mask weight* created in the last step.

``` r
# Dataset split
files_split <- initial_split(files[c('img','mask','mask_weight')],prop = 0.8,strata=mask_weight)

training_dataset <- tensor_slices_dataset(training(files_split)[c('img','mask')])
      #we leave the mask weight out of the tensor dataset, as this component cannot be fed to the CNN
testing_dataset <- tensor_slices_dataset(testing(files_split)[c('img','mask')])
```

One further change of type to single-precision floating-point is
performed to our dataset to ensure a lower memory demand.

``` r
training_dataset <- dataset_map(training_dataset, function(.x)
   list_modify(.x, img = tf$image$convert_image_dtype(.x$img, dtype = tf$float32),
               mask = tf$image$convert_image_dtype(.x$mask, dtype = tf$float32)))

testing_dataset <- dataset_map(testing_dataset, function(.x)
   list_modify(.x, img = tf$image$convert_image_dtype(.x$img, dtype = tf$float32),
                mask = tf$image$convert_image_dtype(.x$mask, dtype = tf$float32)))
```

Finally, we shuffle the training dataset and batch both the training and
testing datasets.

``` r
dataset_iterator <- as_iterator(training_dataset)
dataset_list <- iterate(dataset_iterator)
(training_dataset_size <- length(dataset_list))
#> [1] 3942

training_dataset <- dataset_shuffle(training_dataset, buffer_size = training_dataset_size)
training_dataset <- dataset_batch(training_dataset, 10L)
training_dataset <- dataset_map(training_dataset, unname)

testing_dataset <- dataset_batch(testing_dataset, 10L)
testing_dataset <- dataset_map(testing_dataset, unname)
```

# II.2 CNN model

In this stage we define the CNN architecture that we use for the
modelling. To this end, we use the U-net architecture exposed in the
tutorial [Introduction to Deep Learning in R for the Analysis of
UAV-based Remote Sensing
Data](https://dachro.github.io/ogh_summer_school_2020/Tutorial_DL_UAV.html#from_image_to_pixel-by-pixel_classification).
The meaningful change performed to this architecture is the adaptation
of the `input shape` of the CNN to our tiles of size $128\times 128$.

``` r
## we start with the "contratcing path"##
# input
input_tensor <- layer_input(shape = c(128,128,8))

#conv block 1
unet_tensor <- layer_conv_2d(input_tensor,filters = 64,kernel_size = c(3,3), padding = "same",activation = "relu")
conc_tensor2 <- layer_conv_2d(unet_tensor,filters = 64,kernel_size = c(3,3), padding = "same",activation = "relu")
unet_tensor <- layer_max_pooling_2d(conc_tensor2)

#conv block 2
unet_tensor <- layer_conv_2d(unet_tensor,filters = 128,kernel_size = c(3,3), padding = "same",activation = "relu")
conc_tensor1 <- layer_conv_2d(unet_tensor,filters = 128,kernel_size = c(3,3), padding = "same",activation = "relu")
unet_tensor <- layer_max_pooling_2d(conc_tensor1)

#"bottom curve" of unet
unet_tensor <- layer_conv_2d(unet_tensor,filters = 256,kernel_size = c(3,3), padding = "same",activation = "relu")
unet_tensor <- layer_conv_2d(unet_tensor,filters = 256,kernel_size = c(3,3), padding = "same",activation = "relu")

##  this is where the expanding path begins ##

# upsampling block 1
unet_tensor <- layer_conv_2d_transpose(unet_tensor,filters = 128,kernel_size = c(2,2),strides = 2,padding = "same")
unet_tensor <- layer_concatenate(list(conc_tensor1,unet_tensor))
unet_tensor <- layer_conv_2d(unet_tensor, filters = 128, kernel_size = c(3,3),padding = "same", activation = "relu")
unet_tensor <- layer_conv_2d(unet_tensor, filters = 128, kernel_size = c(3,3),padding = "same", activation = "relu")

# upsampling block 2
unet_tensor <- layer_conv_2d_transpose(unet_tensor,filters = 64,kernel_size = c(2,2),strides = 2,padding = "same")
unet_tensor <- layer_concatenate(list(conc_tensor2,unet_tensor))
unet_tensor <- layer_conv_2d(unet_tensor, filters = 64, kernel_size = c(3,3),padding = "same", activation = "relu")
unet_tensor <- layer_conv_2d(unet_tensor, filters = 64, kernel_size = c(3,3),padding = "same", activation = "relu")

# output
unet_tensor <- layer_conv_2d(unet_tensor,filters = 1,kernel_size = 1, activation = "sigmoid")

# combine final unet_tensor (carrying all the transformations applied through the layers)
# with input_tensor to create model

unet_model <- keras_model(inputs = input_tensor, outputs = unet_tensor)
```

We then compile and run the model with the `RMSprop` weight optimization
algorithm and an optimal learning rate set to $1\times 10^{-4}$. The
loss function minimized by the model learning is set to
`Binary crossentropy` function and the metrics for the evaluation of the
model as the `metric binary accuracy`, as our model predicts on two
classes across each tile.

``` r
# Model compilation
compile(
  unet_model,
  optimizer = optimizer_rmsprop(learning_rate = 1e-4),
  loss = "binary_crossentropy",
  metrics = c(metric_binary_accuracy)
  )
```

``` r
# Model learning and evaluation metrics
diagnostics <- fit(unet_model,
               training_dataset,
               epochs = 10,
               validation_data = testing_dataset)
#> Epoch 1/10
#> 395/395 - 1142s - loss: 0.1969 - binary_accuracy: 0.9388 - val_loss: 0.1894 - val_binary_accuracy: 0.9418 - 1142s/epoch - 3s/step
#> Epoch 2/10
#> 395/395 - 1100s - loss: 0.1673 - binary_accuracy: 0.9490 - val_loss: 0.1897 - val_binary_accuracy: 0.9408 - 1100s/epoch - 3s/step
#> Epoch 3/10
#> 395/395 - 1109s - loss: 0.1620 - binary_accuracy: 0.9495 - val_loss: 0.1737 - val_binary_accuracy: 0.9458 - 1109s/epoch - 3s/step
#> Epoch 4/10
#> 395/395 - 1077s - loss: 0.1528 - binary_accuracy: 0.9502 - val_loss: 0.1754 - val_binary_accuracy: 0.9468 - 1077s/epoch - 3s/step
#> Epoch 5/10
#> 395/395 - 1082s - loss: 0.1448 - binary_accuracy: 0.9516 - val_loss: 0.1537 - val_binary_accuracy: 0.9484 - 1082s/epoch - 3s/step
#> Epoch 6/10
#> 395/395 - 1096s - loss: 0.1400 - binary_accuracy: 0.9518 - val_loss: 0.1617 - val_binary_accuracy: 0.9452 - 1096s/epoch - 3s/step
#> Epoch 7/10
#> 395/395 - 1080s - loss: 0.1370 - binary_accuracy: 0.9525 - val_loss: 0.1487 - val_binary_accuracy: 0.9462 - 1080s/epoch - 3s/step
#> Epoch 8/10
#> 395/395 - 1109s - loss: 0.1340 - binary_accuracy: 0.9521 - val_loss: 0.1484 - val_binary_accuracy: 0.9486 - 1109s/epoch - 3s/step
#> Epoch 9/10
#> 395/395 - 1136s - loss: 0.1313 - binary_accuracy: 0.9523 - val_loss: 0.1586 - val_binary_accuracy: 0.9488 - 1136s/epoch - 3s/step
#> Epoch 10/10
#> 395/395 - 1086s - loss: 0.1300 - binary_accuracy: 0.9521 - val_loss: 0.1399 - val_binary_accuracy: 0.9490 - 1086s/epoch - 3s/step

plot(diagnostics)
```

![](README_files/figure-gfm/unnamed-chunk-12-1.png)<!-- -->

The model can be locally saved for further use:

``` r
# save_model_hdf5(unet_model,filepath = "./FloodsModel.h5")
```

## II.3 Model prediction

In this stage we leverage the learned model to visually evaluate the
class prediction on the testing tiles.

``` r
flood_model <- load_model_hdf5("./FloodsModel.h5")
```

After loading the previously saved model, we select an image to be
tested in this prediction stage. To assemble a visual comparison between
the expected and the actual prediction (mask vs. predicted mask), we use
of the `magick` library to stack these images.

``` r
pred <- predict(object = flood_model,testing_dataset)
#> 99/99 - 51s - 51s/epoch - 512ms/step
image_num <- 30
pred_img <- as.raster(pred[image_num,,,]) |> magick::image_read()
mask <- as.raster(testing(files_split)[image_num,c('mask')][[1]]) |> magick::image_read()
img <- as.raster(testing(files_split)[image_num,c('img')][[1]][,,4:2]) |> magick::image_read()


out <- magick::image_append(c(
  magick::image_append(mask, stack = TRUE),
  magick::image_append(img, stack = TRUE),
  magick::image_append(pred_img, stack = TRUE)
)
)

plot(out)
```

![](README_files/figure-gfm/unnamed-chunk-15-1.png)<!-- -->
