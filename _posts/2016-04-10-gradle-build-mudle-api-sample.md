---
layout: post_layout
title: Gradle 编程模型及 API 实例详解
time: 2016年04月05日
location: 北京
pulished: true
excerpt_separator: "~~~"
---
最近看到一篇关于Gradle讲解的文章，谁写的，邓凡平老师的杰作，在下只有膜拜的份，讲解很好，转一下供你我学习学习，望邓老师对不打一声招呼就转载见谅。

https://docs.gradle.org/current/dsl/ <==这个文档很重要!

Gradle 基于 Groovy，Groovy 又基于 Java。所以，Gradle 执行的时候和 Groovy 一样，会把脚本转换成 Java 对象。Gradle 主要有三种对象，这三种对象和三种不同的脚本文件对应，在 gradle 执行的时候，会将脚本转换成对应的端：

1.Gradle 对象：当我们执行 gradle xxx 或者什么的时候，gradle 会从默认的配置脚本中构造出一个 Gradle 对象。在整个执行过程中，只有这么一个对象。Gradle 对象的数据类型就是 Gradle。我们一般很少去定制这个默认的配置脚本。~~~

2.Project 对象：每一个 build.gradle 会转换成一个 Project 对象。

3.Settings 对象：显然，每一个 settings.gradle 都会转换成一个 Settings 对象。

注意，对于其他 gradle 文件，除非定义了 class，否则会转换成一个实现了 Script 接口的对象。这一点和 3.5 节中 Groovy 的脚本类相似.

当我们执行 gradle 的时候，gradle 首先是按顺序解析各个 gradle 文件。这里边就有所所谓的生命周期的问题，即先解析谁，后解析谁。图 27 是 Gradle 文档中对生命周期的介绍：结合上一节的内容，相信大家都能看明白了。现在只需要看红框里的内容：

![enter description here][1]

## Gradle 对象

我们先来看 Gradle 对象，它有哪些属性呢？如图 28 所示：

![enter description here][2]

我在 posdevice build.gradle 中和 settings.gradle 中分别加了如下输出：

   	//在 settings.gradle 中，则输出"In settings,gradle id is"
    println "In posdevice, gradle id is " + gradle.hashCode() 
    println "Home Dir:" + gradle.gradleHomeDir
    println "User Home Dir:" + gradle.gradleUserHomeDir
    println "Parent: " + gradle.parent

得到结果如图 29 所示：

![enter description here][3]

1.你看，在 settings.gradle 和 posdevice build.gradle 中，我们得到的 gradle 实例对象的 hashCode 是一样的（都是 791279786）。

2.HomeDir 是我在哪个目录存储的 gradle 可执行程序。

3.User Home Dir：是 gradle 自己设置的目录，里边存储了一些配置文件，以及编译过程中的缓存文件，生成的类文件，编译中依赖的插件等等。

## Project 对象

每一个 build.gradle 文件都会转换成一个 Project 对象。在 Gradle 术语中，Project 对象对应的是 Build Script。
Project 包含若干 Tasks。另外，由于 Project 对应具体的工程，所以需要为 Project 加载所需要的插件，比如为 Java 工程加载 Java 插件。其实，一个 Project 包含多少 Task 往往是插件决定的.
所以，在 Project 中，我们要：

1.加载插件。

2.不同插件有不同的行话，即不同的配置。我们要在 Project 中配置好，这样插件就知道从哪里.

3.3读取源文件等设置属性。

### 加载插件

Project 的 API 位于 https://docs.gradle.org/current/javadoc/org/gradle/api/Project.html。加载插件是调用它的 apply 函数.apply 其实是 Project 实现的 PluginAware 接口定义的：

![enter description here][4]

[apply 函数的用法] apply 是一个函数，此处调用的是图 30 中最后一个 apply 函数。注意，Groovy 支持函数调用的时候通过 参数名 1:参数值 2，参数名 2：参数值 2 的方式来传递参数
apply plugin: 'com.android.library' <==如果是编译 Library，则加载此插件
apply plugin: 'com.android.application' <==如果是编译 Android APP，则加载此插件
除了加载二进制的插件（上面的插件其实都是下载了对应的 jar 包，这也是通常意义上我们所理解的插件），还可以加载一个 gradle 文件。为什么要加载 gradle 文件呢？
其实这和代码的模块划分有关。一般而言，我会把一些通用的函数放到一个名叫 utils.gradle 文件里。然后在其他工程的 build.gradle 来加载这个 utils.gradle。这样，通过一些处理，我就可以调用 utils.gradle 中定义的函数了。
加载 utils.gradle 插件的代码如下：
utils.gradle 是我封装的一个 gradle 脚本，里边定义了一些方便函数，比如读取AndroidManifest.xml 中的 versionName，或者是 copy jar 包/APK 包到指定的目录
apply from: rootProject.getRootDir().getAbsolutePath() + "/utils.gradle"  也是使用 apply 的最后一个函数。那么，apply 最后一个函数到底支持哪些参数呢？还是得看图 31 中的 API 说明：

![enter description here][5]

