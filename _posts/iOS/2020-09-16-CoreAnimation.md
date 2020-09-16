---
layout: post
title: CoreAnimation详解
date: 2020-09-16
tag: iOS
---


## 1. 图层树
CALayer 是存在于UIView中的一种平行的层级关系
<img src="http://image.smartjames.cn/20200811234017.jpg" style="zoom:50%" />


## 2.寄宿图
    contens 属性可以给layer赋值CGImage
    contentsGravity属性类似contentModel：kCAGravityResizeAspect
    contentsScale属性用来设置渲染倍数 
    contentsRect用来显示contents中指定区域，适用于雪碧图的展示

```
// 裁剪并显示雪碧图，不用从内存中读多张图片，4张图片在一张雪碧图中
- (void)addSpriteImage:(UIImage *)img withContentRect:(CGRect)rect toLayer:(CALayer *)layer
{
    layer.contents = (__bridge id)(img.CGImage);
    layer.contentsGravity = kCAGravityResizeAspect;
    layer.contentsRect = rect;
}
```


## 3. 图层几何学
UIView的center和 CALayer的position是 一个含义，代表中间点相对于父视图的位置。

anchorPoint是用单位坐标表示的center和position的值作用与本身的那个位置。
    
anchorPoint 可以用来实现时钟的指针转动
<img src="http://image.smartjames.cn/20200812165714.jpeg" style="zoom:50%" />
用layer的 -hitTest: 方法判断当前点击的是那个图层。一般情况下，我们理解layer是不能响应点击时间的，但是layer有这种方法判断点击位置

CALayerDelegate提供如下函数，当frame变化时会调用，用以重新处理layer的位置，但是无法做到像UIView的自适应屏幕。
```
- (void)layoutSublayersOfLayer:(CALayer *)layer;
```


## 4.视觉效果

阴影        
shadowOffset表示阴影的相对位移，默认值{0,-3}， 表示阴影上移3个点，因为Core Animation是Mac OS上出现的，Mac和iOS坐标系是翻转的，所以在Mac上显示的就是向下位移3个点。

shadowRadius是控制阴影的模糊度

shadowPath通过一个CGPath来绘制阴影，因为当视图有多个图层，shadowOffset计算阴影会非常消耗资源。可以通过shadowPath提前绘制一个大概的阴影代替会快很多，涉及到离屏渲染。

蒙版
mask是图层的属性，类型也是图层，可以用来将蒙版以外的内容裁剪掉。而且支持动态的mask
<img src="http://image.smartjames.cn/20200812175605.png" style="zoom:50%" />

拉伸过滤
minificationFilter和magnificationFilter属性，提供3中拉伸函数
- kCAFilterLinear
- kCAFilterNearest
- kCAFilterTrilinear


组透明
UIViewGroupOpacity，当一个控件a内部有另一个控件b, 设置a的alpha, 那么b的透明度也是0.5，但是会叠加控件a的部分颜色，通过在infoplsit中设置UIViewGroupOpacity为yes,或者设置layer的shouldRasterize=yes。
但是在最新的系统上，已经不需要单独处理这种问题。
<img src="http://image.smartjames.cn/20200812183557.png" style="zoom:50%" />


## 5.变换

### 仿射变换：
属于Core Graphics框架，是2D绘图API, 和UIView一样，只有2维空间
```
CGAffineTransformRotate(CGAffineTransform t, CGFloat angle)     
CGAffineTransformScale(CGAffineTransform t, CGFloat sx, CGFloat sy)      
CGAffineTransformTranslate(CGAffineTransform t, CGFloat tx, CGFloat ty)

```

CALayer有三维空间，所以同理CATransform3D属于Core Animation框架可以支持3D变换
API增加了Z轴旋转角度，XYZ轴缩放比例，和Z轴的移动距离
```
CATransform3DMakeRotation(CGFloat angle, CGFloat x, CGFloat y, CGFloat z)
CATransform3DMakeScale(CGFloat sx, CGFloat sy, CGFloat sz)
CATransform3DMakeTranslation(Gloat tx, CGFloat ty, CGFloat tz)
```
<img src="http://image.smartjames.cn/20200812222012.jpeg" style="zoom:50%" />

由图所见，绕Z轴的旋转等同于之前二维空间的仿射旋转，但是绕X轴和Y轴的旋转就突破了屏幕的二维空间，并且在用户视角看来发生了倾斜。
但是旋转得到的结果是等距投影，与真实世界里透视投影不一样。在等距投影中，远处的物体和近处的物体保持同样的缩放比例。引出透视投影的概念。

### 透视投影：
在真实世界中，当物体远离我们的时候，由于视角的原因看起来会变小，理论上说远离我们的视图的边要比靠近视角的边更短。
通过矩阵中一个元素控制：m34。m34的默认值是0，我们可以通过设置m34为-1.0 / d来应用透视效果；d是相机与屏幕之间的距离。
<img src="http://image.smartjames.cn/20200812234846.jpeg" style="zoom:50%" />
```
 CATransform3D transform = CATransform3DIdentity;
    transform.m34 = - 1.0 / 500.0;
    transform = CATransform3DRotate(transform, M_PI_4, 0, 1, 0);
    self.layerView.layer.transform = transform;
```

sublayerTransform属性，使子图层都有相同的trasnform设置。

### 背面
旋转180°后，形成一个镜像，可以用doubleSided属性控制背面是否需要绘制。

