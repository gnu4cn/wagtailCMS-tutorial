# Wagtail之道

**The Zen of Wagtail**

Wagtail诞生于多年的网站开发经验中，诞生于哪些方法好用、哪些方法不好用的试错中，试图在功能的强大与程序的简洁、结构化与灵活性之间找到平衡（Wagtail has been born out of many years of experience building websites, learning approches that work and ones that don't, and striking a balance between power and simplicity, structure and flexibility）。创造Wagtail的人们，希望大家能发现，Wagtail正是处于这一最佳点位。但是，他作为一套软件，Wagtail也只能做到这里了 -- 此时正是大家可以创建一个优美而又有趣的网站的时候。正由于此，尽管大家都在磨拳擦掌，跃跃欲试，还是要花点时间来了解一下Wagtail是何种设计原则来建立的。

本着“[Python之道](https://www.python.org/dev/peps/pep-0020/)”精神，Wagtail哲学也是一套既包含了使用Wagtail构建网站的，也包含了持续进行的Wagtail自身开发的指导原则。

## Wagtail并非一个开箱即用的网站

是不能通过将一些现成模块拼在一起就得到一个漂亮的站点的 -- 需要写代码。

## 要一直清楚自己所扮演的角色

**Always wear the right hat**

有效地使用Wagtail的关键，是要认识到在创建一个网站时，涉及到多个角色：内容作者、站点管理员、开发者与设计师。当然他们可能是不同的人，但也不必非得要是不同的人 -- 比如在使用Wagtail构建自己的个人博客是，就能发现你自己在这写不同角色之间变换。但无论哪种方式，都要注意在某时某刻你正在做什么，并要使用适当的功能来做那件事。

内容作者或站点管理员，会通过Wagtail的管理界面，来完成他们的工作；开发者或设计师则要花大量时间来编写Python、HTML或CSS代码。这实际上是个好事：Wagtail设计初衷不是要取代编程工作。或许未来的某一天，有人搞出来一种通过托放操作，就能建立出与经由代码编写而建立的同样强大的网站，但Wagtail绝不是那样的工具，Wagtail也不会尝试成为那样的工具。

一种常见的错误做法，就是将过多的权力与责任给予到内容作者与站点管理员 -- 实际上，当他们是你的客户时，这往往就是他们会吼你的原因。站点成功有赖于你敢于对此说不。内容管理的真正威力，不是来自将控制权交给内容管理系统的用户，而是来自设定好不同角色之间的边界。抛开其他的不说，这意味着不要让网站编辑在内容编辑界面去搞设计和布局，不要让站点管理员去构建那些可以通过代码来实现的复杂的交互流程。

## 内容管理系统应尽可能有效地、直接地让网站编辑头脑中的想法顺利表达出来，并存入到数据库

** A CMS should get information out of an editor's head and into a database, as efficiently and directly as possible**

不论网站是关于汽车、猫咪、甜点或房屋中介，内容作者都会带着他们想要发布在网站上的那些特定领域的信息，来到Wagtail的管理界面。因此作为网站建设者的目标，就是要将这些信息以他们原始的格式 -- 而不是依照某位特定作者认为信息应该怎样的想法，释放并存储起来。