我这里不遗余力的列出 API 图片，就是希望大家在写脚本的时候，碰到不会的，一定要去查看 API 文档！

### 设置属性

如果是单个脚本，则不需要考虑属性的跨脚本传播，但是 Gradle 往往包含不止一个 build.gradle 文件，比如我设置的 utils.gradle，settings.gradle。如何在多个脚本中设置属性呢？
Gradle 提供了一种名为 extra property 的方法。extra property 是额外属性的意思，在第一次定义该属性的时候需要通过 ext 前缀来标示它是一个额外的属性。定义好之后，后面的存取就不需要 ext 前缀了。ext 属性支持 Project 和 Gradle 对象。即 Project 和 Gradle 对象都可以设置 ext 属性
举个例子：
我在 settings.gradle 中想为 Gradle 对象设置一些外置属性，所以在initMinshengGradleEnvironment 函数中

    def initMinshengGradleEnvironment(){
    //属性值从 local.properites 中读取  
    Properties properties = new Properties()
    File propertyFile = new File(rootDir.getAbsolutePath() + "/local.properties")
    properties.load(propertyFile.newDataInputStream())
    //gradle 就是 gradle 对象。它默认是 Settings 和 Project 的成员变量。可直接获取  
    //ext 前缀，表明操作的是外置属性。api 是一个新的属性名。前面说过，只在  
    //第一次定义或者设置它的时候需要 ext 前缀  
    gradle.ext.api = properties.getProperty('sdk.api')
    println gradle.api  //再次存取 api 的时候，就不需要 ext 前缀了  
    ......
    }

再来一个例子强化一下：

我在 utils.gradle 中定义了一些函数，然后想在其他 build.gradle 中调用这些函数。那该怎么做呢？

      [utils.gradle]
    //utils.gradle 中定义了一个获取 AndroidManifests.xml versionName 的函数  
    def  getVersionNameAdvanced(){
    下面这行代码中的 project 是谁？  
   	def xmlFile = project.file("AndroidManifest.xml") 
   	def rootManifest = new XmlSlurper().parse(xmlFile)
   	return rootManifest['@android:versionName']   
    }
    //现在，想把这个 API 输出到各个 Project。由于这个 utils.gradle 会被每一个 Project Apply，	所以  
        //我可以把 getVersionNameAdvanced 定义成一个 closure，然后赋值到一个外部属性  
    下面的 ext 是谁的 ext？  
    ext{ //此段花括号中代码是闭包  
    //除了 ext.xxx=value 这种定义方法外，还可以使用 ext{}这种书写方法。  
    //ext{}不是 ext(Closure)对应的函数调用。但是 ext{}中的{}确实是闭包。  
    getVersionNameAdvanced = this.&getVersionNameAdvanced
 }
 
上面代码中有两个问题：

project 是谁？

ext 是谁的 ext？

上面两个问题比较关键，我也是花了很长时间才搞清楚。这两个问题归结到一起，其实就是：

加载 utils.gradle 的 Project 对象和 utils.gradle 本身所代表的 Script 对象到底有什么关系？

我们在 Groovy 中也讲过怎么在一个 Script 中 import 另外一个 Script 中定义的类或者函数（见 3.5 脚本类、文件 I/O 和 XML 操作一节）。在 Gradle 中，这一块的处理比 Groovy 要复杂，具体怎么搞我还没完全弄清楚，但是 Project 和 utils.gradle 对于的 Script 的对象的关系是：

1.当一个 Project apply 一个 gradle 文件的时候，这个 gradle 文件会转换成一个 Script 对象。这个，相信大家都已经知道了。

2.Script 中有一个 delegate 对象，这个 delegate 默认是加载（即调用 apply）它的 Project 对象。但是，在 apply 函数中，有一个 from 参数，还有一个 to 参数（参考图 31）。通过 to 参数，你可以把 delegate 对象指定为别的东西。

3.delegate 对象是什么意思？当你在 Script 中操作一些不是 Script 自己定义的变量，或者函数时候，gradle 会到 Script 的 delegate 对象去找，看看有没有定义这些变量或函数。

现在你知道问题 1,2 和答案了：

问题 1：project 就是加载 utils.gradle 的 project。由于 posdevice 有 5 个 project，所以 utils.gradle 会分别加载到 5 个 project 中。所以，getVersionNameAdvanced 才不用区分到底是哪个 project。反正一个 project 有一个 utils.gradle 对应的 Script。

问题 2：ext：自然就是 Project 对应的 ext 了。此处为 Project 添加了一些 closure。那么，在 Project 中就可以调用 getVersionNameAdvanced 函数了

比如：我在 posdevice 每个 build.gradle 中都有如下的代码：

      tasks.getByName("assemble"){
      it.doLast{
        println "$project.name: After assemble, jar libs are copied to local repository"
        copyOutput(true)  //copyOutput 是 utils.gradle 输出的 closure
       }
  }

通过这种方式，我将一些常用的函数放到 utils.gradle 中，然后为加载它的 Project 设置 ext 属性。最后，Project 中就可以调用这种赋值函数了！