### 扁平化图层
当父视图沿着Y轴旋转45°时，子视图沿着Y轴旋转-45°时，预期结果如下，能够抵消旋转的角度
<img src="http://image.smartjames.cn/20200812235931.jpeg" style="zoom:50%" />

但是真实结果如下所示，子视图并没有垂直Z轴，因为CoreAnimation的3D场景其实是扁平化的，3D场景是图层想象出来的，绘制在图层表面的，且各自独立在不同的3D空间。
<img src="http://image.smartjames.cn/20200812235939.jpeg" style="zoom:50%" />
所以用普通的CALayer很难创建非常复杂的3D场景，除非用CATransformLayer。

### 固体对象
通过将不同layer旋转不同角度，拼凑成一个表面看是实心体。

```
CATransform3D perspective = CATransform3DIdentity;
    perspective.m34 = -1.0 / 500.0;
    
    // 都旋转一个角度,看到侧面
    perspective = CATransform3DRotate(perspective, -M_PI_4, 1, 0, 0);
    perspective = CATransform3DRotate(perspective, -M_PI_4, 0, 1, 0);
    
    self.view.layer.sublayerTransform = perspective;
    //add cube face 1
    CATransform3D transform = CATransform3DMakeTranslation(0, 0, 100);
    [self addFace:0 withTransform:transform];
    //add cube face 2
    transform = CATransform3DMakeTranslation(100, 0, 0);
    transform = CATransform3DRotate(transform, M_PI_2, 0, 1, 0);
    [self addFace:1 withTransform:transform];
    //add cube face 3
    transform = CATransform3DMakeTranslation(0, -100, 0);
    transform = CATransform3DRotate(transform, M_PI_2, 1, 0, 0);
    [self addFace:2 withTransform:transform];
    //add cube face 4
    transform = CATransform3DMakeTranslation(0, 100, 0);
    transform = CATransform3DRotate(transform, -M_PI_2, 1, 0, 0);
    [self addFace:3 withTransform:transform];
    //add cube face 5
    transform = CATransform3DMakeTranslation(-100, 0, 0);
    transform = CATransform3DRotate(transform, -M_PI_2, 0, 1, 0);
    [self addFace:4 withTransform:transform];
    //add cube face 6
    transform = CATransform3DMakeTranslation(0, 0, -100);
    transform = CATransform3DRotate(transform, M_PI, 0, 1, 0);
    [self addFace:5 withTransform:transform];
    
```
<img src="http://image.smartjames.cn/20200813003034.jpeg" style="zoom:50%" />


## 6.专用图层

### CAShapeLayer
一个通过矢量图形而不是bitmap来绘制的图层子类。

定义颜色和线宽用CGPath就能渲染，看起来Core Graphics也能实现，但是CAShapeLayer优点如下：
- 渲染快。使用了硬件加速，绘制比CG快很多
- 高效使用内存。不会像普通CALayer一样创建一个寄宿图，所以无论多大，都不会占用太多内存
- 不会被图层边界裁掉。可以在边界之外绘制
- 不会像素化。因为是矢量图。

实现3个圆角，1个直角的矩形。

```
CGRect rect = CGRectMake(50, 50, 100, 100);
    CGSize radii = CGSizeMake(20, 20);
    UIRectCorner corners = UIRectCornerTopRight | UIRectCornerBottomLeft | UIRectCornerTopLeft;
    UIBezierPath *path = [UIBezierPath bezierPathWithRoundedRect:rect byRoundingCorners:corners cornerRadii:radii];
    
CAShapeLayer *layer = [CAShapeLayer layer];
........ // 设置宽度颜色等
layer.path = path.CGPath;

```


### CATextLayer
它以图层的形式包含了UILabel几乎所有的绘制特性，并且额外提供了一些新的特性。同样，CATextLayer也要比UILabel渲染得快得多。很少有人知道在iOS 6及之前的版本，UILabel其实是通过WebKit来实现绘制的，这样就造成了当有很多文字的时候就会有极大的性能压力。而CATextLayer使用了Core text，并且渲染得非常快。



### CATransformLayer
前面想做一个立体图形的旋转状态，需要对立体图形的每一个图层做一次旋转。
现在使用CATransformLayer不需要，只要创建立体图形时把图层加入到CATransformLayer中，旋转CATransformLayer一次即可实现立体图形旋转。
```
- (CALayer *)cubeWithTransform:(CATransform3D)transform
{
  //create cube layer
  CATransformLayer *cube = [CATransformLayer layer];

  //add cube face 1
  CATransform3D ct = CATransform3DMakeTranslation(0, 0, 50);
  [cube addSublayer:[self faceWithTransform:ct]];

// .......

  //add cube face 6
  ct = CATransform3DMakeTranslation(0, 0, -50);
  ct = CATransform3DRotate(ct, M_PI, 0, 1, 0);
  [cube addSublayer:[self faceWithTransform:ct]];

  //center the cube layer within the container
  CGSize containerSize = self.containerView.bounds.size;
  cube.position = CGPointMake(containerSize.width / 2.0, containerSize.height / 2.0);

  //apply the transform and return
  cube.transform = transform;
  return cube;
}

```
<img src="http://image.smartjames.cn/20200813010733.png" style="zoom:50%" />

### CAGradientLayer
用来生成2种或更多颜色平滑渐变的图层。
    
