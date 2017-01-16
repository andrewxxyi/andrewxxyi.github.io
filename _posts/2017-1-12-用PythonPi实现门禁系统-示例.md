---
layout: post
title: 用PythonPi实现门禁系统-示例
date: 2017-1-12
---
和[分布式智能控制系统](http://115.29.52.95/forum.php?mod=forumdisplay&fid=40)类似，首先实现了Python接口的API，基于图形界面的管理界面以后视情况提供。

先看示例的接线图：

![rc522接线图](http://course.pythonpi.top:10008/images/rc522.png)

这个示例是用树莓派的spi接口连接了一个rc522读卡器，以15号gpio口连接了一个led作为电锁动作的指示，16号gpio口连接了一个开关按钮作为出门按钮，1号gpio口连接了一个开关按钮模拟门状态。实现代码如下：

    from cn.ijingxi.corpuscle.python import active
    from cn.ijingxi.corpuscle.python import input
    from cn.ijingxi.corpuscle.python import door
    from cn.ijingxi.corpuscle.python import admin

    #添加卡，卡号：-741925061
    admin.addPeople(admin.PeopleType_RC522,-741925061,"people1")
    #为该卡指定角色
    admin.addRoleMap(admin.PeopleType_RC522,-741925061,"role1")
    
    #定义一个门
    d=door("testDoor")
    #添加锁动作，开门与关门，15号gpio口
    a1=active.getGPIO("/pi1/15",active.HIGH)
    a2=active.getGPIO("/pi1/15",active.LOW)
    d.addLockOpen(a1)
    d.addLockClose(a2)
    #添加出门按钮，16号gpio口，带上拉电阻，提供防抖动功能
    es=input.getGPIO("/pi1/16",input.PULL_UP,input.Flitter)
    d.addExitSwitch(es)
    #添加锁状态，1号gpio口，带上拉电阻，提供防抖动功能
    ls=input.getGPIO("/pi1/1",input.PULL_UP,input.Flitter)
    d.addLockState(ls)
    #添加读卡器
    r=input.getRC522("/pi1/rc522_r1")
    d.addRecognizer(r)
    #许可哪个角色可通过
    d.addRole("role1",r)

    #设置完毕，开始初始化
    d.init()

请参考：[API说明](http://course.pythonpi.top:10008/sectionList.html?course=2&courseware=2)

将上述代码在“在线编程”页面中的代码输入框中输入，点击执行即实现了一个普通的单门门禁系统：

- 刷卡（卡号：-741925061），led亮，代表电锁解除锁门，5秒后led灭，即开门后延时5秒启动电锁锁门（只是下达了锁门的动作，如果此时门还没有开着则电锁不能锁上）

- 按下16号gpio口所连的出门按钮，led亮，代表电锁解除锁门，5秒后led灭，即开门后延时5秒启动电锁锁门

- 门开（led亮）后，如果不按下1号口所连接的开关按钮（代表门关闭了），led熄灭后再不会亮起。因为不关门，代表门一直开着，所以不会再次开门

- 刷卡或按下出门按钮后，不等led自动熄灭就按下1号口所连接的开关按钮，此时led立刻熄灭。即开门后只推了下门却没外出，所以立刻关门

- 刷其它卡无反应

这几行代码就能实现这么多功能？！这是因为door这个组件封装了场景组件并预设了上述功能，下次我们再来详细讲解其是如何实现的

====================================================================================================

`关注我的公众号及时获取推送的最新文章`

  ![公众号](http://course.pythonpi.top:10008/images/qrcode.jpg)
  

