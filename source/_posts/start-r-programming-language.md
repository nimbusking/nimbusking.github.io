---
title: R语言统计分析实战（RStudio）
tags:
  - 统计学
  - R语言
  - RStudio
abbrlink: a93eddc5
updated: 2024-10-28 20:30:21
categories: 统计分析
mathjax: true
date: 2019-12-27 21:42:17
---

### 背景
一位好友找我，问我有没有弄过RStudio，以前了解过R语言，用于统计分析的，但是实际没有实战过。所以嘛，就准备了一下，记录一下探索过程。 :)
### 前言
本片文章简单介绍一些RStudio相关以及少量统计学相关的知识，供上手学习使用。毕竟是一门编程语言，语法看了一下，不是很复杂，具备一定C语言即可，一些代码片段就可能看懂了。
如有相关错误，欢迎评论区留言指正，谢谢。
本文相关运行环境为：
- OS：Microsoft Windows [版本 10.0.18363.535]
- 内存：32GB
- R版本：3.6.2
- RStudio版本：1.2.5033.0
【注】 下文相关截图等数据均基于上述软件版本，如后期版本更新等，类推即可。

### 统计学相关理论（简要）
### 相关概念介绍
#### 什么是R语言
R是R统计计算基金支持的用于统计计算，图形编程的免费软件环境。R语言在统计学家和数据挖掘者中广泛用于数据分析软件和数据挖掘。数据挖掘和学术文献数据库研究表明其普及程度大幅度提高。截至到2019年11月，在TIOBE排行[一个知名的编程语言排行榜，用于衡量编程语言的热度]中，R语言排名第16位。<sup>[1]</sup>
#### 什么是RStudio
RStudio是R语言的一款免费的IDE编写工具。为计算环境提供最灌灌的使用的开源和企业就绪型专业软件。提供了脚本调试、可视化等功能，支持纯R脚本、Rmarkdown（脚本文档混排）、Bookdown（脚本混排混排成书）、Shiny（交互式网络应用）等功能。<sup>[2]</sup>
#### R和RStudio之间的关系
R是一个基础的环境程序，是运行R语言脚本必须的基础运行时环境。类似的像其它的编程语言，类如：Java、C、Python等等，必须安装相关的运行时环境一样，否则你无法直接运行的。而RStudio只是一款可视化编辑工具而已。
<!-- more -->
### 环境准备
#### 安装R
##### 下载R安装包
1. 打开R的官网：https://www.r-project.org/
2. 在左侧菜单栏中找到下载(download)，点击 *CRAN*，打开镜像网站列表页面，如下图所示：
![R语言主站](a93eddc5/mainsite_of_r_project.jpg)
3. 找到中国站点，列了很多国内大学及机构的站点，像清华、中科大、同济，随笔找一个点击连接，如下图所示：
![下载](a93eddc5/mirror_site_of_r_download_page.jpg)
4. 选择指定平台的R语言平台，如果是windows的就选择 *Download R for Windows*就是，如下图所示：
![windows](a93eddc5/different_platform_of_r.jpg)
5. 选择R的版本，直接单击*base*即可
![version](a93eddc5/different_version_of_r.jpg)
6. 直接单击*Download R 3.6.2 for Windows*开始下载

##### 安装
基本一路 **next**下去即可，有三步提一下
1. 安装路径，默认C盘，选择你指定的盘路径即可：
![安装路径](a93eddc5/install_select_install_path.jpg)
2. 选择组件，默认全部勾选即可
3. 启动选项，选择第二个：*No(接受默认选择)*即可
![启动选项](a93eddc5/install_start_option.jpg)

#### 安装RStudio
##### 下载RStudio安装包
1. 打开RStudio官网：https://rstudio.com/products/rstudio/download/
2. 点击页面中 *Free*下的 *DOWNLOAD*按钮，跳转到下载页面
![下载rstudio](a93eddc5/download_site_of_rstudio.jpg)
3. 选择你需要的版本，如windows版本
![选择指定版本](a93eddc5/select_different_version_of_rstudio.jpg)