startPoint和endPoint决定渐变方向，用单位坐标系定义，左上角是{0,0}, 右下角是{1,1}
location定义不同颜色的位置，个数要和颜色个数相同，可选。
```
- (void)viewDidLoad {
	[super viewDidLoad];

	//create gradient layer and add it to our container view
	CAGradientLayer *gradientLayer = [CAGradientLayer layer];
	gradientLayer.frame = self.containerView.bounds;
	[self.containerView.layer addSublayer:gradientLayer];

	//set gradient colors
	gradientLayer.colors = @[(__bridge id)[UIColor redColor].CGColor, (__bridge id) [UIColor yellowColor].CGColor, (__bridge id)[UIColor greenColor].CGColor];

	//set locations
	gradientLayer.locations = @[@0.0, @0.25, @0.5];

	//set gradient start and end points
	gradientLayer.startPoint = CGPointMake(0, 0);
	gradientLayer.endPoint = CGPointMake(1, 1);
}
```
<img src="http://image.smartjames.cn/20200813011439.png" style="zoom:50%" />

### CAReplicatorLayer
为了高效生成许多相似的重复图层。

```
- (void)viewDidLoad
{
    [super viewDidLoad];
    //create a replicator layer and add it to our view
    CAReplicatorLayer *replicator = [CAReplicatorLayer layer];
    replicator.frame = self.containerView.bounds;
    [self.containerView.layer addSublayer:replicator];

    //configure the replicator
    replicator.instanceCount = 10;

    //apply a transform for each instance
    CATransform3D transform = CATransform3DIdentity;
    transform = CATransform3DTranslate(transform, 0, 200, 0);
    transform = CATransform3DRotate(transform, M_PI / 5.0, 0, 0, 1);
    transform = CATransform3DTranslate(transform, 0, -200, 0);
    replicator.instanceTransform = transform;

    //apply a color shift for each instance
    replicator.instanceBlueOffset = -0.1;
    replicator.instanceGreenOffset = -0.1;

    //create a sublayer and place it inside the replicator
    CALayer *layer = [CALayer layer];
    layer.frame = CGRectMake(100.0f, 100.0f, 100.0f, 100.0f);
    layer.backgroundColor = [UIColor whiteColor].CGColor;
    [replicator addSublayer:layer];
}
```
<img src="http://image.smartjames.cn/20200813012855.png" style="zoom:50%" />

instanceCount 代表有复制多少次
instanceTransform属性，代表后续每个图层都要做这个变换。
instanceBlueOffset和instanceGreenOffset代表每次减少蓝色绿色通道多少值。

**镜像**
用CAReplicatorLayer实现镜像效果
```
 CAReplicatorLayer *layer = (CAReplicatorLayer *)self.layer;
    layer.instanceCount = 2;

    //move reflection instance below original and flip vertically
    CATransform3D transform = CATransform3DIdentity;
    CGFloat verticalOffset = self.bounds.size.height + 2;
    transform = CATransform3DTranslate(transform, 0, verticalOffset, 0);
    transform = CATransform3DScale(transform, 1, -1, 0);
    layer.instanceTransform = transform;

    //reduce alpha of reflection layer
    layer.instanceAlphaOffset = -0.6;
```
2个重复对象，移动高度的2倍，缩小一点，颜色透明一点

<img src="http://image.smartjames.cn/20200813013306.png" style="zoom:50%" />
    
### CAScrollLayer
实现类似UIScrollView的图层，但是内容可以滑出边界。
    
### CATiledLayer
iOS中绘制图片最终是转化为OpenGL纹理，最大纹理尺寸是2048×2048或4096×4096， 如果图片超过这个尺寸，会遇到性能问题。

因为Core Animation强制用CPU处理图片而不是GPU。

CATiledLayer会将大图片分解成小图片然后按需载入，类似ScrollView.
    **使用场景**是iOS老版本的地图应用，会一块一块的显示新的之前未曾渲染过的地图，就是用到了这个技术。
    

```
- (void)drawLayer:(CATiledLayer *)layer inContext:(CGContextRef)ctx
{
    //determine tile coordinate
    CGRect bounds = CGContextGetClipBoundingBox(ctx);
    NSInteger x = floor(bounds.origin.x / layer.tileSize.width);
    NSInteger y = floor(bounds.origin.y / layer.tileSize.height);

    //load tile image
    NSString *imageName = [NSString stringWithFormat: @"Snowman_%02i_%02i", x, y];
    NSString *imagePath = [[NSBundle mainBundle] pathForResource:imageName ofType:@"jpg"];
    UIImage *tileImage = [UIImage imageWithContentsOfFile:imagePath];

    //draw tile
    UIGraphicsPushContext(ctx);
    [tileImage drawInRect:bounds];
    UIGraphicsPopContext();
}
```
滑动图层到边缘时自动调用在drawLayer方法，获取对应要显示的图片并渲染。
CATiledLayer支持多线程绘制，这一点不同于UIKit和Core Animation方法。即-drawLayer:inContext: 能在多线程中并发调用


### CAEmitterLayer
是一个高性能的粒子引擎，被用来创建实时粒子动画如：烟雾，火，雨，烟花等。

CAEmitterLayer是CAEmitterCell的容器，后者定义了粒子效果，CAEmitterLayer将不同的粒子效果cell作为模板实例化粒子流。


