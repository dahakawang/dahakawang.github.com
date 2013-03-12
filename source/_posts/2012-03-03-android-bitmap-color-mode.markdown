---
layout: post
title: "【译】Android位图颜色模式的问题"
date: 2012-03-03 20:02
comments: true
categories: [原理探求]
sharing: false
published: true
---
最近开始了android上的编程之旅，在了解2D图形编程时，令人蛋疼的发觉android上仅支持ARGB8888、ARGB4444、RGB565以及Alpha 8这么几种颜色模式，而不支持RGB888这种格式。原本以为即使不支持RGB888我用ARGB8888总行吧，但后来了解到，即使我在内存中用ARGB888颜色模型表示图像，在该图像拷贝到屏幕帧缓冲区的过程中，它也会变成RGB565颜色模式。我们知道，RGB565最多只能表示2^16=65536种图像，这对于RGB888所能表示的2^24=16777216种颜色来说显然在表现力上要略逊一筹。这集中表现在显示某些带有渐变效果的图片时，出现了一条条的颜色带，而不是原始的平滑的渐变效果。后来得知android使用了Dither(抖动)这种技术，以欺骗人类眼球的方式加以补偿。

当然，以这个问题为出发点，后来又引发了诸多问题，而这篇文章解决了我不少问题，特在这里翻译出来供大家分享之。

<!-- more -->

---------------------------------------

在学习AvoidXfermode类的时候，我遇到了图片颜色不能正确显示的问题。我画了一个平滑渐变的色谱图片，然后用AvoidXfermode的Target功能将替换该图片里一种颜色替换为另外的颜色。但问题是，我想要替换的颜色并没有按照我所预想的一样被AvoidXfermode替换掉。原来，我的PNG图片被自动从24位的RGB888颜色模式转换为了16位的RGB565颜色模式，这使得图片中颜色的原始数值被改变了。

现在市面上几乎所有的设备都是16位色的屏幕，这意味着不管你在代码里使用什么颜色格式的图片，在某个时候它们都会被转化为16位图片，这样它们才能被显示在屏幕上。这种转换会占用处理器资源，会以时间和电池寿命的形式给用户造成损失。所以，我们并不期望这种转换在绘图时会发生，相反的，我们希望它还要在尽可能早的阶段里被完成。Android是一个智能的系统，在有些情况下Android会自动帮你完成这些转换工作，所以你并不需要担心这些问题。在大部分情况下，这个特性是相当棒的，它大大减少了开发者的工作量，也减少你的资源大小同时节省了处理器时间和电池寿命。但是，如果你希望在程序里将图片以一种指定的颜色格式处理，这种自动转化机制会让你头大的。幸运的是，我们有办法阻止自动转换的发生，让你的图片以你期望的颜色格式储存。

不过，既然这种自动转换既节省内存和处理器时间又保护电池寿命，那么为什么有人会不希望这种转换发生呢？Android程序中大部分的图片都只需要简单的载入并显示就行了。但是在一些教程以及我之前提到的例子中，我们需要在图片显示之前对它们做一些操作。为了得到最佳的效果，我们希望载入的图片在处理时尽能可能的保持最好的质量，因为在处理过程中过早的降低质量将对最终效果造成不良影响。

好了，这就是说在图片处理好之前，我们不希望被Android横插一脚。但是，Android系统将在什么时候替我们执行这些转换呢?答案是转换将在3个地方发生：

- 编译时图片资源被编译到软件包里去时
- 当图片被从资源中载入为Bitmap时
- 当图片被绘制时

我们将看看这三类情况，并且了解如何避免这些转换。

## 程序编译时

当你在项目“res/drawable”文件夹下放置图片的时候，意味着你告诉Android：如果需要的话，在构建程序的时候将图片转换为16位图片。该转换发生的必要条件有哪些？无论你图片的原始格式是什么，如果你的图片没有alpha通道，在你的软件构建的时候Android会将它转换为本地16位色图。你有两种方法组织转换发生：

- 给图片添加alpha通道
- 将图片放置到“res/raw”目录下而不是“res/drawable”

通过给图片添加alpha通道，Android将不会尝试将图片转换为16位色图，这是因为RGB565颜色模式不带alpha通道。将一张带有alpha通道的图片转换为RGB565颜色格式会使半透明信息丢失，所以任何带有alpha通道的图片将被储存为32位的ARGB8888图片资源。通过将图片放置到“res/raw”目录下,我们告诉Android这个资源包含原始数据，它不应该在构建时更改。所以，对于我们放置到该目录下的任何图片都不会发生自动转换。但不幸的是，如果我们放在raw目录下的图片不带有alpha通道的话，这个方法还是有问题。这个问题在我们载入图片时显露了出来：

