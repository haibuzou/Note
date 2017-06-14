  最近发现再也无法忍受越来越臃肿的Activity代码，越来越来混乱的Activity层的代码，投入到了MVP的怀抱。目前来看MVP的架构还是很适合Android的，在这里记录一下一点心得，希望都给想用MVP的人一点帮助。
##老的MVC架构
刚开始接触Android的时候会觉得Android的整个代码架构就是一个MVC。

- M : 业务层和模型层，相当与javabean和我们的业务请求代码
- V  : 视图层，对应Android的layout.xml布局文件
- C  : 控制层，对应于Activity中对于UI 的各种操作

看起来MVC架构很清晰，但是实际的开发中，请求的业务代码往往被丢到了Activity里面，大家都知道layout.xml的布局文件只能提供默认的UI设置，所以开发中视图层的变化也被丢到了Activity里面，再加上Activity本身承担着控制层的责任。所以Activity达成了MVC集合的成就，最终我们的Activity就变得越来越难看，从几百行变成了几千行。维护的成本也越来越高

##新的MVP架构

- M : 还是业务层和模型层
- V  : 视图层的责任由Activity来担当
- P : 新成员Prensenter 用来代理 C(control) 控制层

MVP与MVC最大的不同，其实是Activity职责的变化，由原来的C (控制层) 变成了 V(视图层)，不再管控制层的问题，只管如何去显示。控制层的角色就由我们的新人 Presenter来担当，这种架构就解决了Activity过度耦合控制层和视图层的问题。

