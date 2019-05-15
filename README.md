# Prometheus_Grafana_Consul
experience in using Prometheus, Grafana and Consul

       公司需要发布一套云环境产品，因此需要选择一套框架完成服务发现、服务监控、服务预警的全过程。
       经过比对，决定选择Prometheus和Consul作为核心组件，Grafana、AlertManager和其他组件，完成全部过程。其完成结构图如下：
       


       先说一下Prometheus监控体系的问题：
       1、结构比较复杂
       组件非常多，从图中可以看到，整个结构组件几十上百个。
       结构上的复杂，也带来了部署上的困难。这个后面讲部署的时候再详细说。
       另外就是结构的复杂还带来了维护的困难，因为组件分散在各处（集中存放的话，同服务端口配置会很麻烦），需要时刻维护一个结构图，以保证整个结构的可维护性。
       2、prometheus的集群稳定性
       包括prometheus和alertmanager组件在内，都存在集群可用性的问题。
       prometheus官方给出的解决方案，要么是不集群，即各服务均单独维护，或者使用统一的数据库，以保证整体的集群数据一致性。前者显然不能保证数据一致性，后者带来的配置复杂性又会成倍提高（配置内置数据库还是外置数据库也同样是个问题），给运维（架构）同学会带来持续的压力。
        alertmanger在官方文档中，其提法是在某个版本后，只需要如下配置即可：


       但是然并卵，经过多次测试，这种配置，同一个信息一定会被多个接收端同时接收到，并不是同一个告警信息所有alertmanager实例只发送一次。必须使用原有的参数配置，即：


       才能完成alertmanager的同步，但是又然并卵，某些情况下仍然存在问题，后面再说。
       另外就是prometheus的默认数据库，官方的说法是：
          Note that a limitation of the local storage is that it is not clustered or replicated. Thus, it is not arbitrarily scalable or durable in the face of disk or node outages and should thus be treated as more of an ephemeral sliding window of recent data. However, if your durability requirements are not strict, you may still succeed in storing up to years of data in the local storage.
          简单说，就是官方自己也不推荐你强依赖这套数据库，除非你的持久化需求不太强烈。也就是说如果你的环境压力比较大，数据量很大，持久性和一致性要求比较高，那么抱歉，你换一个数据库吧，比如infuxDB。
       3、exporter组件来源不统一
       因为prometheus只是给出了数据聚集的解决方案以及存储和查询的解决方案，其本身并不负责从各终端拉取状态服务信息，服务信息的拉取是依靠不同的exporter完成的。而其官方又没有提供全部的exporter（严格说，基本就没几个exporter是官方提供的），这就带来了几个问题。
       首先是exporter需要从不同的渠道进行收集，最大的来源是prometheus文档和github，也存在其他的渠道，所以第一是需要你自己去找exporter，其次是你得选择一个合适的来用（可能找到多个，当然也有可能找不到）。
       其次是标准不同，因为这些都是社区贡献，而始作俑者只是给出了答案的格式而没有约定部署方式，所以各个exporter的部署方式都不一样，需要逐个摸索熟悉和了解，甚至有些连文档都没有。所以工作的最后一步一定是把启动命令用脚本记录下来，并且归档保存以备后用。
       最后是不同的使用方式导致集成的方式也不同，大部分exporter是独立运行的，但也有部分exporter是跟其他服务嵌套在一起使用的，其使用复杂度就更高了，这就导致其维护难度更高，典型的例如jmx插件以及nginx的vts插件，都是典型的嵌套组件，一旦运行环境发生变化，又需要持续进行维护。
       简而言之，整个exporter组件就是难查询、难部署、难维护，是维护是非常耗时耗力的过程。
       4、监控信息需要定制
       标准输出的监控信息更多的普遍化的信息，如果需要更个性化的内容，就需要自己定制开发了，所以这部分工作是无法省去的。例如java后台程序的监控是由插件完成的，但如果你希望收集更详细的，例如500接口的时间和次数等信息，就必须自己开发监控内容以及图标信息了。
       5、集成的匹配度不高
       这点主要表现在consul和prometheus之间的集成，本来是希望自动完美集成，但是实际用到才发现，很多地方是需要二次配置的，最典型的就是如何保证consul自己不进入到prometheus的集成监控列表中，如何调整对应的标签信息等，这种几乎做匹配就需要做处理的地方，官方却没有给出任何的最佳实践的文档，不得不说是一种缺陷。

       上面基本聊完了这套架构的缺点，再说说优点吧。
       1、全部开源
       这点当然是最重要的，我们几乎可以在社区找到我们一般需求的全部组件，不需要自己再次开发，对于成本压力比较大的公司而言，这当然是非常好的选择。
       2、开箱即用
       目前的组件除了极个别需要你下载自己编译之外，所有的组件都是开箱即用的（不管这个过程是不是那么舒服）。
       3、功能基本满足基本需求
       如果你对监控内容的要求不是特别高的话，那么一般的监控信息已经可以满足你绝大部分的需求了，二次开发的工作量可以降到最低。
       4、支持集群
       无论官方给出的集群方案是否完美，但至少是存在开源的免费集群方案的，这比收费和自己建设简单多了。
       一句话总结一下，prometheus监控体系的优点就是入门简单，缺点就是后续成本比较高。

       再聊一下consul这套架构，照例还是先说说里面的坑。
       1、配置冲突
       因为后端都是使用springboot为主的微服务，因此consul的集成依赖了springcloud中关于consul服务发现的部分。这里提示大家一下，如果你使用的springcloud中的其他服务发现模式，例如你很不巧的用了zk作为配置中心，并使用了springcloud的相关组件来注入属性，那么你很快会发现你的服务在引入了consul和zookeeper之后，无法启动了。因为他们都注册了org.springframework.boot.autoconfigure.EnableAutoConfiguration这个bean，同一个bean显然是不能同时存在于spring的context中的，这里推荐一个github上贡献的开源解决策略：
