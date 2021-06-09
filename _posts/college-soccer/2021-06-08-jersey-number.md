---
layout: post
categories: college-soccer
tag: blog
title: "【水技术】Unity自定义球衣和动态号码的实现"
---

自定义球衣的实现，我感觉对于正常业界公司来说应该是个有很成熟解决方案的不是问题的问题。。。
然而我并没有轻松在网上找到直接可以抄的。。。而且前后我其实做了三个方案才得到目前这个我觉得是比较通用的版本。。所以也许写一个小技术分享还是有点价值（虽然我只是又写不动代码了罢。。。）


<!--more-->

{% include imgdesc.html url="assets/blog/jersey-tech/jerseyeditor.gif" description="球衣编辑器" %}

这里就直接介绍最后一个完善的方案了。它能够实现不需要与特定球衣mesh绑定的球衣贴图或自定义纹理+颜色；球衣/短裤号码；以及队徽/赞助商decal贴纸。基本无额外开销（仅对字体/队徽/赞助商纹理采样）

第一步是3D模型的准备。基本的建模，蒙皮等不必赘述。

{% include imgdesc.html url="assets/blog/jersey-tech/blender.png" description="BVB黏土人。。。" %}


对于动态球衣需要做的就是为球衣的skinned mesh准备三套UV，分别对应球衣基本贴图，号码，和队徽赞助商等贴纸。

{% include imgdesc.html url="assets/blog/jersey-tech/uv1.png" description="UV1，对应球衣的基本颜色或纹理样式贴图（这里扒了FIFA的贴图做demo）" %}


{% include imgdesc.html url="assets/blog/jersey-tech/uv2.png" description="UV2，对应号码的部分。这里只需要把号码放在大概在中间的位置就行了（我随便用了能哥的号码图当对齐参考）。在shader里我们会根据uv2是否在一个方块区域内来决定是否对字体texture采样" %}

{% include imgdesc.html url="assets/blog/jersey-tech/uv2-shorts.png" description="UV2，对应号码的部分。这是球裤的展开。也是将号码区域对齐到大概中央的位置。" %}

{% include imgdesc.html url="assets/blog/jersey-tech/uv3.png" description="UV3，对应队徽，赞助商等贴纸的部分。因为我建的mesh比较坑顶点太少面积太小，在shader里我是把又上0.25的方块算作队徽，覆盖到赞助商图片上的" %}

{% include imgdesc.html url="assets/blog/jersey-tech/uv3-shorts.png" description="UV3，裤子上只有队徽，只要把相应区域对上就行" %}

这样就完成了模型的准备工作。接下来导入Unity（或其他实时渲染框架）。我们首先需要一些script和shader来完成动态生成号码的部分。这里主要要解决的问题就是在shader里我们的输入是uv2，输出是一个alpha值，对应到是否有印号。主要的思路是根据uv2确定当前fragment是否在印号的矩形区域内（用`step`函数可以方便做到）。如果在该矩形区域内，我们对uv2做简单的平移和缩放变换，使它的四个角分别对应于字体纹理中给定号码的uv。号码对应的字体纹理的四个角的uv值可以用script通过uniform传入shader。

```c
float4 _NumberCornerUV; // xy: top left uv, zw: bottom right uv
float4 _NumberCornerUV2; // xy: top left uv, zw: bottom right uv
float _NumberOffsetLeftX;   // base X offset of first digit from center
// ...
float getAlphaFromFontTex(float2 uv2, float digitOffsetX, float4 fontUVRect)
{
    float2 digitWidthHeight = fontUVRect.zw - fontUVRect.xy;
    float2 bottomLeftUV = float2(0.5 + digitOffsetX * _DigitScale, 0.5 + _DigitOffsetY)
        - _DigitScale * 0.5 * digitWidthHeight;
    float2 adjustedUV = (uv2 - bottomLeftUV) / _DigitScale;
    adjustedUV.y = -adjustedUV.y * _DigitWidthRatio;
    adjustedUV += fontUVRect.xy;
    float2 hv = step(fontUVRect.xy, adjustedUV) * step(adjustedUV, fontUVRect.zw);
    float a = hv.x * hv.y;  // limit rect for digit
    return a * tex2D(_NumberFontTex, adjustedUV).a;
}
```

对于一位号码和两位号码还需要些特殊处理

```c
if (_HasNumber)
{
    a = getAlphaFromFontTex(IN.uv2_NumberFontTex, _NumberOffsetLeftX, _NumberCornerUV);
    if (_NumberOffsetLeftX < -0.01 && a < 0.001) {
        // 2 digits
        a = getAlphaFromFontTex(IN.uv2_NumberFontTex, -_NumberOffsetLeftX, _NumberCornerUV2);
    }
}

c = lerp(c, _ColorNumber, a);
```

赞助商和队徽的处理类似。step函数取得矩形区域，然后不是单取alpha值而是正常rgb而已。

球衣的纹理颜色我就是用的普通的rgba8 texture，rgb通道分别对应三个颜色进行插值。这里很明显有所谓优化空间，不过普通rgba8在图形处理软件里编辑起来方便，感觉暂时没必要折腾。

{% include imgdesc.html url="assets/blog/jersey-tech/template.png" description="颜色模板借鉴（抄）的是21年博卡青年的客场球衣hhh" %}

```c
fixed4 b = tex2D(_MainTex, IN.uv_MainTex);
// For color template, Albedo comes from a texture tinted by color
fixed4 c = _IsColorTemplateTex
    ? b.r * _Color0 + b.g * _Color1 + b.b * _Color2
    : b;
```

{% include imgdesc.html url="assets/blog/jersey-tech/material.png" description="Unity Material" %}

大概就是这样。并不需要render target啥的。