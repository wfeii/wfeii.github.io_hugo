---
title: "ImageLoader的设计"
date: 2017-11-04
categories:
 - 开源库
tags:
 - ImageLoader
toc: true
toc_label: "ImageLoader的设计"
---

## ImageLoader解决的问题

分析源码之前我们需要知道ImageLoader为什么要出现，或者说ImageLoader是用来解决什么问题的。总结了一下几点:

- 对于加载网络的图片是比较耗时的，而对于耗时的排序基本是 网络>磁盘>内存，因此我们需要尽量减少对于网络的请求次数，我们可以牺牲磁盘和内存来达到尽量减少网络请求的加载，这就是我们所说的三级缓存，先从内存获取，然后从磁盘获取，最后再从网络获取。所以ImageLoader需要对此问题给出解决方案。
- 图片本身会占用大量的内存，如何做到内存的优化？ImageLoader需要对此问题给出解决方案，基本的解决方案是
  - 图片的大小压缩，对应于Bitmap就是其大小
  - 图片的色彩压缩，对应于Bitmap就是像素的标识是565还是8888
- 对于加载图片需要在子线程中运行，如何合理的处理线程需要考虑
- 扩展性，对于此问题不实现也可以，但写出来的代码绝对不是好代码，因为无法扩展
  - 显示的时候，对返回的Bitmap做处理，比如说我们需要倒圆角，圆形图，那ImageLoader是否可以给予支持，给用户更好的使用。(ImageLoader可以不给予实现，但是此实现会极大的提高ImageLoader的易用性)。
  - 对于图片的加载，我们需要从网络加载，也可以从磁盘加载，而且网络加载是否可以引入其他的库比如HttpClient等等。如何做到这些呢？ImageLoader最好给予实现。
- 易用性，既然是库，给client使用时候就需要考虑