```
   CAEmitterLayer *emitter = [CAEmitterLayer layer];
    emitter.frame = self.view.bounds;
    [self.view.layer addSublayer:emitter];

    //configure emitter
    emitter.renderMode = kCAEmitterLayerAdditive;
    emitter.emitterPosition = CGPointMake(emitter.frame.size.width / 2.0, emitter.frame.size.height / 2.0);

    //create a particle template
    CAEmitterCell *cell = [[CAEmitterCell alloc] init];
    cell.contents = (__bridge id)[UIImage imageNamed:@"Spark.png"].CGImage;
    cell.birthRate = 150;
    cell.lifetime = 5.0;
    cell.color = [UIColor colorWithRed:1 green:0.5 blue:0.1 alpha:1.0].CGColor;
    cell.alphaSpeed = -0.4;
    cell.velocity = 50;
    cell.velocityRange = 50;
    cell.emissionRange = M_PI * 2.0;

    //add particle template to emitter
    emitter.emitterCells = @[cell];
```
<img src="http://image.smartjames.cn/20200813020502.png" style="zoom:50%" />

CAEMitterCell的属性基本上可以分为三种：
- 这种粒子的某一属性的初始值。比如，color属性指定了一个可以混合图片内容颜色的混合色。在示例中，我们将它设置为桔色。
- 粒子某一属性的变化范围。比如emissionRange属性的值是2π，这意味着粒子可以从360度任意位置反射出来。如果指定一个小一些的值，就可以创造出一个圆锥形。
- 指定值在时间线上的变化。比如，在示例中，我们将alphaSpeed设置为-0.4，就是说粒子的透明度每过一秒就是减少0.4，这样就有发射出去之后逐渐消失的效果。


### CAEAGLLayer
当想使用GLKit实现OpenGL渲染时，使用CLKView类来处理大部分的设置和绘制工作，但是OpenGL 绘图缓冲的底层可配置项仍然需要使用CAEAGLLayer完成，用来显示任意的Open GL图形。

```
#import "ViewController.h"
#import <QuartzCore/QuartzCore.h>
#import <GLKit/GLKit.h>

@interface ViewController ()

@property (nonatomic, weak) IBOutlet UIView *glView;
@property (nonatomic, strong) EAGLContext *glContext;
@property (nonatomic, strong) CAEAGLLayer *glLayer;
@property (nonatomic, assign) GLuint framebuffer;
@property (nonatomic, assign) GLuint colorRenderbuffer;
@property (nonatomic, assign) GLint framebufferWidth;
@property (nonatomic, assign) GLint framebufferHeight;
@property (nonatomic, strong) GLKBaseEffect *effect;

@end

@implementation ViewController

- (void)setUpBuffers
{
    //set up frame buffer
    glGenFramebuffers(1, &_framebuffer);
    glBindFramebuffer(GL_FRAMEBUFFER, _framebuffer);

    //set up color render buffer
    glGenRenderbuffers(1, &_colorRenderbuffer);
    glBindRenderbuffer(GL_RENDERBUFFER, _colorRenderbuffer);
    glFramebufferRenderbuffer(GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0, GL_RENDERBUFFER, _colorRenderbuffer);
    [self.glContext renderbufferStorage:GL_RENDERBUFFER fromDrawable:self.glLayer];
    glGetRenderbufferParameteriv(GL_RENDERBUFFER, GL_RENDERBUFFER_WIDTH, &_framebufferWidth);
    glGetRenderbufferParameteriv(GL_RENDERBUFFER, GL_RENDERBUFFER_HEIGHT, &_framebufferHeight);

    //check success
    if (glCheckFramebufferStatus(GL_FRAMEBUFFER) != GL_FRAMEBUFFER_COMPLETE) {
        NSLog(@"Failed to make complete framebuffer object: %i", glCheckFramebufferStatus(GL_FRAMEBUFFER));
    }
}

- (void)tearDownBuffers
{
    if (_framebuffer) {
        //delete framebuffer
        glDeleteFramebuffers(1, &_framebuffer);
        _framebuffer = 0;
    }

    if (_colorRenderbuffer) {
        //delete color render buffer
        glDeleteRenderbuffers(1, &_colorRenderbuffer);
        _colorRenderbuffer = 0;
    }
}

- (void)drawFrame {
    //bind framebuffer & set viewport
    glBindFramebuffer(GL_FRAMEBUFFER, _framebuffer);
    glViewport(0, 0, _framebufferWidth, _framebufferHeight);

    //bind shader program
    [self.effect prepareToDraw];

    //clear the screen
    glClear(GL_COLOR_BUFFER_BIT); glClearColor(0.0, 0.0, 0.0, 1.0);

    //set up vertices
    GLfloat vertices[] = {
        -0.5f, -0.5f, -1.0f, 0.0f, 0.5f, -1.0f, 0.5f, -0.5f, -1.0f,
    };

    //set up colors
    GLfloat colors[] = {
        0.0f, 0.0f, 1.0f, 1.0f, 0.0f, 1.0f, 0.0f, 1.0f, 1.0f, 0.0f, 0.0f, 1.0f,
    };

    //draw triangle
    glEnableVertexAttribArray(GLKVertexAttribPosition);
    glEnableVertexAttribArray(GLKVertexAttribColor);
    glVertexAttribPointer(GLKVertexAttribPosition, 3, GL_FLOAT, GL_FALSE, 0, vertices);
    glVertexAttribPointer(GLKVertexAttribColor,4, GL_FLOAT, GL_FALSE, 0, colors);
    glDrawArrays(GL_TRIANGLES, 0, 3);

    //present render buffer
    glBindRenderbuffer(GL_RENDERBUFFER, _colorRenderbuffer);
    [self.glContext presentRenderbuffer:GL_RENDERBUFFER];
}

- (void)viewDidLoad
{
    [super viewDidLoad];
    //set up context
    self.glContext = [[EAGLContext alloc] initWithAPI: kEAGLRenderingAPIOpenGLES2];
    [EAGLContext setCurrentContext:self.glContext];

    //set up layer
    self.glLayer = [CAEAGLLayer layer];
    self.glLayer.frame = self.glView.bounds;
    [self.glView.layer addSublayer:self.glLayer];
    self.glLayer.drawableProperties = @{kEAGLDrawablePropertyRetainedBacking:@NO, kEAGLDrawablePropertyColorFormat: kEAGLColorFormatRGBA8};

    //set up base effect
    self.effect = [[GLKBaseEffect alloc] init];

    //set up buffers
    [self setUpBuffers];

    //draw frame
    [self drawFrame];
}

- (void)viewDidUnload
{
    [self tearDownBuffers];
    [super viewDidUnload];
}

- (void)dealloc
{
    [self tearDownBuffers];
    [EAGLContext setCurrentContext:nil];
}
@end
```