注意：此处我研究的还不是很深，而且我个人感觉：

1 在 Java 和 Groovy 中：我们会把常用的函数放到一个辅助类和公共类中，然后在别的地方 import 并调用它们。

2 但是在 Gradle，更正规的方法是在 xxx.gradle 中定义插件。然后通过添加 Task 的方式来完成工作。gradle 的 user guide 有详细介绍如何实现自己的插件。

### Task 介绍
Task 是 Gradle 中的一种数据类型，它代表了一些要执行或者要干的工作。不同的插件可以添加不同的 Task。每一个 Task 都需要和一个 Project 关联。

Task 的 API 文档位于 https://docs.gradle.org/current/dsl/org.gradle.api.Task.html。关于 Task，我这里简单介绍下 build.gradle 中怎么写它，以及 Task 中一些常见的类型

关于 Task。来看下面的例子：

        [build.gradle]
      //Task 是和 Project 关联的，所以，我们要利用 Project 的 task 函数来创建一个 Task
      task myTask  <==myTask 是新建 Task 的名字  

      task myTask { configure closure }
      task myType << { task action }  <==注意，<<符号 是 doLast 的缩写  

      task myTask(type: SomeType)
      task myTask(type: SomeType) { configure closure }

上述代码中都用了 Project 的一个函数，名为 task，注意：

1.一个 Task 包含若干 Action。所以，Task 有 doFirst 和 doLast 两个函数，用于添加需要最先执行的 Action 和需要和需要最后执行的 Action。Action 就是一个闭包。

2.Task 创建的时候可以指定 Type，通过 type:名字表达。这是什么意思呢？其实就是告诉 Gradle，这个新建的 Task 对象会从哪个基类 Task 派生。比如，Gradle 本身提供了一些通用的 Task，最常见的有 Copy 任务。Copy 是 Gradle 中的一个类。当我们：task myTask(type:Copy)的时候，创建的 Task 就是一个 Copy Task。

3.当我们使用 task myTask{ xxx}的时候。花括号是一个 closure。这会导致 gradle 在创建这个 Task 之后，返回给用户之前，会先执行 closure 的内容。

4.当我们使用 task myTask << {xxx}的时候，我们创建了一个 Task 对象，同时把 closure 做为一个 action 加到这个 Task 的 action 队列中，并且告诉它“最后才执行这个 closure”（注意，<<符号是 doLast 的代表）

图 32 是 Project 中关于 task 函数说明：

![enter description here][6]

陆陆续续讲了这么些内容，我自己感觉都有点烦了。是得，Gradle 用一整本书来讲都嫌不够呢。

anyway，到目前为止，我介绍的都是一些比较基础的东西，还不是特别多。但是后续例子该涉及到的知识点都有了。下面我们直接上例子。这里有两个例子：

1.posdevice 的例子

2.另外一个是单个 project 的例子

## posdevice 实例

现在正是开始通过例子来介绍怎么玩 gradle。这里要特别强调一点，根据 Gradle 的哲学。gradle 文件中包含一些所谓的 Script Block（姑且这么称它）。Script Block 作用是让我们来配置相关的信息。不同的 SB 有不同的需要配置的东西。这也是我最早说的行话。比如，源码对应的 SB，就需要我们配置源码在哪个文件夹里。关于 SB，我们后面将见识到！

posdevice 是一个 multi project。下面包含 5 个 Project。对于这种 Project，请大家回想下我们该创建哪些文件？

1.settings.gradle 是必不可少的

2.根目录下的 build.gradle。这个我们没讲过，因为 posdevice 的根目录本身不包含代码，而是包含其他 5 个子 project。

3.每个 project 目录下包含对于的 build.gradle

4.另外，我把常用的函数封装到一个名为 utils.gradle 的脚本里了。

马上一个一个来看它们。

1.utils.gradle utils.gradle 是我自己加的，为我们团队特意加了一些常见函数。主要代码如下：

        [utils.gradle]
    import groovy.util.XmlSlurper  //解析 XML 时候要引入这个 groovy 的 package

    def copyFile(String srcFile,dstFile){
         ......//拷贝文件函数，用于将最后的生成物拷贝到指定的目录  

    }
    def rmFile(String targetFile){
        .....//删除指定目录中的文件  

    }
    def cleanOutput(boolean bJar = true){
        ....//clean 的时候清理  

    }

    def copyOutput(boolean bJar = true){
        ....//copyOutput 内部会调用 copyFile 完成一次 build 的产出物拷贝  

    }

    def getVersionNameAdvanced(){//老朋友  

       def xmlFile = project.file("AndroidManifest.xml")
       def rootManifest = new XmlSlurper().parse(xmlFile)
       return rootManifest['@android:versionName']   
    }

    //对于 android library 编译，我会 disable 所有的 debug 编译任务  

    def disableDebugBuild(){
      //project.tasks 包含了所有的 tasks，下面的 findAll 是寻找那些名字中带 debug 的 Task。  

      //返回值保存到 targetTasks 容器中  

      def targetTasks = project.tasks.findAll{task ->
          task.name.contains("Debug")
      }
      //对满足条件的 task，设置它为 disable。如此这般，这个 Task 就不会被执行  

      targetTasks.each{
         println "disable debug task  : ${it.name}"
         it.setEnabled false
      }
    }
    //将函数设置为 extra 属性中去，这样，加载 utils.gradle 的 Project 就能调用此文件中定义的函数了  

    ext{
        copyFile = this.&copyFile
        rmFile = this.&rmFile
        cleanOutput = this.&cleanOutput
        copyOutput = this.&copyOutput
        getVersionNameAdvanced = this.&getVersionNameAdvanced
        disableDebugBuild = this.&disableDebugBuild
    }

