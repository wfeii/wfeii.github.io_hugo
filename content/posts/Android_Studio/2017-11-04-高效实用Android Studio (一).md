---
layout: single
title: "高效实用Android Studio (一)"
categories:
 - Android Studio
tags:
  - Android Studio
date: 2017-11-04
toc: true
toc_label: "高效实用Android Studio"
---

![logo](/assets/images/AndroidStudio/android-studio-logo.png){: .align-center}

## 快捷键  

设置Preferences -> Keymap 设置快捷键，可以修改，或者添加keymap等等。

![kemap](http://img.blog.csdn.net/20170719202536443?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvd2ZlaWk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

常用快捷键，每个系统及其设置不同，这里只是说出对应的名称，这样就可以直接搜索来查找快捷键。

- Run  运行安装APK
- Find
     1.  Find 在本文件内查找
        2. Find in Path 在工程中查找
        3. Find Usages 查找引用
- Rename 重命名
- Sync Project with Gradle Files 同步Gradle
- Generate 生成代码或者字段的get或者set方法等等
- File Structure 查看类的方法等
- Type Hierarchy 查看继承关系
- Editor Tabs
   1. Select Next Tab 选择下一个编辑窗口
   2. Select Previous Tab 选择上一个编辑窗口
   3. close 关闭一个编辑窗口
   4. close Others 关闭其他窗口
   5. close All 关闭所有的窗口


## 代码格式  

设置Tab的信息，Use tab character,未勾选将会使用空格代替Tab。
![设置Tab](http://img.blog.csdn.net/20170719202621693?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvd2ZlaWk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

经常的我们会有设置类的字段的命名m开头，我们就可以在Code Style -> Java -> Code Generation路径下，使用Name Prefix设置前缀，也可以使用Name suffix也可以设置后缀。
![设置字段前缀](http://img.blog.csdn.net/20170719202708883?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvd2ZlaWk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

设置文件的特定内容，比如作者等等。可以设置自己的代码风格。
![File Header](http://img.blog.csdn.net/20170719202744623?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvd2ZlaWk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


## 技巧  

设置Live Templates，添加模板代码。
![Live Templates](http://img.blog.csdn.net/20170719202818298?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvd2ZlaWk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

1. 添加并勾选
2. 设置abbreviation，当输入的时候会有补全，输入完成就会有响应的代码生成
3. 设置描述
4. 设置代码的内容
5. 设置什么时候会提醒，根据自己的需要设置相关的功能
