---
layout: post
title: Ansible-variables
date: 2018-07-19 17:57:22.000000000 +08:00
---

前段时间面试卡在了一道dict和list混合的变量问题下，记录如下，变量文件如下：

```yaml
users_with_items:
  - name: "alice"
    personal_directories:
      - "bob"
      - "carol"
      - "dan"
  - name: "bob"
    personal_directories:
      - "alice"
      - "carol"
      - "dan"

common_directories:
  - ".ssh"
  - "loops"
```

先分析一下这个变量文件，由两个list组成，而第一个list里面又包含了两个dictionary,dictionary里面又有一个list , 当时的变量虽然不是这样的，但是大致也是这种类型。首先因为这是一个变量文件，所以需要使用到 var_files 来引用这个变量文件。如果使用 with_items 的话，那只能遍历一个 list ，如下所示：

```yaml
---
- hosts: localhost
  gather_facts: no

  vars_files:
    - /home/nick/ansible/newvars.yml

  tasks:
    - name: "test users_with_items result"
      debug: msg="Hello {{ item.name }} with {{ item.personal_directories}}"
      with_items:
        - "{{ users_with_items }}"
    - name: "test common_directories"
      debug: msg="This is common_directories {{ item }}"
      with_items:
        - "{{ common_directories }}"
```

这段 playbook 遍历变量文件里的两个 list , 因为上面分析过第一个 list 里面包含两个 dictionary , 而 dictionary 是由 key 和value 组成的。因此第一个 task 会打印出来名字和目录 list ，结果如下：

```
PLAY [localhost] ***************************************************************************************************************************

TASK [test users_with_items result] ********************************************************************************************************
ok: [localhost] => (item={u'personal_directories': [u'bob',u'carol', u'dan'], u'name': u'alice'}) => {
    "msg": "Hello alice with [u'bob', u'carol', u'dan']"
}
ok: [localhost] => (item={u'personal_directories': [u'alice',u'carol', u'dan'], u'name': u'bob'}) => {
    "msg": "Hello bob with [u'alice', u'carol', u'dan']"
}

TASK [test common_directories] *************************************************************************************************************
ok: [localhost] => (item=.ssh) => {
    "msg": "This is common_directories .ssh"
}
ok: [localhost] => (item=loops) => {
    "msg": "This is common_directories loops"
}
```

  这个示例是针对读取一个变量文件的，ansible 也支持直接在 playbook 里面定义循环 list ，比如：

```yaml
    - name: "pre-define list variables"
      debug: msg="test {{ item }}"
      with_items:
        - a
        - b
        - c
      tags:
        - list-var
```

执行结果如下：

```
PLAY [localhost] ***************************************************************************************************************************

TASK [pre-define list variables] ***********************************************************************************************************
ok: [localhost] => (item=a) => {
    "msg": "test a"
}
ok: [localhost] => (item=b) => {
    "msg": "test b"
}
ok: [localhost] => (item=c) => {
    "msg": "test c"
}
```

注意，YAML格式里面的 list 一定前面要有 - 这个符号，如果没有这个符合，只是当作空格隔开的几个字符而已。

然后其实上面的例子只是取出了 list 里面的 dictionary , 但是 dictionary 里面的 list 怎么办？先看看我们这个变量文件里面， dictionary 嵌套 list 是在 key 为 personal_directories 这里，所以我们可以使用 with_subelements 来读取嵌套的内容：

```yaml
    - name: "test with_subelements"
      debug: msg="Hello {{item.0.name}} with {{ item.1 }}"
      with_subelements:
        - "{{users_with_items}}"
        - personal_directories
      tags:
        - "subelements"
```

看看 with_subelements 里面，其实是一个 list ，这个 list 第一个内容是这个大的 list ，也就是包含我们两个 dictionary 的这个 list ，然后第二个是我们的子 list 的在这个 dictionary 里的 key , 看看执行结果：

```
PLAY [localhost] ***************************************************************************************************************************

TASK [test with_subelements] ***************************************************************************************************************
ok: [localhost] => (item=[{u'name': u'alice'}, u'bob']) => {
    "msg": "Hello alice with bob"
}
ok: [localhost] => (item=[{u'name': u'alice'}, u'carol']) => {
    "msg": "Hello alice with carol"
}
ok: [localhost] => (item=[{u'name': u'alice'}, u'dan']) => {
    "msg": "Hello alice with dan"
}
ok: [localhost] => (item=[{u'name': u'bob'}, u'alice']) => {
    "msg": "Hello bob with alice"
}
ok: [localhost] => (item=[{u'name': u'bob'}, u'carol']) => {
    "msg": "Hello bob with carol"
}
ok: [localhost] => (item=[{u'name': u'bob'}, u'dan']) => {
    "msg": "Hello bob with dan"
}
```