## 图片载入时

当使用BitmapFactory从你程序的资源中载入一张图片时，同样的自动转换机制会发生。如果你要载入的图片没有alpha通道，Android会将其转换为16位RGB565图片。不幸的是，即使我们将图片放置在“res/raw”目录下，这种转换依然会发生。根据在这个[帖子](http://groups.google.com/group/android-developers/browse_thread/thread/8b1abdbe881f9f71)里一个叫[Romain Guy](http://www.curious-creature.org/)的Android开发者所说，有一个解决之道：

>“你需要做的就是将图片直接载入为ARGB888模式。当你调用
>BitmapFactory.decode*()的某一个重载方法时，你能够指定一系列的
>BitmapFactory.Option对象。你需要将Option对象中的inDither设置为false。这
>样将会使BitmapFactory不去尝试将24位色图转换为16位色图。”

当时，我发觉**这个方法不起作用**，至少在Android 1.6下都不起作用。传入一系列参数使得inDither为false的确使得图片不被抖动（Dither）处理，但是这种方法并不能阻止颜色模式的转换。图片依然会被转换成16位的RGB565，这将会使得转换后的渐变图片出现颜色条带的现象。

既然这个推荐的解决方案并不能满足我们的要求，那么在这个过程中唯一能保持你图片原貌的方法就是让你的图片带上alpha通道。当BitmapFactory发觉图片资源带有alpha通道，它便只能将图片解码为32位的ARGB888位图。

## 绘制时

假设我们有两张图片。图片A是ARGB888位图，图片B是RGB565位图。如果我们敬爱那个图片A绘制到B上，那么图片A将需要被转换为B的颜色格式。辛运的是，这种转换是由Canvas的drawBitmap方法替我们包办了。这对我们来说是个好消息，因为这正是我们想要的。在我们要释放位图资源的时候，我们已经完成调整和操作图片，并且图片将被转换和显示。然而，由于我们从32位转换为16位颜色深度，这将会造成图像的失真。为了减小失真对图片的影响，我们能够控制将图片从32位转换为16位的时机。当将一个高色深的图片绘制到第色深的图片上时，默认是不会进行抖动处理的。对于包含渐变的图片而言，这会使得图片出现众所周知的“色带”问题，这是非常难看的。为了克服这个问题，我们需要告诉Android我们想要对结果进行抖动处理。抖动是这么一种处理，它将原始颜色做出一些改变，以骗过我们的眼睛，让我们在低色深图片中以为自己看到了一个平滑的渐变。

**需要记住的是，当我们处理图片的时候，图片必须总是32位ARGB8888模式。**只有当我们完成处理图片后，他们才应该被转换成16位色的图片。由于任何不带alpha通道的图片，再被BitmapFactory载入的时候将被转换为RGB565，不管它们是在"res/drawable"还是在"res/raw"目录下。而确保他们被解码为32色ARGB8888的唯一手段就是保证你的图片带有alpha通道。

## 例子

为了证明我所说，然我们做一个简单的测试。首先让我们写一个载入并以默认option显示两张图片的activity。两张图片都是平滑渐变的色谱，但是他们将被保存为不同的格式。第一张图将被保存为24位色RGB888的PNG图，第二张图片被保存为32位带有alpha通道的ARGB8888的PNG图片。两张图片有着完全一致的颜色数据，惟一的区别是一张带有alpha通道而另一张没有。保存两张图片，并把他们放置到你项目的"res/raw"目录下。然后，在你的activity中使用下列代码：

``` java
@Override
 public void onCreate(Bundle savedInstanceState) {
       super.onCreate(savedInstanceState);
 
      // Load both of our images from our application's resources.
      Resources r = getResources();
      Bitmap resource= BitmapFactory.decodeResource(r, R.raw.spectrum_gray_nodither_;
      Bitmap resource= BitmapFactory.decodeResource(r, R.raw.spectrum_gray_nodither_;
 
      // Print some log statements to show what pixel format these images were decoded with.
      Log.d("FormatTest","Resource " + resourcegetConfig()); // Resource RGB_
      Log.d("FormatTest","Resource " + resourcegetConfig()); // Resource ARGB_8
 
 // Create two image views to show these bitmaps in.
      ImageView image= new ImageView(this);
      ImageView image= new ImageView(this);
      imagesetImageBitmap(resource;
      imagesetImageBitmap(resource;
 
  
 
      // Create a simple layout to show these two image views side-by-side.
      LayoutParams wrap = new LayoutParams(LayoutParams.WRAP_CONTENT, LayoutParams.WRAP_CONTENT);
      LayoutParams fill = new LayoutParams(LayoutParams.FILL_PARENT, LayoutParams.FILL_PARENT);
      RelativeLayout.LayoutParams params= new RelativeLayout.LayoutParams(wrap);
      paramsaddRule(RelativeLayout.CENTER_VERTICAL);
      paramsaddRule(RelativeLayout.ALIGN_PARENT_LEFT);
      RelativeLayout.LayoutParams params= new RelativeLayout.LayoutParams(wrap);
      paramsaddRule(RelativeLayout.CENTER_VERTICAL);
      paramsaddRule(RelativeLayout.ALIGN_PARENT_RIGHT);
      RelativeLayout layout = new RelativeLayout(this);
      layout.addView(image params;
      layout.addView(image params;
      layout.setBackgroundColor(Color.BLACK);
 
      // Show this layout in our activity.
      setContentView(layout, fill);
 
 }
```

编译此项目并将它部署到你的设备上查看结果。我们看到24位色图片在左边，为32位色图片在右边。但是等一下！32位的图片看上去有一些色带出现，然而24位图片看上去却很平滑。到底是怎么了？让我们仔细看看图片，我们能发现：

![](/images/blogs/2012/original_image.png)

仔细检查后，我们发觉24位图被抖动处理了，而32位色图没有。考虑到对于任何显示到Android设备屏幕的图片，都将被转换为16位色格式，所以这两张图都将发生这个转换。由于24位色图不带有alpha通道，并且它被放置在"res/raw"目录下，所以它将在我们载入activity中的资源时被自动转换为16位色图。我们能通过检查Logcat里的消息来验证这一点。我们从BitmapFactory里得到的Bitmap对象实际上是RGB565格式的，而BitmapFactory足够聪明会帮我们抖动处理图片。而我们带有alpha通道的的32位色图，则被载入为了ARGB8888位图。在该图片对应的ImageView被绘制到屏幕时，它将被转换为本地16位色格式。这里我们看到的是绘制时发生的转换不会进行抖动处理。让我们看一下，如果我们指明转换时进行抖动处理，情况是否会好一些。在上例中19行添加如下几行代码：

``` java
// Enable dithering when our 32-bit image gets drawn.
Drawable drawable32 = image32.getDrawable();
drawable32.setDither(true);
```

编译后上传至设备然后查看结果：

![](/images/blogs/2012/after_compiled.png)

啊，好了。现在我们的32位图也已经被抖动处理好了。这里我们简单地告诉ImageView我们想要Bitmap被被绘制时进行抖动处理。不幸的是这将影响绘图速度，所以这并非一个理想的解决方案。但我们能够消除这个性能问题，通过在我们将图片送至ImageView之前预先进行抖动处理：

``` java
@Override
 public void onCreate(Bundle savedInstanceState) {
 super.onCreate(savedInstanceState);
  
 // Load both of our images from our application's resources.
 Resources r = getResources();
 Bitmap resource24 = BitmapFactory.decodeResource(r, R.raw.spectrum_gray_nodither_24);
 Bitmap resource32 = BitmapFactory.decodeResource(r, R.raw.spectrum_gray_nodither_32);
  
 // Print some log statements to show what pixel format these images were decoded with.
 Log.d("FormatTest","Resource24: " + resource24.getConfig()); // Resource24: RGB_565
 Log.d("FormatTest","Resource32: " + resource32.getConfig()); // Resource32: ARGB_8888
  
 // Save the dimensions of these images.
 int width = resource24.getWidth();
 int height = resource24.getHeight();
  
 // Create a 16-bit RGB565 bitmap that we will draw our 32-bit image to with dithering.
 Bitmap final32 = Bitmap.createBitmap(width, height, Config.RGB_565);
  
 // Create a new paint object we will use to draw our bitmap with. This is how we tell
 // Android that we want to dither the 32-bit image when it gets drawn to our 16-bit final
 // bitmap.
 Paint ditherPaint = new Paint();
 ditherPaint.setDither(true);
  
 // Create a new canvas for our 16-bit final bitmap, and draw our 32-bit image to it with
 // the paint object we just created.
 Canvas canvas = new Canvas();
 canvas.setBitmap(final32);
 canvas.drawBitmap(resource32, 0, 0, ditherPaint);
  
 // Create two image views to show these bitmaps in.
 ImageView image24 = new ImageView(this);
 ImageView image32 = new ImageView(this);
 image24.setImageBitmap(resource24);
 image32.setImageBitmap(final32);
  
 // Create a simple layout to show these two image views side-by-side.
 LayoutParams wrap = new LayoutParams(LayoutParams.WRAP_CONTENT, LayoutParams.WRAP_CONTENT);
 LayoutParams fill = new LayoutParams(LayoutParams.FILL_PARENT, LayoutParams.FILL_PARENT);
 RelativeLayout.LayoutParams params24 = new RelativeLayout.LayoutParams(wrap);
 params24.addRule(RelativeLayout.CENTER_VERTICAL);
 params24.addRule(RelativeLayout.ALIGN_PARENT_LEFT);
 RelativeLayout.LayoutParams params32 = new RelativeLayout.LayoutParams(wrap);
 params32.addRule(RelativeLayout.CENTER_VERTICAL);
 params32.addRule(RelativeLayout.ALIGN_PARENT_RIGHT);
 RelativeLayout layout = new RelativeLayout(this);
 layout.addView(image24, params24);
 layout.addView(image32, params32);
 layout.setBackgroundColor(Color.BLACK);
  
 // Show this layout in our activity.
 setContentView(layout, fill);
 }
```

运行这段代码将会产生之前一样的效果，但是通过提前抖动处理，我们避免了性能瓶颈。如果你只是简单地载入和显示你的图片到你的activity一次，这种处理并不是必要的。但是如果需要频繁地绘制，则需要提前抖动处理以避免浪费处理时间和电池寿命。

这时候你可能会想了，为什么我们要做那么多额外的工作来确保我们的图片被解码为32位色图像，然而其结果却只是还要在我们绘图时做更多的工作来确保图片被抖动处理了呢？既然24位色被自动转换为了效果不错的16位色图而不需要你的任何干预，那么我们使用32位色图的理由是什么呢？实际上，对于上面的例子，我们还没有对这些位图做任何的处理和操作，所以被自动转换的24位色图片看上去和我们手动转换的32位色图片效果一样。他们最终都有着一样的像素数据。但是若是我们想要在显示之前对图片进行一些处理呢？咱们先试试看。在你的设备上运行下述代码：

``` java
@Override
 public void onCreate(Bundle savedInstanceState) {
 super.onCreate(savedInstanceState);
  
 // Load both of our images from our application's resources.
 Resources r = getResources();
 Bitmap resource24 = BitmapFactory.decodeResource(r, R.raw.spectrum_gray_nodither_24);
 Bitmap resource32 = BitmapFactory.decodeResource(r, R.raw.spectrum_gray_dithered_32);
  
 Log.d("FormatTest","Resource24: " + resource24.getConfig()); // Resource24: RGB_565
 Log.d("FormatTest","Resource32: " + resource32.getConfig()); // Resource32: ARGB_8888
  
 // Sadly, the images we have decoded from our resources are immutable. Since we want to
 // change them, we need to copy them into new mutable bitmaps, giving each of them the same
 // pixel format as their source.
 Bitmap bitmap24 = resource24.copy(resource24.getConfig(), true);
 Bitmap bitmap32 = resource32.copy(resource32.getConfig(), true);
  
 // Save the dimensions of these images.
 int width = bitmap24.getWidth();
 int height = bitmap24.getHeight();
  
 // Create a new paint object that we will use to manipulate our images. This will tell
 // Android that we want to replace any color in our image that is even remotely similar to
 // 0xFF307070 (a dark teal) with 0xFF000000 (black).
 Paint avoid1Paint = new Paint();
 avoid1Paint.setColor(0xFF000000);
 avoid1Paint.setXfermode(new AvoidXfermode(0xFF307070, 255, AvoidXfermode.Mode.TARGET));
  
 // Make another paint object, but this one will replace any color that is similar to a
 // 0xFF00C000 (green) with 0xFF0070D0 (skyish blue) instead.
 Paint avoid2Paint = new Paint();
 avoid2Paint.setColor(0xFF0070D0);
 avoid2Paint.setXfermode(new AvoidXfermode(0xFF00C000, 245, AvoidXfermode.Mode.TARGET));
  
 Paint fadePaint = new Paint();
 int[] fadeColors = {0x00000000, 0xFF000000, 0xFF000000, 0x00000000};
 fadePaint.setShader(new LinearGradient(0, 0, 0, height, fadeColors, null,
 LinearGradient.TileMode.CLAMP));
 fadePaint.setXfermode(new PorterDuffXfermode(PorterDuff.Mode.DST_IN));
  
 // Create a new canvas for our bitmaps, and draw a full-sized rectangle to each of them
 // which will apply the paint object we just created.
 Canvas canvas = new Canvas();
 canvas.setBitmap(bitmap24);
 canvas.drawRect(0, 0, width, height, avoid1Paint);
 canvas.drawRect(0, 0, width, height, avoid2Paint);
 canvas.drawRect(0, 0, width, height, fadePaint);
 canvas.setBitmap(bitmap32);
 canvas.drawRect(0, 0, width, height, avoid1Paint);
 canvas.drawRect(0, 0, width, height, avoid2Paint);
 canvas.drawRect(0, 0, width, height, fadePaint);
  
 // Create a 16-bit RGB565 bitmap that we will draw our 32-bit image to with dithering. We
 // only need to do this for our 32-bit image, and not our 24-bit image, because it is
 // already in the RGB565 format.
 Bitmap final32 = Bitmap.createBitmap(width, height, Bitmap.Config.RGB_565);
  
 // Create a new paint object that we will use to draw our bitmap with. This is how we tell
 // Android that we want to dither the 32-bit image when it gets drawn to our 16-bit final
 // bitmap.
 Paint ditherPaint = new Paint();
 ditherPaint.setDither(true);
  
 // Using our canvas from above, draw our 32-bit image to it with the paint object we just
 // created.
 canvas.setBitmap(final32);
 canvas.drawBitmap(bitmap32, 0, 0, ditherPaint);
  
 // Create two image views to show these bitmaps in.
 ImageView image24 = new ImageView(this);
 ImageView image32 = new ImageView(this);
 image24.setImageBitmap(bitmap24);
 image32.setImageBitmap(final32);
  
 // Create a simple layout to show these two image views side-by-side.
 LayoutParams wrap = new LayoutParams(LayoutParams.WRAP_CONTENT, LayoutParams.WRAP_CONTENT);
 LayoutParams fill = new LayoutParams(LayoutParams.FILL_PARENT, LayoutParams.FILL_PARENT);
 RelativeLayout.LayoutParams params24 = new RelativeLayout.LayoutParams(wrap);
 params24.addRule(RelativeLayout.CENTER_VERTICAL);
 params24.addRule(RelativeLayout.ALIGN_PARENT_LEFT);
 RelativeLayout.LayoutParams params32 = new RelativeLayout.LayoutParams(wrap);
 params32.addRule(RelativeLayout.CENTER_VERTICAL);
 params32.addRule(RelativeLayout.ALIGN_PARENT_RIGHT);
 RelativeLayout layout = new RelativeLayout(this);
 layout.addView(image24, params24);
 layout.addView(image32, params32);
 layout.setBackgroundColor(Color.BLACK);
  
 // Show this layout in our activity.
 setContentView(layout, fill);
 }
```

这里我们载入了和之前例子一样的图片，但我们现在使用Android内置的图形函数对他们预先进行了一些处理，而不是直接将它们添加到ImageView中显示。我们在显示两张图片前分别将它们都使用了三个不同的滤镜进行处理。要是你不明白23-52行代码是干嘛的，别担心，我会在以后的文章中详尽的介绍它们。而对于现在而言，你只需要知道我们的原始图片在运行时被我们的程序进行了巨大的变换。下图是最终的效果：

![](/images/blogs/2012/final_image.png)

正如你所看到的，在经过了许多操作后，24位色图有了巨大的失真，而32位色图的效果则依然很不错。除了”条带”和失真，你可以发现24位色图的颜色根本都不正确。那么，为什么24位色图会出现这种现象而32位色图却没有呢？原因就在于24位色图被Android自动转换为了RGB565格式图像。一张16位色图片仅能够显示65,535种不同颜色，而32位色图却能够显示16,777,215种颜色，还带有255级半透明效果。这意味着当我们处理16位色图时，处理过后的颜色不能够被16位色所支持的65,535种颜色精确地表示出来。所以处理后的颜色被截取到了最临近相似的颜色值去了。每次我们对图片进行操作时，这种截取都会发生，所以图片将变得更加不精确。而当时用32位色时，这种现象虽然依然会有，但是由于有着将近17兆的颜色可以表示，这种副作用对于大部分程序来说将可以被忽略。

我希望我的这篇文章能帮到各位。我希望各位能从我的文章中学会的主要思想是：

1. 要意识到Android的自动图片转换和抖动处理机制。当事情不照你所想发展时，知道Android是如何储存、载入以及绘制你的图像的话将大大减轻你的头痛。
2. 如果你要对你的图像被载入后进行任何的处理，确保你的图片是ARGB8888格式的。
3. 通过使用某种图形处理软件将你的图像保存为32位带alpha通道的PNG图片，你可以强制这张图片被载入为32位色。
4. 当你处理完你的32位色图片后，记得启用抖动处理后再将该图片转换为16位RGB565色图。
