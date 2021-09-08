---
layout: post
title: Azure 的各种 Roles
date: 2020-01-18 18:32:24.000000000 +08:00
---

可能大部分 Azure 新人都会觉得它的权限管理很复杂，比如说我刚刚接触 Azure 的时候，我明明有账号密码但是就一直 SSH 不进去机器，后来才发现你要能 SSH 还必须要有一个 Role Assignment 才可以，就是 Virtual Machine User Login, 但是当你想要 SUDO 的时候就会发现又不行了，权限又不够了，这时候就要另外一个 Role Assignment ，这个就是 Virtual Machine Administrator Login. 很坑的就是这两缺一不可，当然有 Owner 权限的又不一样了。

蛮好奇的一个就是各种 Role ，看似权限重叠，但是又各不相关。比如说知道 Service Administrator ,但是 Role Assignment 里面又找不到。比如说我前段时间创建一个新用户， AD 里面，然后我想让这个用户有足够的权限管理我这个 Azure Account 里面的所有资源，天真的以为只要给了他 Global Administrator 的权限就够了，结果发现他登陆的时候根本看不到我的 subscription ，反而 Azure 还提醒他有一年的免费使用要求他自己创建 subcription .这里我就犯了一个错误，没有理解各种 Role 的区别。

其实 Azure 有几种不同的 Role， 主要有 Azure Role, AD Role 还有游离于这两者之外的 Account Admin ， Service Admin 和 Co-Admin 这三个。先聊聊这游离的三个：

1. Account Admin ，这可能算是最大的权限了吧，但是他又不能管理 Azure 的资源。先说说这个 Account ，比如说公司A在 Azure 上面注册了一个账号，那这个公司在 Azure 上面就是一个  Account 。这个Account Admin 可以增加/减少 subscription , 也可以修改 Service Admin, 修改账单什么的。但是记住，他不能管理 subscription 下面的任何资源。这个 Role 在每一个 Account 里面只能有一个用户。
2. 那要管理资源怎么办呢，就是 Service Admin 了，每一个 subscription 里面都有而且只能又一个这个 Role 来管理这个 subscription 下面的各种资源，他也可以删除这个 subscription 但是他没资格创建新的 subscription。默认的情况下，Account Admin 就是新建的 subscription 的 Service Admin。
3. 说起来也挺局限啊，能管理资源的只有一个 Service Admin ,还每个 subscription 只有一个，所以就有了另外一个叫 Co-Admin 的 Role 了， 这个在每一个 subscription 里面最多只能有200个。它的权限跟 Service Admin 几乎是一样的，除了 Service Admin 可以管理 Co-Admin， 他就只能添加 Co-admin而不能修改， Service Admin 可以更改 subscription 的 AD 设置，它也不能做这个。

除了上面这几个 Role， Azure 还支持自建 Role ,而这个对于整个 AD 下面的所有 subscription 都可以使用，每个AD限制最多5000个 custom role.但是其实 Azure 已经提供了很多的 Built-in roles, 一般也都满足了大量的需求，这些 built-in roles 分成了两大类：

1. Azure role, 这个很常见，在所有的 resouce 里面，你点击进去 Access Control（IAM），在这里面添加 Role Assignment 就可以看到，这里面全都是 Azure role, 这些role的一个共同特点就是为了管理某种或者所有资源而存在的，比如说我前面提到说要可以 SSH 进去机器，要有一个 Virtual Machine User Login，这个就是一个 Azure Role，这个Role 就允许了用户可以登陆机器，如果这个 Role 是在 subscription 或者 resouce group 级别添加的 Assignment ， 那这个用户就可以登陆这个 subscription 或者 resouce group 下面的所有机器， 如果说这个是在 VM 级别赋予的，那这个用户就只能登陆对应的那个机器。这个类别里面又分成了四个小类别：
    1. Owner 拥有者的权限，可以对这个对象做任何修改包括管理用户权限什么的。要注意的是 Service Admin 和 Co-Admin 默认就是 Owner
    2. Contributor 除了不能管理用户权限之外，有 Owner 的所有权限
    3. Reader 就是只读了，只能读各种资源
    4. Specific resource role 这有很多个很 specific 的 role, 针对各种各样的资源。比如说刚才说到的 Virtual Machine User Login 就只针对Virtual Machine 的登陆权限。
2. AD role，这个是用来管理 AD 的权限，比如说在 AD 里面创建用户，修改用户密码，设置用户 policy 之类的。我前段时间犯的一个错误就是，在AD里面新建用户，然后我给了这个用户 Global Admin 的权限，私以为他有了最高权限，结果发现他还是访问不了我的 subscription 。所以这个的解决办法就是要另外给他一个 Azure Role。

大概了解这两个分类后，有个很重要也经常会搞混的词就是 tenant，可能很多地方都会把这个词翻译成用户或者租户。但是我觉得不太准确，对于 Azure 来说，一个 AD 就是一个 tenant，有一个很普遍的现象就是通常一个公司也只会有一个 AD ，所以对于 Azure 来说这个 tenant 对应的就是这个 AD 其实也就代表了这一整个公司。但是我们不能排除一个公司有多个 AD 的可能性，所以我觉得把 tenant 翻译成租户什么的还是不太确切。 接下来我们再看 Azure Role 跟 AD Role 一个很大的差别，就是他们有着不同的 scope ，也就是不同的使用范围。
Azure Role 的 scope 可以是 subscription, resource group, 各种资源等等，但是 AD Role 的 scope 却只有一个，就是这个 tenant.也就是说 AD Role不能跨越它自己的这个 AD 。但是 Azure 也提供了一个可以让 AD Role 跨越 scope 的一个方法，就是在 AD 里面的 Property，里面有一个 Access management for Azure resources， 这个只要打上 Yes ，那这个 AD 的Admin就可以管理这个 AD 所连接的所有 subscription 里面的资源了。
不过我觉得这个应该不太会打开，因为对于一定规模的公司而言，AD是有专人管理的，Azure里面的各种计算啊，存储啊，网络资源什么的又是另外的团队管理，所以这个为了权限分离，相信大概率是不会打开的。这个也是 Azure 的设计使然，AD的不能管理 resource ，而 resource 的也不能管理 AD。但是有一个特例就是 AD 的 Global Administrator， 他可以赋予他自己 resource 管理的权限，那对于其他 AD 用户怎么办呢，Azure 又提供了一个叫 PIM 的工具来解决这个权限问题，改天我们再聊聊这个问题。

[官方文档](https://docs.microsoft.com/en-us/azure/role-based-access-control/)