上面的例子都是用 with_items , 也就是说把变量作为 list 来遍历，但是万一我的变量不是 list 而是 dictionary , 咋办呢？ ansible 还提供了 with_dict, 比如说变量如下：

```
languages:
  ruby: Elite
  python: Elite
  dotnet: Lame
```

这是一个 dictionary , 那如何遍历他呢，使用 with_items 的时候结果如下：

```yaml
    - name: "test with dict languages"
      debug: msg="language is {{ item }}"
      with_items:
        - "{{ languages }}"
      tags:
        - "3employee"
```



```
PLAY [localhost] ***************************************************************************************************************************

TASK [test with dict languages] ************************************************************************************************************
ok: [localhost] => (item={u'python': u'Elite', u'dotnet': u'Lame', u'ruby': u'Elite'}) => {
    "msg": "language is {u'python': u'Elite', u'dotnet': u'Lame', u'ruby': u'Elite'}"
}
```

使用 with_dict 的时候则是这样的：

```yaml
    - name: "test with dict languages"
      debug: msg="language is {{ item.key }} and {{ item.value }}"
      with_dict:
        - "{{ languages }}"
      tags:
        - "3employee"
```

结果如下：

```
PLAY [localhost] ***************************************************************************************************************************

TASK [test with dict languages] ************************************************************************************************************
ok: [localhost] => (item={'value': u'Elite', 'key': u'python'}) => {
    "msg": "language is python and Elite"
}
ok: [localhost] => (item={'value': u'Lame', 'key': u'dotnet'}) => {
    "msg": "language is dotnet and Lame"
}
ok: [localhost] => (item={'value': u'Elite', 'key': u'ruby'}) => {
    "msg": "language is ruby and Elite"
}
```

如果刚好我们的 dict 里面嵌套了 list 呢，这时候就可以使用 with_nested 了，比如变量如下：

```
employee:
  name: "Example Developer"
  job: "Developer"
  skill: "Elite"
  employed: True
  foods:
      - Apple
      - Orange
      - Strawberry
      - Mango
```

这个变量首先是一个名为 employee 的 dict , 然后这个字典里面有一个 key 为 foods 的列表，如果我们直接使用 with_dict , 那这个列表就会被整个打印出来而不是分开，因此我们这样写：

```yaml
    - name: "test with employee dict"
      debug: msg="name is {{ item.0 }} and {{ item.1 }} "
      when: employee.employed
      with_nested:
        - "{{ employee.name }}"
        - "{{ employee.foods }}"
      tags:
        - "2employee"
```

item.0 表示名为 employee 的字典，这个字典里面有一个 key 为 name 的 item , 然后 item.1 是这个字典里面 key 为 foods 的列表，因此结果如下：

```
PLAY [localhost] ***************************************************************************************************************************

TASK [test with employee dict] *************************************************************************************************************
ok: [localhost] => (item=[u'Example Developer', u'Apple']) => {
    "msg": "name is Example Developer and Apple "
}
ok: [localhost] => (item=[u'Example Developer', u'Orange']) => {
    "msg": "name is Example Developer and Orange "
}
ok: [localhost] => (item=[u'Example Developer', u'Strawberry']) => {
    "msg": "name is Example Developer and Strawberry "
}
ok: [localhost] => (item=[u'Example Developer', u'Mango']) => {
    "msg": "name is Example Developer and Mango "
}
```

再介绍一下 with_subelements 的用法，首先变量如下：

```yaml
---
# An employee record
employee:
  - name: Martin D'vloper
    job: Developer
    skill: Elite
    employed: True
    foods:
        - Apple
        - Orange
        - Strawberry
        - Mango
    languages:
        perl: Elite
        python: Elite
        pascal: Lame
    education: |
        4 GCSEs
        3 A-Levels
        BSc in the Internet of Things

  - name: Martin D'vloper1
    job: Developer1
    skill: Elite1
    employed: True
    foods:
        - Apple1
        - Orange1
        - Strawbe1rry
        - Mango1
    languages:
        perl: El1ite
        python: E1lite
        pascal: La1me
    education: |
        4 GCSEs1
        3 A-Levels
```

其次代码如下：

```yaml
    - name: "test the dict in var2"
      debug: msg="{{item.0.name}} likes {{item.1}}"
      with_subelements:
        - "{{employee}}"
        - foods
```

执行结果类似上面例子：