<img src="http://image.smartjames.cn/20200813021236.png" style="zoom:50%" />

### AVPlayerLayer
    由AVFoundation框架提供用于播放视频的图层。
因为本身是CALayer的子类，所以也支持layer该有的效果，例如边框，圆角，蒙版，阴影等。



## 7.隐式动画

### 事务
通过CATransaction类来管理，通过begin和commit的入栈和出栈来管理栈中的动画事务，栈中的动画默认执行时长0.25秒    
不显示的调用begin，在runloop一个周期内，也会收集所有属性的变化，做一些0.25秒的动画。    
CATransaction 有completionBlock方法，动画执行完后会自动调用。

UIView会默认禁用关联图层的动画效果，否则对于UIview的属性修改都会形成动画效果，这不是我们想要的。要实现UIView的动画，使用UIView的animateWithDuration：方法；

### CATransition
控制移动的动画类，使用方法如下

```
CATransition *transition = [CATransition animation];
transition.type = kCATransitionPush;
transition.subtype = kCATransitionFromLeft;
self.colorLayer.actions = @{@"backgroundColor": transition};
```

CALayer的属性值变化，其实在设置的当时已经变化，只是视图的更新会延后，CALayer在动画过程中，需要知道当前显示在屏幕上的属性值的记录，所以提供-presentationLayer方法来访问当前呈现的图层。

<img src="http://image.smartjames.cn/mweb/20200916/16002639342476.jpeg" style="zoom:50%" />

大多数情况下，不需要直接访问呈现图层，但在以下俩种情况下呈现图层会很有用，一个是同步动画，一个是处理用户交互。

- 如果实现一个基于定时器的动画，而不仅仅是基于事务的动画，这个时候准确地知道在某个时刻图层显示在什么位置就会对正确摆放图层很有用。
- 想让动画的图层响应用户输入，可以使用++-hitTest:++ 方法判断指定图层是否被触摸，此时响应的是呈现图层，而不是动画结束后的模型图层。


## 8.显式动画
    显式动画相对于隐式动画，是用户自己主动创建的一种动画类型，与隐式动画做的关于属性动画不同。
### 属性动画
    作用于图层的某个单一属性，并指定了它的一个或一连串目标值，分为基础动画和关键帧动画，即CABasicAnimation和CAKeyframeAnimation

### 基础动画
    最简单的形式就是从一个值改变到另一个值，这也是CABasicAnimation最主要的功能。提供了fromValue, toValue, byValue三个属性。fromValue代表了动画开始之前属性的值，toValue代表了动画结束之后的值，byValue代表了动画执行过程中改变的相对值。


```
 CABasicAnimation *animation = [CABasicAnimation animation];
    animation.keyPath = @"backgroundColor";
    animation.toValue = (__bridge id)color.CGColor;
    [self.colorLayer addAnimation:animation forKey:nil];
```
上面这种动画，是使layer变换颜色，但是动画结束后又立刻变回原始值，因为这里使用的显式动画改变的是呈现层，不是模型层（隐式动画通过赋值达到改变模型层）。解决方案是动画完成前赋值给模型层。


### 关键帧动画
    CABasicAnimation揭示了大多数隐式动画背后依赖的机制，这的确很有趣，但是显式地给图层添加CABasicAnimation相较于隐式动画而言，只能说费力不讨好。
CAKeyframeAnimation是另一种UIKit没有暴露出来但功能强大的类。不光能设置单一值，还能设置一串值。


```
CAKeyframeAnimation *animation = [CAKeyframeAnimation animation];
    animation.keyPath = @"backgroundColor";
    animation.duration = 2.0;
    animation.values = @[
                         (__bridge id)[UIColor blueColor].CGColor,
                         (__bridge id)[UIColor redColor].CGColor,
                         (__bridge id)[UIColor greenColor].CGColor,
                         (__bridge id)[UIColor blueColor].CGColor ];
    [self.colorLayer addAnimation:animation forKey:nil];
```
上面给关键帧的values赋值了一个颜色组，每个颜色默认动画0.25秒。


```
CAKeyframeAnimation *animation = [CAKeyframeAnimation animation];
    animation.keyPath = @"position";
    animation.duration = 4.0;
    animation.path = bezierPath.CGPath;
    [shipLayer addAnimation:animation forKey:nil];
```
关键帧不光能接受一组值，还能接受一个路径，使用Core Graphics函数定义运动序列来绘制动画。移动中通过设置rotationMode = kCAAnimationRotateAuto达到图层垂直于路径。

<img src="http://image.smartjames.cn/mweb/20200916/16002639342490.jpeg" style="zoom:40%" />


