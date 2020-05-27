### #前言

之前一直看到别人列github8月份最火项目  9月份最火项目...

一直知道肯定是一个简单的方法,但是一直不知道是什么方法,今天无意之中找到了方法,现在记录一下

先看现象:



![image-20200527194401777](https://github.com/ohyemybaby/blog/blob/master/image/image-20200527195346924.png)

再看搜索条件

```shell
# 按照项目名/仓库名搜索(大小写不敏感)
in:name xxx
# 按照ReadMe搜索(大小写不敏感)
in:readme xxx
# 按照description搜索(大小写不敏感)
in:description xxx
# stars数大于xxx
stars:>xxx
# forks数大于xxx
forks:>xxx
# 编程语言为xxx
language:xxx
# 最新更新时间晚于YYYY-MM-DD
铺设的:>2020-05-20

```