图 33 展示了被 disable 的 Debug 任务的部分信息：

![enter description here][7]

2.settings.gradle

这个文件中我们该干什么？调用 include 把需要包含的子 Project 加进来。代码如下：

   

        [settings.gradle]
    /*我们团队内部建立的编译环境初始化函数  

      这个函数的目的是  

      1  解析一个名为 local.properties 的文件，读取 Android SDK 和 NDK 的路径  

      2  获取最终产出物目录的路径。这样，编译完的 apk 或者 jar 包将拷贝到这个最终产出物目录中  

      3 获取 Android SDK 指定编译的版本  

    */
    def initMinshengGradleEnvironment(){
        println "initialize Minsheng Gradle Environment ....."
        Properties properties = new Properties()
       //local.properites 也放在 posdevice 目录下  

        File propertyFile = new File(rootDir.getAbsolutePath() + "/local.properties")
        properties.load(propertyFile.newDataInputStream())
        /*
          根据 Project、Gradle 生命周期的介绍，settings 对象的创建位于具体 Project 创建之前  

          而 Gradle 底对象已经创建好了。所以，我们把 local.properties 的信息读出来后，通过  

          extra 属性的方式设置到 gradle 对象中  

          而具体 Project 在执行的时候，就可以直接从 gradle 对象中得到这些属性了！  

        */
        gradle.ext.api = properties.getProperty('sdk.api')
        gradle.ext.sdkDir = properties.getProperty('sdk.dir')
         gradle.ext.ndkDir = properties.getProperty('ndk.dir')
         gradle.ext.localDir = properties.getProperty('local.dir')
        //指定 debug keystore 文件的位置，debug 版 apk 签名的时候会用到  

        gradle.ext.debugKeystore = properties.getProperty('debug.keystore')
         ......
        println "initialize Minsheng Gradle Environment completes..."
    }
    //初始化  
    initMinshengGradleEnvironment()
    //添加子 Project 信息  
    include 'CPosSystemSdk' , 'CPosDeviceSdk' , 'CPosSdkDemo','CPosDeviceServerApk', 'CPosSystemSdkWizarPosImpl'

  注意，对于 Android 来说，local.properties 文件是必须的，它的内容如下：
   
  

        [local.properties]
    local.dir=/home/innost/workspace/minsheng-flat-dir/
    //注意，根据 Android Gradle 的规范，只有下面两个属性是必须的，其余都是我自己加的  

    sdk.dir=/home/innost/workspace/android-aosp-sdk/
    ndk.dir=/home/innost/workspace/android-aosp-ndk/
    debug.keystore=/home/innost/workspace/tools/mykeystore.jks
    sdk.api=android-19

 
再次强调，sdk.dir 和 ndk.dir 是 Android Gradle 必须要指定的，其他都是我自己加的属性。当然。不编译 ndk，就不需要 ndk.dir 属性了。

3.posdevice build.gradle

作为 multi-project 根目录，一般情况下，它的 build.gradle 是做一些全局配置。来看我的 build.gradle

        [posdevice build.gradle]
    //下面这个 subprojects{}就是一个 Script Block
    subprojects {
      println "Configure for $project.name"  //遍历子 Project，project 变量对应每个子 Project
      buildscript {  //这也是一个 SB
        repositories { //repositories 是一个 SB
            ///jcenter 是一个函数，表示编译过程中依赖的库，所需的插件可以在 jcenter 仓库中  

           //下载。 
            jcenter()
        }
        dependencies { //SB
            //dependencies 表示我们编译的时候，依赖 android 开发的 gradle 插件。插件对应的  

           //class path 是 com.android.tools.build。版本是 1.2.3
            classpath 'com.android.tools.build:gradle:1.2.3'
        }
       //为每个子 Project 加载 utils.gradle 。当然，这句话可以放到 buildscript 花括号之后  

       apply from: rootProject.getRootDir().getAbsolutePath() + "/utils.gradle"
     }//buildscript 结束  
    }


感觉解释得好苍白，SB 在 Gradle 的 API 文档中也是有的。先来看 Gradle 定义了哪些 SB。如图 34 所示：

![enter description here][8]

你看，subprojects、dependencies、repositories 都是 SB。那么 SB 到底是什么？它是怎么完成所谓配置的呢？

