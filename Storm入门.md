# 1.Storm 基础知识

2016-08-12

[http://wiki.jikexueyuan.com/project/storm/basic-knowledge.html](http://wiki.jikexueyuan.com/project/storm/basic-knowledge.html)

## 1.1 基础知识

Storm 是一个分布式的，可靠的，容错的数据流处理系统。它会把工作任务委托给不同类型的组件，每个组件负责处理一项简单特定的任务。Storm 集群的输入流由一个被称作 spout 的组件管理，spout 把数据传递给 bolt， bolt 要么把数据保存到某种存储器，要么把数据传递给其它的 bolt。你可以想象一下，一个 Storm 集群就是在一连串的 bolt 之间转换 spout 传过来的数据。

这里用一个简单的例子来说明这个概念。昨晚我在新闻节目里看到主持人在谈论政治人物和他们对于各种政治话题的立场。他们一直重复着不同的名字，而我开始考虑这些名字是否被提到了相同的次数，以及不同次数之间的偏差。

想像播音员读的字幕作为你的数据输入流。你可以用一个 spout 读取一个文件（或者 socket，通过 HTTP，或者别的方法）。文本行被 spout 传给一个 bolt，再被 bolt 按单词切割。单词流又被传给另一个 bolt，在这里每个单词与一张政治人名列表比较。每遇到一个匹配的名字，第二个 bolt 为这个名字在数据库的计数加1。你可以随时查询数据库查看结果， 而且这些计数是随着数据到达实时更新的。所有组件（spouts和bolts）及它们之间的关系请参考拓扑图1-1

![](https://img.w3cschool.cn/attachments/image/wk/storm/01.png)

现在想象一下，很容易在整个 Storm 集群定义每个 bolt 和 spout 的并行性级别，因此你可以无限的扩展你的拓扑结构。

## 1.2 Storm 应用案例

* 数据处理流：正如上例所展示的，不像其它的流处理系统，Storm 不需要中间队列。
* 连续计算：连续发送数据到客户端，使它们能够实时更新并显示结果，如网站指标。
* 分布式远程过程调用：频繁的 CPU 密集型操作并行化。
* Storm 组件：对于一个Storm集群，一个连续运行的主节点组织若干节点工作。

在 Storm 集群中，有两类节点：主节点 master node 和工作节点 worker nodes。主节点运行着一个叫做 Nimbus 的守护进程。这个守护进程负责在集群中分发代码，为工作节点分配任务，并监控故障。Supervisor守护进程作为拓扑的一部分运行在工作节点上。一个 Storm 拓扑结构在不同的机器上运行着众多的工作节点。

因为 Storm 在 Zookeeper 或本地磁盘上维持所有的集群状态，守护进程可以是无状态的而且失效或重启时不会影响整个系统的健康（见图1-2）

![](https://img.w3cschool.cn/attachments/image/wk/storm/02.png)

在系统底层，Storm 使用了 zeromq\(0mq, zeromq\([http://www.zeromq.org](http://www.zeromq.org/)\)\)。这是一种先进的，可嵌入的网络通讯库，它提供的绝妙功能使 Storm 成为可能。下面列出一些 zeromq 的特性。

* 一个并发架构的 Socket 库
* 对于集群产品和超级计算，比 TCP 要快
* 可通过 inproc（进程内）, IPC（进程间）, TCP 和 multicast\(多播协议\)通信
* 异步 I / O 的可扩展的多核消息传递应用程序
* 利用扇出\(fanout\), 发布订阅（PUB-SUB）,管道（pipeline）, 请求应答（REQ-REP），等方式实现 N-N 连接

**NOTE**: Storm 只用了 push/pull sockets

## 1.3 Storm 的特性

在所有这些设计思想与决策中，有一些非常棒的特性成就了独一无二的 Storm。

* 简化编程：如果你曾试着从零开始实现实时处理，你应该明白这是一件多么痛苦的事情。使用 Storm，复杂性被大大降低了。
* 使用一门基于 JVM 的语言开发会更容易，但是你可以借助一个小的中间件，在 Storm 上使用任何语言开发。有现成的中间件可供选择，当然也可以自己开发中间件。
* 容错：Storm 集群会关注工作节点状态，如果宕机了必要的时候会重新分配任务。
* 可扩展：所有你需要为扩展集群所做的工作就是增加机器。Storm 会在新机器就绪时向它们分配任务。
* 可靠的：所有消息都可保证至少处理一次。如果出错了，消息可能处理不只一次，不过你永远不会丢失消息。
* 快速：速度是驱动 Storm 设计的一个关键因素
* 事务性：You can get exactly once messaging semantics for pretty much any computation. 你可以为几乎任何计算得到恰好一次消息语义。

## 1.4 Storm学习思维导图

![](/assets/Storm学习思维导图.png)上面这张思维导图，是参考资料storm的百度百科进行绘制的。

# 2.Storm起步

在本章，我们要创建一个 Storm 工程和我们的第一个 Storm 拓扑结构。

**NOTE**: 下面假设你的 JRE 版本在 1.6 以上。我们推荐 Oracle 提供的 JRE。你可以到[http://www.java.com/downloads/](http://www.java.com/downloads/)下载。

## 2.1操作模式 {#7658e8015178d2c0964eff4ebcdee7c4}

开始之前，有必要了解一下 Storm 的操作模式。有下面两种方式。

### 2.1.1本地模式

在本地模式下，Storm 拓扑结构运行在本地计算机的单一 JVM 进程上。这个模式用于开发、测试以及调试，因为这是观察所有组件如何协同工作的最简单方法。在这种模式下，我们可以调整参数，观察我们的拓扑结构如何在不同的 Storm 配置环境下运行。要在本地模式下运行，我们要下载 Storm 开发依赖，以便用来开发并测试我们的拓扑结构。我们创建了第一个 Storm 工程以后，很快就会明白如何使用本地模式了。

**NOTE**: 在本地模式下，跟在集群环境运行很像。不过很有必要确认一下所有组件都是线程安全的，因为当把它们部署到远程模式时它们可能会运行在不同的 JVM 进程甚至不同的物理机上，这个时候它们之间没有直接的通讯或共享内存。

我们要在本地模式运行本章的所有例子。

### 2.1.2远程模式

在远程模式下，我们向 Storm 集群提交拓扑，它通常由许多运行在不同机器上的流程组成。远程模式不会出现调试信息， 因此它也称作生产模式。不过在单一开发机上建立一个 Storm 集群是一个好主意，可以在部署到生产环境之前，用来确认拓扑在集群环境下没有任何问题。

## 2.2编写Storm的Hello World {#b10a8db164e0754105b7a99be72e3fe5}

我们在这个工程里创建一个简单的拓扑，数单词数量。我们可以把这个看作 Storm 的 “Hello World”。不过，这是一个非常强大的拓扑，因为它能够扩展到几乎无限大的规模，而且只需要做一些小修改，就能用它构建一个统计系统。举个例子，我们可以修改一下工程用来找出 Twitter 上的热点话题。

要创建这个拓扑，我们要用一个 spout 读取文本，第一个 bolt 用来标准化单词，第二个 bolt 为单词计数，如图1-1所示。

![](http://wiki.jikexueyuan.com/project/storm/images/03.png)

你可以从这个网址下载源码压缩包，[https://github.com/storm-book/examples-ch02-getting\_started/zipball/master](https://github.com/storm-book/examples-ch02-getting_started/zipball/master)。

**NOTE**: 如果你使用[git](http://git-scm.com/)（一个分布式版本控制与源码管理工具），你可以执行`git clone [git@github.com](git@github.com):storm-book/examples-ch02-getting_started.git`，把源码检出到你指定的目录。

### 2.2.1Java 安装检查

构建 Storm 运行环境的第一步是检查你安装的 Java 版本。打开一个控制台窗口并执行命令：java -version。控制台应该会显示出类似如下的内容：

```
    java -version

    java version "1.6.0_26"
    Java(TM) SE Runtime Enviroment (build 1.6.0_26-b03)

    Java HotSpot(TM) Server VM (build 20.1-b02, mixed mode)
```

如果不是上述内容，检查你的 Java 安装情况。（参考[http://www.java.com/download/](http://www.java.com/download/)）

## 2.2.2创建maven工程 {#f5050010b25ae5ffc86d2048c357a9c9}

开始之前，先为这个应用建一个目录（就像你平常为 Java 应用做的那样）。这个目录用来存放工程源码。

接下来我们要下载 Storm 依赖包，这是一些 jar 包，我们要把它们添加到应用类路径中。你可以采用如下两种方式之一完成这一步：

* 下载所有依赖，解压缩它们，把它 们添加到类路径
* 使用 Apache Maven

**NOTE**: Maven 是一个软件项目管理的综合工具。它可以用来管理项目的开发周期的许多方面，从包依赖到版本发布过程。在这本书中，我们将广泛使用它。如果要检查是否已经安装了maven，在命令行运行 mvn。如果没有安装你可以从[http://maven.apache.org/download.html](http://maven.apache.org/download.html)下载。

没有必要先成为一个 Maven 专家才能使用 Storm，不过了解一下关于 Maven 工作方式的基础知识仍然会对你有所帮助。你可以在 Apache Maven 的网站上找到更多的信息（[http://maven.apache.org/](http://maven.apache.org/)）。

**NOTE**: Storm 的 Maven 依赖引用了运行 Storm 本地模式的所有库。

要运行我们的拓扑，我们可以编写一个包含基本组件的 pom.xml 文件。

```
<project xmlns="http://maven.apache.org/POM/4.0.0"
             xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
             xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
             http://maven.apache.org/xsd/maven-4.0.0.xsd">
             <modelVersion>4.0.0</modelVersion>
             <groupId>storm.book</groupId>
             <artifactId>Getting-Started</artifactId>
             <version>0.0.1-SNAPSHOT</version>

             <build>
                 <plugins>
                     <plugin>
                         <groupId>org.apache.maven.plugins</groupId>
                         <artifactId>maven-compiler-plugin</artifactId>
                         <version>2.3.2</version>
                         <configuration>
                             <source>1.6</source>
                             <target>1.6</target>
                             <compilerVersion>1.6</compilerVersion>
                         </configuration>
                     </plugin>
                 </plugins>
             </build>
             <repositories>
                 <!-- Repository where we can found the storm dependencies -->
                 <repository>
                     <id>clojars.org</id>
                     <url>http://clojars.org/repo</url>
                 </repository>
             </repositories>
             <dependencies>
                 <!-- Storm Dependency -->
                 <dependency>
                      <groupId>org.apache.storm</groupId>
                      <artifactId>storm-core</artifactId>
                      <version>0.9.6</version>
                      <scope>provided</scope>
                </dependency>
             </dependencies>
    </project>
```

开头几行指定了工程名称和版本号。然后我们添加了一个编译器插件，告知 Maven 我们的代码要用** Java1.6 编译**。接下来我们定义了 Maven 仓库（Maven 支持为同一个工程指定多个仓库）。clojars 是存放 Storm 依赖的仓库。Maven 会为运行本地模式自动下载必要的所有子包依赖。

其实，（亲测）只需要添加下面的一个依赖就行：

```
<dependency>
    <groupId>org.apache.storm</groupId>
    <artifactId>storm-core</artifactId>
    <version>0.9.6</version>
    <scope>provided</scope>
</dependency>
```

一个典型的 Maven Java 工程会拥有如下结构：

```
我们的应用目录/
         ├── pom.xml
         └── src
               └── main
                  └── java
               |  ├── spouts
               |  └── bolts
               └── resources
```

java 目录下的子目录包含我们的代码，我们把要统计单词数的文件保存在 resource 目录下。

**NOTE**：命令 mkdir -p 会创建所有需要的父目录。

### 2.2.3创建我们的第一个 Topology

我们将为运行单词计数创建所有必要的类。

#### 2.2.3.1  Spout

WordReader 类实现了 IRichSpout 接口。**WordReader负责从文件按行读取文本，并把文本行提供给第一个 bolt。**

**NOTE**:** 一个 spout 发布一个定义域列表。这个架构允许你使用不同的 bolts 从同一个spout 流读取数据，它们的输出也可作为其它 bolts 的定义域**，以此类推。

例2-1包含 WordRead 类的完整代码（我们将会分析下述代码的每一部分）。

```
       /**
         *  例2-1.src/main/java/spouts/WordReader.java
         */
        package spouts;

        import java.io.BufferedReader;
        import java.io.FileNotFoundException;
        import java.io.FileReader;
        import java.util.Map;
        import backtype.storm.spout.SpoutOutputCollector;
        import backtype.storm.task.TopologyContext;
        import backtype.storm.topology.IRichSpout;
        import backtype.storm.topology.OutputFieldsDeclarer;
        import backtype.storm.tuple.Fields;
        import backtype.storm.tuple.Values;

        public class WordReader implements IRichSpout {
            private SpoutOutputCollector collector;
            private FileReader fileReader;
            private boolean completed = false;
            private TopologyContext context;
            public boolean isDistributed() {return false;}
            public void ack(Object msgId) {
                    System.out.println("OK:"+msgId);
            }
            public void close() {}
            public void fail(Object msgId) {
                 System.out.println("FAIL:"+msgId);
            }
            /**
             * 这个方法做的惟一一件事情就是分发文件中的文本行
             */
            public void nextTuple() {
            /**
             * 这个方法会不断的被调用，直到整个文件都读完了，我们将等待并返回。
             */
                 if(completed){
                     try {
                         Thread.sleep(1000);
                     } catch (InterruptedException e) {
                         //什么也不做
                     }
                    return;
                 }
                 String str;
                 //创建reader
                 BufferedReader reader = new BufferedReader(fileReader);
                 try{
                     //读所有文本行
                    while((str = reader.readLine()) != null){
                     /**
                      * 按行发布一个新值
                      */
                         this.collector.emit(new Values(str),str);
                     }
                 }catch(Exception e){
                     throw new RuntimeException("Error reading tuple",e);
                 }finally{
                     completed = true;
                 }
             }
             /**
              * 我们将创建一个文件并维持一个collector对象
              */
             public void open(Map conf, TopologyContext context, SpoutOutputCollector collector) {
                     try {
                         this.context = context;
                         this.fileReader = new FileReader(conf.get("wordsFile").toString());
                     } catch (FileNotFoundException e) {
                         throw new RuntimeException("Error reading file ["+conf.get("wordFile")+"]");
                     }
                     this.collector = collector;
             }
             /**
              * 声明输入域"word"
              */
             public void declareOutputFields(OutputFieldsDeclarer declarer) {
                 declarer.declare(new Fields("line"));
             }
        }
```

第一个被调用的 spout 方法都是**public void open\(Map conf, TopologyContext context, SpoutOutputCollector collector**\)。它接收如下参数：配置对象，在定义topology 对象时创建；TopologyContext 对象，包含所有拓扑数据；还有SpoutOutputCollector 对象，它能让我们发布交给 bolts 处理的数据。下面的代码是这个方法的实现。

```
    public void open(Map conf, TopologyContext context, SpoutOutputCollector collector) {
        try {
            this.context = context;
            this.fileReader = new FileReader(conf.get("wordsFile").toString());
        } catch (FileNotFoundException e) {
            throw new RuntimeException("Error reading file ["+conf.get("wordFile")+"]");
        }
        this.collector = collector;
    }
```

我们在这个方法里创建了一个 FileReader 对象，用来读取文件。接下来我们要实现**public void nextTuple\(\)**，我们要通过它向 bolts 发布待处理的数据。在这个例子里，这个方法要读取文件并逐行发布数据。

```
    public void nextTuple() {
        if(completed){
            try {
                Thread.sleep(1);
            } catch (InterruptedException e) {
                //什么也不做
            }
            return;
        }
        String str;
        BufferedReader reader = new BufferedReader(fileReader);
        try{
            while((str = reader.readLine()) != null){
                this.collector.emit(new Values(str));
            }
        }catch(Exception e){
            throw new RuntimeException("Error reading tuple",e);
        }finally{
            completed = true;
        }
    }
```

**NOTE**: Values \(\)是一个 ArrarList 实现，它的元素就是传入构造器的参数。

**nextTuple\(\)会在同一个循环内被ack\(\)和fail\(\)周期性的调用。没有任务时它必须释放对线程的控制，其它方法才有机会得以执行。因此 nextTuple 的第一行就要检查是否已处理完成。如果完成了，为了降低处理器负载，会在返回前休眠一毫秒。如果任务完成了，文件中的每一行都已被读出并分发了。**

**NOTE**: 元组\(tuple\)是一个具名值列表，它可以是任意 java 对象（只要它是可序列化的）。默认情况，Storm 会序列化字符串、字节数组、ArrayList、HashMap 和 HashSet 等类型。

#### 2.2.3.2 Bolts

现在我们有了一个 spout，用来按行读取文件并每行发布一个_元组_，还要创建两个 bolts，用来处理它们（看图1-1）。_bolts_实现了接口**backtype.storm.topology.IRichBolt**。

_bolt_最重要的方法是**void execute\(Tuple input\)**，每次接收到元组时都会被调用一次，还会再发布若干个元组。

**NOTE**: 如果有必要，bolt 或 spout 会发布若干元组。当调用**nextTuple**或**execute**方法时，它们可能会发布0个、1个或许多个元组。

第一个 bolt，**WordNormalizer**，**负责得到并标准化每行文本。它把文本行切分成单词，大写转化成小写，去掉头尾空白符。**

首先我们要声明 bolt 的参数：

```
    public void declareOutputFields(OutputFieldsDeclarer declarer){
        declarer.declare(new Fields("word"));
    }
```

这里我们声明 bolt 将发布一个名为 “word” 的域。

下一步我们实现**public void execute\(Tuple input\)**，处理传入的元组：

```
    public void execute(Tuple input){
        String sentence=input.getString(0);
        String[] words=sentence.split(" ");
        for(String word : words){
            word=word.trim();
            if(!word.isEmpty()){
                word=word.toLowerCase();
                //发布这个单词
                collector.emit(new Values(word));
            }
        }
        //对元组做出应答
        collector.ack(input);
    }
```

第一行从元组读取值。值可以按位置或名称读取。接下来值被处理并用collector对象发布。最后，每次都调用collector 对象的**ack\(\)**方法确认已成功处理了一个元组。

例2-2是这个类的完整代码。

```
    //例2-2 src/main/java/bolts/WordNormalizer.java
    package bolts;
    import java.util.ArrayList;
    import java.util.List;
    import java.util.Map;
    import backtype.storm.task.OutputCollector;
    import backtype.storm.task.TopologyContext;
    import backtype.storm.topology.IRichBolt;
    import backtype.storm.topology.OutputFieldsDeclarer;
    import backtype.storm.tuple.Fields;
    import backtype.storm.tuple.Tuple;
    import backtype.storm.tuple.Values;

    public class WordNormalizer implements IRichBolt{
        private OutputCollector collector;
        public void cleanup(){}
        /**
          * *bolt*从单词文件接收到文本行，并标准化它。
          * 文本行会全部转化成小写，并切分它，从中得到所有单词。
          * 把文本行切分成单词，大写转化成小写，去掉头尾空白符
         */
        public void execute(Tuple input){
            String sentence = input.getString(0);
            String[] words = sentence.split(" ");
            for(String word : words){
                word = word.trim();
                if(!word.isEmpty()){
                    word=word.toLowerCase();
                    //发布这个单词
                    List a = new ArrayList();
                    a.add(input);
                    collector.emit(a,new Values(word));
                }
            }
            //对元组做出应答
            collector.ack(input);
        }
        public void prepare(Map stormConf, TopologyContext context, OutputCollector collector) {
            this.collector=collector;
        }

        /**
          * 这个*bolt*只会发布“word”域
          */
        public void declareOutputFields(OutputFieldsDeclarer declarer) {
            declarer.declare(new Fields("word"));
        }
    }
```

**NOTE**: 通过这个例子，我们了解了在一次**execute**调用中发布多个元组。如果这个方法在一次调用中接收到句子 “This is the Storm book”，它将会发布五个元组。

下一个_bolt_，**WordCounter**，负责为单词计数。这个拓扑结束时（**cleanup\(\)**方法被调用时），我们将显示每个单词的数量。

**NOTE**: 这个例子的 bolt 什么也没发布，它把数据保存在 map 里，但是在真实的场景中可以把数据保存到数据库。

```
package bolts;

import java.util.HashMap;
import java.util.Map;
import backtype.storm.task.OutputCollector;
import backtype.storm.task.TopologyContext;
import backtype.storm.topology.IRichBolt;
import backtype.storm.topology.OutputFieldsDeclarer;
import backtype.storm.tuple.Tuple;

public class WordCounter implements IRichBolt{
    Integer id;
    String name;
    Map<String,Integer> counters;
    private OutputCollector collector;

    /**
      * 这个spout结束时（集群关闭的时候），我们会显示单词数量
      */
    @Override
    public void cleanup(){
        System.out.println("-- 单词数 【"+name+"-"+id+"】 --");
        for(Map.Entry<String,Integer>entry : counters.entrySet()){
            System.out.println(entry.getKey()+": "+entry.getValue());
        }
    }

    /**
     *  为每个单词计数
     */
    @Override
    public void execute(Tuple input) {
        String str=input.getString(0);
        /**
         * 如果单词尚不存在于map，我们就创建一个，如果已在，我们就为它加1
         */
        if(!counters.containsKey(str)){
            counters.put(str,1);
        }else{
            Integer c = counters.get(str) + 1;
            counters.put(str,c);
        }
        //对元组作为应答
        collector.ack(input);
    }

    /**
     * 初始化
     */
    @Override
    public void prepare(Map stormConf, TopologyContext context, OutputCollector collector){
        this.counters = new HashMap<String, Integer>();
        this.collector = collector;
        this.name = context.getThisComponentId();
        this.id = context.getThisTaskId();
    }

    @Override
    public void declareOutputFields(OutputFieldsDeclarer declarer) {}
}
```

**execute 方法使用一个 map 收集单词并计数。拓扑结束时，将调用clearup\(\)方法打印计数器 map。（虽然这只是一个例子，但是通常情况下，当拓扑关闭时，你应当使用cleanup\(\)方法关闭活动的连接和其它资源。）**

### 2.2.4编写主类

你可以在主类中创建拓扑和一个本地集群对象，以便于在本地测试和调试。**LocalCluster**可以通过**Config**对象，让你尝试不同的集群配置。比如，当使用不同数量的工作进程测试你的拓扑时，如果不小心使用了某个全局变量或类变量，你就能够发现错误。

**NOTE**：**所有拓扑节点的各个进程必须能够独立运行，而不依赖共享数据（也就是没有全局变量或类变量），因为当拓扑运行在真实的集群环境时，这些进程可能会运行在不同的机器上。**

接下来，**TopologyBuilder**将用来创建拓扑，它决定 Storm 如何安排各节点，以及它们交换数据的方式。

```
    TopologyBuilder builder = new TopologyBuilder();
    builder.setSpout("word-reader", new WordReader());
    builder.setBolt("word-normalizer", new WordNormalizer()).shuffleGrouping("word-reader");
    builder.setBolt("word-counter", new WordCounter()).shuffleGrouping("word-normalizer");
```

**在**_**spout**_**和**_**bolts**_**之间通过shuffleGrouping方法连接。这种分组方式决定了 Storm 会以随机分配方式从源节点向目标节点发送消息。**

下一步，创建一个包含拓扑配置的**Config**对象，它会在运行时与集群配置合并，并通过prepare 方法发送给所有节点。

```
    Config conf = new Config();
    conf.put("wordsFile", args[0]);
    conf.setDebug(true);
```

由 spout 读取的文件的文件名，赋值给**wordFile**属性。由于是在开发阶段，设置**debug**属性为**true**，Strom 会打印节点间交换的所有消息，以及其它有助于理解拓扑运行方式的调试数据。

正如之前讲过的，你要用一个**LocalCluster**对象运行这个拓扑。在生产环境中，拓扑会持续运行，不过对于这个例子而言，你只要运行它几秒钟就能看到结果。

```
    LocalCluster cluster = new LocalCluster();
    cluster.submitTopology("Getting-Started-Topologie", conf, builder.createTopology());
    Thread.sleep(2000);
    cluster.shutdown();
```

调用**createTopology**和**submitTopology**，运行拓扑，休眠两秒钟（拓扑在另外的线程运行），然后关闭集群。

例2-3是完整的代码

```
    //例2-3 src/main/java/TopologyMain.java
    import spouts.WordReader;
    import backtype.storm.Config;
    import backtype.storm.LocalCluster;
    import backtype.storm.topology.TopologyBuilder;
    import backtype.storm.tuple.Fields;
    import bolts.WordCounter;
    import bolts.WordNormalizer;

    public class TopologyMain {
        public static void main(String[] args) throws InterruptedException {
        //定义拓扑
            TopologyBuilder builder = new TopologyBuilder();
            builder.setSpout("word-reader", new WordReader());
            builder.setBolt("word-normalizer", new WordNormalizer()).shuffleGrouping("word-reader");
            builder.setBolt("word-counter", new WordCounter(),2).fieldsGrouping("word-normalizer", new Fields("word"));

        //配置
            Config conf = new Config();
            conf.put("wordsFile", args[0]);
            conf.setDebug(false);

        //运行拓扑
            conf.put(Config.TOPOLOGY_MAX_SPOUT_PENDING, 1);
            LocalCluster cluster = new LocalCluster();
            cluster.submitTopology("Getting-Started-Topologie", conf, builder.createTopology());
            Thread.sleep(1000);
            cluster.shutdown();
        }
    }
```

观察运行情况

你已经为运行你的第一个拓扑准备好了。在这个目录下面创建一个文件，**/src/main/resources/words.txt**，一个单词一行，然后用下面的命令运行这个拓扑：**mvn exec:java -Dexec.mainClass=”TopologyMain” -Dexec.args=”src/main/resources/words.txt**。**举个例子，如果你的 words.txt 文件有如下内容：**Storm test are great is an Storm simple application but very powerful really Storm is great**你应该会在日志中看到类似下面的内容：**is: 2 application: 1 but: 1 great: 1 test: 1 simple: 1 Storm: 3 really: 1 are: 1 great: 1 an: 1 powerful: 1 very: 1**在这个例子中，每类节点只有一个实例。但是如果你有一个非常大的日志文件呢？你能够很轻松的改变系统中的节点数量实现并行工作。这个时候，你就要创建两个**WordCounter\*\* 实例。

```
    builder.setBolt("word-counter", new WordCounter(),2).shuffleGrouping("word-normalizer");
```

程序返回时，你将看到：**— 单词数 【word-counter-2】 — application: 1 is: 1 great: 1 are: 1 powerful: 1 Storm: 3 — 单词数 \[word-counter-3\] — really: 1 is: 1 but: 1 great: 1 test: 1 simple: 1 an: 1 very: 1**棒极了！修改并行度实在是太容易了（当然对于实际情况来说，每个实例都会运行在单独的机器上）。不过似乎有一个问题：单词 is 和 great 分别在每个**WordCounter**各计数一次。怎么会这样？当你调用**shuffleGrouping**时，就决定了 Storm 会以随机分配的方式向你的 bolt 实例发送消息。在这个例子中，理想的做法是相同的单词问题发送给同一个**WordCounter**实例。你把**shuffleGrouping\(“word-normalizer”\)**换成**fieldsGrouping\(“word-normalizer”, new Fields\(“word”\)\)**就能达到目的。试一试，重新运行程序，确认结果。 你将在后续章节学习更多分组方式和消息流类型。

## 2.3结论 {#54bbba803f13eaab0f5441d97b13247a}

我们已经讨论了 Storm 的本地和远程操作模式之间的不同，以及 Storm 的强大和易于开发的特性。你也学习了一些 Storm 的基本概念，我们将在后续章节深入讲解它们。

# 3 Storm Topology（拓扑）

在这一章，你将学到如何在同一个 Storm 拓扑结构内的不同组件之间传递元组，以及如何向一个运行中的 Storm 集群发布一个拓扑。

## 3.1 数据流组 {#550eaba880d75e580686db4b61d3cdde}

设计一个拓扑时，你要做的最重要的事情之一就是定义如何在各组件之间交换数据（数据流是如何被_bolts_消费的）。**一个**_**数据流组**_**指定了每个**_**bolt**_**会消费哪些数据流，以及如何消费它们。**

**NOTE**：一个节点能够发布一个以上的数据流，一个数据流组允许我们选择接收哪个。

数据流组在定义拓扑时设置，就像我们在第2 章看到的：

```
···
    builder.setBolt("word-normalizer", new WordNormalizer())
           .shuffleGrouping("word-reader");
···
```

在前面的代码块里，一个 bolt 由**TopologyBuilder**对象设定， 然后使用随机数据流组指定数据源。**数据流组通常将数据源组件的 ID 作为参数，取决于数据流组的类型不同还有其它可选参数。**

**NOTE**：每个**InputDeclarer**可以有一个以上的数据源，而且每个数据源可以分到不同的组。具体的数据流组的分类如下：

### 3.1.1 随机数据流组

随机流组是最常用的数据流组。它只有一个参数（数据源组件），并且数据源会向随机选择的 bolt 发送元组，保证每个消费者收到近似数量的元组。

随机数据流组用于数学计算这样的原子操作。然而，如果操作不能被随机分配，就像第二章为单词计数的例子，你就要考虑其它分组方式了。

### 3.1.2 域数据流组

域数据流组允许你基于元组的一个或多个域控制如何把元组发送给_bolts_。 它保证拥有相同域组合的值集发送给同一个_bolt_。 回到单词计数器的例子，如果你用_word_域为数据流分组，**word-normalizer **bolt 将只会把相同单词的元组发送给同一个**word-counter**bolt 实例。

```
···
    builder.setBolt("word-counter", new WordCounter(),2)
           .fieldsGrouping("word-normalizer", new Fields("word"));
···
```

**NOTE**: **在域数据流组中的所有域集合必须存在于数据源的域声明中。**

### 3.1.3 全部数据流组

**全部数据流组，为每个接收数据的实例复制一份元组副本。这种分组方式用于向 bolts 发送信号。**比如，你要刷新缓存，你可以向所有的 bolts 发送一个刷新缓存信号。在单词计数器的例子里，你可以使用一个全部数据流组，添加清除计数器缓存的功能（见[拓扑示例](https://github.com/storm-book/examples-ch03-topologies)）

```
    public void execute(Tuple input) {
        String str = null;
        try{
            if(input.getSourceStreamId().equals("signals")){
                str = input.getStringByField("action");
                if("refreshCache".equals(str))
                    counters.clear();
            }
        }catch (IllegalArgumentException e){
            //什么也不做
        }
        ···
    }
```

我们添加了一个 if 分支，用来检查源数据流。 Storm 允许我们声明具名数据流（如果你不把元组发送到一个具名数据流，默认发送到名为 ”**default**“ 的数据流）。这是一个识别元组的极好的方式，就像这\_个例子中，我们想识别**signals**一样。 在拓扑定义中，你要向**word-counter **\_bolt 添加第二个数据流，用来接收从**signals-spout**数据流发送到所有 bolt 实例的每一个元组。

```
    builder.setBolt("word-counter", new WordCounter(),2)
           .fieldsGroupint("word-normalizer",new Fields("word"))
           .allGrouping("signals-spout","signals");
```

**signals-spout**的实现请参考[git仓库](https://github.com/storm-book/examples-ch03-topologies)。

### 3.1.4 自定义数据流组

你**可以通过实现backtype.storm.grouping.CustormStreamGrouping接口创建自定义数据流组，让你自己决定哪些**_**bolt**_**接收哪些元组。**

让我们修改单词计数器示例，使首字母相同的单词由同一个**bolt**接收。

```
    public class ModuleGrouping implements CustormStreamGrouping, Serializable{
        int numTasks = 0;

        @Override
        public List<Integer> chooseTasks(List<Object> values) {
            List<Integer> boltIds = new ArrayList<Integer>();
            if(values.size()>0){
                String str = values.get(0).toString();
                if(str.isEmpty()){
                    boltIds.add(0);
                }else{
                    boltIds.add(str.charAt(0) % numTasks);
                }
            }
            return boltIds;
        }

        @Override
        public void prepare(TopologyContext context, Fields outFields, List<Integer> targetTasks) {
            numTasks = targetTasks.size();
        }
    }
```

这是一个**CustomStreamGrouping**的简单实现，在这里我们采用单词首字母字符的整数值与任务数的余数，决定接收元组的 bolt。

按下述方式**word-normalizer**修改即可使用这个自定义数据流组。

```
    builder.setBolt("word-normalizer", new WordNormalizer())
           .customGrouping("word-reader", new ModuleGrouping());
```

### 3.1.5 直接数据流组

**这是一个特殊的数据流组，数据源可以用它决定哪个组件接收元组。**与前面的例子类似，数据源将根据单词首字母决定由哪个_bolt_接收元组。要使用直接数据流组，在**WordNormalizer **_bolt_中，使用**emitDirect**方法代替**emit**。

```
    public void execute(Tuple input) {
        ...
        for(String word : words){
            if(!word.isEmpty()){
                ...
                collector.emitDirect(getWordCountIndex(word),new Values(word));
            }
        }
        //对元组做出应答
        collector.ack(input);
    }

    public Integer getWordCountIndex(String word) {
        word = word.trim().toUpperCase();
        if(word.isEmpty()){
            return 0;
        }else{
            return word.charAt(0) % numCounterTasks;
        }
    }
```

在**prepare**方法中计算任务数

```
    public void prepare(Map stormConf, TopologyContext context, 
                OutputCollector collector) {
        this.collector = collector;
        this.numCounterTasks = context.getComponentTasks("word-counter");
    }
```

在拓扑定义中指定数据流将被直接分组：

```
    builder.setBolt("word-counter", new WordCounter(),2)
           .directGrouping("word-normalizer");
```

### 3.1.6 全局数据流组

**全局数据流组把所有数据源创建的元组发送给单一目标实例（即拥有最低 ID 的任务）。**

## 3.2 不分组 {#c84edd03a9d724c0e0ca8599e128362b}

写作本书时（Stom0.7.1 版），这个数据流组相当于随机数据流组。也就是说，使用这个数据流组时，并不关心数据流是如何分组的。

## 3.3 LocalCluster VS StormSubmitter {#6d8dcc4853ff1d930fe4217b92bf25f8}

到目前为止，你已经用一个叫做**LocalCluster**的工具在你的本地机器上运行了一个拓扑。**Storm 的基础工具，使你能够在自己的计算机上方便的运行和调试不同的拓扑。但是你怎么把自己的拓扑提交给运行中的 Storm 集群呢？Storm 有一个有趣的功能，在一个真实的集群上运行自己的拓扑是很容易的事情。要实现这一点，你需要把LocalCluster换成StormSubmitter并实现submitTopology方法， 它负责把拓扑发送给集群。**

下面是修改后的代码：

```
    //LocalCluster cluster = new LocalCluster();
    //cluster.submitTopology("Count-Word-Topology-With-Refresh-Cache", conf, 
    //builder.createTopology());
    StormSubmitter.submitTopology("Count-Word-Topology-With_Refresh-Cache", conf,
            builder.createTopology());
    //Thread.sleep(1000);
    //cluster.shutdown();
```

**NOTE**: 当你使用**StormSubmitter**时，你就不能像使用**LocalCluster**时一样通过代码控制集群了。

接下来，把源码压缩成一个 jar 包，运行 Storm 客户端命令，把拓扑提交给集群。如果你已经使用了 Maven， 你只需要在命令行进入源码目录运行：**mvn package**。

现在你生成了一个 jar 包，使用**storm jar**命令提交拓扑（关于如何安装 Storm 客户端请参考附录 A ）。命令格式：**storm jar allmycode.jar org.me.MyTopology arg1 arg2 arg3**。

对于这个例子，在拓扑工程目录下面运行：

```
storm jar target/Topologies-0.0.1-SNAPSHOT.jar countword.TopologyMain src/main/resources/words.txt
```

通过这些命令，你就把拓扑发布集群上了。

如果想停止或杀死它，运行：

```
storm kill Count-Word-Topology-With-Refresh-Cache
```

**NOTE**：拓扑名称必须保证惟一性。

**NOTE**：如何安装Storm客户端，参考附录A

## 3.4 DRPC 拓扑 {#91c11fb7a22712de17c2cbb69a53cbf1}

有一种特殊的拓扑类型叫做分布式远程过程调用（DRPC），它利用 Storm 的分布式特性执行远程过程调用（RPC）（见下图）。Storm 提供了一些用来实现 DRPC 的工具。第一个是 DRPC 服务器，它就像是客户端和 Storm 拓扑之间的连接器，作为拓扑的_spout_的数据源。它接收一个待执行的函数和函数参数，然后对于函数操作的每一个数据块，这个服务器都会通过拓扑分配一个请求 ID 用来识别 RPC 请求。拓扑执行最后的 bolt 时，它必须分配 RPC 请求 ID 和结果，使 DRPC 服务器把结果返回正确的客户端。

![](http://wiki.jikexueyuan.com/project/storm/images/04.png)

**NOTE**：单实例 DRPC 服务器能够执行许多函数。每个函数由一个惟一的名称标识。

Storm 提供的第二个工具（已在例子中用过）是 LineDRPCTopologyBuilder**\*\*，一个辅助构建DRPC 拓扑的抽象概念。生成的拓扑创建**DRPCSpouts**——它连接到 DRPC 服务器并向拓扑的其它部分分发数据——并包装**_**bolts**_**，使结果从最后一个**_**bolt**_**返回。依次执行所有添加到**LinearDRPCTopologyBuilder\*\_对象的\_bolts\*。

作为这种类型的拓扑的一个例子，我们创建了一个执行加法运算的进程。虽然这是一个简单的例子，但是这个概念可以扩展到复杂的分布式计算。

bolt 按下面的方式声明输出：

```
    public void declareOutputFields(OutputFieldsDeclarer declarer) {
        declarer.declare(new Fields("id","result"));
    }
```

因为这是拓扑中惟一的 bolt，它必须发布 RPC  ID 和结果。**execute方法负责执行加法运算**。

```
    public void execute(Tuple input) {
        String[] numbers = input.getString(1).split("\\+");
        Integer added = 0;
        if(numbers.length < 2){
            throw new InvalidParameterException("Should be at least 2 numbers");
        }
        for(String num : numbers){
            added += Integer.parseInt(num);
        }
        collector.emit(new Values(input.getValue(0),added));
    }
```

包含加法 bolt 的拓扑定义如下：

```
    public static void main(String[] args) {
        LocalDRPC drpc = new LocalDRPC();

        LinearDRPCTopologyBuilder builder = new LinearDRPCTopologyBuilder("add");
        builder.addBolt(AdderBolt(),2);

        Config conf = new Config();
        conf.setDebug(true);

        LocalCluster cluster = new LocalCluster();
        cluster.submitTopology("drpcder-topology", conf,
            builder.createLocalTopology(drpc));
        String result = drpc.execute("add", "1+-1");
        checkResult(result,0);
        result = drpc.execute("add", "1+1+5+10");
        checkResult(result,17);

        cluster.shutdown();
        drpc.shutdown();
    }
```

创建一个**LocalDRPC**对象在本地运行 DRPC 服务器。接下来，创建一个拓扑构建器（译者注：LineDRpctopologyBuilder 对象），把 bolt 添加到拓扑。运行 DRPC 对象（LocalDRPC 对象）的**execute**方法测试拓扑。

**NOTE**：使用**DRPCClient**类连接远程 DRPC 服务器。DRPC 服务器暴露了[Thrift API](http://thrift.apache.org/)，因此可以跨语言编程；并且不论是在本地还是在远程运行DRPC服务器，它们的 API 都是相同的。 对于采用 Storm 配置的 DRPC 配置参数的 Storm 集群，调用构建器对象的**createRemoteTopology**向 Storm 集群提交一个拓扑，而不是调用**createLocalTopology**。

# 4 Storm Spouts

2016-08-12

[https://www.w3cschool.cn/storm/jur81jzk.html](https://www.w3cschool.cn/storm/jur81jzk.html)

你将在本章了解到_spout_作为拓扑入口和它的容错机制相关的最常见的设计策略。

## 4.1 可靠的消息 VS 不可靠的消息

在设计拓扑结构时，始终在头脑中记着的一件重要事情就是消息的可靠性。当有无法处理的消息时，你就要决定该怎么办，以及作为一个整体的拓扑结构该做些什么。举个例子，在处理银行存款时，不要丢失任何事务报文就是很重要的事情。但是如果你要统计分析数以百万的 tweeter 消息，即使有一条丢失了，仍然可以认为你的结果是准确的。

对于 Storm 来说，根据每个拓扑的需要担保消息的可靠性是开发者的责任。这就涉及到消息可靠性和资源消耗之间的权衡。高可靠性的拓扑必须管理丢失的消息，必然消耗更多资源；可靠性较低的拓扑可能会丢失一些消息，占用的资源也相应更少。不论选择什么样的可靠性策略，Storm 都提供了不同的工具来实现它。

要在 spout 中管理可靠性，你可以在分发时包含一个元组的消息 ID（**collector.emit\(new Values\(…\),tupleId\)**）。在一个元组被正确的处理时调用 ack**\*\* 方法，而在失败时调用**fail\*\* 方法。当一个元组被所有的靶 bolt 和锚 bolt 处理过，即可判定元组处理成功（你将在第5章学到更多锚 bolt 知识）。

发生下列情况之一时为元组处理失败：

* 提供数据的 spout 调用
  **collector.fail\(tuple\)**
* 处理时间超过配置的超时时间

让我们来看一个例子。想象你正在处理银行事务，需求如下：

* 如果事务失败了，重新发送消息
* 如果失败了太多次，终结拓扑运行

创建一个 spout 和一个 bolt，spout 随机发送100个事务 ID，有80%的元组不会被 bolt 收到（你可以在[例子 ch04-spout](https://github.com/storm-book/examples-ch04-spouts/)查看完整代码）。实现 spout 时利用**Map**分发事务消息元组，这样就比较容易实现重发消息。

```
 public void nextTuple() {
    if(!toSend.isEmpty()){
        for(Map.Entry<Integer, String> transactionEntry : toSend.entrySet()){
            Integer transactionId = transactionEntry.getKey();
            String transactionMessage = transactionEntry.getValue();
            collector.emit(new Values(transactionMessage),transactionId);
        }
        toSend.clear();
    }
}
```

如果有未发送的消息，得到每条事务消息和它的关联 ID，把它们作为一个元组发送出去，最后清空消息队列。值得一提的是，调用 map 的**clear**是安全的，因为**nextTuple**失败时，只有**ack**方法会修改 map，而它们都运行在一个线程内。

维护两个 map 用来跟踪待发送的事务消息和每个事务的失败次数。**ack**方法只是简单的把事务从每个列表中删除。

```
public void ack(Object msgId)
{
    messages.remove(msgId);
    failCounterMessages.remove(msgId);
}
```

**fail**方法决定应该重新发送一条消息，还是已经失败太多次而放弃它。

**NOTE**: 如果你使用**全部数据流组**，而拓扑里的所有 bolt 都失败了，spout 的**fail**方法才会被调用。

```
public void fail(Object msgId) {
    Integer transactionId = (Integer) msgId;
    //检查事务失败次数
    Integer failures = transactionFailureCount.get(transactionId) + 1;

    if(failes >= MAX_FAILS){
        //失败数太高了，终止拓扑
        throw new RuntimeException("错误, transaction id 【"+

         transactionId+"】 已失败太多次了 【"+failures+"】");
    }

    //失败次数没有达到最大数，保存这个数字并重发此消息
    transactionFailureCount.put(transactionId, failures);
    toSend.put(transactionId, messages.get(transactionId));
    LOG.info("重发消息【"+msgId+"】");
}
```

首先，检查事务失败次数。如果一个事务失败次数太多，通过抛出**RuntimeException**终止发送此条消息的工作。否则，保存失败次数，并把消息放入待发送队列（**toSend**），它就会再次调用**nextTuple**时得以重新发送。

**NOTE**: Storm 节点不维护状态，因此如果你在内存保存信息（就像本例做的那样），而节点又不幸挂了，你就会丢失所有缓存的消息。Storm 是一个快速失败的系统。拓扑会在抛出异常时挂掉，然后再由 Storm 重启，恢复到抛出异常前的状态。

## 4.2 获取数据

接下来你会了解到一些设计 spout 的技巧，帮助你从多数据源获取数据。

### 4.2.1 直接连接

在一个直接连接的架构中，spout 直接与一个消息分发器连接。

![](https://img.w3cschool.cn/attachments/image/wk/storm/05.png)

图 直接连接的 spout

这个架构很容易实现，尤其是在消息分发器是已知设备或已知设备组时。**已知设备满足：拓扑从启动时就已知道该设备，并贯穿拓扑的整个生命周期保持不变。未知设备就是在拓扑运行期添加进来的。已知设备组就是从拓扑启动时组内所有设备都是已知的。**

下面举个例子说明这一点。创建一个 spout 使用[Twitter 流 API](https://github.com/storm-book/examples-ch04-spouts/)读取 twitter 数据流。spout 把 API 当作消息分发器直接连接。从数据流中得到符合 track 参数的公共 tweets（参考 twitter 开发页面）。完整的例子可以在链接[https://github.com/storm-book/examples-ch04-spouts/](https://github.com/storm-book/examples-ch04-spouts/)找到。

spout 从配置对象得到连接参数（track，user，password\)，并连接到 API（在这个例子中使用[Apache](http://apache.org/)的[DefaultHttpClient](http://hc.apache.org/httpcomponents-client-ga/httpclient/apidocs/org/apache/http/impl/client/DefaultHttpClient.html)）。它一次读一行数据，并把数据从 JSON 转化成 Java 对象，然后发布它。

```
public void nextTuple() {
    //创建http客户端
    client = new DefaultHttpClient();
    client.setCredentialsProvider(credentialProvider);
    HttpGet get = new HttpGet(STREAMING_API_URL+track);
    HttpResponse response;
    try {
        //执行http访问
        response = client.execute(get);
        StatusLine status = response.getStatusLine();
        if(status.getStatusCode() == 200){
            InputStream inputStream = response.getEntity().getContent();
            BufferedReader reader = new BufferedReader(new InputStreamReader(inputStream));
            String in;
            //逐行读取数据
            while((in = reader.readLine())!=null){
                try{
                    //转化并发布消息
                    Object json = jsonParser.parse(in);
                    collector.emit(new Values(track,json));
                }catch (ParseException e) {
                    LOG.error("Error parsing message from twitter",e);
                }
            }
        }
    } catch (IOException e) {
        LOG.error("Error in communication with twitter api ["+get.getURI().toString()+"], 
           sleeping 10s");
        try {
            Thread.sleep(10000);
        } catch (InterruptedException e1) {}
    }
}
```

**NOTE**:在这里你锁定了**nextTuple**方法，所以你永远也不会执行 ack**\*\* 和**fail\*\* 方法。在真实的应用中，我们**推荐你在一个单独的线程中执行锁定，并维持一个内部队列用来交换数据**（你会在下一个例子中学到如何实现这一点：消息队列）。

棒极了！现在你用一个 spout 读取 Twitter 数据。一个明智的做法是，采用拓扑并行化，多个 spout 从同一个流读取数据的不同部分。那么如果你有多个流要读取，你该怎么做呢？Storm 的第二个有趣的特性（译者注：第一个有趣的特性已经出现过，这句话原文都是一样的，不过按照中文的行文习惯还是不重复使用措词了）是，你可以在任意组件内（spouts/bolts）访问**TopologyContext**。利用这一特性，你能够把流划分到多个 spouts 读取。

```
public void open(Map conf, TopologyContext context,
          SpoutOutputCollector collector) {
    //从context对象获取spout大小
    int spoutsSize = 
context.getComponentTasks(context.getThisComponentId()).size();
    //从这个spout得到任务id
    int myIdx = context.getThisTaskIndex();
    String[] tracks = ((String) conf.get("track")).split(",");
    StringBuffer tracksBuffer = new StringBuffer();
    for(int i=0; i< tracks.length;i++){
        //Check if this spout must read the track word
        if( i % spoutsSize == myIdx){
            tracksBuffer.append(",");
            tracksBuffer.append(tracks[i]);
        }
    }
    if(tracksBuffer.length() == 0) {
        throw new RuntimeException("没有为spout得到track配置" +
 " [spouts大小:"+spoutsSize+", tracks:"+tracks.length+"] tracks的数量必须高于spout的数量");
 this.track =tracksBuffer.substring(1).toString();
    }
 ...
```

利用这一技巧，你可以把 collector 对象均匀的分配给多个数据源，当然也可以应用到其它的情形。比如说，从web服务器收集日志文件

![](https://img.w3cschool.cn/attachments/image/wk/storm/05.png)

图 直连 hash

通过上一个例子，你学会了从一个 spout 连接到已知设备。你也可以使用相同的方法连接未知设备，不过这时你需要借助于一个协同系统维护的设备列表。协同系统负责探察列表的变化，并根据变化创建或销毁连接。比如，从 web 服务器收集日志文件时，web 服务器列表可能随着时间变化。当添加一台 web 服务器时，协同系统探查到变化并为它创建一个新的 spout。

![](https://img.w3cschool.cn/attachments/image/wk/storm/06.png)

图 直连协同

### 4.2.2 消息队列

第二种方法是，通过一个队列系统接收来自消息分发器的消息，并把消息转发给 spout。更进一步的做法是，把队列系统作为 spout 和数据源之间的中间件，在许多情况下，你可以利用多队列系统的重播能力增强队列可靠性。这意味着你不需要知道有关消息分发器的任何事情，而且添加或移除分发器的操作比直接连接简单的多。这个架构的问题在于队列是一个故障点，另外你还要为处理流程引入新的环节。

下图展示了这一架构模型

![](https://img.w3cschool.cn/attachments/image/wk/storm/07.png)

图 使用队列系统

**NOTE**: 你可以**通过轮询队列或哈希队列（把队列消息通过哈希发送给 spouts 或创建多个队列使队列 spouts 一一对应）在多个 spouts 之间实现并行性。**

接下来我们利用 Redis [http://redis.io/](http://redis.io/)和它的 java 库[Jedis](https://github.com/xetorthio/jedis)创建一个队列系统。在这个例子中，我们创建一个日志处理器从一个未知的来源收集日志，利用**lpush**命令把消息插入队列，利用 blpop 命令等待消息。如果你有很多处理过程，**blpop**命令采用了轮询方式获取消息。

我们在 spout 的 open**\*\* 方法创建一个线程，用来获取消息（使用线程是为了避免锁定**nextTuple\*\* 在主循环的调用）：

```
new Thread(new Runnable() {
    @Override
    public void run() {
        try{
           Jedis client= new Jedis(redisHost, redisPort);
           List res = client.blpop(Integer.MAX_VALUE, queues);
           messages.offer(res.get(1));
        }catch(Exception e){
            LOG.error("从redis读取队列出错",e);
            try {
                Thread.sleep(100);
            }catch(InterruptedException e1){}
        }
    }
}).start();
```

**这个线程的惟一目的就是，创建 redis 连接，然后执行blpop命令。每当收到了一个消息，它就被添加到一个内部消息队列，然后会被 nextTuple\*\*\*\* 消费。对于 spout 来说数据源就是 redis 队列，它不知道消息分发者在哪里也不知道消息的数量。**

**NOTE**: 我们**不推荐你在 spout 创建太多线程，因为每个 spout 都运行在不同的线程。一个更好的替代方案是增加拓扑并行性，也就是通过 Storm 集群在分布式环境创建更多线程。**

在**nextTupl**e 方法中，要做的惟一的事情就是从内部消息队列获取消息并再次分发它们。

```
public void nextTuple(){
    while(!messages.isEmpty()){
        collector.emit(new Values(messages.poll()));
    }
}
```

**NOTE**: 你还可以借助 redis 在 spout 实现消息重发，从而实现可靠的拓扑。（译者注：这里是相对于开头的**可靠的消息VS不可靠的消息**讲的）

### 4.2.3 DRPC

DRPCSpout从DRPC 服务器接收一个函数调用，并执行它（见第三章的例子）。对于最常见的情况，使用[backtype.storm.drpc.DRPCSpout](http://nathanmarz.github.io/storm/doc/backtype/storm/drpc/DRPCSpout.html)就足够了，不过仍然有可能利用 Storm 包内的DRPC类创建自己的实现。

## 4.3 小结

现在你已经学习了常见的spout实现模式，它们的优势，以及如何确保消息可靠性。**不存在适用于所有拓扑的架构模式。如果你知道数据源，并且能够控制它们，你就可以使用直接连接；然而如果你需要添加未知数据源或从多种数据源接收数据，就最好使用消息队列。如果你要执行在线过程，你可以使用 DRPCSpout 或类似的实现。**

你已经学习了三种常见连接方式，不过依赖于你的需求仍然有无限的可能。



# 5 Storm Bolts

2016-08-12

[https://www.w3cschool.cn/storm/kas11jzn.html](https://www.w3cschool.cn/storm/kas11jzn.html)



正如你已经看到的，bolts 是一个 Storm 集群中的关键组件。你将在这一章学到 bolt 生命周期，一些 bolt 设计策略，以及几个有关这些内容的例子。

## 5.1 Bolt 生命周期

Bolt 是这样一种组件，它把元组作为输入，然后产生新的元组作为输出。实现一个 bolt 时，通常需要实现**IRichBolt**接口。Bolts 对象由客户端机器创建，序列化为拓扑，并提交给集群中的主机。然后集群启动工人进程反序列化 bolt，调用 prepare\*\*\*\*，最后开始处理元组。

**NOTE**:要创建一个 bolt 对象，它通过构造器参数初始化成员属性，bolt 被提交到集群时，这些属性值会随着一起序列化。

## 5.2 Bolt 结构

Bolts拥有如下方法：

```
declareOutputFields(OutputFieldsDeclarer declarer)//为bolt声明输出模式

prepare(java.util.Map stormConf, TopologyContext context, OutputCollector collector)//仅在bolt开始处理元组之前调用

execute(Tuple input)//处理输入的单个元组

cleanup()//在bolt即将关闭时调用  
```

下面看一个例子，在这个例子中 bolt 把一句话分割成单词列表：

```
class SplitSentence implements IRichBolt {
    private OutputCollector collector;
    public void prepare(Map conf, TopologyContext context, OutputCollector collector) {
        this.collector = collector;
    }

    public void execute(Tuple tuple) {
        String sentence = tuple.getString(0);
        for(String word : sentence.split(" ")) {
            collector.emit(new Values(word));
        }
    }

    public void cleanup(){}

    public void declareOutputFields(OutputFieldsDeclarer declarer) {
        declarer.declare(new Fields("word"));
    }
}  
```

正如你所看到的，这是一个很简单的 bolt。值得一提的是**在这个例子里，没有消息担保。这就意味着，如果 bolt 因为某些原因丢弃了一些消息——不论是因为 bolt 挂了，还是因为程序故意丢弃的——生成这条消息的 spout 不会收到任何通知，任何其它的 spouts 和 bolts 也不会收到。**

然而在许多情况下，你要确保消息在整个拓扑范围内都被处理过了。

## 5.3 可靠的 bolts 和不可靠的 bolts

正如前面所说的，Storm 保证通过 spout 发送的每条消息会得到所有 bolt 的全面处理。基于设计上的考虑，这意味着你要自己决定你的 bolts 是否保证这一点。

拓扑是一个树型结构，消息（元组）穿过其中一条或多条分支。树上的每个节点都会调用**ack\(tuple\)**或**fail\(tuple\)**，Storm 因此知道一条消息是否失败了，并通知那个/那些制造了这些消息的 spout\(s\)。**既然一个 Storm 拓扑运行在高度并行化的环境里，跟踪始发 spout 实例的最好方法就是在消息元组内包含一个始发 spout 引用。这一技巧称做**_**锚定**_\(译者注：原文为Anchoring\)。修改一下刚刚讲过的 SplitSentence，使它能够确保消息都被处理了。

```
class SplitSentence implenents IRichBolt {
    private OutputCollector collector;

    public void prepare(Map conf, TopologyContext context, OutputCollector collector) {
        this.collector = collector;
    }

    public void execute(Tuple tuple) {
        String sentence = tuple.getString(0);
        for(String word : sentence.split(" ")) {
            collector.emit(tuple, new Values(word));
        }
        collector.ack(tuple);
    }

    public void cleanup(){}

    public void declareOutputFields(OutputFieldsDeclarer declarer){
        declar.declare(new Fields("word"));
    }
}  
```

**锚定发生在调用collector.emit\(\)时。**正如前面提到的，**Storm 可以沿着元组追踪到始发spout。collector.ack\(tuple\)和collector.fail\(tuple\)会告知 spout 每条消息都发生了什么。**当树上的每条消息都已被处理了，Storm 就认为来自 spout 的元组被全面的处理了。**如果一个元组没有在设置的超时时间内完成对消息树的处理，就认为这个元组处理失败。默认超时时间为30秒**。

NOTE: 你可以通过修改Config.TOPOLOGY\_MESSAGE\_TIMEOUT修改拓扑的超时时间。

当然了spout需要考虑消息的失败情况，并相应的重试或丢弃消息。

**NOTE**: 你处理的每条消息要么是确认的（译者注：collector.ack\(\)）要么是失败的（译者注：collector.fail\(\)）。**Storm 使用内存跟踪每个元组，所以如果你不调用这两个方法，该任务最终将耗尽内存。**

## 5.4 多数据流

一个 bolt 可以使用**emit\(streamId, tuple\)**把元组分发到多个流，其中参数**streamId**是一个用来标识流的字符串。然后，你可以在**TopologyBuilder**决定由哪个流订阅它。

## 5.5 多锚定

为了用 bolt 连接或聚合数据流，你需要借助内存缓冲元组。为了在这一场景下确保消息完成，你不得不把流锚定到多个元组上。可以向**emit**方法传入一个元组列表来达成目的。

```
...
List anchors = new ArrayList();
anchors.add(tuple1);
anchors.add(tuple2);
collector.emit(anchors, values);
...  
```

**通过这种方式，bolt 在任意时刻调用ack或fail方法，都会通知消息树，而且由于流锚定了多个元组，所有相关的 spout 都会收到通知。**

## 5.6 使用 IBasicBolt 自动确认

你可能已经注意到了，在许多情况下都需要消息确认。简单起见，Storm 提供了另一个用来实现bolt 的接口，IBasicBolt。对于该接口的实现类的对象，会在执行**execute**方法之后自动调用**ack**方法。

```
class SplitSentence extends BaseBasicBolt {
    public void execute(Tuple tuple, BasicOutputCollector collector) {
        String sentence = tuple.getString(0);
        for(String word : sentence.split(" ")) {
            collector.emit(new Values(word));
        }
    }

    public void declareOutputFields(OutputFieldsDeclarer declarer) {
        declarer.declare(new Fields("word"));
    }
}  
```

**NOTE**: **分发消息的 BasicOutputCollector 自动锚定到作为参数传入的元组**。

