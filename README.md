# Pytorch android examples

## Quick start with a HelloWorld example

[HelloWorld](https://github.com/pytorch/android-demo-app/tree/master/HelloWorldApp) is a simple image classification application that demonstrates how to use PyTorch android api.
This application runs TorchScript serialized TorchVision pretrained resnet18 model on static image which is packaged inside the app as android asset.

#### 1. Model preparation

Let’s start with model preparation. If you are familiar with PyTorch, you probably should already know how to train and save your model. In case you don’t, we are going to use a pre-trained image classification model(Resnet18), which is packaged in [TorchVision](https://pytorch.org/docs/stable/torchvision/index.html).
To install it, run the command below:
```
pip install torchvision
```

To serialize the model you can use python [script](https://github.com/pytorch/android-demo-app/blob/master/HelloWorldApp/trace_model.py) in the root folder of HelloWorld app:
```
import torch
import torchvision

model = torchvision.models.resnet18(pretrained=True)
model.eval()
example = torch.rand(1, 3, 224, 224)
traced_script_module = torch.jit.trace(model, example)
traced_script_module.save("app/src/main/assets/model.pt")
```
If everything works well, we should have our model - `model.pt` generated in the assets folder of android application. 
That will be packaged inside android application as `asset` and can be used on the device.

More details about TorchScript you can find in [tutorials on pytorch.org](https://pytorch.org/docs/stable/jit.html) 

#### 2. Cloning from github
```
git clone https://github.com/pytorch/android-demo-app.git
cd HelloWorldApp
```
If [android sdk]() and [android ndk]() are already installed you can install this application to the connected android device or emulator with:
```
./gradlew installDebug
```

We recommend you to open this project in [Android Studio](https://developer.android.com/studio),
in that case you will be able to install android ndk and android sdk using Android Studio UI. 

#### 3. Gradle dependencies

Pytorch android is added to the HelloWorld as [gradle dependencies](https://github.com/pytorch/android-demo-app/blob/master/HelloWorldApp/app/build.gradle#L28-L29) in build.gradle:

```
repositories {
    jcenter()
}

dependencies {
    implementation 'org.pytorch:pytorch_android:1.3.0'
    implementation 'org.pytorch:pytorch_android_torchvision:1.3.0'
}
```
Where `org.pytorch:pytorch_android` is the main dependency with pytorch android api, including libtorch native library for all 4 android abis (armeabi-v7a, arm64-v8a, x86, x86_64).
Further in this doc you can find how to rebuild it only for specific list of android abis. 

`org.pytorch:pytorch_android_torchvision` - additional library with utility functions for converting `android.media.Image` and `android.graphics.Bitmap` to tensors.

#### 4. Reading image from android asset

All logic happens in [`org.pytorch.helloworld.MainActivity`](https://github.com/pytorch/android-demo-app/blob/master/HelloWorldApp/app/src/main/java/org/pytorch/helloworld/MainActivity.java#L31-L69).
As a first step we read `image.jpg` to `android.graphics.Bitmap` using standard android api. 
```
Bitmap bitmap = BitmapFactory.decodeStream(getAssets().open("image.jpg"));
```

#### 5. Loading TorchScript Module
```
Module module = Module.load(assetFilePath(this, "model.pt"));
```
`org.pytorch.Module` represents `torch::jit::script::Module` that can be loaded with `load` method specifying file path to the serialized to file model.

#### 6. Preparing input
```
Tensor inputTensor = TensorImageUtils.bitmapToFloat32Tensor(bitmap,
    TensorImageUtils.TORCHVISION_NORM_MEAN_RGB, TensorImageUtils.TORCHVISION_NORM_STD_RGB);
```
`org.pytorch.torchvision.TensorImageUtils` is part of `org.pytorch:pytorch_android_torchvision` library.
`TensorImageUtils#bitmapToFloat32Tensor` method creates tensor in [torch vision format](https://pytorch.org/docs/stable/torchvision/models.html) using `android.graphics.Bitmap` as a source.

> All pre-trained models expect input images normalized in the same way, i.e. mini-batches of 3-channel RGB images of shape (3 x H x W), where H and W are expected to be at least 224. 
> The images have to be loaded in to a range of `[0, 1]` and then normalized using `mean = [0.485, 0.456, 0.406]` and `std = [0.229, 0.224, 0.225]`

`inputTensor`'s shape is `1x3xHxW`, where `H` and `W` are bitmap height and width appropriately. 

#### 7. Run Inference
 
```
Tensor outputTensor = module.forward(IValue.tensor(inputTensor)).getTensor();
float[] scores = outputTensor.getDataAsFloatArray();
```

`org.pytorch.Module.forward` method runs loaded module's `forward` method and gets result as `org.pytorch.Tensor` outputTensor with shape `1x1000`.

#### 8. Processing results
It's content is retrieved using `org.pytorch.Tensor.getDataAsFloatArray()` method that returns java array of floats with scores for every image net class.
 
After that we just find index with maximum score and retrieve predicted class name from `ImageNetClasses.IMAGENET_CLASSES` array that contains all ImageNet classes.

```
float maxScore = -Float.MAX_VALUE;
int maxScoreIdx = -1;
for (int i = 0; i < scores.length; i++) {
  if (scores[i] > maxScore) {
    maxScore = scores[i];
    maxScoreIdx = i;
  }
}
String className = ImageNetClasses.IMAGENET_CLASSES[maxScoreIdx];
```
 
In the following sections you can find detailed explanation of pytorch android api, code walk through for bigger [demo application](https://github.com/pytorch/android-demo-app/tree/master/PyTorchDemoApp), 
implementation details of api and how to customize and build it from the source. 

## Pytorch demo app

A bit bigger example of application, that does image classification from android camera output and text classification, you can find in the [same github repo](https://github.com/pytorch/android-demo-app/tree/master/PyTorchDemoApp). 

To get device camera output it uses [android CameraX API](https://developer.android.com/training/camerax
). All the logic that works with CameraX is separated to [`org.pytorch.demo.vision.AbstractCameraXActivity`](https://github.com/pytorch/android-demo-app/blob/master/PyTorchDemoApp/app/src/main/java/org/pytorch/demo/vision/AbstractCameraXActivity.java) class.


```
void setupCameraX() {
    final PreviewConfig previewConfig = new PreviewConfig.Builder().build();
    final Preview preview = new Preview(previewConfig);
    preview.setOnPreviewOutputUpdateListener(output -> mTextureView.setSurfaceTexture(output.getSurfaceTexture()));

    final ImageAnalysisConfig imageAnalysisConfig =
        new ImageAnalysisConfig.Builder()
            .setTargetResolution(new Size(224, 224))
            .setCallbackHandler(mBackgroundHandler)
            .setImageReaderMode(ImageAnalysis.ImageReaderMode.ACQUIRE_LATEST_IMAGE)
            .build();
    final ImageAnalysis imageAnalysis = new ImageAnalysis(imageAnalysisConfig);
    imageAnalysis.setAnalyzer(
        (image, rotationDegrees) -> {
          analyzeImage(image, rotationDegrees);
        });

    CameraX.bindToLifecycle(this, preview, imageAnalysis);
  }

  void analyzeImage(android.media.Image, int rotationDegrees)
```

Where `analyzeImage` method processes camera output, `android.media.Image`.

It uses aforementioned [`TensorImageUtils.imageYUV420CenterCropToFloat32Tensor`](https://github.com/pytorch/pytorch/blob/master/android/pytorch_android_torchvision/src/main/java/org/pytorch/torchvision/TensorImageUtils.java#L90) method to convert `android.media.Image` in `YUV420` format to input tensor.

After getting predicted scores from the model it finds top K classes with the highest scores and shows on the UI.