```
PLAY [localhost] ***************************************************************************************************************************

TASK [test the dict in var2] ***************************************************************************************************************
ok: [localhost] => (item=[{u'name': u"Martin D'vloper", u'languages': {u'python': u'Elite', u'pascal': u'Lame', u'perl': u'Elite'}, u'job': u'Developer', u'employed': True, u'skill': u'Elite', u'education': u'4 GCSEs\n3 A-Levels\nBSc in the Internet of Things\n'}, u'Apple']) => {
    "msg": "Martin D'vloper likes Apple"
}
ok: [localhost] => (item=[{u'name': u"Martin D'vloper", u'languages': {u'python': u'Elite', u'pascal': u'Lame', u'perl': u'Elite'}, u'job': u'Developer', u'employed': True, u'skill': u'Elite', u'education': u'4 GCSEs\n3 A-Levels\nBSc in the Internet of Things\n'}, u'Orange']) => {
    "msg": "Martin D'vloper likes Orange"
}
ok: [localhost] => (item=[{u'name': u"Martin D'vloper", u'languages': {u'python': u'Elite', u'pascal': u'Lame', u'perl': u'Elite'}, u'job': u'Developer', u'employed': True, u'skill': u'Elite', u'education': u'4 GCSEs\n3 A-Levels\nBSc in the Internet of Things\n'}, u'Strawberry']) => {
    "msg": "Martin D'vloper likes Strawberry"
}
ok: [localhost] => (item=[{u'name': u"Martin D'vloper", u'languages': {u'python': u'Elite', u'pascal': u'Lame', u'perl': u'Elite'}, u'job': u'Developer', u'employed': True, u'skill': u'Elite', u'education': u'4 GCSEs\n3 A-Levels\nBSc in the Internet of Things\n'}, u'Mango']) => {
    "msg": "Martin D'vloper likes Mango"
}
ok: [localhost] => (item=[{u'name': u"Martin D'vloper1", u'languages': {u'python': u'E1lite', u'pascal': u'La1me', u'perl': u'El1ite'}, u'job': u'Developer1', u'employed': True, u'skill': u'Elite1', u'education': u'4 GCSEs1\n3 A-Levels\n'}, u'Apple1']) => {
    "msg": "Martin D'vloper1 likes Apple1"
}
ok: [localhost] => (item=[{u'name': u"Martin D'vloper1", u'languages': {u'python': u'E1lite', u'pascal': u'La1me', u'perl': u'El1ite'}, u'job': u'Developer1', u'employed': True, u'skill': u'Elite1', u'education': u'4 GCSEs1\n3 A-Levels\n'}, u'Orange1']) => {
    "msg": "Martin D'vloper1 likes Orange1"
}
ok: [localhost] => (item=[{u'name': u"Martin D'vloper1", u'languages': {u'python': u'E1lite', u'pascal': u'La1me', u'perl': u'El1ite'}, u'job': u'Developer1', u'employed': True, u'skill': u'Elite1', u'education': u'4 GCSEs1\n3 A-Levels\n'}, u'Strawbe1rry']) => {
    "msg": "Martin D'vloper1 likes Strawbe1rry"
}
ok: [localhost] => (item=[{u'name': u"Martin D'vloper1", u'languages': {u'python': u'E1lite', u'pascal': u'La1me', u'perl': u'El1ite'}, u'job': u'Developer1', u'employed': True, u'skill': u'Elite1', u'education': u'4 GCSEs1\n3 A-Levels\n'}, u'Mango1']) => {
    "msg": "Martin D'vloper1 likes Mango1"
}
```

如果在这里使用 with_nested 估计就不行了，因为 nested 的两个需要是 list , 所以如果像最后这个例子里面包含两个dict作为 list , with_nested 就写不了了。所以如果有嵌套还是推荐用 with_subelements , 这样第二个只要写出 key 的名字就可以，前提是这个 key 所对应的 value 是一个 list , 比如这样写执行起来就报错：

```
    - name: "test the dict in var2"
      debug: msg="{{item.0.name}} likes {{item.1}}"
      with_subelements:
        - "{{employee}}"
        - languages
        
nick@nick-ThinkPad-T61:~/ansible$ ansible-playbook readvar2.yml
 [WARNING]: provided hosts list is empty, only localhost is available. Note that the implicit localhost does not match 'all'


PLAY [localhost] ***************************************************************************************************************************

TASK [test the dict in var2] ***************************************************************************************************************
fatal: [localhost]: FAILED! => {"msg": "the key languages should point to a list, got '{u'python': u'Elite', u'pascal': u'Lame', u'perl': u'Elite'}'"}

PLAY RECAP *********************************************************************************************************************************
localhost                  : ok=0    changed=0    unreachable=0    failed=1   
```

我把 foods 改成了 languages , 因为 languages 对应是一个 dict , 因此报错。
