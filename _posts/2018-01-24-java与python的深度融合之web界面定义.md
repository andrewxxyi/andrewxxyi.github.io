---
layout: post
title: java与python的深度融合之web界面定义
date: 2018-01-24
---
最近对在PythonPi平台上将对java与python进行深度融合非常感兴趣的原因在于10年前的一个认识：在大环境和IT技术的发展变化都异常迅猛的这个时代，信息系统应当具备**快速的低成本定制**的能力才能快速随动这种变化、充分发挥出管理、协作、支撑方面的作用。

信息系统本质上还是一个形式自动机，所以我认为**快速低成本定制**的关键是在能否将信息系统简单的定义出来。也就是说一个信息系统主要的行为应该是静态的、简易的描述出来的，当然也应该是可以简单修改、验证的。最好是能由业务人员甚至是客户完成这个主要的定义，只有少量的动态复杂逻辑由程序员编程处理。

这个定义的内容应该至少包括：

- 界面

- 数据

- 流程

- 权限

基于这个认识，当年就做了web界面的定义之类的，甚至为了支持自定义，还自己写了一个业务语言的编辑编译器，但总的来说，还是很麻烦、太过专业化。

这次由于为了简化PythonPi上的数据查询操作，无意间完成了数据查询的定义工作，简直是欣喜若狂，终于能够再次尝试之前的想法了。

目前在界面方面实现了：

- text：文本，显示指定的文本，如果绑定了数据项，则当在脚本中修改数据时，自动刷新到前端，可作为tag设置标记颜色

    #加了警告标记的普通大小的文本
    web test_tip_1 type text top=100,left=200,text='名字',tag=warning
    web test_tip_2 type text top=200,left=200,text='123 456 789'
    #H2的文本
    web test_tip_3 type text top=300,left=200,size=2,text="1 2 3 4 5 6 7 8 9 "
    #H3的文本
    web test_tip_4 type text top=400,left=200,size=3,text=987654321
    #绑定了数据项的文本，当后台python脚本修改该数据项时，自动刷新
    web data_test1 bind test1 type text top=100,left=300,text='test1的显示内容'

- checkbox：选择框

    #disp是显示格式：5-yes/no，3-on/off
    web test_info_2 bind testid type checkbox top=200,left=300,checked=true,disp=5

- input：文本输入，绑定了数据项后，自动刷新到脚本中的数据缓存，并通知感兴趣的脚本函数数据已更新

    #定义了一个输入框，如果有输入会自动将输入提交到后台，校验功能也有，但是是在后台校验:(
    web data_testid bind testid type input top=500,left=200,placeholder="请输入一个数字"

显示效果：

    ![显示效果](http://course.pythonpi.top:10008/images/webControlText.png)

- button：按钮，点击即可自动执行脚本中的相应函数并完成参数的动态赋值

    #点击执行t1函数，由于t1声明了调用参数，所以会自动将参数进行装定后再调用
    web test_op_1 type button top=100,left=500,motion=t1,text='业务场景操作1'

    #t1函数
    @jx.define
    def t1(str):
        """
         /* 定义调用本函数应当送入的参数 */
        param testid
        """
        #对表的操作：第一行
        #由于后面的表定义中将本列定义了标记，所以会按所定义标记为绿色（success）
        biz.table1.row(0).test1="你是谁"
        biz.table1.row(0).testid=123
        #第二行
        biz.table1.row(1).test1="爱学习"
        #显式的将本列标记为橙色（warning）
        biz.table1.row(1).test1.tag="warning"
        biz.table1.row(1).testid=789
        #然后弹出了一个对话框显示用户在上面绑定了testid的输入框中的输入
        jx.info("输入变化","你输入了:{}",biz.testid)

- table：表格，根据指定的类sql语句从后台自动加载表数据，双击某表格可输入数据，同样实现了自动刷新自动提交

    #定义了一个表格，包括两列（数据的装定是在t1函数中）
    web table1 type table top=200,left=500,title="人员表"
    with table1 col test1 head '人名' width=100,tag=success
    with table1 col testid head '员工ID' width=200

显示效果(表中的数据已被刷新了，注意人名一列的标记)：

    ![显示效果](http://course.pythonpi.top:10008/images/webControlButton.png)

- dtpicker：日期时间选择器

    web test_dtpicker type dtpicker top=450,left=1000,width=200,time=false

- info：信息提示框，可折叠可移除

    web test_info_1 type info top=100,left=1000,collapse=true,close=true,title='提示板',text='定义web端和数据关联的控件，名字要和alias中的一致，否则无法自动关联'

- tree：树型列表，可折叠展开，自动加载后台数据

    #定义了一个树型菜单，会执行testquery函数以请求相应的菜单项数据
    web test_tree_1 type tree top=250,left=1000,width=400,motion=testquery,title='数型菜单'
    #对应的python函数：定义了两个folder型的菜单项，点击后会调用tree1函数并分别送入各种的treeid以请求加载各自的子菜单
    def testquery():
        tree=biz.getTree()
        tree.addSub("treeItem1","你想咋的？",tree.folder)
        tree.setSubParam("treeItem1","motion","tree1")
        tree.setSubParam("treeItem1","treeid",11)
        tree.addSub("treeItem2","我想揍你！",tree.folder)
        tree.setSubParam("treeItem2","motion","tree1")
        tree.setSubParam("treeItem2","treeid",12)
        return tree.getData()
    #动态加载子菜单项的函数
    @jx.define
    def tree1(tid):
        """
         /* 定义调用本函数应当送入的参数 */
        param treeid
        """
        if tid==11:
            tree=biz.getTree()
            tree.addSub("t1","xxxxxxx",tree.item)
            #tree.setSubParam("treeItem1","res","dbtest")
            tree.setSubParam("t1","motion","tree12")
            tree.addSub("t2","yyyyyyy",tree.item)
            tree.setSubParam("t2","motion","tree12")
            return tree.getData()
        else:
            tree=biz.getTree()
            tree.addSub("t1","xxqqrqerewf",tree.item)
            tree.setSubParam("t1","motion","tree22")
            tree.addSub("t2","r4efafasf",tree.item)
            tree.setSubParam("t2","motion","tree22")
            return tree.getData()

显示效果：

    ![显示效果](http://course.pythonpi.top:10008/images/webControlTree.png)

而在增加这些控件之时，其扩展是平台无关的，也就是说，在平台的相关支撑工作做完后，所有的控件定义是在javascript脚本中实现的，使用则是在python脚本中，所以在我增加这些界面控件时，后台服务一直运行着，两边的脚本写好了，随时刷一下页面就可以看到效果而不需要重启服务即实现了新增控件的扩展:)

====================================================================================================

`关注我的公众号及时获取推送的最新文章`

  ![公众号](http://course.pythonpi.top:10008/images/qrcode.jpg)
