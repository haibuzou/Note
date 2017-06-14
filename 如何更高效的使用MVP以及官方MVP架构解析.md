>转载请标明出处： 
>http://blog.csdn.net/dantestones/article/details/51445208

[Android mvp 架构的自述](http://blog.csdn.net/dantestones/article/details/50899235)中我简单的介绍了mvp，以及怎么写mvp。我自己也将mvp运用到了项目中，其实mvp并没有固定的写法，正确的去理解架构的思想，都可以有自己独特的mvp写法。git上也有很多例子，比如google的[android-architecture](https://github.com/googlesamples/android-architecture)，simple哥的Android 源码设计模式解析与实战中也有mvp的讨论。这里参考了simple哥做了一个通用版的mvp，并对google的MVP做了一点自己的解析。

##关于presenter一直持有Activity对象导致的内存泄漏问题
只要用过mvp这个问题可能很多人都知道。写mvp的时候，presenter会持有view，如果presenter有后台异步的长时间的动作，比如网络请求，这时如果返回退出了Activity，后台异步的动作不会立即停止，这里就会有内存泄漏的隐患，所以会在presenter中加入一个销毁view的方法。现在就在之前的项目中做一下修改
```java
//presenter中添加mvpView 置为null的方法
public void onDestroy(){    
    mvpView = null;
}

//退出时销毁持有Activity
@Override
protected void onDestroy() {    
    mvpPresenter.onDestroy();    
    super.onDestroy();
}
```
presenter中增加了类似的生命周期的方法，用来在退出Activity的时候取消持有Activity。

####但是在销毁后需要思考一点，后台的延时操作返回时，这个时候view被销毁了，如果接着去调用view的方法就  会抛出空指针异常。所以在后台的延时操作中需要考虑到这种可能产生空指针的情况，尤其是网络请求。

##BasePresenter
如果每一个Activity都需要做绑定和解绑操作就太麻烦了，现在我希望可以有一个通用的presenter来为我们添加view的绑定与销毁。
```java
public abstract class BasePresenter<T> {   

     public T mView;    

     public void attach(T mView) {       
         this.mView = mView;    
     }    

     public void dettach() {        
         mView = null;    
     }
}
```
因为不能限定死传入的View，所以使用泛型来代替传入的对象。通过这个通用的presenter我就可以把原来的[MvpPresenter](https://github.com/haibuzou/MVPSample/blob/master/app/src/main/java/haibuzou/mvpsample/mvp/presenter/MvpPresenter.java)简化成下面的样子

```java
public class NewMvpPresenter extends BasePresenter<NewMvpView> {   
 
    private RequestBiz requestBiz;    
    private Handler mHandler;    

    public NewMvpPresenter() {        
        requestBiz = new RequestBiziml();       
        mHandler = new Handler(Looper.getMainLooper());    
   }    

    public void onResume(){        
        requestBiz.requestForData(new OnRequestListener() {                        
             @Override            
             public void onSuccess(final List<String> data) {                
                 mHandler.post(new Runnable() {                    
                    @Override                   
                    public void run() {                       
                        mView.hideLoading();                        
                        mView.setListItem(data);                    
                   }               
               });            
            }            

             @Override            
             public void onFailed() {                
                 mView.showMessage("请求失败");            
             }        
        });    
      }    

    public void onItemClick(int position){        
        mView.showMessage("点击了item"+position);    
    }
}
```
##BaseView
界面需要提供的UI方法中会有很多类似的UI方法，可以把它们提取到一个公共的父类接口中。比如提取显示loading界面和隐藏loading界面的方法，其他的view层接口就可以直接继承BaseView接口，不必重复的写显示和隐藏loading界面方法。
```java
public interface BaseView {    
    void showLoading();   
    void hideLoading();
}
```
##BaseMvpActivity
presenter绑定到activity和View的绑定和解绑操作是每个Activity都会去做的，同样这里我也希望能有一个父类来完成这个统一的操作。
```java
public abstract class BaseMvpActivity<V,T extends BasePresenter<V>> extends AppCompatActivity {   
 
    public T presenter;    

     @Override   
     protected void onCreate(Bundle savedInstanceState) {        
         super.onCreate(savedInstanceState);        
         presenter = initPresenter();   
     }    

     @Override    
     protected void onResume() {        
          super.onResume();        
          presenter.attach((V)this);    
     }    

     @Override    
     protected void onDestroy() {       
        presenter.dettach();        
        super.onDestroy();    
     }   

     // 实例化presenter
     public abstract T initPresenter();

}
```
同样使用泛型来提取通用的逻辑，presenter的初始化，以及view的绑定和解绑操作都提取到父类Activity中。向外部提供了一个 ``` initPresenter(); ``` 方法用来初始化presenter，如果想创建不同参数的构造函数都可以随意去创建。
##更加通用的例子
通过上面的base父类，对之前的例子进行优化，写一个更加好用的例子。

 *  NewMvpView  继承BaseView接口，添加自己的初始化ListView和Toast信息方法

```java
public interface NewMvpView extends BaseView {    
        void setListItem(List<String> data);    
        void showMessage(String message);
}
```
* NewMvpPresenter 继承BasePresenter类，增加网络请求和处理点击事件的方法
```java
public class NewMvpPresenter extends BasePresenter<NewMvpView> {    
        private RequestBiz requestBiz;    
        private Handler mHandler;    

        public NewMvpPresenter() {        
            requestBiz = new RequestBiziml();       
           mHandler = new Handler(Looper.getMainLooper());    
        }    

        public void onResume(){        
            requestBiz.requestForData(new OnRequestListener() {            
                @Override            
                public void onSuccess(final List<String> data) {                
                   mHandler.post(new Runnable() {                    
                      @Override                    
                       public void run() {                        
                           mView.hideLoading();                        
                           mView.setListItem(data);                    
                        }                
                    });            
                 }            

                 @Override            
                 public void onFailed() {                
                     mView.showMessage("请求失败");            
                 }        
            });    
       }   

        public void onItemClick(int position){        
              mView.showMessage("点击了item"+position);     
        }
}
```
* NewMvpActivity 
```java
public class NewMvpActivity extends BaseMvpActivity<NewMvpView,NewMvpPresenter> implements NewMvpView,AdapterView.OnItemClickListener{    
        ListView mvpListView;    
        ProgressBar pb;    

        @Override    
        protected void onCreate(Bundle savedInstanceState) {        
              super.onCreate(savedInstanceState);        
              setContentView(R.layout.activity_mvp);        
              mvpListView = (ListView)findViewById(R.id.mvp_listview);                 
              mvpListView.setOnItemClickListener(this);       
              pb = (ProgressBar) findViewById(R.id.mvp_loading);    
       }    

       @Override    
        protected void onResume() {        
             super.onResume();        
             presenter.onResume();    
        }    

      @Override    
       public NewMvpPresenter initPresenter() {        
             return new NewMvpPresenter();    
       }    

      @Override    
       public void onItemClick(AdapterView<?> parent, View view, int position, long id) {         
                presenter.onItemClick(position);   
      }    

      @Override    
       public void setListItem(List<String> data) {        
           ArrayAdapter adapter = new ArrayAdapter(this,android.R.layout.simple_list_item_1,data);        
           mvpListView.setAdapter(adapter);    
       }    

      @Override    
       public void showMessage(String message) {        
           Toast.makeText(this,message,Toast.LENGTH_SHORT).show();    
       }    

      @Override    
       public void showLoading() {        
           pb.setVisibility(View.VISIBLE);    
       }    

       @Override    
       public void hideLoading() {        
          pb.setVisibility(View.GONE);    
      }
}
```
最终的成果，我们只需要在Acitivity中传入泛型对象，并写好``` initPresenter()``` Presenter的初始化的方法就可以直接去使用presenter，当然View的接口还是要自己去实现。
以上的方法只是一些比较简单的封装，下面来看看官方的MVP架构是怎么写的。

##官方的MVP架构
先来看看官方的代码目录(这里只是功能模块的目录，不包括测试模块，毕竟这里分析的是官方的实现代码)

![这里写图片描述](http://img.blog.csdn.net/20160531113556829)
谷歌的todomvp工程实现了一个类似记事本的功能，整体的目录结构：

*  tasks 包可以显示任务列表
*  taskdetail包显示任务详情
*  addedittask包添加和编辑任务
* statistics包用来显示任务的完成情况
* data包数据模块对应mvp的M
* Util包就是通用的方法。

这里todomvp也有BasePresenter和BaseView2个基类，不过看的出来这2个都是接口。先来看看这2个接口
```java
public interface BasePresenter {    
    void start();
}
```
```java
public interface BaseView<T> {    
    void setPresenter(T presenter);
}
```
各自简单的声明了一个方法，start()方法用来给Presenter做一些初始化的操作不用特别在意，BaseView声明的方法就很有意思了，```setPresenter(T presenter)``` 很明显是给View绑定Presenter，而前文我们使用的方式给Presenter传入View的方式来完成View和Presenter绑定的操作，这里谷歌采用了相反的方式来操作，为什么呢？

先把这个问题留下，看看单个模块的具体的文件结构。

![这里写图片描述](http://img.blog.csdn.net/20160531113627360)

有Activity，Fragment，Presenter，View哪里去了？ 其实谷歌的mvp是将Fragment作为View层来实现的，这一点在官方的Readme中也有说明，为什么要用Fragment？
>官方认为Fragement和Activity相比更像是MVP中的的View层，刚好可以满足MVP的View层的要求，Activity则作为最高指挥官用来创建和联系View和Presenter。
>
>Fragment在平板或者屏幕上有多个View时更有优势

Activity只作为创建和联系View和PresenterView而存在，将Fragment作为显示UI而存在。Activity主指挥，Fragment主显示。这也是谷歌的sample中的一贯做法。

View的问题解释完了，再看看TaskDetailContract 这种**Contract 接口，这也是官方独有的管理方法
##Contract

google的todomvp 工程中每个模块都会有一个 **Contract 接口，来看看他的代码
```java
public interface TaskDetailContract {    
    interface View extends BaseView<Presenter> {        
        void setLoadingIndicator(boolean active);        
        void showMissingTask();        
        void hideTitle();        
        void showTitle(String title);        
        void hideDescription();        
        void showDescription(String description);        
        void showCompletionStatus(boolean complete);        
        void showEditTask(String taskId);        
        void showTaskDeleted();        
        void showTaskMarkedComplete();        
        void showTaskMarkedActive();    
        boolean isActive();
}     
    interface Presenter extends BasePresenter {        
        void editTask();        
        void deleteTask();        
        void completeTask();        
        void activateTask();    
    }
}
```
Contract其实就是一个包涵了Presenter和View的接口，Presenter实现的逻辑层方法，View实现的UI层的方法都能在Contract接口中一目了然的看明白，具体的Presenter和View的实现类都是通过实现Contract接口来完成。这种方式既方便了管理和维护，也给开发点了一个导航灯。

下面来看看Presenter如何引用到Fragment中以及View如何与Presenter建立联系

##TaskDetailActivity
```java
/**
 * Displays task details screen.
 */
public class TaskDetailActivity extends AppCompatActivity {
    public static final String EXTRA_TASK_ID = "TASK_ID";

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.taskdetail_act);

        // Set up the toolbar.
        //.... 设置toolbar 代码省略
        // Get the requested task id
        String taskId = getIntent().getStringExtra(EXTRA_TASK_ID);

        // 实例化taskDetailFragment 
        TaskDetailFragment taskDetailFragment = (TaskDetailFragment) getSupportFragmentManager()
                                                     .findFragmentById(R.id.contentFrame);
        if (taskDetailFragment == null) {
            //taskDetailFragment 添加到Activity
            taskDetailFragment = TaskDetailFragment.newInstance(taskId);
            ActivityUtils.addFragmentToActivity(getSupportFragmentManager(),
            taskDetailFragment, R.id.contentFrame);
        }

        // Create the presenter  初始化Presenter
       new TaskDetailPresenter(taskId，Injection.provideTasksRepository(getApplicationContext()),
                               taskDetailFragment);
}
..... 省去不重要代码
```
TaskDetailActivity主要做了taskDetailFragment 初始化和TaskDetailPresenter的初始化操作，在做TaskDetailPresenter初始化时直接将taskDetailFragment作为参数传入，因为taskDetailFragment实现了View层的接口，看到这里有一个疑问，new TaskDetailPresenter()可以实例化一个Presenter，但是怎么传入到taskDetailFragment中呢？为了解开疑惑我查看了TaskDetailPresenterde 构造方法
```java
public class TaskDetailPresenter implements TaskDetailContract.Presenter {    
     //....省去不重要的变量声明

    public TaskDetailPresenter(@Nullable String taskId,                               
                               @NonNull TasksRepository tasksRepository,                               
                               @NonNull TaskDetailContract.View taskDetailView) {       
        this.mTaskId = taskId;        
        mTasksRepository = checkNotNull(tasksRepository, "tasksRepository cannot be null!");        
        mTaskDetailView = checkNotNull(taskDetailView, "taskDetailView cannot be null!");        
        //实现绑定Presenter的关键方法
        mTaskDetailView.setPresenter(this);    
}
```
关键方法就是在BaseView中声明的```void setPresenter(T presenter);``` 方法实现的绑定，当然这里只是调用View的方法，具体的绑定还要追踪到实体类中，来看TaskDetailFragment

##TaskDetailFragment
```java
public class TaskDetailFragment extends Fragment implements TaskDetailContract.View {
     ......省去不重要代码

    //声明了一个mPresenter
    private TaskDetailContract.Presenter mPresenter;

    public View onCreateView(LayoutInflater inflater, ViewGroup container, Bundle savedInstanceState) {
         View root = inflater.inflate(R.layout.taskdetail_frag, container, false);
         ....省去初始化代码
         return root;
    }

    @Override
    public void setPresenter(@NonNull TaskDetailContract.Presenter presenter) {   
         //完成mPresenter的绑定
         mPresenter = checkNotNull(presenter);
    }

.....
}
```
TaskDetailFragment 实现了TaskDetailContract接口中的View接口，这样在TaskDetailPresenter 构造方法中调用mTaskDetailView.setPresenter(this)方法后，TaskDetailFragment 的setPresenter也会调用，TaskDetailPresenter 就成功绑定到了TaskDetailFragment 中。

>这种绑定方式也解释了上文提到的问题，为什么谷歌采用了相反的方式操作，就是为了这后面的绑定操作。

##Acitivity对象内存泄漏的问题
谷歌的项目同样会有当有后台异步任务时可能导致Acitivity内存泄漏的问题。谷歌也有自己的处理方法，上面提到的TaskDetailContract 接口有声明一个方法 ```isActive()```
```java
public interface TaskDetailContract { 
    interface View extends BaseView<Presenter> {
        ....省去其他的方法
        boolean isActive();
}
```
看看这方法的具体实现类
```java
public class TaskDetailFragment extends Fragment implements TaskDetailContract.View {
     .....
     @Override
     public boolean isActive() {    
          return isAdded();
     }
```
isAdded()方法如果返回true代表Fragment添加到了Activity，false代表没有添加，通过调用isActive()方法就可以给后台异步任务添加判断避免内存泄漏。
##UML结构图
到这里整体的架构就已经清楚了配上这张UML图可以更好的理解。

![这里写图片描述](http://img.blog.csdn.net/20160531113653345)

1. Activity最外层负责Presenter和View的创建和联系。
2. Fragment实现了View的接口并与Presenter绑定
3. Presenter从数据层中获取数据与View进行交互

google的todomvp不光给了我们一个mvp的架构演示，还附带了一个测试模块，整体的架构非常适合用来开发。强烈建议大家去github上下载源码来看看，效果更佳

相关代码
[https://github.com/haibuzou/MVPSample](https://github.com/haibuzou/MVPSample)
[https://github.com/googlesamples/android-architecture/tree/todo-mvp/](https://github.com/googlesamples/android-architecture/tree/todo-mvp/)