以[Android-Universal-Image-Loader](https://github.com/nostra13/Android-Universal-Image-Loader)为例

[Android-Universal-Image-Loader](https://github.com/nostra13/Android-Universal-Image-Loader)流程
![流程](http://img.blog.csdn.net/20170521155701391?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvd2ZlaWk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

ImageLoder对外充当高层接口，ImageLoaderEngine管理线程，而ImageDecoder和ImageDownloader用于加载图片，而BitmapDisplayer用于显示Bitmap到ImageView上。

对外接口结构
![对外接口](http://img.blog.csdn.net/20170521155933665?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvd2ZlaWk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


供客户端接口是通过Facade模式进行封装，把底层细节隐藏起来，提供给用户使用的是ImageLoader类和ImageLoaderConfiguraton类。客户端不需要涉及底层细节，提高易用性。

## 源码分析  

下面以ImageLoader的displayImage为例，看看如何从网络获取图片，如何把图片设置到ImageView上。
```
    public void displayImage(String uri, ImageAware imageAware, DisplayImageOptions options,
          ImageSize targetSize, ImageLoadingListener listener, ImageLoadingProgressListener progressListener) {
     ......
       // 获取控件大小，后面我们需要根据此来压缩图片
       if (targetSize == null) {
          targetSize = ImageSizeUtils.defineTargetSizeForView(imageAware, configuration.getMaxImageSize());
       }
      // 根据Uri生成一个key，这个key非常重要，在ListView中itemView复用的时候通过key识别
       String memoryCacheKey = MemoryCacheUtils.generateKey(uri, targetSize);
       engine.prepareDisplayTaskFor(imageAware, memoryCacheKey);

      //现在先从memory中看是否能获取到。
       Bitmap bmp = configuration.memoryCache.get(memoryCacheKey);
       if (bmp != null && !bmp.isRecycled()) {
          // 这里是可以获取到Bitmap的代码，把Bitmap展示到ImageView中，可以查看ImageLoader的代码详细了解
       } else {
          if (options.shouldShowImageOnLoading()) {
             imageAware.setImageDrawable(options.getImageOnLoading(configuration.resources));
          } else if (options.isResetViewBeforeLoading()) {
             imageAware.setImageDrawable(null);
          }

          ImageLoadingInfo imageLoadingInfo = new ImageLoadingInfo(uri, imageAware, targetSize, memoryCacheKey,
                options, listener, progressListener, engine.getLockForUri(uri));
          LoadAndDisplayImageTask displayTask = new LoadAndDisplayImageTask(engine, imageLoadingInfo,
                defineHandler(options));
         //如果是同步加载，直接运行task，否则直接ImageLoaderEngine来提交工作线程处理
          if (options.isSyncLoading()) {
             displayTask.run();
          } else {
             engine.submit(displayTask);
          }
       }
    }

```
从上面的代码中可以看出先从内存中获取，获取不到的话会让LoadAndDisplayImageTask处理，我们一起分析一下LoadAndDisplayImageTask。

#### LoadAndDisplayImageTask分析
```
   public void run() {
      //如果pause就需要直接退出，比如我们的Activity进入pause，我们可以调用ImageLoader的
       if (waitIfPaused()) return;
       if (delayIfNeed()) return;
       Bitmap bmp;
       try {
         //先从内存中获取，
          bmp = configuration.memoryCache.get(memoryCacheKey);
          if (bmp == null || bmp.isRecycled()) {
            //没有获取到通过相应的方法获取。
             bmp = tryLoadBitmap();
          } else {
             loadedFrom = LoadedFrom.MEMORY_CACHE;
             L.d(LOG_GET_IMAGE_FROM_MEMORY_CACHE_AFTER_WAITING, memoryCacheKey);
          }

          if (bmp != null && options.shouldPostProcess()) {
             L.d(LOG_POSTPROCESS_IMAGE, memoryCacheKey);
             bmp = options.getPostProcessor().process(bmp);
             if (bmp == null) {
                L.e(ERROR_POST_PROCESSOR_NULL, memoryCacheKey);
             }
          }
          checkTaskNotActual();
          checkTaskInterrupted();
       } catch (TaskCancelledException e) {
          fireCancelEvent();
          return;
       } finally {
          loadFromUriLock.unlock();
       }
    	// 创建DisplayBitmapTask去加载Bitmap到ImageView上
       DisplayBitmapTask displayBitmapTask = new DisplayBitmapTask(bmp, imageLoadingInfo, engine, loadedFrom);
       runTask(displayBitmapTask, syncLoading, handler, engine);
    }
```

1. 还是先从内存中获取Bitmap，存在多线程操作，因此这里会重新从内存总获取一次。
2. 调用tryLoadBitmap获取Bitmap
3. 通过DisplayBitmapTask对象来显示Bitmap

对于tryLoadBitmap方法的操作如下
```
    private Bitmap tryLoadBitmap() throws TaskCancelledException {
       Bitmap bitmap = null;
       try {
          File imageFile = configuration.diskCache.get(uri);
          if (imageFile != null && imageFile.exists() && imageFile.length() > 0) {
             L.d(LOG_LOAD_IMAGE_FROM_DISK_CACHE, memoryCacheKey);
             loadedFrom = LoadedFrom.DISC_CACHE;

             checkTaskNotActual();
            //通过磁盘缓存获取
             bitmap = decodeImage(Scheme.FILE.wrap(imageFile.getAbsolutePath()));
          }
          if (bitmap == null || bitmap.getWidth() <= 0 || bitmap.getHeight() <= 0) {
             L.d(LOG_LOAD_IMAGE_FROM_NETWORK, memoryCacheKey);
             loadedFrom = LoadedFrom.NETWORK;
    		 ......
              // 通过网络获取
             bitmap = decodeImage(imageUriForDecoding);

             if (bitmap == null || bitmap.getWidth() <= 0 || bitmap.getHeight() <= 0) {
                fireFailEvent(FailType.DECODING_ERROR, null);
             }
          }
       }
      ......
       return bitmap;
    }
```

1. 先从磁盘中获取Bitmap
2. 然后从网络获取

下面分析一下decodeImage方法

```
    private Bitmap decodeImage(String imageUri) throws IOException {
       ViewScaleType viewScaleType = imageAware.getScaleType();
       ImageDecodingInfo decodingInfo = new ImageDecodingInfo(memoryCacheKey, imageUri, uri, targetSize, viewScaleType,
             getDownloader(), options);
       return decoder.decode(decodingInfo);
    }
```

分析一下ImageDecoder的decode方法，而我们默认的BaseImageDecoder。

以BaseImageDecoder为例
```
    public Bitmap decode(ImageDecodingInfo decodingInfo) throws IOException {
       Bitmap decodedBitmap;
       ImageFileInfo imageInfo;

      //通过ImageDownloader获取到InputStream对象
       InputStream imageStream = getImageStream(decodingInfo);
       if (imageStream == null) {
          L.e(ERROR_NO_IMAGE_STREAM, decodingInfo.getImageKey());
          return null;
       }
      //创建Bitmap
       try {
          imageInfo = defineImageSizeAndRotation(imageStream, decodingInfo);
          imageStream = resetStream(imageStream, decodingInfo);
          Options decodingOptions = prepareDecodingOptions(imageInfo.imageSize, decodingInfo);
          decodedBitmap = BitmapFactory.decodeStream(imageStream, null, decodingOptions);
       } finally {
          IoUtils.closeSilently(imageStream);
       }

      // 处理Bitmap类。
       if (decodedBitmap == null) {
          L.e(ERROR_CANT_DECODE_IMAGE, decodingInfo.getImageKey());
       } else {
          decodedBitmap = considerExactScaleAndOrientatiton(decodedBitmap, decodingInfo, imageInfo.exif.rotation,
                imageInfo.exif.flipHorizontal);
       }
       return decodedBitmap;
    }
```

最主要的通过ImageDownloader获取到InputStream对象。后面还会详细分析。

还有两个没有分析，一个是DisplayBitmapTask如何显示Bitmap的，一个是ImageLoaderEngine如何加载管理线程的。

### DisplayBitmapTask分析

DisplayBitmapTask继承Runnable
```
    public void run() {
       if (imageAware.isCollected()) {
          L.d(LOG_TASK_CANCELLED_IMAGEAWARE_COLLECTED, memoryCacheKey);
          listener.onLoadingCancelled(imageUri, imageAware.getWrappedView());
       } else if (isViewWasReused()) {
          L.d(LOG_TASK_CANCELLED_IMAGEAWARE_REUSED, memoryCacheKey);
          listener.onLoadingCancelled(imageUri, imageAware.getWrappedView());
       } else {
          L.d(LOG_DISPLAY_IMAGE_IN_IMAGEAWARE, loadedFrom, memoryCacheKey);
          displayer.display(bitmap, imageAware, loadedFrom);
          engine.cancelDisplayTaskFor(imageAware);
          listener.onLoadingComplete(imageUri, imageAware.getWrappedView(), bitmap);
       }
    }
```

通过BitmapDisplayer的对象来显示的。

```
    public final class SimpleBitmapDisplayer implements BitmapDisplayer {
       @Override
       public void display(Bitmap bitmap, ImageAware imageAware, LoadedFrom loadedFrom) {
          imageAware.setImageBitmap(bitmap);
       }
    }
```

### ImageLoaderEngine

ImageLoaderEngine主要是task的分发执行
```
    void submit(final LoadAndDisplayImageTask task) {
       taskDistributor.execute(new Runnable() {
          @Override
          public void run() {
             File image = configuration.diskCache.get(task.getLoadingUri());
             boolean isImageCachedOnDisk = image != null && image.exists();
             initExecutorsIfNeed();
             if (isImageCachedOnDisk) {
                taskExecutorForCachedImages.execute(task);
             } else {
                taskExecutor.execute(task);
             }
          }
       });
    }
```
## 模块结构

主要功能有一下几点

### 线程池管理

线程池的管理主要是由ImageLoaderEngine负责。
![线程池](http://img.blog.csdn.net/20170521160445980?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvd2ZlaWk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
对于ImageLoaderEngine的线程池主要有三个

- taskDistributor 用于分发Task,具体可查看ImageLoaderEngine的submit方法

- taskExecutor 用于从网络上下载
- taskExecutorForCachedImages 用于从磁盘缓存中获取，或者提交ProcessAndDisplayImageTask时候使用

考虑一种情况，在Activity中有一个ListView而itemView中会从Uri中获取图片，在Activity中我们需要做什么操作呢？

- 当Activity处于paused的时候，现在界面不需要刷新，我们可以调用ImageLoader的pause来暂定新的请求加载
- 当Activity处于stop的时候，界面不可见，我们可以调用stop来暂定所有的线程
- 当Activity处于destory的时候，调用destroy来销毁

这样做到了加载与Activity声明周期相关联。

### 缓存
这里以内存缓存为例
![缓存](http://img.blog.csdn.net/20170521160536497?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvd2ZlaWk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

抽象出MemoryCache接口，而外界只是引用其接口，DiskCache与此类似

### 下载解析
![这里写图片描述](http://img.blog.csdn.net/20170521160630324?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvd2ZlaWk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


对于ImageDecoder解析是根据ImageDowloader的getStream获取InputStream，然后根据InputStream来解析出Bitmap。对外也都是引用均为接口。

而ImageDownloader是通过装饰模式进行组织，BaseImageDownloader为ConcreateComponent。而SlowNetWorkImageDowloader为Decorator。

- 实现自己的下载，可以根据ImageDownloader来实现。

- 而实现自己的图片解析器，可以根据ImageDecoder。

### 显示

对于显示除了DisplayBitmapTask类外最主要的就是下面类图。

![显示](http://img.blog.csdn.net/20170521160659997?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvd2ZlaWk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

对于BitmapDisplayer类图是对Bitmap进行进一步处理的过程，处理成圆形等等。

对于ImageAware的类图主要是处理View的显示，弱引用View避免内存泄露，其中是通过**模板方法**模式组织。

ViewAware
```
    @Override
    public boolean setImageBitmap(Bitmap bitmap) {
       if (Looper.myLooper() == Looper.getMainLooper()) {
          View view = viewRef.get();
          if (view != null) {
             setImageBitmapInto(bitmap, view);
             return true;
          }
       } else {
          L.w(WARN_CANT_SET_BITMAP);
       }
       return false;
    }
```
对于ViewAware子类只需要实现相应的抽象方法。



## ImageLoader问题解答

- 加载网络的图片是比较耗时，ImageLoader使用三级缓存进行处理。
- 图片内存占用优化，Android-Universal-Image-Loader中ImageLoaderConfigue可以配置最大的长宽，而且还会根据View的大小计算出相应的图片大小。在ImageSizeUtils中获取ImageSize，而对于ImageDecoder中根据ImageSize进行计算取样（Glide会缓存两份，一份是大图，一份是缩略图，对于控件比较小，直接从缩略图中加载，以后分析Glide的时候在详细讲解）。

 ```
public static ImageSize defineTargetSizeForView(ImageAware imageAware, ImageSize maxImageSize) {
           int width = imageAware.getWidth();
           if (width <= 0) width = maxImageSize.getWidth();

           int height = imageAware.getHeight();
           if (height <= 0) height = maxImageSize.getHeight();

           return new ImageSize(width, height);
        }
```

  - 对于色彩的优化，这里没有指定，默认是Bitmap.Config.ARGB_8888，如果需要优化需要自己指定。（对[Glide](https://github.com/bumptech/glide)默认是565）。
- 线程的优化
  - 对于Activity的生命周期有优化，不过需要自己手动设置pause，还有stop等，（[Glide](https://github.com/bumptech/glide)实现了自动管理）
  - 对于ListView的滑动，优化加载可见View的图片，具体可参考ImageLoader类的cancelDisplayTask方法（也是需要自己去实现，[Glide](https://github.com/bumptech/glide)实现自动管理）
- 扩展性与易用性，对于扩展性，使用面向接口，从而摆脱对具体实现的依赖。对于易用性，通过Facade模式来完成，隐藏底层细节，对外高层级的接口。
