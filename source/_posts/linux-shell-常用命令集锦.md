---
title: Linux shell 常用命令集锦
date: 2021-08-31 13:50:48
tags:
  - Linux
  - shell
---

## 常用指令

### sort

`sort [OPTION]... [FILE]...`

对输入进行**排序**

常用的参数:

* `-n` 使用数值进行排序
* `-u` 去重，相同的情况仅仅输出第一个
* `-r` 从大到小排序
* `-k` 指定一个特定的key进行排序，而不是整一行
  * 比如想要使用空格作为分隔符，然后第二个作为数字键，那就使用`sort -k 2n`
  * 如果要指定指定其他分隔符 ，可以使用-t参数，例如使用冒号作为分隔符，`sort -t: -k 3n /etc/passwd`

### awk

```bash
awk '
BEGIN { actions } 
/pattern/ { actions }
/pattern/ { actions }
……….
END { actions } 
' filenames  
```

 BEGIN表示在所有匹配前先做一些事情，END表示在所有匹配结束后再做一些事情

例如有下面的文件a.txt，像求出水果的平均价格，可以这么写`awk '/apple/{cnt+=1;sum+=$2}END{print sum/cnt} a.txt' `

```tex
apple 3.5
peach 4.6
apple 4.7
pear 5.3
```

要点：

* 如果需要改变分割符，可以使用`-F`参数，举例：`awk -F ',' '{print $NF}' filename`

* awk内有特殊变量，常见的有

  | 特殊变量 | 描述                                                         |
  | -------- | ------------------------------------------------------------ |
  | \$number  | 表示记录的字段。比如，\$1表示第1个字段，\$2表示第2个字段，如此类推。而\$0比较特殊，表示整个当前行 |
  | FS       | 表示字段分隔符                                               |
  | NF       | 表示当前记录中的字段数量                                     |
  | NR       | 表示当前记录的编号（awk将第一个记录算作记录号1）             |


### uniq

一般与sort一起使用，常用以下参数

* -c 对排序后的数据进行计数
* -d 对排序后的数据只统计重复过的数据
* -u 对排序后的数据去重

### xargs

`xargs [options] [command [initial-arguments]]`

对标准输入的数据进行处理后作为参数给command，如果没有command就直接将参数输出到标准输出流

 常用参数：

* -n max-args 设置执行命令的最大参数个数

### head

取文件头10行

* -n [-]NUM

  取前NUM行，有-就表明输出开头到最后倒数第NUM行

### tail

取文件尾10行

* -n [+]NUM

  取后NUM行，有+就表明从第NUM开始输出到最后

## 经典问题与解答

1. 对输入文件里面的所有单词进行统计个数，并且按照单词个数从大到小进行输出

   * 第一种

     `cat words.txt | xargs -n 1 | sort | uniq -c | sort -nr | awk '{print $2" "$1}'`

   * 第二种

     ```bash
      awk '{
          for(i = 1; i <= NF; i++){
              res[$i] += 1
          }
      }
      END{
          for(k in res){
              print k" "res[k]
          }
      }' words.txt | sort -nr -k2
     ```

       注意到awk内置有执行控制语句，以及数组类型，字符串可以作为key