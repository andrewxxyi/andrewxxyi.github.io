---
layout: post
title: java与python的深度融合之数据查询
date: 2018-01-17
---
最近迷上了在PythonPi平台上将java和python进行深度融合的尝试，首先就是数据查询的操作。PythonPi由于jxpi提供了ORM功能，已经能在java中很简单的就实现了数据操作，甚至不管数据对象是否存在各种继承关系，都是一样的操作。但在python中进行操作时，开始就是把相应的ORM接口开放出来而已，没能实现更为简便的操作。后来，翻了半天python的相关特性，看到两个好东东，一个是__doc__，一个是@修饰符，哈，感觉一下就高大上起来啦:)

先看下最终的实现效果：

    from cn.ijingxi.intelControl.python import jxpython

    @jx.defSQL
    def test():
        """
        from ObjTag,People
        select People.Name as peopleName type string,ObjTag.TagID as tag type int
        where ObjTag.ID == People.ID and People.Name as pname != ?
        """
        pass

这段代码定义了一个名为test的空函数，它唯一的功能就是在注释中，定义了一个类sql语句，然后用@修饰了下，脚本加载后即会自动将这个类sql语句编译为jxpi中的查询操作对象。该查询操作对象接收一个名为pname的参数，执行后返回列名分别为peopleName和tag的数据集合，如果指定了limit和offset还可以自动实现翻页功能。

可这个该怎么用呢？思来想去，我干脆将它和jxpi的REST功能进行了结合，这就实现了一个通用的数据查询接口，前端定义查询参数，查询语句在后端的python脚本中进行定义、动态加载，使用既简单又和平台相互隔离，而且功能的增加与修改也不需要重启服务啦，甚至还自带访问权限控制能力，这块以后再详细说。

实现的java代码，其中的注释换成了#开头：

    #接口访问策略：只要是正常的登录用户都可访问
    @ActiveRight(policy = ActiveRight.Policy.NormalUser)
    #为该函数提供REST功能，访问点的url是：/jxc/query
    @REST
    public static jxHttpData query(Map<String, Object> ps, jxJson Param) throws Exception {
        #从session中取得当前用户
        String sid = (String) ps.get("SessionID");
        UUID pid = jxSession.getPeopleID(sid);
        #由于是NormalUser许可，所以这两个404是画蛇添足，这是为了防止哪天没想开将访问策略修改为不需登录也可访问
        if (pid == null)
            return new jxHttpData(404, "用户未找到");
        People p = (People) People.getByID(People.class, pid);
        if (p == null)
            return new jxHttpData(404, "用户未找到");

        SelectSql sql = null;
        jxSession s = jxSession.search(sid);
        #res是脚本名
        String res = Param.getSubValueString("res");
        #安装脚本
        scene ms = installToSession(s, p, res);

        String active = Param.getSubValueString("filter");
        if (active != null) {
            #用户自定义的查询筛选器
            sql = getSQL(p, active);
            utils.checkAssert(sql != null, "{}未定义筛选器：{}", p.Name, active);
        } else {
            #执行一个预定义好的查询语句
            active = Param.getSubValueString("active");
            utils.checkAssert(active != null, "{} 未指定明确的查询入口", res);

            #权限检查。jxpi的权限检查分为操作权限和读权限，这里检查的是操作权限，在执行每个python函数时都会自动进行权限检查
            #但由于这里的pyghon函数只相当于是给类sql语句起了个名字，并不会被真正执行，所以类sql语句的查询需要显式进行权限检查
            boolean accept = authorize(py, p, active);
            if (!accept)
                return new jxHttpData(403, "{}没有权限执行：{}-{}", p.Name, res, active);

            sql = getSQL(ms, active);
            utils.checkAssert(sql != null, "{}的{}未正确定义查询语句", res, active);
        }

        jxHttpData rs = new jxHttpData(200, "OK");
        for (jxJson sub : Param)
            setParam(sql, sub.getName(), sub.getValue());

        #执行查询，结果是名为result的json数组。根据各数据项的显示格式自动进行格式化，如果用户没有权限读取某数据项则显示为：没有访问权限
        jxJson json = sceneUtils.query(ms, sql);
        rs.addJson(json);

        return rs;
    }

这样一来，最频繁的数据查询操作就可以简化为：

- 确定需要访问的数据以及查询参数

- 定义一个类sql语句，写到后端相应的python脚本中的某个函数的注释中

- 前端在javaScript脚本中写rest访问函数

    #添加session信息
    var url=getUrlWithQueryString("/jxc/query");
    $.jxREST(url,{res:"脚本名",active:"函数名",limit:20,pname:"testName"},callbackfunc);
    #然后就可以在callbackfunc中获得名为result的数组进行使用了

====================================================================================================

`关注我的公众号及时获取推送的最新文章`

  ![公众号](http://course.pythonpi.top:10008/images/qrcode.jpg)