仔细研究，你会发现 SB 后面都需要跟一个花括号，而花括号，恩，我们感觉里边可能一个 Closure。由于图 34 说，这些 SB 的 Description 都有“Configure xxx for this project”，所以很可能 subprojects 是一个函数，然后其参数是一个 Closure。是这样的吗？

Absolutely right。只是这些函数你直接到 Project API 里不一定能找全。不过要是你好奇心重，不妨到 https://docs.gradle.org/current/javadoc/，选择 Index 这一项，然后 ctrl+f，输入图 34 中任何一个 Block，你都会找到对应的函数。比如我替你找了几个 API，如图 35 所示：

![enter description here][9]

特别提示：当你下次看到一个不认识的 SB 的时候，就去看 API 吧。

下面来解释代码中的各个 SB：

subprojects：它会遍历 posdevice 中的每个子 Project。在它的 Closure 中，默认参数是子 Project 对应的 Project 对象。由于其他 SB 都在 subprojects 花括号中，所以相当于对每个 Project 都配置了一些信息。

buildscript：它的 closure 是在一个类型为 ScriptHandler 的对象上执行的。主意用来所依赖的 classpath 等信息。通过查看 ScriptHandler API 可知，在 buildscript SB 中，你可以调用 ScriptHandler 提供的 repositories(Closure )、dependencies(Closure)函数。这也是为什么 repositories 和 dependencies 两个 SB 为什么要放在 buildscript 的花括号中的原因。明白了？这就是所谓的行话，得知道规矩。不知道规矩你就乱了。记不住规矩，又不知道查 SDK，那么就彻底抓瞎，只能到网上到处找答案了！

关于 repositories 和 dependencies，大家直接看 API 吧。后面碰到了具体代码我们再来介绍
4.CPosDeviceSdk build.gradle

CPosDeviceSdk 是一个 Android Library。按 Google 的想法，Android Library 编译出来的应该是一个 AAR 文件。但是我的项目有些特殊，我需要发布 CPosDeviceSdk.jar 包给其他人使用。jar 在编译过程中会生成，但是它不属于 Android Library 的标准输出。在这种情况下，我需要在编译完成后，主动 copy jar 包到我自己设计的产出物目录中。

        //Library 工程必须加载此插件。注意，加载了 Android 插件就不要加载 Java 插件了。因为 Android
    //插件本身就是拓展了 Java 插件  
    apply plugin: 'com.android.library'  
    //android 的编译，增加了一种新类型的 Script Block-->android
    android {
           //你看，我在 local.properties 中设置的 API 版本号，就可以一次设置，多个 Project 使用了  

          //借助我特意设计的 gradle.ext.api 属性  

           compileSdkVersion = gradle.api  //这两个红色的参数必须设置  

           buildToolsVersion  = "22.0.1"
           sourceSets{ //配置源码路径。这个 sourceSets 是 Java 插件引入的  

            main{ //main：Android 也用了  

                manifest.srcFile 'AndroidManifest.xml'  //这是一个函数，设置 manifest.srcFile
                aidl.srcDirs=['src'] //设置 aidl 文件的目录  

                java.srcDirs=['src'] //设置 java 文件的目录  

            }
         }
        dependencies {  //配置依赖关系  

           //compile 表示编译和运行时候需要的 jar 包，fileTree 是一个函数，  

          //dir:'libs'，表示搜索目录的名称是 libs。include:['*.jar']，表示搜索目录下满足*.jar 名字的 jar
         //包都作为依赖 jar 文件  

            compile fileTree(dir: 'libs', include: ['*.jar'])
       }
    }  //android SB 配置完了  

    //clean 是一个 Task 的名字，这个 Task 好像是 Java 插件（这里是 Android 插件）引入的。  
    //dependsOn 是一个函数，下面这句话的意思是 clean 任务依赖 cposCleanTask 任务。所以  
    //当你 gradle clean 以执行 clean Task 的时候，cposCleanTask 也会执行  
    clean.dependsOn 'cposCleanTask'
    //创建一个 Task，  
    task cposCleanTask() <<{
        cleanOutput(true)  //cleanOutput 是 utils.gradle 中通过 extra 属性设置的 Closure
    }
    //前面说了，我要把 jar 包拷贝到指定的目录。对于 Android 编译，我一般指定 gradle assemble
    //它默认编译 debug 和 release 两种输出。所以，下面这个段代码表示：  
    //tasks 代表一个 Projects 中的所有 Task，是一个容器。getByName 表示找到指定名称的任务。  
    //我这里要找的 assemble 任务，然后我通过 doLast 添加了一个 Action。这个 Action 就是 copy
    //产出物到我设置的目标目录中去  
    tasks.getByName("assemble"){
        it.doLast{
            println "$project.name: After assemble, jar libs are copied to local repository"
            copyOutput(true)
         }
    }
    /*
      因为我的项目只提供最终的 release 编译出来的 Jar 包给其他人，所以不需要编译 debug 版的东西  

      当 Project 创建完所有任务的有向图后，我通过 afterEvaluate 函数设置一个回调 Closure。在这个回调  

      Closure 里，我 disable 了所有 Debug 的 Task
    */
    project.afterEvaluate{
        disableDebugBuild()
    }


