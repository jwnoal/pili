---
title: 'Unity 文字'
titleColor: '#aaa,#20B2AA'
titleIcon: 'asset:unity'
tags: [ 'Untiy' ]
categories: [ 'Code' ]
description: 'Unity 文字组件的使用'
---

#### 显示中文
1. 生成Font Asset
将MSYH ttf字体文件拖入 Unity 编辑器中，可以放在 Assets/Rseources/Fonts 目录下，右键Create -> TextMeshPro -> Font Asset 即可生成 Font Asset。

2. 在我们初次打开的时候会自动弹出提示框，我们点击Import TMP Essentials即可
点击字体文件SDF，在 Inspector 面板中，设置图集大小，4096x4096，可以勾选Multi Atlas Textures和Clear Dynamic Data, 然后点击Apply

3. 创建文字组件 Text-TextMeshPro
右键 GameObject -> UI -> Text -> TextMeshPro 创建文字组件，然后在 Inspector 面板中设置文字内容和字体样式。将刚才创建的 Font Asset 拖入 Font Asset 字段中。

4. 设置默认中文
项目顶部 Edit -> Project Settings -> TextMeshPro -> Settings -> Default Font Asset 选择刚才创建的 Font Asset。

#### 表情乱码
将SEGUIEMJ ttf字体文件拖入 Unity 编辑器中，并生成SDF文件  
在MSYH SDF中设置Fallback List, 添加SEGUIEMJ SDF