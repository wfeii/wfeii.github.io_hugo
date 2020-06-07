---
layout: single
title: "Bound Services"
categories:
 - 外功招式
 - Android
 - Android基础

date: 2015-06-20 00:00:00
---

### 基础  
#### 如何绑定service?  
1. 创建一个bound service  
2. 在组件中调用bindService()方法进行绑定  

#### 如何创建一个bound Service呢？  
继承Service，用于绑定的类，首先，必须提供一个IBinder用于与客户端(例如Activity)  
交互,有三种方式可以提供此IBinder,每一种方式又有不同的实现方式。  

 1. 继承Binder(实现了IBinder)类  
 2. 使用Messenger  
 3. 使用AIDL  

其次是在onBind()中返回IBinder

> **Notice:**多个clients连接到service时，只在第一次绑定时调用onBind()方法获取   
>           IBinder对象，对于其他的clients，系统直接传递一个IBinder，不会再调  
	            用onBind()方法.

<!--more-->

 #### 创建bound service三种方式可以提供此IBinder,如何选择呢？  
- 继承Binder： 在servie只能在本应用中使用，并且不涉及多进程时使用（也就是与 客户端处于相同的进程中）  
- 使用Messenger：当想要IBinder工作于不同的进程中，并且不在意请求的同时执行，使用 此方法.这是最简单方式对于IPC，因为Messenger队列请求都是单线程的                       无需考虑线程安全问题。  
- 使用AIDL：首先，使用Messenger是基于AIDL，当想要同时处理多个请求时可以使用此方式，当 要自己处理多线程和线程安全问题。  

> **Notice:**不是所有组件都能绑定service，广播就是个例外，但广播可以启动service。   

### bound service
 #### 继承Binder如何定义bound service，组件中如何使用？
 1. service定义  
 1.1 在Service中Binder的对象，此对象中包含组件可以使用的共有方法，在公有方法中返回service对象，或者返回有service托管的类（类中要有public方法）。  
1.2：在onBind()方法中返回Binder对象。
 2. 组件中绑定service  
 2.1组件中onServiceConnected（）方法中获取IBinder并操作。  
```
	/******继承Binder方式bound service定义的模板******/

	public class LocalService extends Service {
    //返回到给客户端(例如Activity)
    private final IBinder mBinder = new LocalBinder();
    /*
     * 继承Binder
     */
    public class LocalBinder extends Binder {
        LocalService getService() {
            //返回service对象客户端能调用service的public方法。
            return LocalService.this;
        }
    }

    @Override
    public IBinder onBind(Intent intent) {
        return mBinder;
    }

    /*客户端（例如activity）中能够调用此方法*/
    public int getTest {
      return 12;
    }
}
```

```
	public class BindingActivity extends Activity {
	//保存获取service的对象
    LocalService mService;
	//用于标志是否service绑定了
    boolean mBound = false;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.main);
    }

    @Override
    protected void onStart() {
        super.onStart();
        // 绑定service
        Intent intent = new Intent(this, LocalService.class);
		//第二个参数Serviceonnection对象
        bindService(intent, mConnection, Context.BIND_AUTO_CREATE);
    }

    @Override
    protected void onStop() {
        super.onStop();
        // 解除绑定
        if (mBound) {
            unbindService(mConnection);
            mBound = false;
        }
    }

    //button
    public void onButtonClick(View v) {
        if (mBound) {
		    int num = service.getTest();
            Toast.makeText(this, "number: " + num, Toast.LENGTH_SHORT).show();
        }
    }

    /*由于bindService是立即的返回，没有值返回，而ServiceConnection用于监听  
     *连接的当建立连接了就会调用onServiceConnected(),从而可以获取到IBinder  
     */
    private ServiceConnection mConnection = new ServiceConnection() {

        @Override
        public void onServiceConnected(ComponentName className,
                IBinder service) {
            LocalBinder binder = (LocalBinder) service;
            mService = binder.getService();
            mBound = true;
        }

        @Override
        public void onServiceDisconnected(ComponentName arg0) {
            mBound = false;
        }
    };
}
```