##一个demo
理念终归要用代码来实现，来看一个很典型的例子，打开一个有列表的Activity界面，请求数据然后刷新界面的过程。先来看看工程的架构
![这里写图片描述](http://img.blog.csdn.net/20160317163144720)
我分了MVC 和 MVP2种写法的目录，MVP与MVC不同就在于多了2个结构presenter和view。
这里业务层大家就公用了一个biz，先来看看相同的biz 也就是业务层
####RequestBiz
声明了一个接口，带有请求数据业务的方法
```java
public interface RequestBiz {
    //请求数据业务
    void requestForData(OnRequestListener listener);
}
```

####RequestBiziml
请求的实现类为了模拟网络请求，开启了一个会sleep2秒的线程，然后装填请求的数据，通过OnRequestListener 接口回调出去，与我们平时开发的方式一致。
```java
public class RequestBiziml implements RequestBiz{

    @Override
    public void requestForData(final OnRequestListener listener) {

        new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    Thread.sleep(2000);
                    ArrayList<String> data = new ArrayList<String>();
                    for(int i = 1 ; i< 8 ; i++){
                        data.add("item"+i);
                    }
                    if(null != listener){
                        listener.onSuccess(data);
                    }
                }catch(Exception e){
                    e.printStackTrace();
                }
            }
        }).start();
    }
}
```
####OnRequestListener
数据请求的回掉接口，声明了成功和失败的方法 。
```java
public interface OnRequestListener {

    void onSuccess(List<String> data);
    void onFailed();
}
```

到此业务层完成，开始实现功能了,先来看看传统写法

###MVC的写法
```java
public class MVCActivity extends AppCompatActivity{

    private ListView mvcListView;
    private RequestBiz requestBiz;
    private ProgressBar pb;
    private Handler handler;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_mvc);
        mvcListView = (ListView)findViewById(R.id.mvc_listview);
        pb = (ProgressBar) findViewById(R.id.mvc_loading);
        pb.setVisibility(View.VISIBLE);
        handler = new Handler(Looper.getMainLooper());
        requestBiz = new RequestBiziml();
        requestForData();
    }

    public void requestForData(){
        requestBiz.requestForData(new OnRequestListener() {
            @Override
            public void onSuccess(final List<String> data) {
             //由于请求开启了新线程，所以用handler去更新界面
                handler.post(new Runnable() {
                    @Override
                    public void run() {
                        pb.setVisibility(View.GONE);
                        ArrayAdapter adapter = new ArrayAdapter(MVCActivity.this,
                        android.R.layout.simple_list_item_1,data);
                        mvcListView.setAdapter(adapter);
                        mvcListView.setOnItemClickListener(itemClickListener);
                    }
                });

            }

            @Override
            public void onFailed() {
                pb.setVisibility(View.GONE);
                Toast.makeText(MVCActivity.this,"加载失败",Toast.LENGTH_SHORT).show();
            }
        });
    }

    AdapterView.OnItemClickListener itemClickListener = new AdapterView.OnItemClickListener() {
        @Override
        public void onItemClick(AdapterView<?> parent, View view, int position, long id) {
            Toast.makeText(MVCActivity.this,"点击了item"+(position+1),Toast.LENGTH_SHORT).show();
        }
    };
}


```

比较常见的Activity里面的写法：

1. 界面的初始化
2. 发起请求以及请求完成后的界面更新
3. 点击的监听设置方法。

C: 控制层(点击事件，网络请求),  V : 视图层(界面刷新) 。就这样耦合到了Activity里面。

###MVP的写法
由于Activity变成了view层不再去控制界面，但是具体的界面的改变api其实还是由Activity来提供的，所以在写MVP之前需要思考，View层需要哪些方法。

- 显示loading
-  隐藏loading
- listview的初始化
- 弹出Toast消息

####MvpView
```java
public interface MvpView {
     //显示loading progress
     void showLoading();
     //隐藏loading progress
     void hideLoading();
     //ListView的初始化
     void setListItem();
     //Toast 消息
     void showMessage();
}
```
通过上面的MVC的例子，我们可以总结出View层需要的接口。我们的Activty就是View层，所以直接用Activity来实现上面的方法。
```java
public class MVPActivity extends AppCompatActivity implements MvpView{

    ListView mvpListView;
    MvpPresenter mvpPresenter;
    ProgressBar pb;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_mvp);
        mvpListView = (ListView)findViewById(R.id.mvp_listview);
        pb = (ProgressBar) findViewById(R.id.mvp_loading);
    }

    @Override
    public void showLoading() {
        pb.setVisibility(View.VISIBLE);
    }

    @Override
    public void hideLoading() {
        pb.setVisibility(View.GONE);
    }

    @Override
    public void setListItem(List<String> data) {
        ArrayAdapter adapter = new ArrayAdapter(MVPActivity.this,
        android.R.layout.simple_list_item_1,data);
        mvpListView.setAdapter(adapter);
    }


    @Override
    public void showMessage(String message) {
        Toast.makeText(this,message,Toast.LENGTH_SHORT).show();
    }
}
```
View视图层就完成了。接下来开始写presenter层， 同样在写presenter之前想想控制层需要哪些方法

- 网络请求
- 点击事件

####MvpPresenter
```java
public class MvpPresenter {

    private MvpView mvpView;
    RequestBiz requestBiz;
    private Handler mHandler;

    public MvpPresenter(MvpView mvpView) {
        this.mvpView = mvpView;
        requestBiz = new RequestBiziml();
        mHandler = new Handler(Looper.getMainLooper());
    }

    public void onResume(){
        mvpView.showLoading();
        requestBiz.requestForData(new OnRequestListener() {
            @Override
            public void onSuccess(final List<String> data) {
               //由于请求开启了新线程，所以用handler去更新界面
                mHandler.post(new Runnable() {
                    @Override
                    public void run() {
                        mvpView.hideLoading();
                        mvpView.setListItem(data);
                    }
                });

            }

            @Override
            public void onFailed() {
                mvpView.showMessage("请求失败");
            }
        });
    }

    public void onItemClick(int position){
        mvpView.showMessage("点击了item"+position);
    }
    
}
```
Presenter完成，现在就剩下一件事，Activity中使用Presenter

####完整版MVPActivity
```java
public class MVPActivity extends AppCompatActivity implements MvpView ,AdapterView.OnItemClickListener{

    ListView mvpListView;
    MvpPresenter mvpPresenter;
    ProgressBar pb;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_mvp);
        mvpListView = (ListView)findViewById(R.id.mvp_listview);
        mvpListView.setOnItemClickListener(this);
        pb = (ProgressBar) findViewById(R.id.mvp_loading);
        mvpPresenter = new MvpPresenter(this);
    }

    @Override
    protected void onResume() {
        super.onResume();
        mvpPresenter.onResume();
    }

    @Override
    public void onItemClick(AdapterView<?> parent, View view, int position, long id) {
        mvpPresenter.onItemClick(position);
    }

    @Override
    public void showLoading() {
        pb.setVisibility(View.VISIBLE);
    }

    @Override
    public void hideLoading() {
        pb.setVisibility(View.GONE);
    }

    @Override
    public void setListItem(List<String> data) {
        ArrayAdapter adapter = new ArrayAdapter(MVPActivity.this,
        android.R.layout.simple_list_item_1,data);
        mvpListView.setAdapter(adapter);
    }


    @Override
    public void showMessage(String message) {
        Toast.makeText(this,message,Toast.LENGTH_SHORT).show();
    }
}

```
静静的享受MVP的感觉吧，就是这么清爽，没有乱七八糟的业务，没有各种点击处理逻辑，Activity只需要提供View层的方法就可以了。

##一点心得
我们的实际发开中需求往往瞬息万变，每次还要等美工出界面，这样往往会浪费大量时间，但是在MVP的世界这些都不是事，视图层与控制层完全分离，可以让我们在界面还是很粗糙的情况下，先进行控制层的开发，甚至可以先让View层先提供方法出来，这样可以节省很多时间。后面的复杂需求变化，也就是构建接口的时候会变的复杂，但是在有这么多优势的情况下，这点复杂度完全可以接受
**还等什么快来加入MVP吧!!!**

[代码下载](https://github.com/haibuzou/MVPSample/tree/master)
