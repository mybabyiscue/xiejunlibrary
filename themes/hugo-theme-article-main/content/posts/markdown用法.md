---
title: 静态站点中markdown用法
date: 2025-06-05T10:10:00+0800
description: "本文详细介绍该静态站点markdown常用的语法"
tags: [使用教程]
---

# 1. 说明
markdown文件的开头必须按照以下格式补全对应的内容：
```yaml
---
title: 静态站点中markdown用法
date: 2025-06-05T10:10:00+0800
description: "本文详细介绍该静态站点markdown常用的语法"
tags: [使用教程]
---
```

# 2. 语法
支持markdown的所有语法，但不支持mermaid。

需要特别说明`图片的插入`方法。图片必须放入`static`目录下，然后通过`![图片描述](/images/图片)`的方式插入图片。当然也可以使用url方式插入网上的图片。如果图片存放在github上，建议使用`![图片描述](https://raw.githubusercontent.com/用户名/仓库名/分支名/图片路径)`的方式插入图片。