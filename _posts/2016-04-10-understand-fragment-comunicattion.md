---
layout: post_layout
title: Fragment 之间数据专递的五种方法
time: 2016年04月10日
location: 北京
pulished: true
excerpt_separator: "~~~"
---
听到五种方法你是不是被吓到了，哪有五种，你胡说！我真的没有胡说，接下来我们就看看吧。说到fragment之间的通信，你肯定也会想到一些方法，到底都有哪些情况我们需要fragment之间通信，又有哪些fragment之间通信的方法呢。

## 不同Activity中的Fragment 数据传递

这种情况类似于activity之间的数据传递，fragment 也支持 startActivityForResult(intent, REQUEST_CODE) 到别的activity中获取数据，只不过当我们的目标activity中fragment真正处理并返回数据的时候，我们需要借助目标activity来返回数据，看下列代码：~~~

    请求数据的fragment：
        public class RequestFragment extends ListFragment
    {
        public static final int REQUEST_CODE = 0x110; 
        //...
        @Override
        public void onActivityCreated(Bundle savedInstanceState)
        {
            //...
            startActivityForResult(intent, REQUEST_CODE);
        }

        @Override
        public void onActivityResult(int requestCode, int resultCode, Intent data)
        {
            super.onActivityResult(requestCode, resultCode, data);
            if(requestCode == REQUEST_CODE)
            {
            //...
            }
        }
    }
---
    处理并返回数据的fragment：
    public class HandleFragment extends Fragment
    {
        //...
        @Override
        public void onCreate(Bundle savedInstanceState)
        {
            //...
            if (bundle != null)
            {
            //...
                getActivity().setResult(RequestFragment.REQUEST_CODE, intent);
            }
        }

        @Override
        public View onCreateView(LayoutInflater inflater, ViewGroup container,
                Bundle savedInstanceState)
        {
            //...
    }
---
      目标activity：
      public class HandleActivity extends FragmentActivity
      {
      private HandleFragment mHandleFragment;

      @Override
      protected void onCreate(Bundle savedInstanceState)
      {
          super.onCreate(savedInstanceState);
          setContentView(R.layout.activity_single_fragment);

          FragmentManager fm = getSupportFragmentManager();
          mContentFragment = (HandleFragment) 							fm.findFragmentById(R.id.id_fragment_container);

          if(mHandleFragment == null )
          {
              String title = getIntent().getStringExtra(HandleFragment.ARGS);
              mHandleFragment = HandleFragment.newInstance(title);
              	fm.beginTransaction().add(R.id.id_fragment_container,mHandleFragment).commit();
          }

      }
  }	

请求数据的额fraagment 通过startActivityForResult(intent, REQUEST_CODE) 请求数据，而目标activity 需要把接受的bundle中数据需要的部分传递给handleFragment，供其使用，当出完数据后在通过getActivity().setResult(RequestFragment.REQUEST_CODE, intent) 来设置返回数据，这里fragment没有饭后数据的能力，必须使用载体Activity 返回。

## 两个Fragment在同一个container中交替出现

这种情况也是比较常见的，在两个fragment交换的过程中同参数来传递数据，看一下代码：

    public class MainActivity extends ActionBarActivity{
     
    private Button btn;
     
    private MyFragment1 fragment1;
    private MyFragment2 fragment2;
    private FragmentManager fragmentManager;
    private Fragment currentFragment;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        fragmentManager = getSupportFragmentManager();        
        FragmentTransaction fragmentTransaction = fragmentManager.beginTransaction();
        Bundle data = new Bundle();
        data.putString("TEXT", "这是Activiy通过Bundle传递过来的值");
        fragment1.newInstance(data);//通过Bundle向Activity中传递值
        fragmentTransaction.add(R.id.rl_container,fragment1);//将fragment1设置到布局上
        fragmentTransaction.addToBackStack(null);
        fragmentTransaction.commitAllowingStateLoss();
        currentFragment = fragment1;
        //初始化button控件
        btn = (Button)findViewById(R.id.btn);
        btn.setOnClickListener(new OnClickListener() {
            @Override
            public void onClick(View v) {
                if(currentFragment instanceof MyFragment1){
                    switchFragment();
                }else{//当前是fragment2，因此，只需要将fragment2出栈即可变成fragment1
                    fragmentManager.popBackStack();
                    currentFragment = fragment1;
                }
            }
        });
    }
    /**
     * 切换Fragment
     */
    private void switchFragment(){
        Bundle data = new Bundle();
        data.putString("TEXT", "传递给fragment2");
        fragment2.newInstance(data);
        FragmentTransaction fragmentTransaction = fragmentManager.beginTransaction();
        fragmentTransaction.add(R.id.rl_container,fragment2);
        fragmentTransaction.addToBackStack(null);
        fragmentTransaction.commitAllowingStateLoss();
        currentFragment = fragment2;
    }
}