#### 使用Messenger如何定义bound service，组件中如何使用？
1. 在Service中定义一个Handler用于处理来自client的请求，  
2. 使用Handler对象作为参数来创建一个Messenger对象  
3. 在onBind()方法中使用Messenger对象返回一个IBinder对象，
4. 在clients中的ServiceConnection的onServiceConnetion()中根据IBinder 对象来实例化Messenger,  
```
    public class MessengerService extends Service {

    static final int MSG_SAY_HELLO = 1;

    /**
     * 1. 定义一个Handler用于处理来自client的请求
     */
    class IncomingHandler extends Handler {
        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case MSG_SAY_HELLO:
                    Toast.makeText(getApplicationContext(), "hello!",  
					     Toast.LENGTH_SHORT).show();
                    break;
                default:
                    super.handleMessage(msg);
            }
        }
    }

    /**
     * 2. 使用Handler对象作为参数来创建一个Messenger对象  
     */
    final Messenger mMessenger = new Messenger(new IncomingHandler());

    /**
     * 3. 在onBind()方法中使用Messenger对象返回一个IBinder对象，
     */
    @Override
    public IBinder onBind(Intent intent) {
        return mMessenger.getBinder();
    }
}
```

```
public class ActivityMessenger extends Activity {
    Messenger mService = null;

    /**用于判断是否已连接*/
    boolean mBound;

    /**
     * Class for interacting with the main interface of the service.
     */
    private ServiceConnection mConnection = new ServiceConnection() {
        public void onServiceConnected(ComponentName className,   
		       IBinder service) {
           //4 根据IBinder对象来实例化Messenger,  
            mService = new Messenger(service);
            mBound = true;
        }

        public void onServiceDisconnected(ComponentName className) {
            // This is called when the connection with the service has been
            // unexpectedly disconnected
            mService = null;
            mBound = false;
        }
    };

	//Button
    public void sayHello(View v) {
        if (!mBound) return;
        Message msg = Message.obtain(null,  
		              MessengerService.MSG_SAY_HELLO, 0, 0);
        try {
            mService.send(msg);
        } catch (RemoteException e) {
            e.printStackTrace();
        }
    }

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.main);
    }

    @Override
    protected void onStart() {
        super.onStart();
        // Bind to the service
        bindService(new Intent(this, MessengerService.class),   
		                mConnection, Context.BIND_AUTO_CREATE);
    }

    @Override
    protected void onStop() {
        super.onStop();
        // Unbind from the service
        if (mBound) {
            unbindService(mConnection);
            mBound = false;
        }
    }
}
```

> **Notice：**这里没有演示service对client的回应，一个很好的例子时APIDemo中MessengerService.java和MesengerServiceActivity的例子。  

#### 上面对于clients绑定service很模糊，究竟怎么连接？
1. 实现ServiceConnection  
1.1  onServiceConnected() client-service建立连接就会调用此函数， 传递onBind()中创建的IBinder。   
1.2  onServiceDisconnected() client-service之间的连接断开时会调用这个函数。
2. 调用bindService时传递一个ServiceConnection实现。
3. 当解除绑定时调用unbindService方法。  
```
Intent intent = new Intent(this, LocalService.class);  
//BIND_AUTO_CREATE:如果Service没有在运行，就自动创建。
bindService(intent, mConnection, Context.BIND_AUTO_CREATE);
```

>**Skill:**  * Activity与service交互是在Activity可视时应该bindService() 在onCreate（）,unbindService()在onStop()中 ,Activity在后台也能接受service的回应，那可以在onCreate()中调用bindService，在onDestory()中调用unbindService().  

> **Notice:** 不应该绑定在onResume()，在onPause()解除定，在多个Activity来回切换的话可能导致service不断的销毁，然后在重建，       

### service生命周期  
#### started service
![started service声明周期](https://raw.githubusercontent.com/wangfei1991/Blog/gh-pages/img/android/android_knowledge/startedServiceflow.png)  

#### bound service
![bound service生命周期](https://raw.githubusercontent.com/wangfei1991/Blog/gh-pages/img/android/android_knowledge/bindserverflow.png)  

#### 开启服务与绑定了服务共存
[查看图片](https://raw.githubusercontent.com/wangfei1991/Blog/gh-pages/img/android/android_knowledge/service_binding_tree_lifecycle.png)   

> **Notice:**在开启service和绑定service都存在时，
1. 先解除所有绑定，会调用onUnbind(),但不会调用onDestroy（）方法，当调用stopService（）或者stopSelf()时才会调用onDestroy()，
2. 先调用stopService（）或者stopSelf()时，由于service没有解除绑定, 此时不会调用onDestroy(),当所有绑定都解除，才会调用onDestroy（）。  
3. 也就说只要servie使用中，就不会调用onDestroy().

### 参考文档
[参考文档1](http://developer.android.com/guide/components/bound-services.html)
