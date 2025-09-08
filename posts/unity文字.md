---
title: "Unity 文字"
titleColor: "#aaa,#20B2AA"
titleIcon: "asset:unity"
tags: ["Untiy"]
categories: ["Code"]
description: "Unity 文字组件的使用"
publishDate: "2023-03-10"
updatedDate: "2023-03-10"
---

#### 显示中文

1. 生成 Font Asset
   将 MSYH ttf 字体文件拖入 Unity 编辑器中，可以放在 Assets/Rseources/Fonts 目录下，右键 Create -> TextMeshPro -> Font Asset 即可生成 Font Asset。

2. 在我们初次打开的时候会自动弹出提示框，我们点击 Import TMP Essentials 即可
   点击字体文件 SDF，在 Inspector 面板中，设置图集大小，4096x4096，可以勾选 Multi Atlas Textures 和 Clear Dynamic Data, 然后点击 Apply

3. 创建文字组件 Text-TextMeshPro
   右键 GameObject -> UI -> Text -> TextMeshPro 创建文字组件，然后在 Inspector 面板中设置文字内容和字体样式。将刚才创建的 Font Asset 拖入 Font Asset 字段中。

4. 设置默认中文
   项目顶部 Edit -> Project Settings -> TextMeshPro -> Settings -> Default Font Asset 选择刚才创建的 Font Asset。

#### 表情乱码

将 SEGUIEMJ ttf 字体文件拖入 Unity 编辑器中，并生成 SDF 文件  
在 MSYH SDF 中设置 Fallback List, 添加 SEGUIEMJ SDF。

#### 下标符号乱码

noto_Sans

字体下载网站：
[https://fonts.google.com/noto/specimen/Noto+Sans?preview.text=H%E2%82%87.%E9%9B%A8%E6%B2%AB#glyphs](https://fonts.google.com/noto/specimen/Noto+Sans?preview.text=H%E2%82%87.%E9%9B%A8%E6%B2%AB#glyphs)

#### 字体图片

使用网站[https://snowb.org/](https://snowb.org/) 将零散图转为图集  
设置图集为 Multiple 并 Sprite Editor 进行拆分  
图集右键新建 sprite asset, 在 sprite Character Table 中可以设置图片 Name (按回车确定)  
使用脚本加载图片文字

```csharp
public string GetImageString(string str)
{
   string[] imageTarget = new string[11]
   {
      "-",
      "0",
      "1",
      "2",
      "3",
      "4",
      "5",
      "6",
      "7",
      "8",
      "9",
   };
   char[] astr = str.ToCharArray();
   string tar = "";
   for (int i = 0; i < astr.Length; i++)
   {
      string idstr = astr[i].ToString();
      if (!imageTarget.Contains(idstr))
      {
            tar += astr[i].ToString();
            continue;
      }
      else
      {
            tar += $"<sprite name=\"{idstr}\">";
      }
   }
   return tar;
}

```