### 虚拟属性
    前面的属性动画都针对真实属性，例如backgroundColor, position等。如果想对一个属性的路径，或不是真实属性做动画，就是虚拟属性。


```
CABasicAnimation *animation = [CABasicAnimation animation];
animation.keyPath = @"transform";
animation.toValue = [NSValue valueWithCATransform3D: CATransform3DMakeRotation(M_PI, 0, 0, 1)];
```
正常的属性赋值是上面这样，使用虚拟属性可以向下面一样使用

```
CABasicAnimation *animation = [CABasicAnimation animation];
animation.keyPath = @"transform.rotation";
animation.byValue = @(M_PI * 2);
```
用transform.rotation而不是transform做动画的好处如下：
- 我们可以不通过关键帧一步旋转多于180度的动画。
- 可以用相对值而不是绝对值旋转（设置byValue而不是toValue）。
- 可以不用创建CATransform3D，而是使用一个简单的数值来指定角度。
- 不会和transform.position或者transform.scale冲突（同样是使用关键路径来做独立的动画属性）。


### 过渡/变幻
    过渡并不像属性动画那样平滑地在两个值之间做动画，而是影响到整个图层的变化。过渡动画首先展示之前的图层外观，然后通过一个交换过渡到新的外观。

使用CATransition类，有4种变换类型。
- kCATransitionFade 
- kCATransitionMoveIn 
- kCATransitionPush 
- kCATransitionReveal

和4个方向
- kCATransitionFromRight 
- kCATransitionFromLeft 
- kCATransitionFromTop 
- kCATransitionFromBottom

<img src="http://image.smartjames.cn/mweb/20200916/16002639342502.jpeg" style="zoom:50%" />


### 自定义动画
    过渡是一种对那些不太好做平滑动画属性的强大工具，但是CATransition的提供的动画类型太少了。
UIView +transitionFromView:toView:duration:options:completion: 提供的过渡选项却很多
- UIViewAnimationOptionTransitionFlipFromLeft 
- UIViewAnimationOptionTransitionFlipFromRight
- UIViewAnimationOptionTransitionCurlUp 
- UIViewAnimationOptionTransitionCurlDown
- UIViewAnimationOptionTransitionCrossDissolve 
- UIViewAnimationOptionTransitionFlipFromTop 
- UIViewAnimationOptionTransitionFlipFromBottom



## 9.图层时间

### CAMediaTiming 协议
CAMediaTiming协议定义了在一段动画内用来控制逝去时间的属性的集合，CALayer和CAAnimation都实现了此协议。
    
常用属性
- duration      动画持续时间
- repeatCount   重复次数
- speed         倍速，0暂停，负数回放动画
- beginTime     动画开始之前的的延迟时间
- timeOffset    动画快进到某一点
- fillMode      动画结束是否保持之前的状态，定义了几个类型

```
kCAFillModeForwards
kCAFillModeBackwards
kCAFillModeBoth
kCAFillModeRemoved
```
- autoreverses  是否进行反向动画，实现门的开关

<img src="http://image.smartjames.cn/mweb/20200916/16002639342517.jpeg" style="zoom:50%" />


## 10.时间函数
    为了让动画显得更自然，而不是很机械，需要使用时间函数控制动画速率
    
### CAMediaTimingFunction
    CAAnimation的timingFunciont属性接收一个CAMediaTimingFunction对象。
包含以下类型
- kCAMediaTimingFunctionLinear 
- kCAMediaTimingFunctionEaseIn 
- kCAMediaTimingFunctionEaseOut 
- kCAMediaTimingFunctionEaseInEaseOut
- kCAMediaTimingFunctionDefault

### UIView的时间函数
    UIView的动画也支持时间函数，参考CAAnimation的函数
- UIViewAnimationOptionCurveEaseInOut 
- UIViewAnimationOptionCurveEaseIn 
- UIViewAnimationOptionCurveEaseOut 
- UIViewAnimationOptionCurveLinear

### 自定义时间函数
除了+functionWithName:之外，CAMediaTimingFunction同样有另一个构造函数，一个有四个浮点参数的+functionWithControlPoints::::
三次贝塞尔时间函数，自定义函数的入参就是图中的几个控制点
参数就是图中的几个控制点

<img src="http://image.smartjames.cn/mweb/20200916/16002639342534.jpeg" style="zoom:50%" />
## 12.性能调优

### 影响GPU， 降低图层绘制
- 太多的几何结构 - 这发生在需要太多的三角板来做变换，以应对处理器的栅格化的时候。现代iOS设备的图形芯片可以处理几百万个三角板，所以在Core Animation中几何结构并不是GPU的瓶颈所在。但由于图层在显示之前通过IPC发送到渲染服务器的时候（图层实际上是由很多小物体组成的特别重量级的对象），太多的图层就会引起CPU的瓶颈。这就限制了一次展示的图层个数（见本章后续“CPU相关操作”）。

- 重绘 - 主要由重叠的半透明图层引起。GPU的填充比率（用颜色填充像素的比率）是有限的，所以需要避免重绘（每一帧用相同的像素填充多次）的发生。在现代iOS设备上，GPU都会应对重绘；即使是iPhone 3GS都可以处理高达2.5的重绘比率，并任然保持60帧率的渲染（这意味着你可以绘制一个半的整屏的冗余信息，而不影响性能），并且新设备可以处理更多。