通过参数在fragment之间以及activity间，这也会我们最常使用的方法。

## 同一个activity中两个container的fragment间通信

这里有三种方法，但是也会因情况而选择合适的方法：

### 1.获取其他fragment的组件的引用，并操作

我们可以再当前的fragment中通activity来获取当前activity的其他fragment中组件的引用，并进行直接操作。看一下代码：

    public void onActivityCreated(Bundle savedInstanceState) {
    super.onActivityCreated(savedInstanceState);
    mFragment2_tv = (TextView) getActivity().findViewById(R.id.fragment2_tv);
    //获取其它	fragment中的控件引用的唯一方法!!!
     mFragment2_tv.setText(“hello”);
	}

 代码很简单，但还需要注意：如果你查看关于fragment生命周期的官方文档，你会发现onActivityCreated 的执行是在主activity的onCreate执行完成后，也就是说主activity的初始化完成后，fragment才能操作activity的中的数据。不过我们并不提倡这么做因为这加大了组件之间的耦合，违背了模块分割的设计原则。
 
###  2.使用接口并借助activity作为桥梁通信

使用接口进行fragment和activity之间的通信我想我们并不陌生，而fragment之间通过接口通信呢？哈哈，其实原理大同小异，我们还是简略的讲一下，看代码：

通信接口：

      // 用来存放fragment的Activtiy必须实现这个接口
      public interface OnCallBackListener {
          public void onDoAction(int position);
      }
---
  在fragment1中：
  
        @Override
          public void onAttach(Activity activity) {
              super.onAttach(activity);
              // 这是为了保证Activity容器实现了用以回调的接口。如果没有，它会抛出一个异常。
              try {
                  setCallBackListener((OnCallBackListener) activity);
              } catch (ClassCastException e) {
                  throw new ClassCastException(activity.toString()
                          + " must implement OnHeadlineSelectedListener");
              }
          }
---
   看一下Activity：
    
        public static class MainActivity extends Activity
            implements calbackFragment.OnCallBackListener{
        ...
        public void onDoAction(int position) {
            // 做一些必要的业务操作
            Fragment2 fragment2= (Fragment2)
                    getSupportFragmentManager().findFragmentById(R.id.article_fragment);
            if (fragment2 != null) {
                // 如果 fragment2不为空，说明fragment1和fragment2存在于两个container中
                // 调用fragment2中的方法去更新它的内容
                fragment2.update(position);
            } else {
                // 否则，两个fragment共用给一个container
                fragment2 = new Fragment2();
                Bundle args = new Bundle();
                args.putInt(ArticleFragment.ARG_POSITION, position);
                newFragment.setArguments(args);
                FragmentTransaction transaction = 	getSupportFragmentManager().beginTransaction();
                transaction.replace(R.id.fragment_container, fragment2);
                transaction.addToBackStack(null);
                // 提交事务
                transaction.commit();
            }
        }
    }

这是一种近乎完美的概括性方法，我们把参数通过回调接口传递给activity，接着让activity操作fragment处理参数，到达了两个fragment之间通信的目的。
 
###  使用setTargetFragment()，getTargetFragment()
 
 发现这两个函数也是由于一次阅读Android源码，我发现他们的使用大多关联着FragmentDialog，后来我查阅文档，看了一下他们的函数说明：
 
 

    public void setTargetFragment (Fragment fragment, int requestCode) 
    Added in API level 11Optional target for this fragment. This may be used, for example, if this fragment is being started by another, and when done wants to give a result back to the first. The target set here is retained across instances via FragmentManager.putFragment().

    Parameters
    fragment  The fragment that is the target of this one. 
    requestCode  Optional request code, for convenience if you are going to call back with onActivityResult(int, int, Intent).  

