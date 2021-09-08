---
layout: post
title: AWS系列之Well Architecting Frameworks
---

终于决定要考AWS的Solution Architect Associate 了，仔细阅读了 [考纲](https://d1.awsstatic-china.com/training-and-certification/docs-sa-assoc/AWS_Certified_Solutions_Architect_Associate_Feb_2018_%20Exam_Guide_v1.5.2.pdf) 发现主要考点还是围绕着 [Well Architected](https://amazonaws-china.com/architecture/well-architected/) 的五个支柱理论，因此这次考试的准备就先从 [Well Architected Framework](https://d1.awsstatic-china.com/whitepapers/architecture/AWS_Well-Architected_Framework.pdf) 开始了。

首先简单介绍一下这个framework , 这是一套可供用户设计服务系统架构时参考或者评估用的标准。它提出了一系列的问题来帮助用户更全面的了解系统架构是否符合服务的最优使用方案， 同时也可以协助用户在现代云计算环境中设计和部署适合业务的系统架构。它的框架主要基于五个支柱，分别是：

1. Operational Excellence  The ability to run and monitor systems to deliver business value and to continually improve supporting processes and procedures
2. Security  The ability to protect information, systems, and assets while delivering business value through risk assessments and mitigation strategies.
3. Reliability  The ability of a system to recover from infrastructure or service disruptions, dynamically acquire computing resources to meet demand, and mitigate disruptions such as misconfigurations or transient network issues.
4. Performance Efficiency  The ability to use computing resources efficiently to meet system requirements, and to maintain that efficiency as demand changes and technologies evolve.
5. Cost Optimization   The ability to avoid or eliminate unneeded cost or suboptimal resources.

其实我相信各位在做架构设计的时候都会或多或少的考虑到上面几个方面 ， 但是真的这种东西没有总结出来一个成体系的东西，始终还是有点没底。

首先Operational Excellence，要求的是通过运行服务系统来传递商业价值，通过监控来不断改进流程和步骤的能力，近几年来很火的DevOps，其实就有这么一个味道，通过快速的迭代快速的试错，从而达到持续的改进。但是改进也是有个参考值的啊，这里就引入了监控了，通过监控来给不断的迭代和实验提供一个准确的反馈，然后根据反馈再不断的改进，这样才是一个架构生态。相信这个世界上没有一成不变的架构，随着业务的增长，技术的革新，架构不随着变化总会有一天跟不上需求从而被淘汰的。

第二点安全，通过风险评估和降级策略来保证信息，系统和资产的安全的能力。还记得CSDN的密码泄露事件么？还记得heart bleeding 的事件么？安全没做好，服务没办法有保障的提供，用户的信息都没有安全保障，又谈何发展？？？

第三点是可靠性，就是说在系统或者服务崩溃时可以自动恢复，在系统负载很高用户请求很大时能自动应用新的资源满足用户请求，在网络故障或者配置出错的时候能降低影响。这里可能涉及到 AutoScaling , CloudFormation之类的自动化工具，在发现系统问题时，可以快速自动的去相应调整。

第四点是高效的性能， 通过有效的利用计算资源来满足业务需求，在需求变更技术发展的时候可以有效的相应需求。

第五点是费用优化，通过减少和避免不必要的资源浪费来节省费用，老板省点钱，员工多点奖金，大家都happy 。

综合这几点来看呢，其实就是要把一个服务架构设计成一个有活力的系统，不是死的俄罗斯方块似的堆叠服务，而是说能自动相应部署或者减少资源，效率高，费用低，自发自动的一个架构。比如说刚才说的DevOps的持续集成，部署，反馈，优化等等。这里就要求用到 CloudWatch , AutoScaling , IAM , VPC 等等服务了。

下面我打算分五个部分来聊聊这五个支柱，可能写起来文章会特别长，所以看情况划分了。

***Operational Excellence***

上面说过，这其实就是一个可以传递商业价值可以不断改善流程的一种能力。如果是我的理解呢，应该跟运维自动化，监控之类有关。书里提供了一个六个方面的设计准则：

1. Perform operation as code ， 近几年大热的SDX，比如SDN， SDS之类的各种网络定义。为什么要 as code呢，首先代码实现可以实现版本控制，增删修改比较简单，实现容易；其次可以无差别部署，只要是一样的环境，一样的 code 部署出来的肯定是一样的，这样便于维护；最后还可以避免 human error 。
2. Annotate documentation , 不知道大家有没有这样的经历，一个服务出问题了，刚刚好这个服务的负责人走了，然后这时候就很头疼了，你要不去慢慢查log去找shooting , 要不去看看乱的要命的文档，甚至还严重落后，而且还不一定有这个文档。所以这里提倡的就是云端自动生成文档，这样就可以将服务文档规范化，下次有新人接手也方便些，甚至可以作为输出提供给其他服务调用。
3. Make frequent , small , reversible changes , 允许各个系统组件可以单独升级，出问题时随时可以回滚。这样对于团队合作来说对于系统效率来说都是最好的，因为不会出现A团队的服务升级导致B团队的服务出西西， 服务出问题还可以随时回滚，这样要求很好的解耦。我觉得这有点像电脑，CPU，内存，硬盘各种配件都可以单独升级而不会有依赖说我CPU升级了，内存硬盘也得跟着升级。当然CPU升级了，最好内存也跟着升级，但是就算内存不变也没问题。所以这样对于整个服务系统的更新是最好的，出错的代价也是最低的。
4. Refine operations procedures frequently , 经常审核操作流程，通过定期的审核流程和业务来寻找优化的空间。
5. Anticipate failure , 中文有句古话叫居安思危，跟这一条很相似。其实都是要经常假设某个服务出问题了，某个机器down掉了，然后看系统是否可以无缝提供服务，如果不行的话，要赶紧改进。不要等到真正出问题才来改，那就代价很高了，所以要定期的测试系统负载和团队相应能力。
6. Learn from all operation failure , 我的前东家就很注重这些，只要有人出现什么失误，总会要求人家写RCA， root cause analyze , 不为别的，就为了可以让后来者吸取教训。所以这个在云环境中也很重要，既然已经有人踩坑了，就要确保后来人不会又踩这个坑，至少前面的踩坑人要把这个坑给填了或者树立一个警示牌。


说完了这6个设计规范，接下来就说到了最佳实践的三点了。这里面有一句话我觉得很重要，就是运维团队要知道自己的价值和客户的需求，这样才能更有效率的提供客户服务。我碰见过很多运维的其实都不知道自己在干吗，为了打patch而打patch ,而且那种鄙视的价值链，开发的鄙视运维的，这种都是普遍存在的。所以我觉得很重要的一点就是知道自己能干嘛，自己这么做能给整个项目带来什么样子的帮助，这样对自己也是一种资本。

最佳实践第一条就是 preparation , 充分的准备才能带来高效的维护。需要有可靠的流程来确保任务的进行，需要有认可的标准来衡量任务的结果，需要有适当的机制来实时反映任务的进展，像我们已经有 change 的流程，测试机到生产机的部署规范及流程，机器上下架的流程等等。但是这一切在 aws 里面变得更加容易实现，它可以将这是繁琐复杂的东西都变成代码存在，比如说 CloudFormation 可以将整个架构保存成为代码，需要改动时只要修改代码就可以，可以很明确的版本控制各种环境的变化。而且 CloudWatch , CloudTrail , VPC Flow Logs 可以实时监控各种日志，指标和流量 。这里有三个问题可控参考：

1. What factors drive your operational priorities?
2. How do you design your workload to enable operability?
3. How do you know that you are ready to support a workload?

尽量用代码来实现呢，首先可以提高生产效率，其次可以降低 human error ， 还可以实现自动相应 。这样才能跟云环境的便利跟自由相得益彰。