- 离屏绘制 - 这发生在当不能直接在屏幕上绘制，并且必须绘制到离屏图片的上下文中的时候。离屏绘制发生在基于CPU或者是GPU的渲染，或者是为离屏图片分配额外内存，以及切换绘制上下文，这些都会降低GPU性能。对于特定图层效果的使用，比如圆角，图层遮罩，阴影或者是图层光栅化都会强制Core Animation提前渲染图层的离屏绘制。但这不意味着你需要避免使用这些效果，只是要明白这会带来性能的负面影响。

- 过大的图片 - 如果视图绘制超出GPU支持的2048x2048或者4096x4096尺寸的纹理，就必须要用CPU在图层每次显示之前对图片预处理，同样也会降低性能。

### 影响CPU，延迟动画开始时间
- 布局计算 - 如果你的视图层级过于复杂，当视图呈现或者修改的时候，计算图层帧率就会消耗一部分时间。特别是使用iOS6的自动布局机制尤为明显，它应该是比老版的自动调整逻辑加强了CPU的工作。

- 视图惰性加载 - iOS只会当视图控制器的视图显示到屏幕上时才会加载它。这对内存使用和程序启动时间很有好处，但是当呈现到屏幕上之前，按下按钮导致的许多工作都会不能被及时响应。比如控制器从数据库中获取数据，或者视图从一个nib文件中加载，或者涉及IO的图片显示（见后续“IO相关操作”），都会比CPU正常操作慢得多。

- Core Graphics绘制 - 如果对视图实现了-drawRect:方法，或者CALayerDelegate的-drawLayer:inContext:方法，那么在绘制任何东西之前都会产生一个巨大的性能开销。为了支持对图层内容的任意绘制，Core Animation必须创建一个内存中等大小的寄宿图片。然后一旦绘制结束之后，必须把图片数据通过IPC传到渲染服务器。在此基础上，Core Graphics绘制就会变得十分缓慢，所以在一个对性能十分挑剔的场景下这样做十分不好。

- 解压图片 - PNG或者JPEG压缩之后的图片文件会比同质量的位图小得多。但是在图片绘制到屏幕上之前，必须把它扩展成完整的未解压的尺寸（通常等同于图片宽 x 长 x 4个字节）。为了节省内存，iOS通常直到真正绘制的时候才去解码图片（14章“图片IO”会更详细讨论）。根据你加载图片的方式，第一次对图层内容赋值的时候（直接或者间接使用UIImageView）或者把它绘制到Core Graphics中，都需要对它解压，这样的话，对于一个较大的图片，都会占用一定的时间。


### 性能优化实例
UITabelViewCell 使用了阴影和圆角等触发离屏渲染，影响性能。这是可以打开shouldRasterize， 这是光栅化，缓存阴影和圆角到位图，下次重绘时直接取用缓存。虽然光栅会也会触发离屏渲染，但是能够缓解cell的滑动不断渲染，也能一定程度上提高性能。


## 13. 高效绘图

软件绘图指不由GPU协助的绘图，iOS中一般通过Core Graphics完成，但是和Core Animation和 OpenGL相比，Core Graphics要慢不少。
软件绘图还会消耗不少内存，一个Context上下文相当于图层宽×高×4字节，分辨率为2048*1526的iPad来说，就是12MB内存，且每次重绘都需要抹掉内存重新分配，代价很大，所以要尽量避免绘图。

### 使用CAShapeLayer 提高效率
例如实现手指滑动绘制线条，简单做法如下：

```
- (void)touchesMoved:(NSSet *)touches withEvent:(UIEvent *)event
{
    //get the current point
    CGPoint point = [[touches anyObject] locationInView:self];

    //add a new line segment to our path
    [self.path addLineToPoint:point];

    //redraw the view
    [self setNeedsDisplay];
}

- (void)drawRect:(CGRect)rect
{
    //draw path
    [[UIColor clearColor] setFill];
    [[UIColor redColor] setStroke];
    [self.path stroke];
}
```
使用了Core Graphics，画的越多，程序越慢，每次都会重绘整个路径，效率很低。
使用CAShapeLayer替代Core Graphics，性能大幅提高。

```
+ (Class)layerClass
{
    //this makes our view create a CAShapeLayer
    //instead of a CALayer for its backing layer
    return [CAShapeLayer class];
}
- (void)touchesMoved:(NSSet *)touches withEvent:(UIEvent *)event
{
    //get the current point
    CGPoint point = [[touches anyObject] locationInView:self];

    //add a new line segment to our path
    [self.path addLineToPoint:point];

    //update the layer with a copy of the path
    ((CAShapeLayer *)self.layer).path = self.path.CGPath;
}
```

### 脏矩形
为了减少不必要的绘制，MacOS和iOS设备将会把屏幕区分为需要重绘的区域和不需要重绘的区域。那些需要重绘的部分被称作『脏区域』。

当一个视图被改动过了，TA可能需要重绘。但是很多情况下，只是这个视图的一部分被改变了，所以重绘整个寄宿图就太浪费了。所以提供了-setNeedsDisplayInRect:来把脏区域作为参数传入，能够提高部分效率。

相比依赖于Core Graphics为你重绘，裁剪出自己的绘制区域可能会让你避免不必要的操作。那就是说，如果你的裁剪逻辑相当复杂，那还是让Core Graphics来代劳吧，记住：当你能高效完成的时候才这样做。

CGRectIntersectsRect()能判断指定的rect在不在脏区域内，从而判断是否需要重新绘制这一块。

