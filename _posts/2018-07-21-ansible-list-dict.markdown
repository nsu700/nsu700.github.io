---
layout: post
title: Ansible-list 和 dict 的写法区别
date: 2018-07-21 18:14:16.000000000 +08:00
---

相信不管学什么语言，老师都会要求先熟悉数据结构，我前段时间一直没分清 ansible 里面 字典和列表的写法区别，经过一段时间的实验，现在记录如下，变量文件：

```yaml
# An employee record
employee1:
  - name: Martin D'vloper
  - job: Developer
  - skill: Elite
  - employed: True

employee2:
  - name: Martin D'vloper
    job: Developer
    skill: Elite
    employed: True

employee3:
    name: Martin D'vloper
    job: Developer
    skill: Elite
    employed: True
```

在这里 employee1 是一个列表，剩下两个 employee2 和 employee3 都是字典，区别就在于 employee2 是一个内容为一个字典的单项列表，而 employee3 则是一个字典，怎么区别呢，看下面的例子，我使用的是 with_dict 来判断是否为字典。

```
  tasks:
    - name: "test employee1 in var3"
      debug: msg="{{ item.key }}"
      with_dict:
        - "{{ employee1 }}"
      ignore_errors: yes
    - name: "test employee2 in var3"
      debug: msg="{{ item.key }}"
      with_dict:
        - "{{ employee2.0 }}"
      ignore_errors: yes
    - name: "test employee3 in var3"
      debug: msg="{{ item.key }}"
      with_dict:
        - "{{ employee3 }}"
      ignore_errors: yes
```



```python
  tasks:
    - name: "test employee1 in var3"
      debug: msg="{{ item.key }}"
      with_dict:
        - "{{ employee1 }}"
      ignore_errors: yes
    - name: "test employee2 in var3"
      debug: msg="{{ item.key }}"
      with_dict:
        - "{{ employee2.0 }}"
      ignore_errors: yes
    - name: "test employee3 in var3"
      debug: msg="{{ item.key }}"
      with_dict:
        - "{{ employee3 }}"
      ignore_errors: yes
```

结果如下：

```
TASK [test employee1 in var3] **************************************************************************************************************
fatal: [localhost]: FAILED! => {"msg": "with_dict expects a dict"}
...ignoring

TASK [test employee2 in var3] **************************************************************************************************************
ok: [localhost] => (item={'value': True, 'key': u'employed'}) => {
    "msg": "employed"
}
ok: [localhost] => (item={'value': u'Elite', 'key': u'skill'}) => {
    "msg": "skill"
}
ok: [localhost] => (item={'value': u'Developer', 'key': u'job'}) => {
    "msg": "job"
}
ok: [localhost] => (item={'value': u"Martin D'vloper", 'key': u'name'}) => {
    "msg": "name"
}

TASK [test employee3 in var3] **************************************************************************************************************
ok: [localhost] => (item={'value': True, 'key': u'employed'}) => {
    "msg": "employed"
}
ok: [localhost] => (item={'value': u'Elite', 'key': u'skill'}) => {
    "msg": "skill"
}
ok: [localhost] => (item={'value': u'Developer', 'key': u'job'}) => {
    "msg": "job"
}
ok: [localhost] => (item={'value': u"Martin D'vloper", 'key': u'name'}) => {
    "msg": "name"
}
```

这里很明显，employee1 的时候报错，说 with_dict expects a dict ， 也就是说 with_dict 只能处理字典结构，因此我们的 employee1 这里不是 ansible 的字典。然后在处理 employee2 的时候，跟 employee3 有个小区别，就是 with_dict 的时候我写的是 employee2.0 , 为什么要加个 .0呢？这是因为我们上面分析了 employee2 是个单项列表，所以如果我只传给 with_dict 的是 employee2 ， 那会报第一个一样的错误， with_dict expects a dict 。这里也可以写成 employee2[0] , 这个也是读取 python 列表的办法。处理 employee3 就很清楚了，这里就是一个字典，连 key value都写得很清楚。

希望通过这篇文章，我说清楚了 ansible 里的字典和列表的写法区别。

再给变量文件添加点内容看看区别：

```yaml
---
# An employee record
employee1:
  - name: Martin D'vloper
  - job: Developer
  - skill: Elite
  - employed: True

employee2:
  - name: Martin D'vloper
    job: Developer
    skill: Elite
    employed: True
  - name: Martin D'vloper
    job: Developer
    skill: Elite
    employed: True

employee3:
    name: Martin D'vloper
    job: Developer
    skill: Elite
    employed: True
    name: Martin D'vloper
    job: Developer
    skill: Elite
    employed: True
```

