title: 从零开始Android Jenkins持续集成
date: 2018-03-15 20:02:24
---
# 从零开始用Jenkins做Android开发CI系统

[TOC]

## 1. 什么是持续集成(CI)

说到持续集成(Continuous Integration)，就不得不提到相关的持续交付(Continuous Delivery)和持续部署(Continuous Deployment)。

要区分和理解这三者的关系，简单的概括成一下三点：

* 持续集成（Continuous Integration）指的是，频繁地（一天多次）将代码集成到主干。 
  持续集成的目的，就是让产品可以快速迭代，同时还能保持高质量。  
  它的核心措施是，代码集成到主干之前，必须通过自动化测试。只要有一个测试用例失败，就不能集成。

* 持续交付（Continuous delivery）指的是，频繁地将软件的新版本，交付给质量团队或者用户，以供评审。如果评审通过，代码就进入生产阶段。
  持续交付可以看作持续集成的下一步。它强调的是，不管怎么更新，软件是随时随地可以交付的。

* 持续部署（continuous deployment）是持续交付的下一步，指的是代码通过评审以后，自动部署到生产环境。

  持续部署的目标是，代码在任何时刻都是可部署的，可以进入生产阶段。

  持续部署的前提是能自动化完成测试、构建、部署等步骤。

他们在整个开发过程中的相互关系如下图：

