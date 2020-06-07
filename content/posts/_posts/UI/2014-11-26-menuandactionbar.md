---
layout: single
title: "menu和actionbar"
categories:
 - 外功招式
 - Android
 - Android基础
date: 2016-2-25 16:15:00
---


### context menu的创建

context menu可以使用到任意view上，但通常使用在ListView，GridView的items上，或者其他用户能够执行item操作的view集合。  

有两种方式可以创建context menu  

> - In a ***floating context menu***. A menu appears as a floating list of menu items (similar to a dialog) when the user performs a long-click (press and hold) on a view that declares support for a context menu. Users can perform a contextual action on one item at a time.   
> - In the ***contextual action mode***. This mode is a system implementation of ActionMode that displays a contextual action bar at the top of the screen with action items that affect the selected item(s). When this mode is active, users can perform an action on multiple items at once (if your app allows it).

>**NOTICE:** ***contextual action mode***在android3.0以后才可使用。   

<!-- more -->

#### 创建***floating context menu***
- 注册view到context menu中  

```
    /*1. 注册于context menu关联的iew*/
    registerForContextMenu(textview);
```

 - 实现onCreateContextMenu()方法在activity或者fragment中    
```
   /*2.实现onCreateContextMenu()方法,当注册的view长按时,就会调用onCreateContextMenu()方法*/
    @Override
    public void onCreateContextMenu(ContextMenu menu, View v,
	                     ContextMenu.ContextMenuInfo menuInfo) {
        super.onCreateContextMenu(menu, v, menuInfo);
        MenuInflater inflater = getMenuInflater();
        inflater.inflate(R.menu.menu_share_items, menu);
    }
```

 - 实现onContextItemSelected()方法    
```
    /*3. 实现onContextItemSelected(),处理menu item选择的响应*/
    @Override
    public boolean onContextItemSelected(MenuItem item) {
        AdapterView.AdapterContextMenuInfo menuInfo =
            (AdapterView.AdapterContextMenuInfo) item.getMenuInfo();
        switch (item.getItemId())
        {
            case R.id.menu_share:
                /*在这里作相应的处理*/
                return true;
            default:
                return super.onContextItemSelected(item);
        }

    }
```

#### 创建***contextual action mode***  

对于android3.0及以上版本，应该使用***contextual action mode***代替***floating context menu***  

为view提供conteual操作，通常调用下面两种contextual操作中的一个或者两个：  

- 长按View  
- 选择了一个checkbox或者类似的iew。  

**Tips**: How your application invokes the contextual action mode and defines the behavior for each action depends on your design. There are basically two designs:

- For contextual actions on individual, arbitrary views.
- For batch contextual actions on groups of items in a ListView or GridView (allowing the user to select multiple items and perform an action on them all).  


***对于单个view使用Context menu***  
为了当用户选择一个特定的View，调用Contextual action mode,需要做下面的操作  

- 实现ActionMode.Callback接口   

```
  private ActionMode.Callback mCallback = new ActionMode.Callback() {

        /*当action mode被创建的的时候调用*/
        @Override
        public boolean onCreateActionMode(ActionMode mode, Menu menu) {
            mode.getMenuInflater().inflate(R.menu.menu_share_items,menu);
            return true;
        }

        /*每次调用onCreateActionMode()之后调用此方法,当mode无效的时候会调用多次*/
        @Override
        public boolean onPrepareActionMode(ActionMode mode, Menu menu) {
            /*这里没有相应的处理*/
            return false;
        }

        /*当选择context item时,会调用此方法*/
        @Override
        public boolean onActionItemClicked(ActionMode mode, MenuItem item) {
            switch (item.getItemId())
            {
                case R.id.menu_share:
                    Toast.makeText(ContextMenuActionBarModeActivity.this,
                            "share",Toast.LENGTH_LONG).show();
                    mode.finish();
                    return true;
                default:
                    return false;
            }
        }

        @Override
        public void onDestroyActionMode(ActionMode mode) {
            mCallback = null;
        }
    };
```

- 显示时调用startActionMode()方法（例如长按特定view）  

```
/*长按时显示action bar*/
        textView.setOnLongClickListener(new View.OnLongClickListener() {
            @Override
            public boolean onLongClick(View v) {
                if (mActionMode != null) {
                    return false;
                }
                mActionMode = startActionMode(mCallback);
                textView.setSelected(true);
                return true;
            }
        });
```

***对于listiew和GridView使用Context menu***  
对于批量处理listiew和GridView时需要做下面操作
- 实现AbaListView.MultiChiceModeListener接口，并调用ListView或者GridView的setMultiChioceModeListener()设置。
- ListView或者GridView调用setChoiceMode()方法，设置参数为CHOICE_MODE_MULTIPLE_MODE.

###  popup menu的创建
实现的步骤：  
1. 创建一个PopupMenu对象  
2. 使用menu资源填充menu  
3. 调用PopupMenu.show().  

### Styling the action bar  
[style action bar](http://developer.android.com/guide/topics/ui/actionbar.html#Style)  


###参考文档：
[参考文档1](http://developer.android.com/guide/topics/ui/actionbar.html)  
[参考文档2](http://developer.android.com/guide/topics/ui/menus.html)  
