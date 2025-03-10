---
title: 'Unity 图片'
titleColor: '#aaa,#0acee9'
titleIcon: 'asset:unity'
tags: [ 'Untiy' ]
categories: [ 'Code' ]
description: 'Unity 图片组件的使用'
---

#### 默认设置导入图片为 Sprite
在项目路径下新建文件夹 Assets/Editor 创建DefaultSpriteImporter.cs

``` c#
using UnityEditor;

public class DefaultSpriteImporter : AssetPostprocessor
{
    void OnPreprocessTexture()
    {
        // 获取当前 Texture 的导入器
        TextureImporter importer = (TextureImporter)assetImporter;

        // 设置默认类型为 Sprite
        importer.textureType = TextureImporterType.Sprite;
    }
}
```

也可以批量选中更改

#### Image和RawImage的区别
Image和RawImage都是用来显示图片的组件，但是它们的区别在于：

- Image：Image组件仅支持Sprite类型资源（如精灵图或图集中的子图），可以设置图片的缩放模式，拉伸、填充、适应等，并且可以设置图片的颜色。 
- RawImage：RawImage组件只能显示原始图片，不支持缩放、拉伸、填充等操作，并且只能显示纯色。

性能与适用场景：

- ​Image：推荐用于常规UI元素（按钮、图标等），尤其是需要图集优化、九宫格适配或填充动画的场景
- RawImage：适用于需动态更新纹理的场景（如AR/VR中的实时图像、背景图），或需要直接操作Texture的底层需求。

#### Image的各个属性
Source Image：设置图片资源，可以是Sprite类型资源，也可以是Texture类型资源。  
Color：设置图片的颜色。  
Meterial：设置图片的材质。  
Raycast Target：设置图片是否可以被射线检测。  
Raycast Padding：设置图片的射线检测范围。  
Maskable：设置图片是否可以蒙版。  
Image Type：设置图片的类型，可以是Simple、Sliced、Tiled、Filled。  
- Simple：图片不做任何处理，显示原图。
    - Use Sprite Mesh：勾选后，图片会使用精灵网格，可以减少绘制的面数，提高性能。
    - Preserve Aspect Ratio：勾选后，图片会保持纵横比，不失真。
- Sliced：图片被切割成多张小图，可以设置切割方向、间隔、填充等。 （点击Sprite, 在Inspector中点击Sprite Editor）
    - Pixeles Per Unit：设置每单位像素的切割数量。
- Tiled：图片平铺，可以设置平铺方向、间隔、填充等。
- Filled：图片被填充成矩形，可以设置填充方向、间隔、填充等。

#### RawImage的属性
Texture：设置图片资源，只能是Texture类型资源。  
Color：设置图片的颜色。  
Meterial：设置图片的材质。  
Raycast Target：设置图片是否可以被射线检测。  
Raycast Padding：设置图片的射线检测范围。  
Maskable：设置图片是否可以蒙版。  
UV Rect：设置图片的UV范围。  

#### 圆形头像 (推荐)
[github：https://github.com/kirevdokimov/Unity-UI-Rounded-Corners/tree/master](https://github.com/kirevdokimov/Unity-UI-Rounded-Corners/tree/master)    
先添加Image/RawImage组件, 然后添加ImageWithRoundedCorners组件。    
ImageWithIndependentRoundedCorners 可以分别设置4个角的圆角。


#### 使用shader实现圆形头像 （可以用来学习shader的编写）
1. 创建shader文件 放入 Assets/Resources/Shaders/RoundShape.shader shader 文件在最下面

2. 创建材质文件 Round Material， 放入 Assets/Resources/Materials

3. 打开材质文件，选择shader为 UI/RoundShape

4. 创建图片组件，设置材质。

``` c#
Shader "UI/RoundShape"
{
    Properties
    {
        [PerRendererData] _MainTex ("Sprite Texture", 2D) = "white" {}
        _Color ("Tint", Color) = (1,1,1,1)
 
       
 
        _RoundRadius("Round Radius", Range(0,0.5)) = 0.5
        _Width("Width", Float) = 100
        _Height("Height", Float) = 100
    }
 
    SubShader
    {
        Tags
        { 
            "Queue"="Transparent" 
            "IgnoreProjector"="True" 
            "RenderType"="Transparent" 
            "PreviewType"="Plane"
            "CanUseSpriteAtlas"="True"
        }
 
        Stencil
        {
            Ref [_Stencil]
            Comp [_StencilComp]
            Pass [_StencilOp] 
            ReadMask [_StencilReadMask]
            WriteMask [_StencilWriteMask]
        }
 
        Cull Off
        Lighting Off
        ZWrite Off
        Blend SrcAlpha OneMinusSrcAlpha
 
        Pass
        {
            Name "Default"
        CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #pragma target 2.0
 
            #include "UnityCG.cginc"
            #include "UnityUI.cginc"
 
            #pragma multi_compile __ UNITY_UI_ALPHACLIP
 
            struct appdata_t
            {
                float4 vertex   : POSITION;
                float4 color    : COLOR;
                float2 texcoord : TEXCOORD0;
                UNITY_VERTEX_INPUT_INSTANCE_ID
            };
 
            struct v2f
            {
                float4 vertex   : SV_POSITION;
                fixed4 color    : COLOR;
                float2 texcoord  : TEXCOORD0;
                float4 worldPosition : TEXCOORD1;
                UNITY_VERTEX_OUTPUT_STEREO
            };
 
            fixed4 _Color;
            fixed4 _TextureSampleAdd;
            float4 _MainTex_ST;
            float4 _ClipRect;
            float _RoundRadius;
            float _Width;
            float _Height;
            
            v2f vert(appdata_t IN)
            {
                v2f OUT;
                UNITY_SETUP_INSTANCE_ID(IN);
                UNITY_INITIALIZE_VERTEX_OUTPUT_STEREO(OUT);
                OUT.worldPosition = IN.vertex;
                OUT.vertex = UnityObjectToClipPos(OUT.worldPosition);
 
                //OUT.texcoord = IN.texcoord;
                OUT.texcoord = TRANSFORM_TEX(IN.texcoord, _MainTex);
                OUT.color = IN.color * _Color;
                return OUT;
            }
 
            sampler2D _MainTex;
 
            fixed4 frag(v2f IN) : SV_Target
            {
                half4 color = (tex2D(_MainTex, IN.texcoord) + _TextureSampleAdd) * IN.color;
 
                color.a *= UnityGet2DClipping(IN.worldPosition.xy, _ClipRect);
 
                #ifdef UNITY_UI_ALPHACLIP
                clip (color.a - 0.001);
                #endif
 
                float aspect = _Height/_Width;
 
                float2 center = float2(abs(round(IN.texcoord.x) - _RoundRadius*aspect),abs(round(IN.texcoord.y) - _RoundRadius));
                float dis = distance(fixed2(IN.texcoord.x * _Width,IN.texcoord.y * _Height),fixed2(center.x * _Width,center.y * _Height));
                
                float pwidth = fwidth(dis);
                float alpha = saturate((_RoundRadius * _Height - dis) / pwidth);
                
                //float alpha = step(dis,_RoundRadius * _Height);
                float a = color.a* alpha;
                
 
                float oy = max(step(IN.texcoord.y,_RoundRadius),step((1-_RoundRadius),IN.texcoord.y));
                float ox = max(step(IN.texcoord.x,_RoundRadius*aspect),step((1-_RoundRadius*aspect),IN.texcoord.x));
                color.a = ox * (oy * a + (1-oy) * color.a) + (1-ox) * color.a;
 
                return color;
                
            }
        ENDCG
        }
    }
}
```