![](http://ww2.sinaimg.cn/large/a15b4afegy1fm775l3tybj20m80d90ti)

## 2. 什么是Jenkins

用[Jenkins](https://jenkins.io/)官网的一句话来概括就是:

> Jenkins is a self-contained, open source automation server which can be used to automate all sorts of tasks related to building, testing, and delivering or deploying software.
>
> Jenkins can be installed through native system packages, Docker, or even run standalone by any machine with a Java Runtime Environment (JRE) installed.

Jenkins是一个独立开源的自动化服务，可用于自动执行各种任务包括：构建、测试、交付或部署软件。

Jenkins可以通过系统软件包或Docker来安装，甚至可以通过任何安装了Java运行时环境（JRE）的计算机独立运行。

## 3. 安装Jenkins

以下各种操作都默认服务器环境为Centos7，代码仓库使用社区开源版GitLab

###　3.1 使用war包安装

很多人都是直接从jenkins官网下载下来war包，用如下java命令行来运行：

```bash
java -jar jenkins.war --httpPort=8080
```

这样是很方便，但是不便于后期管理，所以我们要利用好系统自身的包管理系统来安装jenkins。这样做有以下几点好处：

1. 可以使用系统本身的Systemd服务来管理jenkins服务状态
   想要重启jenkins服务只需以下命令：

   ```bash
   sudo systemctl restart jenkins
   ```

   想要查看jenkins当前的运行状态只需以下命令：

   ```bash
   sudo systemctl status jenkins
   ```

   得到的Jenkins运行信息

   ![](http://ww2.sinaimg.cn/large/a15b4afegy1fm78cm6uvxj20oi0diaas)

2. 可以使用系统自身的包管理系统来安装、卸载、升级等等
   例如想要升级Jenkins的话一行命令即可：`sudo yum update jenkins`

### 3. 2 使用os自身的包管理器安装

由于Jenkins不在centos的默认软件仓库中，所以我们要先将jenkins 源引入：

```bash
sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
```

再导入Jenkins源对应的密钥：

```bash
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io.key
```

最后就可以用yum包管理器直接安装了：

```bash
sudo yum install jenkins
```

## 4 配置Jenkins

### 4.1 jenkins初始化配置

不管你是用war包安装，还是用包管理器安装，这时候你只要使用你服务器的ip或这DNS解析的域名加上默认端口8080，就可以看到如下页面了：

如：http://www.example.com:8080

![](http://ww2.sinaimg.cn/large/a15b4afely1fm7a26u7jhj20ri0ivtaf)

这时候再到服务器上运行一下命令`cat /var/lib/jenkins/secrets/initialAdminPassword`拿到密钥填入passwd处顶点击continue，进入插件选择页

![](http://ww2.sinaimg.cn/large/a15b4afegy1fm7a84yy2pj20qe0enaco)

这里我们直接选择默认配置，以后有需求再慢慢安装其他插件。

![](http://ww2.sinaimg.cn/large/a15b4afegy1fm7af82dd4j20rm0fbq54)

等走完上面的安装进度条我们就进到管理员账号设置页，设好管理员账号后我们就进入jenkins 的首页了。

![](http://ww2.sinaimg.cn/large/a15b4afegy1fm7algkh6jj20rw0cfjsd)

### 4.2 Jenkins编译环境配置

首先我们到jenkins的全局工具配置页 Manage Jenkins->Global Tool Configuration，如下图所示配好JDK和Gradle(注意版本要与Android项目用到的一样)，jenkins中使用的JDK强烈建议使用Oracle的JDK而不是OpenJDK，因为jenkins和OpenJDK有很多兼容性问题。

![](http://ww2.sinaimg.cn/large/a15b4afely1fm7au9onrcj216e0f23z5)

接着到jenkins的系统配置页 Manage Jenkins->Configure System，增加如下ANDROID_HOME的环境变量

![](http://ww2.sinaimg.cn/large/a15b4afegy1fm7b2f8garj216707it8t)

### 4.3 Jenkins代码仓库证书配置

这里我们使用的是开源的社区版GitLab仓库，而仔Jenkins的插件库中也有专门对接Gitlab的插件。

首先我们到jenkins的插件管理页 Manage Jenkins->Plugin Manager，点击Available标签，搜索gitlab，并选中Gitlab Hook Plugin、 Gitlab Plugin 两个插件，点击安装。

![](http://ww2.sinaimg.cn/large/a15b4afegy1fm80b9hdetj213b09vt99)

安装完毕，我们再到证书管理页，Credentials->System->Global credentials，如图：

![](http://ww2.sinaimg.cn/large/a15b4afegy1fm80ezjg81j216u0azjsi)

再点击添加证书(Add Credentials):

![](http://ww2.sinaimg.cn/large/a15b4afegy1fm80g4epgvj21h509cq40)

证书类型选择Gitlab API token，范围直接默认全局，再填入从gitlab中获取的API token，保存。

![](http://ww2.sinaimg.cn/large/a15b4afegy1fm80l8ujpgj21h40asq3k)

接着我们再按照以上方式添加一个pull代码用的SSH key。

最后我们再到jenkins的系统配置页 Manage Jenkins->Configure System，会发现多了一个Gitlab的栏目，填入相应的配置，Gitlab host URL填上Gitlab的url，如果没有绑定域名直接填上ip加端口号也可以，我这是是用的现成gitlab.com，证书选择我们刚刚设置的Gitlab API Token证书。

![](http://ww2.sinaimg.cn/large/a15b4afegy1fm80wskag9j216e0bgaaf)

###　 4.4 Jenkins项目配置

接下来我们回到jenkins主页，新建一个Freestyle project 的项目，在GitLab connection处选择我们刚刚配置好的Gitlab API token，被我模糊的地方是后面要讲到的Git Parameter Plug-In插件的配置，这里先抹掉免得弄混淆了。

![](http://ww2.sinaimg.cn/large/a15b4afegy1fm817bspsrj21430lw761)

再配置号代码仓库的url 和 ssh key证书，在构建分支指定栏中填入`origin/${gitlabSourceBranch}`

![](http://ww2.sinaimg.cn/large/a15b4afegy1fm81cdrvpfj21410jvabl)

在构建触发器中勾选Gitlab选项，并选择想要触发jenkins自动构建的条件，这里我选择了push事件和Accepted Merge Request事件。同时将此处的Gitlab CI Service URL 填入GitLab相应的Hook链接处。

![](http://ww2.sinaimg.cn/large/a15b4afegy1fm81ik2cz0j21400hc3zl)

在构建选项中选择调用Gradle脚本(Invoke Gradle script)，并选择项目对应的Gradle版本号，填写Gradle构建的Task任务。

![](http://ww2.sinaimg.cn/large/a15b4afegy1fm81u5wy6xj20n30ar74l)

最后在构建后动作(Post-build Actions)中选择`Archive the artifacts` 和 `Publish build status to GitLab commit`两项，归档文件中填入构建成功后Apk所在的位置，路径可以使用通配符。

![](http://ww2.sinaimg.cn/large/a15b4afegy1fm81rq6kunj21440dn0tq)

![](http://ww2.sinaimg.cn/large/a15b4afely1fm81lfalfkj213z0kjq3w)

其中第二项选选中后可以在Gitlab中看到每次触发构建的结果。如下图

![](http://ww2.sinaimg.cn/large/a15b4afegy1fm822tsrcyj20zh0dljt7)

最后点击保存，我们就可以在首页看到自己的项目了。

这时候在Android Studio上做出修改并push到Gitlab，Gitlab就会通过Web Hook触发Jenkins进行自动构建，当Jenkins构建完成后又会调用Gitlab 的API反馈某一次构建的结果。如果构建成功，Jenkins还会自动把生成的Apk文件提取出来放到项目首页。

![](http://ww2.sinaimg.cn/large/a15b4afegy1fm82girj38j21gz0lj0va)

## 5. 几个好用的插件

### 5.1 Git Parameter 

当我们按照上面的方法配置完成后可以在项目的首页看到一个Build Now按钮，这时候点击是不会构建成功的，因为我们在指定构建分支时填了 需要Gitlab传递的分支参数 的路径`origin/${gitlabSourceBranch}`， 而此时再到项目首页点击Build Now按钮时，由于不是来自Gitlab的调用，无法把分支参数实例化，自然也就无法构建了。

此时我们只要在Branches to build一栏下再增加一个默认的`*/master`分支，再点击Build Now，这时候Jenkins就会进行正常的构建了。

![](http://ww2.sinaimg.cn/large/a15b4afegy1fm8613gajrj214r03awef)

可是如果我不想build master分支呢？我们平时开发肯定一般都会在dev分支或test分支上不断迭代开发，我们经常发版的分支肯定不会是master分支就是了。我们要在不同的分支上切换并进行构建的时候总不能每次都到配置里修改构建分支吧。

所以就有了Git Parameter Plug-In这个插件，依然按照上面的方法在插件库中安装好插件。这时候再进到项目配置页，勾选参数化并添加Git Parameter，就会出现Git参数话配置。

![](http://ww2.sinaimg.cn/large/a15b4afegy1fm881uo0hkj214908zq3j)

填写参数名，并在Parameter Type中选择Branch。

![](http://ww2.sinaimg.cn/large/a15b4afegy1fm86a6au6sj21430ls75e)

然后在源码管理配置中增加构建分支参数`origin/$Branch` 。注意，此处的参数名应该和上面的Git Parameter Name一致。

![](http://ww2.sinaimg.cn/large/a15b4afegy1fm885rlo1sj21490cat9i)

保存，并回到项目首页，我们可以看到之前的Build Now按钮已经变成了Build with Parameters。

![](http://ww2.sinaimg.cn/large/a15b4afegy1fm887r0mlpj21gx0ef40g)

点击该按钮，就会进到分支选择页面，选中某一个分支，点击Build，即可开始对特定分支进行构建了。

![](http://ww2.sinaimg.cn/large/a15b4afegy1fm888wfjndj21fu0co0tt)

### 5.2 Commit Message Trigger

我们现在已经解决了push代码的时候自动构建项目和手动选择分支构建的问题，可是我们有时候向仓库push代码的时候不想让他自动构建怎么办呢？比如说，当一个完整的功能中写完了比较独立的公共部分，此时我们提交代码只是为了和协同开发的同事同步代码而已，并不需要触发构建。

那有什么办法让jenkins知道我们是否真的想构建项目呢，这时候这个插件就能很好的解决这个问题了。

依旧按上面的方法在插件库中找到Commit Message Trigger插件并安装。

再次进入 项目配置，在Build Environment配置项中我们可以看到多了Enable Commit Message Trigger复选框。选中该项并填入自定义的KeyWord，如build(不区分大小写)。

以后我们只要在commit message中包含`[ci keyword]`,Jenkins就能自动识别并构建项目了。

![](http://ww2.sinaimg.cn/large/a15b4afegy1fm88mygqvnj21410bt3z7)

此时我们到Android Studio 中做出修改不在commit message中填写任何内容并push到仓库，发现Jenkins触发了构建，但是却很快就结束了，且状态显示为灰色的Not Build状态。

![](http://ww2.sinaimg.cn/large/a15b4afely1fm88udbjmnj209l0buaao)

我们打开这次构建的Console Output来瞧瞧，原来Commit Message Trigger插件没有在commit message中检测到`[ci build]`字符串，而自动跳过了构建步骤。

![](http://ww2.sinaimg.cn/large/a15b4afegy1fm88t8fhkzj21b40fgwgu)

这时候我们再次修改项目，并在commit message 中添加`[ci build]`标记，push到仓库，就能看到Jenkins成功的构建了。