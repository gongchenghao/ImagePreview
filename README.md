这是鸿蒙中的大图预览功能，具体功能点如下：
1：双指捏合图片放大和缩小
2：双击放大和缩小
3：放大后的单指拖动和松开手指后的惯性滑动
4：可与轮播图共用

遇到的主要问题点如下：
1：图片怎么缩放
2：图片怎么滑动
3：缩放后滑动时的边界怎么处理
4：与轮播图的滑动事件冲突
5：怎么在手指松开之后判断是否进行惯性滑动
6：怎么进行惯性滑动
7：放大后松开手指时怎么不让图片滑动
8：怎么监听双击事件 ？

问题解答：
1：图片缩放
使用Image的 transform() 进行缩放，具体捏合手势通过PinchGesture管理

2：图片滑动
使用Image的 offset() 进行滑动，具体滑动手势通过PanGesture管理

3：边界处理
在图片缩放的过程中实时计算出X轴和Y轴的最大滑动距离，大概思路如下：

// PinchGesture的onActionUpdate()方法
    this.maxScrollX = (this.imageWidth * this.scaleInfo.scaleValue - this.screenWidth) / 2
    this.maxScrollY = (this.imageHeight * this.scaleInfo.scaleValue - this.screenHeight) / 2
    if (this.maxScrollX < 0){
      this.maxScrollX = 0
    }
    if (this.maxScrollY < 0){
      this.maxScrollY = 0
    }
图片放大后滑动到上下左右的某个边界，然后进行缩小，缩小时会滑出边界并且显露出父组件的背景色，这时就需要边缩小边对图片进行移动——移动到边界处，具体思路如下：

// PinchGesture的onActionUpdate()方法
if (event.scale < 1) { //缩小
      if (this.offsetInfo.currentX > this.maxScrollX) {
        this.offsetInfo.currentX = this.maxScrollX
      }
      if (this.offsetInfo.currentX < -this.maxScrollX) {
        this.offsetInfo.currentX = -this.maxScrollX
      }
      if (this.offsetInfo.currentY > this.maxScrollY) {
        this.offsetInfo.currentY = this.maxScrollY
      }
      if (this.offsetInfo.currentY < -this.maxScrollY) {
        this.offsetInfo.currentY = -this.maxScrollY
      }
    }

  
4：与轮播图的滑动事件冲突
图片在轮播图中放大之后，本来只是想滑动图片的，但是却进行了轮播图的换页，从这里看是图片的滑动事件和轮播图的滑动事件冲突了，解决方法如下：
首先通过Swiper的disableSwipe()方法，禁用轮播图自带的切换功能，然后在图片滑动时进行控制，在图片滑动到左右边界的时候，通过SwiperController来进行上一页、下一页的操作，这样会有切换的动画，切记不要直接操作Swiper的index()方法，这样会导致没有切换动画

swiperController.showNext()
swiperController.showPrevious()
图片滑动到边界时直接切换轮播图的下一页或者上一页，就会导致滑动的时候很容易直接切换了轮播图，但是我们此时还是想看当前图片，因此需要滑动到边界之后进行二次滑动，再进行轮播图的翻页，通过增加isSliding这个boolean值进行判断，图片滑动时设置为true，在PanGesture的onActionEnd()中设置为false——注意这里需要延迟设置为false，只有为false的时候才允许切换轮播图

  //滑动图片时进行轮播图翻页
  onSlidPic(event: GestureEvent){
    if (this.isSliding == false) { //一次滑动只允许翻一页
      if (Math.abs(event.offsetX) > this.mMinimumSlideSwiper){ //必须大于最小滑动距离，才能翻页
        if (this.onSlideSwiper != null){
          if (event.offsetX > 0) { //右滑
            this.onSlideSwiper(false)
          }
          if (event.offsetX < 0) { //左滑
            this.onSlideSwiper(true)
          }
        }
        this.isSliding = true
      }
    }
  }

  
5：怎么在手指松开之后判断是否进行惯性滑动
首先需要明确进行惯性滑动的时机为PanGesture的onActionEnd()方法，这里是手指滑动操作结束的标志；如果直接在这里进行惯性滑动话，在手指滑动很慢然后抬起的情况下也会进行惯性滑动，这不符合操作习惯，所以就需要判断手指抬起时在屏幕上的滑动速度，通过event.velocityX 和 event.velocityY我们可以获取到手指抬起时Image在X轴和Y轴的滑动速度，当着两个值中的一个大于最小滑动速度时，我们就进行惯性滑动

if (Math.max(Math.abs(event.velocityX), Math.abs(event.velocityY)) > this.mMinimumVelocity){
  //进行惯性滑动             
}


6：怎么进行惯性滑动
鸿蒙的官方开发文档中有惯性滑动的方法，但是效果不甚理想，首先是只能滑动event.offsetX的距离，这就导致滑动距离很短；其次我建议使用EaseOut作为滑动效果，因为我们是全方向滑动；滑动的动画时长建议设置成300毫秒；滑动距离我们使用event.offsetX * 15 和 event.offsetY * 15，这样滑动距离长，效果更好；非惯性滑动时还是滑动event.offsetX和event.offsetY，这里注意区分。
具体滑动代码示例如下：

//PanGesture的onActionEnd()方法中
//惯性滑动动画：当手指抬起时执行，手指滑动过程中不执行
            if (Math.max(Math.abs(event.velocityX), Math.abs(event.velocityY)) > this.mMinimumVelocity){
              animateTo({
                duration: 300,
                curve: Curve.EaseOut, //表示动画以低速结束
                iterations: 1 ,
                playMode: PlayMode.Normal,
                onFinish: () => {
                  //动画结束100毫秒之后再保存移动距离，立即保存的话会产生动画卡顿
                  setTimeout(() => {
                    this.offsetInfo.stash()
                    this.isSliding = false
                  }, 100)
                }
              }, () => {
                this.setOffSet(event,true)
              })
            }else {
              this.offsetInfo.stash()  //动画结束
            }

            
7：放大后松开手指时怎么不让图片滑动
双指对图片进行缩放完松开手指时，因为手指离开屏幕时有先后顺序可能会触发滑动事件，导致图片的突然移动，因此我在PinchGesture的onActionEnd()方法中加了一个延迟以控制是否正在捏合操作的变量，示例代码如下：

setTimeout(() => {
              this.isScaling = false //缩放完成150毫秒后再重置，防止刚缩放完手指离开屏幕时有先后顺序导致的滑动
            }, 150)

            
8：怎么监听双击事件 ？
这个很简单，直接上代码：

//双击
        TapGesture({ count: 2 })
          .onAction((event: GestureEvent) => {
            this.onDoubleClick()
          })
