---
title: "Spine的纹理压缩和半透显示"
subtitle: "学习一下spine的纹理相关"
date: 2018-05-29 12:00:00
layout: post
author: "aibow"
header-style: text
tags:
  - spine
  - learning
---
![img](https://oscimg.oschina.net/oscnet/26359ed4e5dab41ab86791aec03bee5a051.jpg)

## 纹理压缩

如果使用Spine的默认输出格式，是这样的

![img](https://oscimg.oschina.net/oscnet/fd1cfd287e7b72b9a2ee006a0101c556c49.jpg)

但是输出纹理的格式必须是未压缩的RGBA，如果你将图片格式改为ETC2或者ASTC，就会变成这样：

![img](https://oscimg.oschina.net/oscnet/3debd4bf177950422f3c21b6ed5a4df1c07.jpg)

而这是绝对不能容忍的画面瑕疵。



ETC2的压缩质量绝不至于这样差，之所以会变成这样，在于Spine的默认输出格式是勾选了“预乘（premultiplied alpha）”参数的，这种做法会将图片原始的RGB通道预先乘以透明度保存成文件，显示时再用特殊的Shader乘回去。而ETC2的压缩算法并没有考虑过这种情况，由此导致了压缩质量降低。

而之所以Spine要使用预乘（premultiplied alpha），目的是为了解决透明图片的采样问题。

![img](https://oscimg.oschina.net/oscnet/55f797791427487602a2199719d7c910660.jpg)

如图，左边的像素点是纯透明，右边的像素点是白色点，如果在红点处采样，获得的颜色值就是(0.5,0.5,0.5,0.5)

![img](https://oscimg.oschina.net/oscnet/0a240b64f9c701561db38fc79a7dd29ea92.jpg)

而正确的颜色值应该是(1,1,1,0.5)，因为变化的只应该是透明度，不能因为达到边缘，连图像本身的亮度都降低。

![img](https://oscimg.oschina.net/oscnet/dfc05fc2b720cf20eb161b4ba9ad9690c58.jpg)

预乘是能完美的解决这个问题的，它生成的像素点是(0x0,0x0,0x0,0),(1x1,1x1,1x1,1)，采样后的颜色是0.5,0.5,0.5,0.5，但还原的时候还要再除以透明度，所以结果是(0.5/0.5,0.5/0.5,0.5/0.5,0.5)=(1,1,1,0.5)，能够得到正确的值。

而除了预乘以外，还有另一个方法，就是让透明度为0的区域也填充有颜色。虽然有颜色，但因为透明度是0，看上去依然是透明，如图：

![img](https://oscimg.oschina.net/oscnet/6f6336b3b3c9df14bc0a4f39b618b9e1f88.jpg)

这种操作叫做出血(bleeding)。排除掉透明通道，可以看到出血后图片会复制边缘的像素到临近的透明区域。

![img](https://oscimg.oschina.net/oscnet/8c4743b76edcacb4a12788941330ae4752a.jpg)

虽然在某些特殊的采样角度下会有错误，但大部分情况没问题，而这种方法可以正常支持压缩纹理。

Bleeding可以在Spine的输出窗体设置，也可以勾选Unity图片的Alpha Is Transparency激活。



所以，我们应该取消Spine的premultiplied alpha选项，并勾选Bleeding选项来生成普通的图片（SpineEditorUtilities.cs有自动导入脚本，需要修改，否则图片参数会被它重新设置回去）。

Shader内，直接用最普通的方式绘制就好了。

![img](https://oscimg.oschina.net/oscnet/8195f9a84017127f1c13d02d2c2536e0c7f.jpg)

瑕疵依然存在，但是回到了普通不透明图片的压缩质量水准。（具体质量要看使用的压缩格式，比如ASTC肯定要比ETC2更好）



——但是，Spine并不能直接用普通的Alpha Blend方式绘制贴图。

Spine的整个Shader都是基于Blend One OneMinusSrcAlpha，也就是预乘混合模式。它要求frag输出的颜色通道必须是预乘过Alpha通道的，所以，如果输入的图片是未预乘的，就要在采样后乘上透明度，再输出。

```text
col = tex2D(_MainTex,xxxxx);
col.rgb *= col.a;
```

倒也简单，但为什么不直接将混合模式直接改成普通的Blend SrcAlpha OneMinusSrcAlpha呢？这就是下个话题的内容——





## 如何用一个PASS同时绘制Alpha Blend和Additive物体？

Alpha Blend 对应的是 Blend SrcAlpha OneMinusSrcAlpha，

相当于lerp(x.rgb,y.rgb,x.a)，也就是result = (x.r, x.g, x.b) * x.a + (y.r, y.g, y.b) * (1 - x.a)



Additive则是result = (x.r, x.g, x.b) *x.a + (y.r, y.g, y.b)



而预乘混合Blend One OneMinusSrcAlpha是result = (x.r, x.g, x.b) + (y.r, y.g, y.b) * (1 - x.a)，由于我们的纹理并不是预乘过的，而是在shader内相乘，所以是za = x.a，result = (x.r, x.g, x.b) * za + (y.r, y.g, y.b) * (1 - x.a)，结果上和Alpha Blend相同。

但这里，za和x.a其实是不同的参数，我们可以让它是不同的值。如果za和x.a始终相等，就是AlphaBlend，如果让za=x.a，同时把x.a赋值为0，那么结果就是result = (x.r, x.g, x.b) *x.a + (y.r, y.g, y.b) * (1 - 0)。

和Additive是相同的。



Spine就是用这种方式同时显示AlphaBlend和Additive的图片的，它的顶点色是预乘过的，比如白色50%透明度就是（0.5,0.5,0.5,0.5），而如果是Additive的部分，白色50%透明度就是(0.5,0.5,0.5,0)，因为透明度为0，所以输出的x.a永远都是0，效果就和Additive相同。

这样直接乘在颜色值上，然后用Blend One OneMinusSrcAlpha输出就可以了。



如果并不是将透明度重置为0，而是取一个中间值，还可以得到Alpha Blend和Additive中间的一个结果，在Alpha Blend太暗，Additive太亮的时候可以考虑（一般特效美术会选择用两个不同材质的纹理叠加，其实没必要）





## 如何做Spine的半透显示？

GrabPass是不可取的，因为存在透明物体的半透叠加。所以唯一的方法就是绘制到RT上再显示。

这时候就会遇到同时绘制Alpha Blend和Additive的问题。需要将摄像机的ClearColor设置成0,0,0,0（而不是0,0,0,1），Alpha的物体不能有ColorMask，Additive的物体需要ColorMask RGB（或者Alpha值为0）。

而显示RT时，必须用Blend One OneMinusSrcAlpha才能正确，否则Addtive的部分会没效果。

![img](https://oscimg.oschina.net/oscnet/4f54f8fae6eae1848ab3060ba9382918e17.png)用SrcAlpha OneMunusSrcAlpha绘制

![img](https://oscimg.oschina.net/oscnet/f2640f3b4ecef72589eba445eabb7b71689.png)用One OneMunusSrcAlpha绘制

在这种情况下，设置为半透不能只修改alpha值，而且是要同时修改全部通道。

Blend One OneMinusSrcAlpha的物体只修改Color的Alpha值，相当于在AlphaBlend和Additive的显示效果之间调整，其实还挺便利的。这个中间的部分其实很适合用来模拟科幻场景的空气投影屏幕，因为AlphaBlend太实，Additive又太像光。

而真正的空气投影屏幕谁也没见过，毕竟都没诞生。





## 动态立绘应当启用Mipmap

很多人对Mipmap有误解，认为这是一个优化手段，而且是在处理远近物体的时候用的。

Mipmap最初的目的其实是为了处理纹理走样问题，优化只是一个副产品。

比较著名的示例是这个，又称为摩尔纹。

![img](https://oscimg.oschina.net/oscnet/2614f0ad5e211cec6e3128dad4bce817d63.jpg)

在立绘的情况下是这个，也算是纹理锯齿的一种。

![img](https://oscimg.oschina.net/oscnet/8a12c7310355c2d5d2e8a7b2dbd373b249a.png)

至于出现的原因？纹理过滤时，图像的放大其实和PS里是类似的，也就是普通的2次线性插值，但是缩小时则完全不同。PS将图像缩小一倍，目标像素是对应的4个像素的平均值，但是在纹理过滤的时候，只会取的最近的1个像素的值。

很显然，这是为了效率考虑。

mipMap则将这4个像素的平均值预先储存了一份，这种情况下就可以直接取值了。其实效率是差不多的，但只有mipMap的才是正确的。

但在两个mipMap层级中间的时候，即使使用三向过滤或者多重采样过滤，会从两个mipMap层级之间插值，依然还是不正确的。因为采样了低分辨率的图像，还会导致图像产生模糊。

也就是在信号学所说的“在不提高分辨率的情况下，我们只能把信号中无法还原的高频部分抛弃掉，避免出现剧烈的变化，来实现所谓的反走样”，即是，想要反走样，就必须承担一定程度的模糊。想要反走样又不想模糊，只有一个办法，提升分辨率。



出现走样并不需要3D，只要你的图片存在两个像素被并进一个像素显示的情况（这通过缩放也可以达成），如果你的图片分辨率高于屏幕分辩率就更是如此。

不过，大部分走样效果并不明显，因为人眼会自己修正。但是，如果你的物体是运动的，走样就会产生另一种效果“闪烁”，这就非常明显了。上面的锯齿其实也算是一种线段的时隐时现。

如果你的图片是静态的，用一点点走样换取图片的“清晰”其实更合算。但是动态立绘，请不要。

当然，如果你的分辨率极高，高到人眼无法分辨，走样确实也无所谓了。

但这样的话，模糊也同样无所谓。

而且别忘了，Mipmap确实在很多情况下可以提高性能。



此外，开了MipMap请务必开启纹理各项异性过滤（也就是那个Anisotropic filtering），这能显著缓解mipMap模糊的问题。

现世代这东西的成本已经可以忽略了，建议直接在Quailty里设置为forced On，这也是目前大部分项目的做法。