Android 自己定义了好多 ScriptBlock。Android 定义的 DSL 参考文档在

https://developer.android.com/tools/building/plugin-for-gradle.html 下载。注意，它居然没有提供在线文档。

图 36 所示为 Android 的 DSL 参考信息。

![enter description here][10]

图 37 为 buildToolsVersion 和 compileSdkVersion 的说明：

![enter description here][11]

从图 37 可知，这两个变量是必须要设置的.....

5.CPosDeviceServerApk build.gradle

再来看一个 APK 的 build，它包含 NDK 的编译，并且还要签名。根据项目的需求，我们只能签 debug 版的，而 release 版的签名得发布 unsigned 包给领导签名。另外，CPosDeviceServerAPK 依赖 CPosDeviceSdk。
虽然我可以先编译 CPosDeviceSdk，得到对应的 jar 包，然后设置 CPosDeviceServerApk 直接依赖这个 jar 包就好。但是我更希望 CPosDeviceServerApk 能直接依赖于 CPosDeviceSdk 这个工程。这样，整个 posdevice 可以做到这几个 Project 的依赖关系是最新的。

        [build.gradle]
    apply plugin: 'com.android.application'  //APK 编译必须加载这个插件  
    android {
           compileSdkVersion gradle.api 
           buildToolsVersion "22.0.1"
           sourceSets{  //差不多的设置  

            main{
                manifest.srcFile 'AndroidManifest.xml'
              //通过设置 jni 目录为空，我们可不使用 apk 插件的 jni 编译功能。为什么？因为据说  

              //APK 插件的 jni 功能好像不是很好使....晕菜  

               jni.srcDirs = []  
                jniLibs.srcDir 'libs'
                aidl.srcDirs=['src']
                java.srcDirs=['src']
                res.srcDirs=['res']
            }
        }//main 结束  

        signingConfigs { //设置签名信息配置  

            debug {  //如果我们在 local.properties 设置使用特殊的 keystore，则使用它  

               //下面这些设置，无非是函数调用....请务必阅读 API 文档  

               if(project.gradle.debugKeystore != null){ 
                   storeFile file("file://${project.gradle.debugKeystore}")
                   storePassword "android"
                   keyAlias "androiddebugkey"
                   keyPassword "android"
                }
            }
       }//signingConfigs 结束  

         buildTypes {
            debug {
                signingConfig signingConfigs.debug
                jniDebuggable false
            }
        }//buildTypes 结束  

        dependencies {
            //compile：project 函数可指定依赖 multi-project 中的某个子 project
            compile project(':CPosDeviceSdk')
            compile fileTree(dir: 'libs', include: ['*.jar'])
       } //dependices 结束  

      repositories {
       flatDir { //flatDir：告诉 gradle，编译中依赖的 jar 包存储在 dirs 指定的目录  

                name "minsheng-gradle-local-repository"
                dirs gradle.LOCAL_JAR_OUT //LOCAL_JAR_OUT 是我存放编译出来的 jar 包的位置  

       }
      }//repositories 结束  

    }//android 结束  

    /*
       创建一个 Task，类型是 Exec，这表明它会执行一个命令。我这里让他执行 ndk 的  

       ndk-build 命令，用于编译 ndk。关于 Exec 类型的 Task，请自行脑补 Gradle 的 API
    */
    //注意此处创建 task 的方法，是直接{}喔，那么它后面的 tasks.withType(JavaCompile)
    //设置的依赖关系，还有意义吗？Think！如果你能想明白，gradle 掌握也就差不多了  

    task buildNative(type: Exec, description: 'Compile JNI source via NDK') {
            if(project.gradle.ndkDir == null) //看看有没有指定 ndk.dir 路径  

               println "CANNOT Build NDK"
            else{
                commandLine "/${project.gradle.ndkDir}/ndk-build",
                    '-C', file('jni').absolutePath,
                    '-j', Runtime.runtime.availableProcessors(),
                    'all', 'NDK_DEBUG=0'
            }
      }
     tasks.withType(JavaCompile) {
            compileTask -> compileTask.dependsOn buildNative
      }
      ......   
     //对于 APK，除了拷贝 APK 文件到指定目录外，我还特意为它们加上了自动版本命名的功能  

     tasks.getByName("assemble"){
            it.doLast{
            println "$project.name: After assemble, jar libs are copied to local repository"
            project.ext.versionName = android.defaultConfig.versionName
            println "\t versionName = $versionName"
            copyOutput(false)
         }
    }


6.结果展示

在 posdevice 下执行 gradle assemble 命令，最终的输出文件都会拷贝到我指定的目录，结果如图 38 所示：

![enter description here][12]

图 38 所示为 posdevice gradle assemble 的执行结果：

1.library 包都编译 release 版的，copy 到 xxx/javaLib 目录下

2.apk 编译 debug 和 release-unsigned 版的，copy 到 apps 目录下

3.所有产出物都自动从 AndroidManifest.xml 中提取 versionName。

## 实例 2