然后 playbook 改成这样：

```python
  tasks:
    - name: "test employee1 in var3"
      debug: msg="{{item.name}}"
      with_items:
        - "{{employee1}}"
      ignore_errors: yes
    - name: "test employee2 in var3"
      debug: msg="{{item.name}}"
      with_items:
        - "{{employee2}}"
      ignore_errors: yes
    - name: "test employee3 in var3"
      debug: msg="{{item.name}}"
      with_items:
        - "{{employee3}}"
      ignore_errors: yes
```

先解释一下这个 playbook 的意图，用的是 with_items 来处理，也就是说假设所有的变量都是列表 list , 这个说法可能不太准确，因为列表是没有 key 这个概念的，然后 debug 的时候使用的是 item.name ,因为我们每一段都有一个 name , 希望可以打印出来 name 对应的内容。结果如下：

```yaml
nick@nick-ThinkPad-T61:~/ansible$ ansible-playbook readvar3.yml
 [WARNING]: provided hosts list is empty, only localhost is available. Note that the implicit localhost does not match 'all'

 [WARNING]: While constructing a mapping from /home/nick/ansible/var3.yml, line 20, column 5, found a duplicate dict key (name). Using last
defined value only.

 [WARNING]: While constructing a mapping from /home/nick/ansible/var3.yml, line 20, column 5, found a duplicate dict key (job). Using last
defined value only.

 [WARNING]: While constructing a mapping from /home/nick/ansible/var3.yml, line 20, column 5, found a duplicate dict key (skill). Using
last defined value only.

 [WARNING]: While constructing a mapping from /home/nick/ansible/var3.yml, line 20, column 5, found a duplicate dict key (employed). Using
last defined value only.


PLAY [localhost] ***************************************************************************************************************************

TASK [test employee1 in var3] **************************************************************************************************************
ok: [localhost] => (item={u'name': u"Martin D'vloper"}) => {
    "msg": "Martin D'vloper"
}
fatal: [localhost]: FAILED! => {"msg": "The task includes an option with an undefined variable. The error was: 'dict object' has no attribute 'name'\n\nThe error appears to have been in '/home/nick/ansible/readvar3.yml': line 8, column 7, but may\nbe elsewhere in the file depending on the exact syntax problem.\n\nThe offending line appears to be:\n\n  tasks:\n    - name: \"test employee1 in var3\"\n      ^ here\n"}
...ignoring

TASK [test employee2 in var3] **************************************************************************************************************
ok: [localhost] => (item={u'employed': True, u'skill': u'Elite', u'job': u'Developer', u'name': u"Martin D'vloper"}) => {
    "msg": "Martin D'vloper"
}
ok: [localhost] => (item={u'employed': True, u'skill': u'Elite', u'job': u'Developer', u'name': u"Martin D'vloper"}) => {
    "msg": "Martin D'vloper"
}

TASK [test employee3 in var3] **************************************************************************************************************
ok: [localhost] => (item={u'employed': True, u'skill': u'Elite', u'job': u'Developer', u'name': u"Martin D'vloper"}) => {
    "msg": "Martin D'vloper"
}
```

我们先暂时略过前面的几行 warning , 看 employee1 的执行结果，确实打印出来 name 对应的内容，但是后面还有一些报错，说什么没有这个 key 之类的，这是因为 with_items 把这些 employee1 当成了多个键值对列表，当处理到 name 这一行的时候， key 是 name 没错，打印出内容。但是接下来的几行 key 都不是 name ,所以报错。然后看 employee2 , 这里打印了两个结果，并非循环出错，而是因为我们这里是有两个内容为字典的 item 组成的一个列表，因此读列表里面的地一个值时，这个字典有一个 key 为 name ,打印出来 。然后处理这个列表的下一项，又有一个 key 为 name  的字典，再打印处理， employee2 处理结束。好了，来到 employee3 了，我们看回前面略过的 warning ,说发现了重复的字典 key , 有 employed , skill , job, name 。就是说这个 employee3 里面是一整个字典，并非 employee2 那样的分成列表的两项，因此虽然这里也有两个 name ,但是只打印出来一个结果。注意 warning  里面有这么一句话， using last define value only , 也就是说不管你重复多少个键，ansible 都只会打印最后一个对应的值出来。

最后总结一下，ansible 使用缩进来分割，如果有多行前面带有 - 的话，那就是 list , 如果不是则是 dict 。如果是多个 dict 并列组成一个 list , 那每个 dict 的第一对 key:value 需要有一个 - ，请参考最后这个例子里的变量内容。