### 异步绘制
- CATiledLayer 分片显示的layer，在多线程中为每个小块同时调用drawLayer:inContext:方法，避免阻塞交互。
- drawsAsynchronously 它自己的-drawLayer:inContext:方法只会在主线程调用，但是CGContext并不等待每个绘制命令的结束。相反地，它会将命令加入队列，当方法返回时，在后台线程逐个执行真正的绘制。==根据苹果的说法。这个特性在需要频繁重绘的视图上效果最好（比如我们的绘图应用，或者诸如UITableViewCell之类的），对那些只绘制一次或很少重绘的图层内容来说没什么太大的帮助。==

    


## 14.图像IO
图像耗费时间主要包括2个部分，加载到内存和解压。

### 解决加载问题

**文件读取**: 使用异步加载的方式，GCD或NSOperationQueue，在子线程中读取图片，在主线程中显示图片。

**延迟解压**: 当加载图片之后，iOS通常会延迟解压图片的时间。这就会在准备绘制图片的时候影响性能，因为需要在绘制之前进行解压（通常是消耗时间的问题所在）。

使用+imageNamed:方法能避免延时加载，会在加载图片后立刻解压。但对于生成的图片，相册照片或下载的图片无效。
此时可以使用ImageIO框架实现：

```
NSInteger index = indexPath.row;
NSURL *imageURL = [NSURL fileURLWithPath:self.imagePaths[index]];
NSDictionary *options = @{(__bridge id)kCGImageSourceShouldCache: @YES}; 
CGImageSourceRef source = CGImageSourceCreateWithURL((__bridge CFURLRef)imageURL, NULL);
CGImageRef imageRef = CGImageSourceCreateImageAtIndex(source, 0,(__bridge CFDictionaryRef)options);
UIImage *image = [UIImage imageWithCGImage:imageRef]; 
CGImageRelease(imageRef);
CFRelease(source);
```
这样就可以使用kCGImageSourceShouldCache来创建图片，强制图片立刻解压，然后在图片的生命周期保留解压后的版本。

### 分辨率替换
在滑动的时候使用分辨率图片，滑动停止时替换为高分辨率图片。

### 图片缓存
imageNamed: 在内存中自动缓存了解压后的图片，但也有以下几个问题
- [UIImage imageNamed:]方法仅仅适用于在应用程序资源束目录下的图片，但是大多数应用的许多图片都要从网络或者是用户的相机中获取，所以[UIImage imageNamed:]就没法用了。

- [UIImage imageNamed:]缓存用来存储应用界面的图片（按钮，背景等等）。如果对照片这种大图也用这种缓存，那么iOS系统就很可能会移除这些图片来节省内存。那么在切换页面时性能就会下降，因为这些图片都需要重新加载。对传送器的图片使用一个单独的缓存机制就可以把它和应用图片的生命周期解耦。

- [UIImage imageNamed:]缓存机制并不是公开的，所以你不能很好地控制它。例如，你没法做到检测图片是否在加载之前就做了缓存，不能够设置缓存大小，当图片没用的时候也不能把它从缓存中移除。


### NSCache
NSCache和NSDictionary类似，你可以通过-setObject:forKey:和-object:forKey:方法分别来插入，检索。和字典不同的是，NSCache在系统低内存的时候自动丢弃存储的对象。
-setCountLimit: 设置缓存大小
-setObject:forKey:cost:来对每个存储的对象指定消耗的值来提供一些参考，数值越大说明很昂贵，降低丢弃的优先级。
-setTotalCostLimit: 指定全体缓存的大小

### PNG和JPEG区别
JPEG不包含透明度通道，压缩算法复杂，文件小，解压时会耗费时间多
PNG包含透明度通道，算法简单，文件大，解压时耗费时间少

JPEG适合噪点大的图片， PNG适合扁平颜色，锋利的线条和渐变色图片。



## 15.图层性能

### 光栅化

在第四章『视觉效果』中我们提到了`CALayer`的`shouldRasterize`属性，它可以解决重叠透明图层的混合失灵问题。同样在第12章『速度的曲调』中，它也是作为绘制复杂图层树结构的优化方法。

启用`shouldRasterize`属性会将图层绘制到一个屏幕之外的图像。然后这个图像将会被缓存起来并绘制到实际图层的`contents`和子图层。如果有很多的子图层或者有复杂的效果应用，这样做就会比重绘所有事务的所有帧划得来得多。但是光栅化原始图像需要时间，而且还会消耗额外的内存。

当我们使用得当时，光栅化可以提供很大的性能优势（如你在第12章所见），但是一定要避免作用在内容不断变动的图层上，否则它缓存方面的好处就会消失，而且会让性能变的更糟。

为了检测你是否正确地使用了光栅化方式，用Instrument查看一下Color Hits Green和Misses Red项目，是否已光栅化图像被频繁地刷新（这样就说明图层并不是光栅化的好选择，或则你无意间触发了不必要的改变导致了重绘行为）。


### 混合和过度绘制

GPU每一帧可以绘制的像素有最大显示fill rate, 所以必须有选择的绘制区域。GPU会放弃绘制哪些被其他图层遮挡的像素，但是要计算出一个图层是否被遮挡也是相当复杂并且会消耗处理器资源。同样，合并不同图层的透明重叠像素(混合)消耗的资源也是相当可观的。所以为了加速处理进程，非必要时刻不要用透明图层。

- 给视图`backgroundColor` 设置一个固定的不透明颜色。
- 设置`opaque`属性为YES

这样做减少了混合行为（因为编译器知道在图层之后的东西都不会对最终的像素颜色产生影响），且计算得到了加速。
