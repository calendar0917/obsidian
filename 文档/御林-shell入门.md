---
title: "御林招新题：shell 入门"
subtitle: "御林招新题：shell 入门"
summary: "学习 shell 脚本的编写"
description: "学习 shell 脚本的编写"
image: ""
date: 2025-10-18
lastmod: 2025-10-18
draft: false
toc:
 enable: true
hiddenFromHomePage: True
weight: false
categories: ["CTF"]
tags: ["CTF"]
---

## **重定向与管道**

- 了解标准输入、标准输出、标准错误的概念。

> - **标准输入（Standard Input，stdin）**：默认是键盘，是程序获取输入数据的地方。例如，`cat`命令如果没有指定文件，就会从标准输入读取内容，此时你可以在键盘上输入文字，输入结束后按`Ctrl + D`（类 Unix 系统）表示输入结束，`cat`会将输入的内容输出。
> - **标准输出（Standard Output，stdout）**：默认是终端屏幕，是程序输出正常结果的地方。比如执行`ls`命令，会在终端显示当前目录下的文件和文件夹列表。
> - **标准错误（Standard Error，stderr）**：默认也是终端屏幕，用于输出程序的错误信息。例如，执行`ls /nonexistent`（假设`/nonexistent`是不存在的目录），会在终端显示类似`ls: cannot access '/nonexistent': No such file or directory`的错误信息，这就是标准错误输出。

- 使用 `>` 和 `>>` 操作符将命令的输出重定向到文件。

> - `>`操作符：用于将命令的标准输出重定向到文件，如果文件已存在，会**覆盖**文件原有内容；如果文件不存在，会创建新文件。
> - `>>`操作符：用于将命令的标准输出以**追加**的方式重定向到文件，即把输出内容添加到文件原有内容的末尾，不会覆盖原有内容。

- 使用 `2>` 或 `2>>` 操作符将标准错误重定向到文件。

```shell
[root@localhost hello]# ls /invalid 2> error.txt
[root@localhost hello]# ls /invalid_2 2>> error.txt
[root@localhost hello]# cat error.txt
ls: cannot access /invalid: No such file or directory
ls: cannot access /invalid_2: No such file or directory
```

- 使用 `|` 管道操作符将一个命令的输出作为另一个命令的输入。

```shell
[root@localhost hello]# ls | grep 'txt'
digit_count.txt
error.txt
```

## **变量与引号**

- 学习如何在 Shell 中定义、引用和取消定义变量。

> - **定义变量**：基本语法是：`变量名=值`，等号两边不能有空格。
>
> - **引用变量**：使用 `$变量名` 或者 `${变量名}` 的形式。
>
> - **取消定义变量**
>
> 使用 `unset` 命令可以取消定义变量，语法为：`unset 变量名`。

- 理解单引号 (`'`)、双引号 (`"`) 和反引号 (``) 的区别与作用。

> - **单引号（`'`）**：用于包裹字符串，单引号内的所有内容都被视为普通字符，不会进行变量替换、命令替换等操作。
>
> - **双引号（`"`）：**双引号内会进行变量替换和命令替换（需要结合`$()`或反引号）
>
> - **反引号（\``）**：反引号主要用于命令替换，即执行反引号内的命令，并将命令的输出作为结果返回。不过现在更推荐使用`$()`来进行命令替换，因为它的可读性更好，而且可以嵌套使用（反引号嵌套使用比较复杂）。

## **参数与条件判断**

- 了解如何获取命令行参数（例如 `$1`, `$2`, `$#`）。

> - **`$1`、`$2`、`$3`……**：分别表示第 1 个、第 2 个、第 3 个…… 命令行参数。
> - **`$#`**：表示命令行参数的个数。
> - **`$0`**：表示脚本本身的名称。
> - **`$\*`**：把所有的命令行参数当作一个整体的字符串
> - **`$@`**：把每个命令行参数当作独立的字符串

- 掌握基本的条件判断语句（`if` 语句）。

```bash
if [条件1]; then
    # 条件1为真时执行的命令
elif [条件2]; then
    # 条件2为真时执行的命令
else
    # 所有条件都为假时执行的命令
fi
```

- 使用 `test` 或 `[]` 进行文件、字符串或数字的比较。

> `test` 命令和 `[]`（方括号）在 Shell 中用于进行条件测试，可以对文件、字符串、数字等进行比较。`[]` 是 `test` 命令的另一种写法，使用起来更简洁。
>
> - 文件：
>   - -e 存在、-f  普通文件、-d 目录、-r 可读、-w 可写、-x 可执行
> - 字符串：
>   - = 相等、！=不等、-z 长度是否为0、-n 长度是否不为零
> - 数字：
>   - -eq 相等、-ne 不相等、-gt 大于、-ge 大于等于、-lt 小于、-le 小于等于

## **循环**

- 学习 `for` 循环和 `while` 循环的基本用法。

```bash
for 变量 in 元素1 元素2 元素3 ...; do
    # 循环体，对每个元素执行的操作
done

for 变量 in {起始值..结束值}; do
    # 循环体
done
```

```bash
while 条件; do
    # 循环体，条件为真时执行的操作
done
```

- 能够编写简单的循环脚本来处理文件列表或执行重复任务。

## **编写一个 Shell 脚本**，实现以下功能：

1. **参数检查**：
   - 检查脚本是否接收到至少一个命令行参数。如果没有，打印使用说明（例如：`Usage: ./file_processor.sh <directory>`）并退出。

2.  **目录遍历**：
   - 脚本接收一个目录路径作为参数。进入该目录后，使用 `for` 循环遍历目录下的所有文件。

3. **文件处理**：

   - 对于遍历到的每一个文件，进行以下判断：

   - 如果文件是**普通文件**（非目录），则将该文件的文件名（不包含路径）打印到标准输出。

   - 如果文件是**可执行文件**，则在文件名后面追加 `(Executable)` 字样。

4. **结果输出**：

   - 将所有被处理的文件的输出结果（包含可执行标记）重定向到一个名为 `processed_files.txt` 的新文件中。

   - 最后，打印一句话，提示用户处理已完成，并说明结果已保存在 `processed_files.txt` 中。 

```bash
#!/bin/bash

# 判断是否未传入任何参数
if [ -z "$*" ]; then
    echo 'Usage： ./file_processor.sh <directory>'
    echo '是不是没有写文件路径？'
# 判断传入的参数是否为目录
elif [ -d "$1" ]; then
    # 遍历目录下的所有文件
    for file in "$1"/*; do
        # 判断是否为普通文件
        if [ -f "$file" ]; then
            echo -n "$file"
            echo -n "$file" >> processed_files.txt
            # 判断文件是否可执行
            if [ -x "$file" ]; then
                echo -n 'Executable'
                echo -n 'Executable' >> processed_files.txt
            fi
        fi
        echo '' # 美化的作用
    done
    echo '批处理已完成，结果已保存在 processed_files.txt 中'
    echo '-----------------'
else
    echo '是不是文件夹路径输错了？'
fi
```

> 写错的点：
>
> - `[ -z $* ]`，要先写 `-<...>`
> - `$file` 忘记改单引号为双引号了
> - `$1` 只是目录路径，`"$1"/*` 才是目录下所有文件
> - `echo` 默认带换行，添加 -n 参数取消

结果如下图：

![image-20251018185932433](https://raw.githubusercontent.com/calendar0917/images/master/image-20251018185932433.png)

> 又发现 .txt 中没有换行，得添加一下手动换行