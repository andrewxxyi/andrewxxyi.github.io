---
layout: post
title: 用PythonPi实现门禁系统-场景
date: 2017-1-3
---
在[分布式智能控制系统](http://115.29.52.95/forum.php?mod=forumdisplay&fid=40)中，我们介绍了控制逻辑组件。该组件可以完成智能控制的功能。但控制逻辑组件存在一个问题：它不具备基于个体识别然后据此进行管控的能力，控制逻辑组件并不关心参与者是谁，它对所有人都是一视同仁的，因此控制逻辑组件是无法用来进行门禁管控的。

为了实现门禁控制，我们将具有个体识别能力的识别点、基于角色的权限管理和一个事件驱动的控制逻辑组件组装到一起就成为一个场景。和控制逻辑相比，场景具有针对特别的人来执行不同功能的能力。它包括：

- 个体识别，根据卡片或生物特征对场景中的参与者进行个体识别

- 权限管理，根据识别出的个体进行基于角色的权限裁决

- 控制逻辑，实现具体的控制功能

由于控制逻辑是基于状态机的事件驱动，因此将权限裁决可视为出现特定事件，这样控制逻辑不需做任何修改即可引用。

考虑到一个应用场景中可能会存在多个识别点（如进出都需要刷卡的门禁），所以个体权限的裁决是和识别点的位置相关的，比如要求单向行进的参观路径上，游客是不允许后向出门的，但工作人员则可以，所以游客角色在反向（门后，相当于行进方向）的识别点是没有权限开门的，在正向（门前）的识别点是有权限开门的；而工作人员角色则不管正反向都有开门的权限。

考虑到传统门禁系统人员和角色只能单一映射会限制智能控制系统随动业务系统的管控能力，所以允许一个人可以映射为多个角色，但由于控制系统的特点，只要在这些角色中有一个在识别点得到了授权许可即视为有通过权限并终止后继的检查。

在上述的实现中意味着：`默认没有授权`，只有显式的将某角色在某识别点的出现绑定为某特殊事件才是一个特定的授权。但这个所谓的授权也并不就一定意味着开门动作。比如，在很多的资金交接区域，开门是一个有很多约束前提的动作，如需双门互锁（一个封闭的资金交接区域有两道门，各自有门禁管控，两门只能同时打开一道）、远程确认（刷卡得到授权后还需保安通过视频进行确认并在安保中心按下同意按钮才能开门，单一刷卡或远程按下按钮都不能开门）等等。

从双门互锁、远程确认这些功能我们可以看到状态机相比产生式在控制领域的优势：

- 状态机的输入源只是发出一个事件，无论要实现的功能有多复杂，它就是简单的发出一个事件通知

- 复杂些的功能都是一系列连贯的动作，对状态机来说这是天然的，而产生式是平行的条件检测，哪个满足执行哪个，执行路径难以控制

- 我们想象的智能系统要具备随动环境变化而变化，这就可能经常需要对各种复杂功能进行调整、升级与新增。状态机可以通过图上作业，甚而我们在有了足够的积累之后，可以开发图形化的辅助设计工具来帮助分析、自动推导来降低设计难度。但产生式的功能是分布在非常多的规则中的，这些规则的彼此平行，需要非常强的专家知识来提供帮助

====================================================================================================

`关注我的公众号及时获取推送的最新文章`

  ![公众号](http://course.pythonpi.top:10008/images/qrcode.jpg)