下面这个实例也是来自一个实际的 APP。这个 APP 对应的是一个单独的 Project。但是根据我前面的建议，我会把它改造成支持 Multi-Projects Build 的样子。即在工程目录下放一个 settings.build。

另外，这个 app 有一个特点。它有三个版本，分别是 debug、release 和 demo。这三个版本对应的代码都完全一样，但是在运行的时候需要从 assets/runtime_config 文件中读取参数。参数不同，则运行的时候会跳转到 debug、release 或者 demo 的逻辑上。

注意：我知道 assets/runtime_config 这种做法不 decent，但，这是一个既有项目，我们只能做小范围的适配，而不是伤筋动骨改用更好的方法。另外，从未来的需求来看，暂时也没有大改的必要。

引入 gradle 后，我们该如何处理呢？

解决方法是：在编译 build、release 和 demo 版本前，在 build.gradle 中自动设置 runtime_config 的内容。代码如下所示：

        [build.gradle]
    apply plugin: 'com.android.application'  //加载 APP 插件  

    //加载 utils.gradle
    apply from: rootProject.getRootDir().getAbsolutePath() + "/utils.gradle"
    //buildscript 设置 android app 插件的位置  

    buildscript {
        repositories { jcenter() }
        dependencies { classpath 'com.android.tools.build:gradle:1.2.3' }
    }
    //android ScriptBlock
    android {
        compileSdkVersion gradle.api
        buildToolsVersion "22.0.1"
       sourceSets{ //源码设置 SB
            main{
                manifest.srcFile 'AndroidManifest.xml'
                jni.srcDirs = []
                jniLibs.srcDir 'libs'
                aidl.srcDirs=['src']
                java.srcDirs=['src']
                res.srcDirs=['res']
                assets.srcDirs = ['assets'] //多了一个 assets 目录  

            }
        }
        signingConfigs {//签名设置  

            debug {  //debug 对应的 SB。注意  

                if(project.gradle.debugKeystore != null){
                    storeFile file("file://${project.gradle.debugKeystore}")
                    storePassword "android"
                    keyAlias "androiddebugkey"
                    keyPassword "android"
                }
            }
        }
        /*
         最关键的内容来了： buildTypes ScriptBlock.
         buildTypes 和上面的 signingConfigs，当我们在 build.gradle 中通过{}配置它的时候，  

         其背后的所代表的对象是 NamedDomainObjectContainer<BuildType> 和  

         NamedDomainObjectContainer<SigningConfig>
         注意，NamedDomainObjectContainer<BuildType/或者 SigningConfig>是一种容器，  

         容器的元素是 BuildType 或者 SigningConfig。我们在 debug{}要填充 BuildType 或者  

        SigningConfig 所包的元素，比如 storePassword 就是 SigningConfig 类的成员。而 proguardFile 等  

        是 BuildType 的成员。  

        那么，为什么要使用 NamedDomainObjectContainer 这种数据结构呢？因为往这种容器里  

        添加元素可以采用这样的方法： 比如 signingConfig 为例  

        signingConfig{//这是一个 NamedDomainObjectContainer<SigningConfig>
           test1{//新建一个名为 test1 的 SigningConfig 元素，然后添加到容器里  

             //在这个花括号中设置 SigningConfig 的成员变量的值  

           }
          test2{//新建一个名为 test2 的 SigningConfig 元素，然后添加到容器里  

             //在这个花括号中设置 SigningConfig 的成员变量的值  

          }
        }
        在 buildTypes 中，Android 默认为这几个 NamedDomainObjectContainer 添加了  

        debug 和 release 对应的对象。如果我们再添加别的名字的东西，那么 gradle assemble 的时候  

        也会编译这个名字的 apk 出来。比如，我添加一个名为 test 的 buildTypes，那么 gradle assemble
        就会编译一个 xxx-test-yy.apk。在此，test 就好像 debug、release 一样。  

       */
        buildTypes{
            debug{ //修改 debug 的 signingConfig 为 signingConfig.debug 配置  

                signingConfig signingConfigs.debug
            }
            demo{ //demo 版需要混淆  

                proguardFile 'proguard-project.txt'
                signingConfig signingConfigs.debug
            }
           //release 版没有设置，所以默认没有签名，没有混淆  

        }
          ......//其他和 posdevice 类似的处理。来看如何动态生成 runtime_config 文件  

       def  runtime_config_file = 'assets/runtime_config'
       /*
       我们在 gradle 解析完整个任务之后，找到对应的 Task，然后在里边添加一个 doFirst Action
       这样能确保编译开始的时候，我们就把 runtime_config 文件准备好了。  

       注意，必须在 afterEvaluate 里边才能做，否则 gradle 没有建立完任务有向图，你是找不到  

       什么 preDebugBuild 之类的任务的  

       */
        project.afterEvaluate{
          //找到 preDebugBuild 任务，然后添加一个 Action  
          tasks.getByName("preDebugBuild"){
                it.doFirst{
                    println "generate debug configuration for ${project.name}"
                    def configFile = new File(runtime_config_file)
                    configFile.withOutputStream{os->
                        os << I am Debug\n'  //往配置文件里写 I am Debug
                     }
                }
            }
           //找到 preReleaseBuild 任务  

            tasks.getByName("preReleaseBuild"){
                it.doFirst{
                    println "generate release configuration for ${project.name}"
                    def configFile = new File(runtime_config_file)
                    configFile.withOutputStream{os->
                        os << I am release\n'
                    }
                }
            }
           //找到 preDemoBuild。这个任务明显是因为我们在 buildType 里添加了一个 demo 的元素  

          //所以 Android APP 插件自动为我们生成的  

            tasks.getByName("preDemoBuild"){
                it.doFirst{
                    println "generate offlinedemo configuration for ${project.name}"
                    def configFile = new File(runtime_config_file)
                    configFile.withOutputStream{os->
                        os << I am Demo\n'
                    }
                }
            }
        }
    }
     .....//copyOutput