##### 安装
过程很简单，除了第一步你需要选择安装路径之外，其余默认即可，过程3步就可以安装完成。
此过程就不贴安装过程了。
##### 基本运行调试
运行RStudio（页面没有快捷方式的话，去开始菜单栏里找一下），如下图所示：
![主界面](a93eddc5/rstudio_main_page.jpg)
在左侧的 *Console*选项卡下，闪烁的光标后输入：*1+1*后，按键盘 *回车(Enter)*，如果下一行显示[1] 2，证明安装好了。

### R语言语法操作相关
本语法小节参考R语言官方手册<sup>[3]</sup>中关于语法相关(基本上就是翻译了一下)，更多相关介绍直接请查阅手册其余内容。
#### 基础语法
##### 基础运算符(Simple manipulations)
##### 对象模型和属性(Objects, their modes and attributes)
##### 有序和无序操作符(Ordered and unordered factors)
##### 数组和矩阵(Arrays and matrices)
##### 列表和数据片段(Lists and data frames)
#### 基础操作
##### 从文件读取数据(Reading data from files)
##### 可能的数据分布(Probability distributions)
##### 数据分组、循环和条件运算(Grouping, loops and conditional execution)
##### 编写自己的运算函数(Writing your own functions)
##### R中的统计模型(Statistical models in R)
##### 图形处理(Graphical procedures)
### RStudio使用教程
#### 基本界面介绍
#### 基础篇
#### 进阶篇
### 样例实战
#### 样例1：一例黑客团队邮箱列表分析
原始参考案例：http://rpubs.com/F789GH/HackingTeamEmails
##### 原文分析目的
- 统计列表中所有邮箱域名个数
- 统计邮箱域名所属区域，分析这些黑客邮件地址都来自什么地方
##### 获取数据源
直接从这个链接：[黑客邮箱列表](https://wikileaks.org/hackingteam/emails/emailid/144932)，中获取黑客邮箱名单。
Copy邮箱列表到txt文件中，保存准备处理。需要简单处理一下里面的邮箱地址，列表中有#号和空白行等等。
处理后的列表文件，https://github.com/nimbusking/start_r_programming/blob/master/hack_team_email_list.txt
##### 开始分析
###### 引用必要的类库
分析过程中使用了一些函数，因此需要引用相关类库
```R
library(rvest)
# ggplot2绘图函数包
library(ggplot2)
library(stringi)
library(magrittr)
library(plyr)
library(dplyr)
```
第一次运行，控制台中键入上述代码后，会自动联网下载相应类库并安装
##### 分析汇总域名频率分布柱状图
分析完成后的展示，如下图所示：
![柱状图1](a93eddc5/Histogram_email_domain_frequence.jpg)
##### 分析域名对应国家出现频率
分析完成后的展示，如下图所示：
![柱状图2](a93eddc5/Histogram_email_domain_site_frequence.jpg)
##### 结论
排名第一的居然是it（意大利），其次是德国、英国、美国、捷克...
##### 完整代码
```R
# 导入函数库
library(rvest)
library(ggplot2)
library(stringi)
library(magrittr)
library(plyr)
library(dplyr)
# 从txt数据源中读取数据到变量email_list中
email_list <- read.table("hack_team_email_list.txt", header = T)
# 打印前6行数据
head(email_list)
# 按符号@分割电子邮件地址到变量splitted_email中
splitted_email <- data.frame(stri_split_fixed(tolower(email_list$V1), "@", 2, omit_empty = NA, simplify = TRUE))
# 复制完整邮箱名到第三列
splitted_email$X3 <- paste(tolower(splitted_email$X1), tolower(splitted_email$X2), sep = "@")
head(splitted_email)
# 绘制域名频率柱状图
## 表格转换数据帧
splitted_email_freq <- as.data.frame(table(splitted_email$X2))
## 调用plyr函数包中的arrange方法进行排序
splitted_email_freq <- plyr::arrange(splitted_email_freq, desc(splitted_email_freq$Freq))
head(splitted_email_freq)
## 绘制柱状图
splitted_email_freq_plot <- ggplot(data=splitted_email_freq[1:20,], aes(x=reorder(Var1, Freq), y=Freq))
## 设置图形展示样式
splitted_email_freq_plot <- splitted_email_freq_plot + geom_bar(stat = "identity") + coord_flip(ylim= c(0,430)) + theme(
  axis.text.y = element_text(size = 12), 
  axis.text.x = element_text(size = 12))
## 设置图形标题横纵坐标
splitted_email_freq_plot <- splitted_email_freq_plot + xlab("域名") + ylab("各域名提供商所拥有的黑客数量") + ggtitle("从黑客邮件列表中能发现多少域名提供商？")
## 图形坐标范围
splitted_email_freq_plot <- splitted_email_freq_plot + scale_y_continuous(breaks = seq(0, 430, 30))
## 生成图形
splitted_email_freq_plot

# 绘制国家出现频率
## 先过滤一些公共域名
blackListDomains <- c("outlook|gmail|yahoo|live|hotmail|googlemail")
## 过滤列表域名列表
splitted_email_freq_bld <- splitted_email[!stri_detect_regex(splitted_email$X2, blackListDomains), ]
## 转换X2为数据帧
splitted_email_freq_bld <- as.data.frame(table(splitted_email_freq_bld$X2))
## 排序
splitted_email_freq_bld <- plyr::arrange(splitted_email_freq_bld, desc(splitted_email_freq_bld$Freq))
## 打印前15行
head(splitted_email_freq_bld, 15)
# 处理国家归属
domain_list <- data.frame(stri_split_fixed(tolower(splitted_email_freq_bld$Var1), ".", -1, omit_empty = NA, simplify = TRUE))
domain_list[domain_list == ""] <- NA
domain_list$X1 <- as.character(domain_list$X1)
domain_list$X2 <- as.character(domain_list$X2)
domain_list$X3 <- as.character(domain_list$X3)
domain_list$X4 <- as.character(domain_list$X4)
domain_list$X5 <- as.character(domain_list$X5)
domain_list$new_NEW <- domain_list[-1][cbind(1:nrow(domain_list), max.col(!is.na(domain_list[-1]), ties.method = "last"))]
## 合并行
domain_list$new_NEW <- domain_list[-1][cbind(1:nrow(domain_list), max.col(!is.na(domain_list[-1]), ties.method = "last"))]

## freq again
domain_list <- as.data.frame(table(domain_list$new_NEW))
domain_list <- plyr::arrange(domain_list, desc(domain_list$Freq))
domain_list <- domain_list[!stri_detect_regex(domain_list$Var1, "com|org|net|edu|gov|mil"), ]
## 打印前10行
head(domain_list, 10)
## 绘制域名
domain_list_plot <- ggplot(data=domain_list[1:20,], aes(x=reorder(Var1, Freq), y=Freq)) 
domain_list_plot <- domain_list_plot + geom_bar(stat = "identity") + coord_flip(ylim= c(0,100)) + theme(
  axis.text.y = element_text(size = 14), 
  axis.text.x = element_text(size = 14)) 
domain_list_plot <- domain_list_plot + xlab("域名Top数") + ylab("特殊域名数") + ggtitle("从黑客邮件列表中能发现多少域名分布？")
domain_list_plot <- domain_list_plot + scale_y_continuous(breaks = seq(0, 100, 5))
## 生成图形
domain_list_plot

# 从图形中看，cz（捷克Czech Republic）归属的比较特别，统计看一下
splitted_email_czech <- splitted_email[grepl(".cz", splitted_email$X2), ]
head(splitted_email_czech, 10)


```

#### 样例2：贝叶斯统计理论探讨
原文参考：https://www.statsjoke.me/posts/philosophy-of-bayesian-statistics/

##### 概念相关
###### 贝叶斯定理
![贝叶斯定理](a93eddc5/Bayes_Theorem.jpg)

###### 伯努利分布
![伯努利分布](a93eddc5/Bernoulli_distribution.jpg)

###### 二项式定理
![二项式定理](a93eddc5/Binomial_pdf.jpg)

###### β分布
![β定理](a93eddc5/Beta_pdf.jpg)

##### 探究课题
通过模拟硬币反转来绘制这项后验概率的更新图
代码片段如下
```R
# 设置随机种子
set.seed(1234)
# 随机样本次数
n_trials <- c(0, 1, 2, 3, 4, 5, 8, 15, 50, 500, 1000, 2000, 5000)
# 设置二项分布
data <- rbinom(n_trials[length(n_trials)], 1, 0.5)
x <- seq(0, 1, length.out=100)
par(mfrow=c(1, 2))
for (N in n_trials) {
  heads <- sum(data[1:N])
  # β
  y <- dbeta(x, 1+heads, 1+N-heads)
  plot(x, y, type = 'l', col='blue')
  legend('topright', sprintf("observe %d tosses,\n %d heads", N, heads), bg='transparent', box.lty=0)
  # 绘制斜线
  abline(v=0.5, col="red", lty=2, lwd=1)
}
```
观察每次样本数后的的分布图
![1](a93eddc5/AutoCapture_2019-12-28_014018.jpg)
![2](a93eddc5/AutoCapture_2019-12-28_014024.jpg)
![3](a93eddc5/AutoCapture_2019-12-28_014028.jpg)
![4](a93eddc5/AutoCapture_2019-12-28_014034.jpg)
![5](a93eddc5/AutoCapture_2019-12-28_014039.jpg)
![6](a93eddc5/AutoCapture_2019-12-28_014044.jpg)
![7](a93eddc5/AutoCapture_2019-12-28_014049.jpg)

可以看出，随着样本数的不断增加，可以得出概率越来越趋近p = 0.5，尽管不是全部。

一个简单的贝叶斯推断：
- 先验概率：P(A) = p
- 后验概率：P(A|X)
- P(X|A)
- P(X)

推理如下图所示：
![Bayesian Inference1](a93eddc5/Bayesian_Inference1.jpg)

近似为：P(X| ~ A) = 0.5，因此：
![Bayesian Inference2](a93eddc5/Bayesian_Inference2.jpg)

演示代码为：
```R
p <- seq(0, 1, length.out = 50)
plot(p, 2*p/(1+p),col='blue',type='l',xlab = 'Prior',ylab = 'Posterior')
```
RStudio中演示为
![Bayesian Inference3](a93eddc5/Bayesian_Inference3.jpg)

##### 泊松分布
###### 概率质量函数
![probability mass function](a93eddc5/probability_mass_function.jpg)

示例代码：
```R
a <- 0:15
lambda <- c(1.5, 4.25)
barplot(dpois(a, lambda[1]), names.arg = a, col = '#348ABD')
barplot(dpois(a, lambda[2]), names.arg = a, col = '#A60628', add = T)
legend('topright', c('lambda=1.5', 'lambda=4.25'), fill = c('#348ABD', '#A60628'))
```

![Poisson Distribution](a93eddc5/Poisson_Distribution.jpg)

##### 指数分布
###### 概率密度函数
![probability density function](a93eddc5/probability_density_function.jpg)

示例代码：
```R
a <- seq(0, 4, length.out = 100)
lambda <- c(0.5, 1)
plot(a, dexp(a, 0.5), col='#348ABD', type = 'l', xlab = 'z', ylab = 'PDF at z', ylim = c(0,1))
lines(a, dexp(a, 1), col='#A60628')
```

![Exponential Distribution](a93eddc5/Exponential_Distribution.jpg)

### 引用
[1] R (programming language):[EB/OL]. 2004.10[2019-12]. https://en.wikipedia.org/wiki/R_(programming_language)
[2] About RStudio:[EB/OL]. [2019-12]. https://rstudio.com/about/
[3] An Introduction to R:Notes On R: A Programming Environment for Data Analysis And Graphics.[EB/OL].2019.12.12[2019-12]. https://mirrors.tuna.tsinghua.edu.cn/CRAN/doc/manuals/r-release/R-intro.pdf