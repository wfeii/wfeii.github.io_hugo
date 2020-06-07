---
layout: single
title: "menu和action bar"
categories:
 - 外功招式
 - Android
 - Android基础

date: 2016-2-22 16:15:00
---

从android3.0开始传统的6-items的菜单被action bar的形式所取代，在这篇文章中，将使用action bar方式实现menu，并且使用的是支持库的方式，这样可以解决版本兼容问题。  

### 三种菜单类型  
- options menu  
- context menu  
- popup menu  

### 如何定义menu  
- 在res/menu/下使用xml中定义menu item, 代码中填充菜单  
- 直接在代码中创建menu，添加menu条目  

> **NOTICE：** 最好使用menu资源来定义menu items  

> - It's easier to visualize the menu structure in XML.  
> - It separates the content for the menu from your application's behavioral code.  
> - It allows you to create alternative menu configurations for different platform versions, screen sizes, and other configurations by leveraging the app resources framework.  

<!-- more -->

下面就是不同类型menu的创建方式  

###  创建options menu  
options menu既可以在activity子类中定义，又可以在fragment子类中定义，如果两个同时定义了menu，那最后的menu形式将是二者的结合。activity的menu items先出现，然后是根据fragment在activity添加的次序来添加它们的menu items。  

源码可以查看[Androidbase/UI/](https://github.com/wangfei1991/Androidbase/tree/master/UI/app/src/main/java/cn/edu/wyu/ui/menu_actionbar)下OptionsMenuUsingActionbar相关的代码  

-  继承v7ActionBarActivity,在activity覆写onCreateOptionsMenu（）方法（对于Fragment不仅要覆写onCreateOptionsMenu()方法,而且要在onCreate()中调用 setHasOptionsMenu(true),否则将不能显示）
```
         @Override
    public boolean onCreateOptionsMenu(Menu menu) {  
        // 使用menu资源来填充menu  
        getMenuInflater().inflate(R.menu.menu_refresh_setting_menu, menu);  
        //动态的menu item,并以action bar形式显示  
        MenuItem locationMenuItem = menu.add(0,R.id.menu_location,0,"Location");  
        locationMenuItem.setIcon(R.drawable.ic_action_location);  
        MenuItemCompat.setShowAsAction(locationMenuItem,  
                MenuItemCompat.SHOW_AS_ACTION_IF_ROOM);  
        return true;  
    }
```
- 覆写onOptionsItemSelected()方法,响应item点击事件（Fragment也是覆写onOptionsItemSelected()方法）  

```
 @Override
    public boolean onOptionsItemSelected(MenuItem item) {
        int id = item.getItemId();
        switch (id)
        {
            case R.id.menu_refresh:
                return true;
            case R.id.menu_location:
                return true;
            case R.id.menu_settings:
                return true;
        }

        return super.onOptionsItemSelected(item);
    }
```
-  使用Theme.AppCompat的主题，这里打开了action bar，有的主题把action bar关闭了，一定要注意这一点。   

###  与options menu相关的内容  
下面基本在options menu中使用，在另外两种类型中比较少使用。  


####  添加tabs
使用场景类似于：[点击查看](https://raw.githubusercontent.com/wangfei1991/Blog/gh-pages/img/android/android_knowledge/ui/menu_actionbar/actionbar-tabs-stacked@2x.png)   
源码可以查看[Androidbase/UI/](https://github.com/wangfei1991/Androidbase/tree/master/UI/app/src/main/java/cn/edu/wyu/ui/menu_actionbar)的TabsAndSplitActionBar相关的代码  
使用步骤：

-  实现ActionBar.TabListener接口，用于处理tab的事件  

```
    /*
     * 实现TabListener方法
     */
    @Override
    public void onTabSelected(ActionBar.Tab tab,
	                          FragmentTransaction fragmentTransaction)
    {
        //tab选择时调用这个方法
    }

    @Override
    public void onTabUnselected(ActionBar.Tab tab,
	                            FragmentTransaction fragmentTransaction)
    {
        //tab没有选择后调用的方法
    }

    @Override
    public void onTabReselected(ActionBar.Tab tab,
	                            FragmentTransaction fragmentTransaction)
    {
        //重复选择时调用的方法
    }
```

-  获取action bar添加tabs，并设置ActionBar.TabListener。
```
    ActionBar actionBar = getSupportActionBar();
    actionBar.setNavigationMode(ActionBar.NAVIGATION_MODE_TABS);
    for (int i=1; i<4; i++) {
        actionBar.addTab(actionBar.newTab()
                 .setText("TAB "+i).setTabListener(this));
    }
```


#### 添加Action View  
使用场景类似于：[点击查看](https://raw.githubusercontent.com/wangfei1991/Blog/gh-pages/img/android/android_knowledge/ui/menu_actionbar/actionbar-searchview@2x.png)  
源码可以查看[Androidbase/UI/](https://github.com/wangfei1991/Androidbase/tree/master/UI/app/src/main/java/cn/edu/wyu/ui/menu_actionbar)的OptionsMenuAddActionView相关的代码  
实现的步骤:   

-  在menu资源文件中添加actionViewClass属性，用于声明action view  

```
 <item android:id="@+id/action_search"
    android:title="search"
    android:icon="@android:drawable/ic_menu_search"
    android:orderInCategory="100"
    app:showAsAction="ifRoom"
    app:actionViewClass="android.support.v7.widget.SearchView"
    />
```
-  在onCreateOptionsMenu()中配置action view(例如监听事件)  

```
@Override
    public boolean onCreateOptionsMenu(Menu menu) {
         //使用menu资源来填充menu
         getMenuInflater().inflate(
		                R.menu.menu_options_add_action_view,menu);
         MenuItem menuItem = (MenuItem) menu.findItem(R.id.action_search);
         //在android11以后，可以使用menuItem.getActionView()的方式获取action iew
         SearchView searchView = (SearchView)
                                  MenuItemCompat.getActionView(menuItem);
        /*searchView事件处理*/
        return super.onCreateOptionsMenu(menu);
    }
```
#### 添加Action Provider  
这里以添加ShareActionProvider为例。  
使用场景类似于：[点击查看](https://raw.githubusercontent.com/wangfei1991/Blog/gh-pages/img/android/android_knowledge/ui/menu_actionbar/actionbar-shareaction@2x.png)   
源码可以查看[Androidbase/UI/](https://github.com/wangfei1991/Androidbase/tree/master/UI/app/src/main/java/cn/edu/wyu/ui/menu_actionbar)的OptionsMenuAddActionProvider相关的代码  
实现步骤:  

-  在menu资源的\<item\>定义actionProvider，添加***ShareActionProvider***

```
<item android:id="@+id/action_share"
        android:title="share"
        android:icon="@android:drawable/ic_menu_share"
        android:orderInCategory="100"
        app:showAsAction="ifRoom"        
        app:actionProviderClass=
                 "android.support.v7.widget.ShareActionProvider"
        />
```

-  在onCreateOptionsMenu( )中获取***ShareActionProvider***，并调用setShareIntent()方法来设置要分享的Intent。  
```
@Override
    public boolean onCreateOptionsMenu(Menu menu) {
        // Inflate the menu; this adds items to the action bar if it is present.
        getMenuInflater().inflate(
		               R.menu.menu_options_add_action_provider, menu);
        MenuItem menuItem = menu.findItem(R.id.action_share);
        ShareActionProvider actionProvider;
        /*获取ShareActionProvider*/
        actionProvider = (ShareActionProvider)
                     MenuItemCompat.getActionProvider(menuItem);
        /*来设置要分享的Inte*/
        actionProvider.setShareIntent(getDefaultInten());
        return true;
    }

    public Intent getDefaultInten()
    {
        Intent intent = new Intent(Intent.ACTION_SEND);
        intent.setType("*/*");
        return intent;
    }
```

#### 添加Drow-down Navigation  
drow-down list使用在内容重要但不频繁改变的情况，对于频繁改变的内容要使用tabs来代替。
源码可以查看[Androidbase/UI/](https://github.com/wangfei1991/Androidbase/tree/master/UI/app/src/main/java/cn/edu/wyu/ui/menu_actionbar)的OptionsMeneAddDropdown相关的代码  
实现步骤：  

1. 创建一个SpinnerAdapter提供drow-down的item，
2. 实现ActionBar.onNavigiationListener定义用户选择item的行为
3. 在Activity的onCreate()的方法中，开启此模式通过调用
4. 获取action bar，调用setListNavigationCallbacks()方法。

```
 @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_options_mene_add_dropdown);
        ActionBar actionBar = getSupportActionBar();

		/*1. 创建一个SpinnerAdapter提供drow-down的item，*/
        spinnerAdapter = new ArrayAdapter<String>(this,
                android.R.layout.simple_spinner_dropdown_item,
                android.R.id.text1,data);

		/*3. 在Activity的onCreate()的方法中，开启此模式通过调用*/
        actionBar.setNavigationMode(ActionBar.NAVIGATION_MODE_LIST);

		/*4. 获取action bar，调用setListNavigationCallbacks()方法。*/
        actionBar.setListNavigationCallbacks(spinnerAdapter,
                                             onNavigationListenerImp );
    }

    /*2. 实现ActionBar.onNavigiationListener定义用户选择item的行为*/
    private ActionBar.OnNavigationListener onNavigationListenerImp =
                              new ActionBar.OnNavigationListener() {
        @Override
        public boolean onNavigationItemSelected(int i, long l) {
             /*这里做相应的选择处理*/             
            Log.e("OptionsMeneAddDropdownActivity","选择"+i+"位置");
            return true;
        }
    };
```


### 参考文档：  
[参考文档1](http://developer.android.com/guide/topics/ui/actionbar.html)  
[参考文档2](http://developer.android.com/guide/topics/ui/menus.html)  
