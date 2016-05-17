---
layout: post
title: Shell脚本中处理参数Options
comments: true
tags: [Shell,Shell脚本]
header-img: img/post-bg-js-module.jpg
---

对于Shell脚本的只读参数，Shell只把它当作普通只读数组，放在变量$@中。

```python

$@	#参数列表(数组)
$*	#参数串接字符串(空格隔开)
$#	#参数个数(实际个数)
$0	#脚本文件名
$1-$9	#第一到第九个参数(大于9的无法通过位置直接访问)
$$	#进程ID
$?	#上条命令的返回状态

```

使用Shell脚本参数可以直接用上述方式，然而由于位置访问方式只支持1－9参数，因此需要使用shift来转换。

####shift

shift提供对只读的位置参数的移位赋值的操作，将1=\$2，2=\$3，…，可以使用shift N来制定移位的数目，例如shift 3，则表示1=\$4，2=\$5，…。如果命令行中有[-options]的，我们可以对他们进行判断，并进行移位处理。一个简单例子如下：

```python

if [ $1 = -o ]; then 
    #process the -o option 
    shift 
fi 
#normal processing of arguments...

```

使用shift处理参数通常的做法如下：

```python

while [ -n "$(echo $1 | grep '-')" ]; do 
    case $1 in 
        -a ) [process option -a] ;; 
        -b ) [process option –b, $2 is the option's argument]   
              shift ;; 
        -c ) [process option -c] ;; 
        *  ) echo 'usage: alice [-a] [-b barg] [-c] args...' 
             exit 1 
    esac 
    shift 
done 
[normal processing of arguments...]

```

但是这种方式不能解决下面两个问题：一、无法使用-abc来代替-a –b -c，二、无法使用-barg来代替-b arg，但是这两种方式在Linux命令中都是比较常见的。我们可以使用getopts这个built-in的命令来处理。

####getopts
getopts有两个参数，第一个参数是一个字符串(optstring)，包括字符和“：”，每一个字符都是一个有效的选项，如果字符后面带有“：”，表示这个字符有自己的参数。getopts从命令中获取这些参数，并且删去了“-”，并将其赋值在第二个参数中，如果带有自己参数，这个参数赋值在“OPTARG”中。

1. optstring;可以选择是否“：”开头可以如果我们使用"ab:c"，如果option并非所需，则报告illegal option —d之类的，如果我们去掉相关信息，getopts的第一个参数以":"开头即可。当然也可以通过设置环境参数OPTERR为0，如果不匹配，则将获取的参数值置为"?"，在下面中case的第四个选择可以为/?)或者*)。
2. OPTARG;
3. OPTIND.
