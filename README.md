
# 安卓技术架构的演变与实现（模块化）
## 背景
- #### 代码量持续成量级增加的时候，代码冗余、耦合就会非常严重，而且逻辑会越来越不清晰，Android Studio 更新优化提高的编译速度完全赶不上新增代码所带来的编译速度，这样大大降低了工作效率。
    - #### 核心思想：分而治之，降低耦合
- #### 这时组件化、模块化、插件化应运而生，组件化与模块化相较起来可以简单理解为同一种，插件化为组件化的多功能形态。
    - 组件化为将公共组件抽取为底层，单个功能、产品抽取为单个模块的方式去实现。
    - 分拆出来的多个模块即可组合也可单个打包成为独立APP，单独运行，为编译期组合。
    - 插件化是以组件化为基础，增加运行时组合拆装能力。
    - 两者的区别在于插件化具有组件化所有功能，比组件化多一项运行时组装模块的功能
----
#### 以上为组件化与插件化的区别，下面着重来讨论组件化的实现。
----
- 上面提到组件化的单个模块又可组合又可单独运行，这让团队开发实现了分工而为，互不干扰。开发时只编译自己负责的单个模块，编译运行速度将会大大提高，出现模块间耦合的情况将消失！
- 开发时首先将自己负责的模块修改为 asAPP(是否单独运行)，然后结合内部Maven库的使用，将所需模块依赖到此模块，进行开发，由于其余支持模块为已编译完成的成品，所以将会大大提高编译此模块的速度，内部Maven库，后面会单独介绍。
- #### 模块的划分--基本样例
    ![image](http://7xplbb.dl1.z0.glb.clouddn.com/mokuai1.png)
    - BaseModle：包括核心库（三方库等）、各种帮助类以及自定义View
    - NetWorK：网络层的操作
    - 上层拆分模块，可单独组装运行，合成的APP即为最终的APP
- #### 组件的单独调试
    - 由于一个APP只有一个Activity入口，所以这里要用一个变量去区分：  
    >     在gradle.properties文件中配置：mainmodulename=app  //通过Modle名去判断
    - 在Modle中src下新建release、debug文件夹用来存放不同的AndroidMinifest.xml文件
    - ![image](http://7xplbb.dl1.z0.glb.clouddn.com/mokuai2.png)
    - 在build.gradle文件中
    
    ```
    def asApp = false  //是否作为单独的APP this_modle需设置为此modle name
    if (mainmodulename == "this_modle") {
        asApp = true
    }
    if (asApp) {
        apply plugin: 'com.android.application'
     else {
        apply plugin: 'com.android.library'
    }
    ```
- ### 组件的数据传输UI跳转
    - 组件化的第一目标就是  **解耦**
    - 通过scheme和host的匹配关系负责分发路由-这种方式的缺点是定义规则太复杂，维护成本高，出错率较高。
    - 通过全局路由的方式，定义对应路由表，通过反射机制去进行对应跳转和携带数据，这里推荐使用阿里开源的ARoute路由库。
        - 实现的功能
            1. 支持直接解析标准URL进行跳转，并自动注入参数到目标页面中
            1. 支持多模块工程使用
            1. 支持添加多个拦截器，自定义拦截顺序
            1. 支持依赖注入，可单独作为依赖注入框架使用
            1. 支持InstantRun
            1. 支持MultiDex(Google方案)
            1. 映射关系按组分类、多级管理，按需初始化
            1. 支持用户指定全局降级与局部降级策略
            1. 页面、拦截器、服务等组件均自动注册到框架
            1. 支持多种方式配置转场动画
            1. 支持获取Fragment
            1. 完全支持Kotlin以及混编(配置见文末 其他#5)
            1. 支持第三方 App 加固(使用 arouter-register 实现自动注册)
        - 其主要的应用有：
            1. 从外部URL映射到内部页面，以及参数传递与解析
            1. 跨模块页面跳转，模块间解耦
            1. 拦截跳转过程，处理登陆、埋点等逻辑
            1. 跨模块API调用，通过控制反转来做组件解耦
    - github地址：https://github.com/alibaba/ARouter
    - 使用方式：
        ```
        // 在支持路由的页面上添加注解(必选)
        // 这里的路径需要注意的是至少需要有两级，/xx/xx
        @Route(path = "/test/activity")
        public class YourActivity extend Activity {
            ...
        }
        ```

        ```
        // 1. 应用内简单的跳转(通过URL跳转在'进阶用法'中)
        ARouter.getInstance().build("/test/activity").navigation();
        
        // 2. 跳转并携带参数
        ARouter.getInstance().build("/test/1")
        			.withLong("key1", 666L)
        			.withString("key3", "888")
        			.withObject("key4", new Test("Jack", "Rose"))
        			.navigation();
        ```
        如此简单，只需定义要跳转的类的名字，在任何一个地方任何一个Modle都可以进行跳转，并支持各种方式传参。关键一点是路由表维护可以通过Apt插件去自动生成，自己无需维护，还有更多进阶用法，自己探索。
### 其他问题
- 资源文件名冲突
    #### 可以通过统一设置资源名来解决，在实际开发中，资源名尽量不重名即可，大范围使用的资源可以统一在BaseCore中统一放置，例如返回键什么的。
    
    ```
    defaultConfig {
       ...
       resourcePrefix "module_name_"
       ...
    }
    ```
- Butterknife R文件问题
    #### 之前存在的问题就不提了，因为最新的8.8.1版本已修复，当前所需要做的只有将子Modle中的 R 修改为 R2,手动修改，工具修改都行。下面注释有个注意点：
    
    ```
    @BindView(R2.id.login_tv_title)
        TextView mTvTitle;
        @OnClick({R2.id.btn_login}){
            //这里只能用 if else 方式，不能使用Swich case 方式
            //因为R2文件里的id不是final的case需要final,所以这里使用if else方式去解决此问题
        }
    ```
----
### 附加1： Nexus 实现本地Maven库管理

最终实现的结果：
由  
```
compile project(':basemodle')
```
改为
```
compile 'com.test.android:basecoremodle:1.0.0'
```
实现代码隔离，防止代码冲突，保证版本的稳定性,引入的为已编译后的代码，加快模块编辑速度。
### 实现方式：
在modle的build,gradle中添加

```
apply plugin: 'maven-publish'
apply plugin: 'maven'

uploadArchives {
    repositories {
        mavenDeployer {
            repository(url: "http://127.0.0.1:8081/repository/android/") {
                authentication(userName: "admin", password: "admin")      //账号，密码
            }
            pom.project {
                packaging 'aar'
                description 'modle1'
                groupId = GROUP
                artifactId = POM_ARTIFACT_ID_LOGIN
                version = VERSION_LOGIN
            }

        }
    }
}
```
重新编译，执行gradle
- ![image](http://7xplbb.dl1.z0.glb.clouddn.com/mokuai3.png)
- Nexus的配置这里就不在赘述了。

   

### 附加2： Nexus + gitlab + jenkens实现自动化构建，部署
先附上思路，具体实现下一篇
![image](http://7xplbb.dl1.z0.glb.clouddn.com/mokuai4.png)

###封装有三方库
#### 3.RxBus
事件总线，传递信息

	使用方式：

	发送：RxBus2.getInstance().post(“finish_home”);
	
	接收：RxBus2.getInstance().toObservable(String.class)
                .subscribe(new Observer<String>() {
                    @Override
                    public void onSubscribe(@NonNull Disposable d) {
                        mPresenter.addDispose(d);
                    }
                    @Override
                    public void onNext(@NonNull String s) {
                        if ("finish_home".equals(s)) {
                            HomeActivity.this.finish();
                        }
                    }
                    @Override
                    public void onError(@NonNull Throwable e) {
                    }
                    @Override
                    public void onComplete() {
                    }
                });
#### 4.MLog 日志帮助类
在BaseNetModel或BaseCoreModel初始化时，需传入setSaveFile(true)，来控制是否将日志输出到文件保存，待上传服务器。存储位置默认为 cache/log 路径下。

1.项目中的Mlog，应将**所有**try catch下的异常进行MLog.e(e);输出，用来提供项目异常监测。

2.其余使用方式，请参照MLog使用文档

#### 5.AppManager Activity栈管理类

**提供能力：**

1. currentActivity()--获取当前Activity（堆栈中最后一个压入的）
2. finishActivity()--结束当前Activity（堆栈中最后一个压入的）
3. finishActivity(Activity activity)--结束指定的Activity  
4. finishAllActivity()--结束所有Activity
5. getActivity(Class<?> cls)--获取指定的Activity
6. AppExit(Context context)--退出应用程序
7. ......

---
#### 第三方引入库
#### 1.Gson
项目地址：https://github.com/google/gson，项目当前使用版本 "2.8.2"

Gson则是谷歌提供的用来解析Json数据的一个Java类库

    // 常用方式
    Gson gson = new Gson();
    // toJson 将bean对象转换为json字符串
    String jsonStr = gson.toJson(user, User.class);
    // fromJson 将json字符串转为bean对象
    Student user= gson.fromJson(jsonStr, User.class);
注：项目中网络框架已对接口返回Json字符串做了内部解析操作
#### 2.Glide
项目地址：https://github.com/bumptech/glide，项目当前使用版本 "4.6.1"

Glide 4.0版本中取消掉了部分链式布局，但增加的东西是很值得的

例如：在RecyclerView中Glide 自动处理了 View 的复用和请求的取消，传入的 Activity 或 Fragment 实例销毁时，Glide 会自动取消加载并回收资源，等等。

**使用方式：**

1.在BaseCore中封装有GlideUtils,可以直接实例化其对象进行链式调用。

2.直接使用Glide4.0+提供的注解生成的 GlideApp类来直接进行链式调用（此方法须事先新增MyAppGlideModule类实现@GlideModel注解）

    // 常用方式 1
    new GlideUtils().with(activity)
                .asBitmap()
                .load(mDataBeanList.get(position).getImg_url())
                .centerCrop()
                .placeholder(R.drawable.home_banner_logo)
                .error(R.drawable.home_banner_logo)
                .into(holder.mIvSelectedPic);
	// 常用方式 2 (需build之后才会生成GlideApp类)
    GlideApp.with(activity)
                .asBitmap()
                .load(mDataBeanList.get(position).getImg_url())
                .centerCrop()
                .placeholder(R.drawable.home_banner_logo)
                .error(R.drawable.home_banner_logo)
                .into(holder.mIvSelectedPic);
注：使用Glide的时候GlideApp.with(context);建议传入Activity，而不是Application,因为Glide对实例销毁时回收资源已经做了处理，传入Application将会导致Glide的所占用的资源无法被回收。
#### 3.PictureSelector 相册文件选择
项目地址：https://github.com/LuckSiege/PictureSelector  项目当前使用版本‘2.2.3’

	示例：
    PictureSelector.create(CameraActivity.this) 
				    .openGallery(PictureMimeType.ofImage()) //选择方式
				    .isCamera(false)  //是否包含相机选项
				    .selectionMode(PictureConfig.SINGLE) //选择文件个数
				    .compress(true) //是否压缩
				    .enableCrop(true) //是否裁剪
				    .freeStyleCropEnabled(true) //裁剪框是否可自由移动
				    .forResult(PictureConfig.CHOOSE_REQUEST); //回调ResultCode
	//回调
	@Override
    public void onActivityResult(int requestCode, int resultCode, Intent data) {
        super.onActivityResult(requestCode, resultCode, data);
        if (resultCode == RESULT_OK && requestCode == PictureConfig.CHOOSE_REQUEST) {
                    // 图片、视频、音频选择结果回调
                    List<LocalMedia> selectList = PictureSelector.obtainMultipleResult(data);
                    // 例如 LocalMedia 里面返回三种path
                    // 1.media.getPath(); 为原图path
                    // 2.media.getCutPath();为裁剪后path，需判断media.isCut();是否为true  注意：音视频除外
                    // 3.media.getCompressPath();为压缩后path，需判断media.isCompressed();是否为true  注意：音视频除外
                    // 如果裁剪并压缩了，以取压缩路径为准，因为是先裁剪后压缩的
                    String path = selectList.get(0).getCutPath());
        }
    }
#### 4.PickerView 下拉选择器（时间选择器和选项选择器）
项目地址：https://github.com/Bigkoo/Android-PickerView  项目当前使用版本‘3.2.7’

	//条件选择器
	 OptionsPickerView pvOptions = new  OptionsPickerView.Builder(this, new OptionsPickerView.OnOptionsSelectListener() {
	            @Override
	            public void onOptionsSelect(int options1, int option2, int options3 ,View v) {
	                //返回的分别是三个级别的选中位置
	                String tx = options1Items.get(options1).getPickerViewText()
	                        + options2Items.get(options1).get(option2)
	                        + options3Items.get(options1).get(option2).get(options3).getPickerViewText();
	                tvOptions.setText(tx);
	            }
	        }).build();
	 pvOptions.setPicker(options1Items, options2Items, options3Items);
	 pvOptions.show(); 
#### 5.banner 轮播图
待补充
#### 6.xReclerView 下拉刷新、上拉加在更多列表
待确认全部统一所用
#### 7.ARouter 页面路由
通过全局路由的方式，定义对应路由表，通过反射机制去进行对应跳转和携带数据。

	1. 从外部URL映射到内部页面，以及参数传递与解析
	2. 跨模块页面跳转，模块间解耦
	3. 拦截跳转过程，处理登陆、埋点等逻辑
	4. 跨模块API调用，通过控制反转来做组件解耦

**使用方式：**

        // 在支持路由的页面上添加注解(必选)
        // 这里的路径需要注意的是至少需要有两级，/xx/xx
        @Route(path = "/test/activity")
        public class YourActivity extend Activity {
            ...
        }

        // 1. 应用内简单的跳转(通过URL跳转在'进阶用法'中)
        ARouter.getInstance().build("/test/activity").navigation();
        
        // 2. 跳转并携带参数
        ARouter.getInstance().build("/test/1")
        			.withLong("key1", 666L)
        			.withString("key3", "888")
        			.withObject("key4", new Test("Jack", "Rose"))
        			.navigation();


##目标
- ### 代码解耦。如何将一个庞大的工程拆分成有机的整体？
    - 经讨论得出
- ### 组件单独运行。上面也讲到了，每个组件都是一个完整的整体，如何让其单独运行和调试呢？
    - 通过gradle动态赋值的方式
- ### 数据传递。因为每个组件都会给其他组件提供的服务，那么主项目（Host）与组件、组件与组件之间如何传递数据？
    - ARoute路由传递数据
- ### UI跳转。UI跳转可以认为是一种特殊的数据传递，在实现思路上有啥不同？
    - ARoute路由页面跳转
- ### 组件的生命周期。我们的目标是可以做到对组件可以按需、动态的使用，因此就会涉及到组件加载、卸载和降维的生命周期。
    -
- ### 集成调试。在开发阶段如何做到按需的编译组件？一次调试中可能只有一两个组件参与集成，这样编译的时间就会大大降低，提高开发效率。
    - 内部Maven库的方式
- ### 代码隔离。组件之间的交互如果还是直接引用的话，那么组件之间根本没有做到解耦，如何从根本上避免组件之间的直接引用呢？也就是如何从根本上杜绝耦合的产生呢？只有做到这一点才是彻底的组件化。
    - 拒绝耦合使用路由表的方式加载





