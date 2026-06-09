+++
title = '更换开机动画LOGO'
date = '2026-06-08T16:35:00+08:00'
draft = false
+++

开机LOGO

```
Z:\android\aml\s905x5\aml-s905x5-androidu-v2\device\amlogic\ross\logo_img_files
OR
Z:\android\aml\s905x5\aml-s905x5-androidu-v2\device\amlogic\common\logo_img_files
```

取决于TARGET_AMLOGIC_RES_PACKAGE的值

```
Z:\android\aml\s905x5\aml-s905x5-androidu-v2\device\amlogic\planck\BoardConfig.mk
```



开机动画路径定义以及查找优先级相关都在这个文件内

```
Z:\android\aml\s905x5\aml-s905x5-androidu-v2\frameworks\base\cmds\bootanimation\BootAnimation.cpp
```

