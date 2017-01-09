---
layout: post
title: 用PythonPi实现门禁系统-模式识别
date: 2017-1-9
---
模式识别听起来很高大上，其实就是特征识别。人类对事物的识别过程其实就是提取特征、根据特征对事物进行分类的过程，然后人类就可以将这类事物的特点与规律套用到这个事物上。

我们在门禁课程中曾提出过一个[双人开门](http://course.pythonpi.top:10008/coursePlay.html?course=3&courseware=4&Order=6)的练习：在某些安全性要求较高的门禁控制点，要求两人以上才能开门。但如果用通常的状态机来实现这个功能，我在思考题中也说，会有非常多的问题难以解决。这些问题包括：

- 有权限的人连刷两次怎么办（如何将两个不同的人识别为事件）

- 几十个有权限的人中只要有两个就可以双人开门（几十个人中随便挑两个的数字是一个组合问题，如果有20个人就有190种可能，好不容易做好了，再增加一个人怎么办？）

要完美解决双人开门的问题，就需要使用模式识别。当然我们用到的是最简单的模式：序列。所谓的`序列，就是一个连续的事物组合`。最常用的序列就是密码了：一组连续的特定数字或字母。

[PythonPi平台](http://course.pythonpi.top:10008/coursePlay.html?course=1&courseware=1&Order=1)中的序列也被称为一个窗口，就是一个观察连续输入的数据流的窗口，如果输入数据流中的某一段数据符合窗口规定的特定条件即被被该窗口所识别为合法的输入数据序列。如0101窗口即可从下面的输入流中识别出相应的序列：0110100011`0101`0011。

一个识别特定序列的窗口由一组窗格组成，而每个窗格包括了三个对输入数据进行筛选的要素：

- 输入数据来源：如连接到1号树莓派1号GPIO口的开关量输入源:/pi1/1、连接到2号树莓派下连的单片机上的i2c总线72号地址2号寄存器处的模数转换器：/pi2/nodeMCU_n1/i2c_72_2

- 判别条件：等于、不等于、大于、大于等于、小于、小于等于，甚至是大于5小于9(？>5)And(?<9)这样的筛选条件

- 预设的判别比较值：即上述识别表达式中(？>5)And(?<9)的5、9

窗口同时保存了一个输入数据的队列，每次有数据输入，则将输入数据推入到该队列（队列长度即为窗口大小），然后让每个窗格一一对应的对队列中的各个数据进行检测，如果能够通过所有窗格的检测，即实现了对序列的识别。

序列是对控制逻辑单元中的特定输入进行识别，即我们可以定义待识别的序列为：

    1、/pi1/1的输入为0（即按钮按下）
    2、/pi2/nodeMCU_n1/i2c_72_2的值大于128（即摇杆向右打）

用口语的说法就是：启动并向右搬动摇杆，则如何如何。下面的代码就是如何设置上面的识别序列：

    #导入依赖包
    from cn.ijingxi.corpuscle.python import input
    from cn.ijingxi.corpuscle.python import sequence
    from cn.ijingxi.corpuscle.python import condition
    from cn.ijingxi.corpuscle.python import logic
    #定义启动开关
    inputSwitch=input.getGPIO("/pi1/1",input.PULL_UP,input.Flitter)
    #定义每0.5秒读取模数转换器的一个字节来感知摇杆的x轴位置
    inputDirection=input.getI2C("/pi2/nodeMCU_n1/i2c_72_2",1,500)
    #定义开关输入值的检测条件
    c0=condition.getCompare(condition.Equal,input.LOW)
    #定义摇杆位置输入值的检测条件
    c1=condition.getCompare(condition.Greate,128)
    #定义一个序列，该序列从输入流中匹配成功，则触发start事件
    s=sequence("start")
    #序列的第一个窗格是检测启动开关是否启动
    s.addDetection(inputSwitch,c0)
    #序列的第二个窗格是检测摇杆x轴位置
    s.addDetection(inputDirection,c1)
    #定义控制逻辑
    l=logic("smtest")
    #将序列添加到控制逻辑中
    l.addSequence(s)

  `这段代码在启动并向右搬动摇杆将触发start事件`

由于个体识别设备的引入，就有了基于个体身份的权限管理的问题。而双人开门功能中，就有一个很明确的要求：必须是两个不同的有权限的人在一定时间内先后刷卡才可以开门。这里不要求识别具体的人，而是要求识别不同的人、相同的权限组。所以就有了一种特殊的序列：缓存序列。

缓存序列和普通序列的不同在于：普通序列的条件检测值是预置的、固定的，如上面代码中的128，而缓存序列的条件检测值则是前一个输入值，即比较前后两次的输入是否相同。

目前，PythonPi平台提供的缓存序列在场景中提供了四种预设的检测事件：

- SamePeople：前后是同一个人

- DifferPeople：前后不是同一个人

- SameRole：前后是同一个组有权限的人

- DifferRole：前后不是同一个组有权限的人

`这些事件是识别为有权限通过者才会进行检测`

利用这些预设事件，则双人开门功能的实现，可以通过如下的状态机定义来实现：

    d.addTrans("closed","open","waitOther","第一个有权限的人刷卡",startTimer1)#open事件是有权限才会发出的
    d.addTrans("waitOther","DifferPeople","otherDetect","有另外的人刷卡")
    d.addTrans("otherDetect","SameRole","open","第二个刷卡的人也有权限，开门",openDoor,stopTimer1)
    d.addTrans("otherDetect","DifferRole","closed","第二个刷卡的人没有权限",stopTimer1)#用超时可以省掉这条代码
    d.addTrans("waitOther","timeover","closed","超时")
    d.addTrans("otherDetect","timeover","closed","超时")

====================================================================================================

`关注我的公众号及时获取推送的最新文章`

  ![公众号](http://course.pythonpi.top:10008/images/qrcode.jpg)
  
  