https://github.com/cloud-ready/spring-cloud-service-discovery
       但里面的代码可能并不是都用得上，大家可以自行调整，抓住关键即可。
       2、管理复杂
       为了简化各项目配置，consul的服务地址均使用了默认的localhost:8500这个地址，这就意味着每台宿主机都必须有一个consul的client端在工作。可以想见的是，随着宿主机的增多，consul-client的数量也会持续增加，对它们的管理势必会持续增大管理成本。
       这个问题还容易出现端口冲突，因为consul占用的端口比较多（5个端口），且位于8000-9000号段（8500,8300,8301,8302,8600），是比较容易跟其他应用的端口发生冲突的，实际操作中，如果对该机制不熟悉，则很容易在配置时提前占用相关接口。
       3、注销机制
       consul的注销机制是存在问题的，但是最近的测试发现似乎最新的server端已经增加了一段时间不在线即注销对应的服务实例的操作，不过并没有专门进行测试。但无论如何，某一个服务实例宕机，如果想通过consul-server端进行api调用注销仍然是不可行的，你只能通过该实例注册的consul-client进行注销。
       4、pull机制
       这个问题在prometheus也存在，默认情况下，数据都是由server端从client端拉数据的，因此client端向server端注册时，必须保证逆向可达，这一点在ip和网络配置上是存在一定环境要求的，这种模式虽然解决server端可能会被client同时请求而压垮的场景，但是也带来了逆向必须可达的环境要求，给开发和调试会带来一定困难。

       再说说consul架构的优势
       1、与终端解耦
       consul不是spring开发，开发之处就是可以接入各种终端的，这方面能力不需要担心。
       2、接入便捷
       默认环境下，java后端接入成本极低，开发人员无需感知。
       3、服务稳定
       在集群模式和拉数据模式下，server压力可控，不用太过于担心其稳定性问题。
       4、和prometheus接入简单
       prometheus官方支持consul接入，因此在初步接入层面比较容易，不需要额外开发和配置。（这也是选择这两套搭伴使用的很重要原因）
       5、开源和开箱即用
       与prometheus相同。

       再说一下其他组件的优缺点：
       1、Grafana
        Grafana的可用性很高，图表插件非常多，完全可以根据自己的需求进行选用，并且社区上传了大量的Dashboard可供使用，因此其优势与其他组件类似，开源、开箱即用等。但其社区贡献也带来了如exporter一般的问题，首先是Dashboard和exporter的匹配度普遍不高，下载之后需要大量的二次调整（花费了相当的时间），其次就是接入的api不友好，例如接入prometheus，其接入api即为prometheus的时序数据库的查询语句，由于不熟悉，也导致需要大量时间进行熟悉的理解。最后就是缺少列表信息的内容，社区的Dashboard主要由各种图表构成，做整体分析是ok的，但是问题定位的时候就显得力不从心了。
       2、alertmanager
       刚才已经提到了alertmanager的集群问题，alertmanager集群即便使用了官方的cluster配置，仍然会在某些情况下出现重复发送消息的问题。实测是在某服务down后，启动该服务。观察日志发现，down的时候确实只收到了一条消息，但是很奇怪的是启动的时候也会收到信息，并且此时是两条信息。因为并没有配置服务上线的告警信息，所以只可能是服务down的提示信息，暂时还没有去深入了解相关文档（也不确定有没有相关文档）。
       其次就是alertmanager的匹配规则配置很复杂，是在prometheus的时序数据库的基础上做二次定制，也意味着如果你中间更换数据库，那么之前的配置需要重新调整。这个对于后期配置开发是非常不利的，尤其其键值还依赖exporter的metrics信息，从编码的角度而言，这里缺少接口的封装，对于程序的可维护性是非常不利的。
       
       大致就是这些了，下面再列一下目前使用到的组件以及相关信息。
       以下是用到的exporter的组件列表。
