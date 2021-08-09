# Bash删除字符串中的第一个和最后一个字符 - Ubuntu问答
## 问题描述

我有一个这样的字符串：

```null
|abcdefg|
```

我想要得到一个新的字符串 (如 string2) 与原始字符串调用没有两个 | 在开始和结束时

所以我会有这个

```null
abcdefg
```

在 bash 中可能吗？

## 最佳解决方法

你可以做

```null
string="|abcdefg|"
string2=${string
string2=${string2%"|"}
echo $string2
```

或者如果你的字符串长度不变，你可以做

```null
string="|abcdefg|"
string2=${string:1:7}
echo $string2
```

此外，这应该工作

```null
echo "|abcdefg|" | cut -d "|" -f 2
```

另外这个

```null
echo "|abcdefg|" | sed 's/^|\(.*\)|$/\1/'
```

## 次佳解决方法

这是一个独立于字符串长度 (bash) 的解决方案：

```null
string="|abcdefg|"
echo "${string:1:${#string}-2}"
```

## 第三种解决方法

在这里列出的几个帖子看起来最简单的方法是：

```null
string="|abcdefg|"
echo ${string:1:-1}
```

编辑：工程与 bash 4.2 Ubuntu 的; 不能用 bash 4.1 在 centOS 上工作

## 第四种方法

另一个：

```null
string="|abcdefg|"
echo "${string//|/}"
```

## 第五种方法

另一种方式与头 & 尾巴：

```null
$ echo -n "|abcdefg|" | tail -c +2 | head -c -1
abcdefg
```

## 第六种方法

您也可以使用 sed 删除 | 不只是引用符号本身，而是使用位置引用，如下所示：

```null
$ echo "|abcdefg|" | sed 's:^.\(.*\).$:\1:'
abcdefg
```

如果’:’是分隔符 (你可以用 / 或者不是在查询中的任何字符替换它们，任何跟在 s 后面的符号都会这样做) 这里 ^(caret)表示输入字符串的开头，$(dollar)表示在结束。这个。 (点)它在插入符号之后，并且在美元符号代表单个字符之前。换句话说，我们正在删除第一个和最后一个字符。请记住，即使 | 也会删除任何字符它不存在于字符串中。

EX：

```null
$ echo "abcdefg" | sed 's:^.\(.*\).$:\1:'
bcdef
```

## 参考资料

-   [Bash remove first and last characters from a string](https://askubuntu.com/questions/89995/bash-remove-first-and-last-characters-from-a-string) 
    [https://ubuntuqa.com/article/908.html](https://ubuntuqa.com/article/908.html)