setTargetFragment函数说明大意为如果这个fragment正在被另一个fragment启动，并且它想要在完成是方会给调用者一个结果，被设置在这里的target fragment的实例会被通过FragmentManager.putFragment()而暂时保留状态。两个参数分别为目标及调用这的fragment引用，另一个参数为请求码，做什么用，我们可以使用它来直接调用目标target的onActivityResult(int, int, Intent)，是不是很NB，是不是一下子想起好多情景都可以通过这种方式来完成了，我们来举一个例子：
 
 

      /**
     * Dialog to request user confirmation before setting
     * {@link NetworkPolicy#limitBytes}.
     */
    public static class ConfirmLimitFragment extends DialogFragment {

        public static void show(parentFragment parent) {
            final ConfirmLimitFragment dialog = new ConfirmLimitFragment();
            dialog.setArguments(args);
            dialog.setTargetFragment(parent, 0);
            dialog.show(parent.getFragmentManager(), TAG_CONFIRM_LIMIT);
        }

        @Override
        public Dialog onCreateDialog(Bundle savedInstanceState) {
            final Context context = getActivity();
            final AlertDialog.Builder builder = new AlertDialog.Builder(context);
            builder.setTitle(R.string.data_usage_limit_dialog_title);
            builder.setMessage(message);
            builder.setPositiveButton(android.R.string.ok, new DialogInterface.OnClickListener() {
                @Override
                public void onClick(DialogInterface dialog, int which) {
                    final parentFragment parent = (parentFragment) getTargetFragment();
                    if (parent != null) {
						//....
                    }
                }
            });
            return builder.create();
        }
    }

这是一种最常见的Fragment+DialogFragment的使用用法，我们可以通过DialogFragment来提示信息，并展示可选项，并根据用的操作直接操作parentFragment。但是又要说了，如果这样子用那也很局限啊，哈哈，之前我是说大多数情况下，Fragment+DialogFragment使用常用个用法，那有没有Fragment+Fragment 的使用用法呢，当然了，看下面代码示例：

       public static class AppDetailsFragment extends Fragment {
          private static final String EXTRA_APP = "app";

          public static void show(parentFragment parent,data data) {
              if (!parent.isAdded()) return;

              final Bundle args = new Bundle();
             args.putParcelable(EXTRA_APP, data);
              final AppDetailsFragment fragment = new AppDetailsFragment();
              fragment.setArguments(args);
              fragment.setTargetFragment(parent, 0);
              final FragmentTransaction ft = parent.getFragmentManager().beginTransaction();
              ft.add(fragment, TAG_APP_DETAILS);
              ft.addToBackStack(TAG_APP_DETAILS);
              ft.setBreadCrumbTitle(label);
              ft.commitAllowingStateLoss();
          }
          @Override
          public void onCreate(Bundle savedInstanceState) {
              super.onCreate(savedInstanceState);
              final parentFragment target = (parentFragment) getTargetFragment();
              target.mCurrentApp = getArguments().getParcelable(EXTRA_APP);
          }

          @Override
          public void onStart() {
              super.onStart();
              final parentFragment target = (parentFragment) getTargetFragment();
              //target.mCurrentApp = getArguments().getParcelable(EXTRA_APP);
              Log.d(TAG,"AppDetailsFragment start ");
              target.updateBody();
          }

          @Override
          public void onStop() {
              super.onStop();
              final parentFragment target = (parentFragment) getTargetFragment();
              //target.mCurrentApp = null;
              target.updateBody();
          }

          @Override
          public void onDestroy() {
              super.onDestroy();
              final parentFragment target = (parentFragment) getTargetFragment();
              target.mCurrentApp = null;
              target.mNeedAnimMonthDS = true;
              target.mNeedAnimDayDS = true;
          }
      }
      
	//使用：
    private OnItemClickListener mListListener = new OnItemClickListener() {
        @Override
        public void onItemClick(AdapterView<?> parent, View view, int position, long id) {
            AppDetailsFragment.show(AppDetailsFragment.this, data);
        }
    };

 这段代码通过一个空的fragment来控制parent faragment中数据的更新与显示，重要的是，可以每次现实的信息都作为一个“一个图片”保存下来，我可以通过按返回键来使parent faragment显示之前“图层”的数据，这里的parent Fragment其实始终都有一个，这样节约了大量的内存，只是AppDetailsFragment有多个，但由于他们并没有显示出来，所有占用了较小的内存，通过这种方法，我们可以把浏览数据的过程保存下来，并可以回退查看。
另外推荐一篇关于Fragment生命周期的文章，比较简练：

[理解Fragment生命周期][1]
 

 
 
 


  [1]: http://blog.csdn.net/forever_crying/article/details/8238863/