组件名 来源 启动命令 备注
jmx_exporter https://github.com/prometheus/jmx_exporter 需要内嵌至容器启动命令中JAVA_OPTS="$JAVA_OPTS  -javaagent:/*/prometheus_exporter/jmx/jmx_prometheus_javaagent-0.11.0.jar=9151:/*/prometheus_exporter/jmx/tomcat.yaml" 不同的jmx监控对象需要配置不同的yaml文件以约定其输出内容，配置文档见最后的附件
node_exporter https://github.com/prometheus/node_exporter nohup ./node_exporter --web.listen-address=":8909" >nohup.out 2>&1 & 
nginx_vts_exporter https://github.com/hnlq715/nginx-vts-exporter nohup ./nginx-vts-exporter -metrics.namespace=node1 -nginx.scrape_uri=localhost/status/format/json  >nohup.out 2>&1 & 这个插件需要依赖nginx支持相关功能，当nginx支持后，启动该exporter即可
php-fpm-exporter https://github.com/bakins/php-fpm-exporter nohup ./php-fpm-exporter.linux.amd64 --addr x.x.x.x:xxxx --endpoint http://x.x.x.x:xxxx/php_fpm_status >nohup.out 2>&1 & 依赖php-fpm的功能，一般需要在nginx进行配置后方可使用
mysql_exporter https://github.com/prometheus/mysqld_exporter /mysqld_exporter --log.level=debug --web.listen-address=:8908 --config.my-cnf=mysql.cfg --collect.auto_increment.columns --collect.global_status 1、需要一个有权限的账号2、需要配置一个配置文件，输入mysql的信息我的配置信息如下：[client]host=port=user=password=
redis_exporter https://github.com/oliver006/redis_exporter nohup ./redis_exporter -web.listen-address=":8912" -redis.file=redisconfig.cfg >nohup.out 2>&1 &  配置文档信息：redis://xx.xx.2.xx:xxxx,password
zookeeper_exporter https://github.com/carlpett/zookeeper_exporter ./zookeeper_exporter -bind-addr=:8902 -zookeeper=localhost:8182 貌似不支持单机集群采集，待验证
mongodb_exporter https://github.com/dcu/mongodb_exporter ./mongodb_exporter --web.listen-address=:8919 --mongodb.uri=mongodb://user:password@uri:port 需要构建用户到mongo数据库中

下面是grafana的内容模板：

http://note.youdao.com/noteshare?id=996753825eac4fd3e7c6dafcf498bb64
A9QZ

没有贴编号的原因是里面大多数都有过修改，如果直接使用前面的exporter，应该是可以直接对接的。

