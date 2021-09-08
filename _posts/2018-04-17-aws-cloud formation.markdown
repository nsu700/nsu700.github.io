---
layout: post
title: AWS系列之CloudFormation
date: 2018-04-17 15:32:24.000000000 +09:00
---

不知道手动配置一台EC2，然后配置好网络，存储和IAM，还有auto scaling group，需要多长时间，要点击多少次鼠标，敲多少次键盘。但是这仅仅是一台，如果说业务需求要成千上万，而且还不同的需求不同的配置，那就是一件很崩溃的事情了。所以我今天打算聊聊CloudFormation , 就这一个服务解决痛点。

简单的说，它是根据预先配置好的template来生成对应的stack ， 这是什么话，还记得我们的auto scaling不，创建auto scaling需要提前创建好launch configuration , 然后在符合指标触发伸缩的时候，根据launch configuration去增加或者减少EC2的实例。今天我们的主角CloudFormation也是类似的东西，但是它的功能更加强大，不只是配置EC2.

进到CloudFormation界面的时候，默认就是Stack的页面，而创建stack的时候就会要求你选择一个template , 创建template有个很方便的工具，就是aws提供的一个图形界面，只要把所需的服务拖到图面上，然后修改对应的需求就可以。但是用惯了terminal的我们，能敲命令就不用鼠标的我们，怎么能满足于这样的工具呢。所以其实我更建议直接编辑json来生成需要的template .

我们就拿 LAMP_single_template来解释一下吧。顶级目录，

1. AWSTemplateFormatVersion ， 这个是版本号，有人可能会嫌麻烦，但是当你有很多不同的template的时候，你就只要版本号是多么的重要，而且，重要的代码一定一定一定要放在github上面，不管是私有还是公有的；
2. Description , 这个是template的描述信息，想怎么写怎么写，但是最好写清楚这个template是干嘛的，虽然你自己知道，但是总会有别人会看到的，能不能一眼看出来这是干嘛的，就看你的description了；
3. Parameters , 这里定义一些变量 ， 以便后面引用；
4. Mappings  , 配置EC2的AZ， 类型和对应使用的AMI；
5. Resources , 这个就是直接操作EC2或者说对应服务的配置了 ；
6. Output , 定义一些这个template完成后的输出，其他的template也可以读取这个输出作为自己的变量。这样可以减少一些硬编码，让template更加灵活一些。

接着我们再往下看，Parameters这里呢， [parameters-structure][parameters-structure]

{% highlight json %}

"Parameters" : {
  "ParameterLogicalID" : {
    "Type" : "DataType",
    "ParameterProperty" : "value"
  }
}

{% endhighlight %}

需要声明的两个就是LogicalID和Type ,这两个指明了这个Parameter的ID，以便后面引用，而Type则限定了这个Parameter的类型，需要注意的是，当定义密码相关的比如DB密码，用户密码什么的，可以在声明里面加上NoEcho为true，这样用户在输入密码的时候就会显示星号。



[ parameters-structure ] : https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/parameters-section-structure.html