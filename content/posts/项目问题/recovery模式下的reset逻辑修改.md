+++
title = 'recovery模式下的reset逻辑修改'
date = '2026-06-08T16:32:00+08:00'
draft = false
+++

对应到目录

```
Z:\android\aml\s905x5\aml-s905x5-androidu-v2\bootable\recovery\recovery_ui\ui.cpp
```

对应代码

这里在对返回的按键做判定，判定之后根据需求返回信号

```
RecoveryUI::KeyAction RecoveryUI::CheckKey(int key, bool is_long_press) 
{
    ...
    //这是目前reset键的判定，这个if判定成功说明按下了reset，返回了TOGGLE
    if ((key == KEY_VOLUMEUP || key == KEY_UP) && IsKeyPressed(KEY_POWER)) 
    {
      return TOGGLE;
    }
    ...
}
```

在这里对返回的信号进行处理

> ​	原有的逻辑是现实/关闭UI文字，现在修改为移动索引，由于之前索引的移动是按下KEY_DOWN，所以可以直接复用逻辑，直接发送KEY_DOWN即可。另外还需要实现一个长按3s确认的功能。所以在接收到这个信号的时候进行一次判定如果是长按到来的信号，那么就走KEY_ENTER逻辑，否则就走KEY_DOWN逻辑

```
void RecoveryUI::ProcessKey(int key_code, int updown) 
{
    ...
    case RecoveryUI::TOGGLE:
        ShowText(!IsTextVisible());
        break;
    ...
}
```

```
void RecoveryUI::ProcessKey(int key_code, int updown) 
{
    ...
    case RecoveryUI::TOGGLE:
        ShowText(true);
        if (long_press) 
        {
            EnqueueKey(KEY_ENTER);  // 长按确认
        } 
    	else
        {
            EnqueueKey(KEY_DOWN);   // 短按下移
        }
        break;
    ...
}
```

另外对应到长按的逻辑在

```
void RecoveryUI::TimeKey(int key_code, int count) 
{
  ...
  std::this_thread::sleep_for(750ms);  // 750 ms == "long"
  ...
}
```