最终的结果如图 39 所示：

![enter description here][13]

几个问题，为什么我知道有 preXXXBuild 这样的任务？

答案：gradle tasks --all 查看所有任务。然后，多尝试几次，直到成功

# 总结

到此，我个人觉得 Gradle 相关的内容都讲完了。很难相信我仅花了 1 个小时不到的时间就为实例 2 添加了 gradle 编译支持。在一周以前，我还觉得这是个心病。回想学习 gradle 的一个月时间里，走过不少弯路，求解问题的思路也和最开始不一样：

1.最开始的时候，我一直把 gradle 当做脚本看。然后到处到网上找怎么配置 gradle。可能能编译成功，但是完全不知道为什么。比如 NameDomainObjectContainer，为什么有 debug、release。能自己加别的吗？不知道怎么加，没有章法，没有参考。出了问题只能 google，找到一个解法，试一试，成功就不管。这么搞，心里不踏实。

2.另外，对语法不熟悉，尤其是 Groovy 语法，虽然看了下快速教材，但总感觉一到 gradle 就看不懂。主要问题还是闭包，比如 Groovy 那一节写得文件拷贝的例子中的 withOutputStream，还有 gradle 中的 withType，都是些啥玩意啊？

3.所以后来下决心先把 Groovy 学会，主要是把自己暴露在闭包里边。另外，Groovy 是一门语言，总得有 SDK 说明吧。写了几个例子，慢慢体会到 Groovy 的好处，也熟悉 Groovy 的语法了。

4.接着开始看 Gradle。Gradle 有几本书，我看过 Gradle in Action。说实话，看得非常痛苦。现在想起来，Gradle 其实比较简单，知道它的生命周期，知道它怎么解析脚本，知道它的 API，几乎很快就能干活。而 Gradle In Action 一上来就很细，而且没有从 API 角度介绍。说个很有趣的事情，书中有个类似下面的例子

        task myTask  <<  {
       println ' I am myTask'
    }

书中说，如果代码没有加<<，则这个任务在脚本 initialization（也就是你无论执行什么任务，这个任务都会被执行，I am myTask 都会被输出）的时候执行，如果加了<<，则在 gradle myTask 后才执行。

尼玛我开始完全不知道为什么，死记硬背。现在你明白了吗？？？？

这和我们调用 task 这个函数的方式有关！如果没有<<，则闭包在 task 函数返回前会执行，而如果加了<<，则变成调用 myTask.doLast 添加一个 Action 了，自然它会等到 grdle myTask 的时候才会执行！

现在想起这个事情我还是很愤怒，API 都说很清楚了......而且，如果你把 Gradle 当做编程框架来看，对于我们这些程序员来说，写这几百行代码，那还算是事嘛？？

转自 极客学院 [深入理解 Android 之 Gradle][14] 系列文章




  [1]: http://wiki.jikexueyuan.com/project/deep-android-gradle/images/29.jpg
  [2]: http://wiki.jikexueyuan.com/project/deep-android-gradle/images/30.jpg
  [3]: http://wiki.jikexueyuan.com/project/deep-android-gradle/images/31.jpg
  [4]: http://wiki.jikexueyuan.com/project/deep-android-gradle/images/32.jpg
  [5]: http://wiki.jikexueyuan.com/project/deep-android-gradle/images/33.jpg
  [6]: http://wiki.jikexueyuan.com/project/deep-android-gradle/images/34.jpg
  [7]: http://wiki.jikexueyuan.com/project/deep-android-gradle/images/35.jpg
  [8]: http://wiki.jikexueyuan.com/project/deep-android-gradle/images/36.jpg
  [9]: http://wiki.jikexueyuan.com/project/deep-android-gradle/images/37.jpg
  [10]: http://wiki.jikexueyuan.com/project/deep-android-gradle/images/38.jpg
  [11]: http://wiki.jikexueyuan.com/project/deep-android-gradle/images/39.jpg
  [12]: http://wiki.jikexueyuan.com/project/deep-android-gradle/images/40.jpg
  [13]: http://wiki.jikexueyuan.com/project/deep-android-gradle/images/41.jpg
  [14]: http://wiki.jikexueyuan.com/project/deep-android-gradle/four-four.html