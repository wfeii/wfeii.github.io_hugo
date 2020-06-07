---
layout: single
title: "ContentProvider"
categories:
 - 外功招式
 - Android
 - Android基础

author: 王飞

date: 2016-04-20 00:00:00
---

####  **什么是Android中的Content Provider**

Content Provider是用于管理访问结构化数据集的一种组件，最经常管理数据集是数据库。content provider会封装数据，提供数据安全的机制。content provider用于一个进程与另一个运行中的进程的数据连接。  	  
**Notice:**   
当不打算共享数据给其他应用时，不需要定义自己的Content Provider.   

#### **Content Provider如何表示数据的呢？**
Content Provider呈现数据是通过一个或多个表，类似与数据库，使用Content Provider跟使用数据库相类似。  

#### **如何访问Content Povider?**
Content Provider的访问很简单  
1. 定义Uri  
2. 获取ContentResolver对象  
3. 使用ContentResolver对象进行相关的操作。  
4. 配置访问时要求的权限  


#### **如何定义Uri，以及Uri到底是做什么用的？**
Uri是URI的引用，URI是Uniform Resource Identifier(统一资源标识符)简称，用于标识某一资源名称的,
组成形式：协议://authority／path。  
ContentResolver使用的Uri是用于标识内容提供者的数据，组成形式：content://authority/path如果使用Content Provider共享的数据是数据库，那么path就是操作的table  
```
/*填入要操作的数据的内容提供者的字符串*/
Uri uri = Uri.parse(" ");
```


#### **ContentResolver能够进行哪些相关的操作**  
可能（这里使用可能是因为要看Content Provider具体实现，也许访问的Content Provider只支持其中部分操作）支持的操作如下：
1. insert
2. delete
3. update
4. query   

#### **如何insert插入数据？**  

可以像这样：

{% highlight ruby %}
/*1. 定义要操作的Uri（记得初始化）*/
Uri mUri;
/*2. 定义ContentValues对象，用于保存要插入的数据*/
ContentValues contentValues= new ContentValues();
contentValues.put();
contentValues.pust();
...
/*3 获取ContentResolver对象*/
ContentResolver resolver = getContentResolver();
/*4 插入数据*/
resolver.insert(mUri,contentValues);
{% endhighlight %}  

- ####**如何delete数据？**  
- {% highlight ruby linenos %}
/*1. 定义要操作的Uri（记得初始化）*/
Uri mUri;
/*2. 定义ContentValues对象，用于保存要插入的数据*/
...
/*3 获取ContentResolver对象*/
ContentResolver resolver = getContentResolver();
/*4 插入数据*/
resolver.delete(mUri, where, selectionArgs);
{% endhighlight %}
