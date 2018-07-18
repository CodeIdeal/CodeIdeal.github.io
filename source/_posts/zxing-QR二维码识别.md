title: zxing-QR二维码识别
date: 2017-03-15 18:30:14
---
## zxing二维码库

* GitHub地址：https://github.com/zxing/zxing
* maven 地址：https://repo1.maven.org/maven2/com/google/zxing/
* 概该库最开始是由Google 开源出来的，后来一直被各位大神维护至今

### Android 项目需要用到的jar包

* core.jar 是二维码解析及生成的核心类库

* android-core.jar 提供了Android相关的相机配置工具类

* glass 项目  提供了二维码扫描的界面activity


### 遇到的问题

1. surfaceView 获取相机预览 当手机为竖屏时预览图像方向旋转了90度，参见网络文章[^1]：

   * 解决方法：在通过调用Camera 类中的camera.setDisplayOrientation(90)来使预览的相机湖面旋转90与手机屏幕坐标系吻合。
   * 原因：手机的屏幕坐标系和手机相机传感器的坐标系的差异导致的
     * 屏幕方向：在Android系统中，屏幕的左上角是坐标系统的原点（0,0）坐标。原点向右延伸是X轴正方向，原点向下延伸是Y轴正方向。

     * 相机传感器方向：手机相机的图像数据都是来自于摄像头硬件的图像传感器，这个传感器在被固定到手机上后有一个默认的取景方向，如下图2所示，坐标原点位于手机横放时的左上角，即与横屏应用的屏幕X方向一致。换句话说，与竖屏应用的屏幕X方向呈90度角。

2. 相机预览画面无法自动对焦

   * 原因：由于使用了CameraConfigurationUtils来配置camera相关的参数，没有把相机设置为自动对焦


*    解决方法：调用android-core.jar中的

     ```java
     CameraConfigurationUtils.setFocus(Camera.Parameters parameters,boolean autoFocus, boolean disableContinuous,boolean safeMode)
     ```

     方法，把第二个参数和第三个参数分别设置为true和false即可。  

3. 相机资源的及时释放（在二维码扫面界面进入后台或手机直接锁屏的情况下为了减少不必要的界面渲染以及手机电量的耗费）

   * 解决方法：在该activity 的onpause 方法中及时释放 camera资源 以及相关实时解析逻辑的runnable任务

     ```java
     @Override
       public synchronized void onPause() {
         result = null;
         if (decodeRunnable != null) {
           decodeRunnable.stop();
           decodeRunnable = null;
         }
         if (camera != null) {
           camera.stopPreview();
           camera.release();
           camera = null;
         }
         if (holderWithCallback != null) {
           holderWithCallback.removeCallback(this);
           holderWithCallback = null;
         }
         super.onPause();
       }
     ```

4. 二维码很难扫描出结果

   * 原因：想当然的认为 二维码的识别区域是整个camera的预览区域，不知到zxing库中把camera预览区域分为了信息采集区域和额外区域(详见zxing 二维码解析 预览画面的byte数组 的具体代码):、

     ```java
     int subtendedWidth = width / CameraConfigurationManager.ZOOM;
     int subtendedHeight = height / CameraConfigurationManager.ZOOM;
     int excessWidth = width - subtendedWidth;
     int excessHeight = height - subtendedHeight;
     ```

     其中CameraConfigurationManager.ZOOM 是一个配置的参数，默认为2，那么在camera预览画面中只有 长宽均为Camera预览画面长宽的一半 的 中心区域才是真正的二维码信息采集区域！


*    解决办法：在整个预览画面前面加上一层遮罩用来提示 真正的二维码信息采集区域。


### 生成二维码相关

1. 趁热写的 生成代码

   ```java
   HashMap<EncodeHintType, Integer> config = new HashMap<>();
   config.put(EncodeHintType.MARGIN,1);
   BitMatrix bitMatrix = new QRCodeWriter().encode("ahhahahaha", BarcodeFormat.QR_CODE, QR_WIDTH, QR_HEIGHT,config);
   int[] pixels = new int[QR_WIDTH * QR_H
   for (int y = 0; y < 39; y++) {
       for (int x = 0; x < 39; x++) {
           pixels[y * QR_WIDTH + x] = bitMatrix.get(x, y) ? 0xff000000 : 0xffffffff;
       }

   Bitmap bitmap = Bitmap.createBitmap(QR_WIDTH, QR_HEIGHT, Bitmap.Config.ARGB_8888);
   bitmap.setPixels(pixels,0,QR_WIDTH,0,0,QR_WIDTH,QR_HEIGHT);
   Matrix matrix = new Matrix();
   matrix.setScale(10,10);
   Bitmap bitmap1 = Bitmap.createBitmap(bitmap, 0, 0, QR_WIDTH, QR_HEIGHT, matrix, false);
   File file = new File(appContext.getExternalCacheDir(),"test.png");
   BufferedOutputStream bos = new BufferedOutputStream(new FileOutputStream(file));
   bitmap1.compress(Bitmap.CompressFormat.JPEG, 100, bos);
   bos.flush();
   bos.close();
   ```

   其中

   * `new QRCodeWriter().encode("ahhahahaha", BarcodeFormat.QR_CODE, QR_WIDTH, QR_HEIGHT,config);`为生成二维码 原始 像素矩阵 的核心代码，是zxing库中的方法调用。
   * 两层for循环是用来从该矩阵变换为一维像素信息数组
   * 由于源始像素信息是单一点阵，如果直接拿来生成二维码图片则，会由于分辨率太小而看不清，甚至 无法扫描还原自然信息，所以需要通过Matrix缩放来增大图片大小。

   ​


[^1]: 参见http://gold.xitu.io/entry/56aa36fad342d300542e7510