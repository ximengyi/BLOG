---
title: laravel 底层auth源码
date: 2019-03-25 10:17:16
tags: laravel auth 
---

### 1 我们首先来看下laravel下的auth源码的目录结构
##### 打开vendor->laravel->framework->src->llluminate-> 即可看到如下图所示的文件夹结构
![](/images/auth组件/auth1.png)
<!--more-->
#####  在这个文件夹下可以看到几个文件夹 Access（权限控制）、Console（控制台用于artisan生成代码）、Event（权限控制的一些事件） Listeners（监听主要发邮件） Middleware（中间件） Notification（消息通知） Passwords（密码生成销毁等） 和一些类文件
为了方便理解auth组件，在讲auth组件的开始我们先想一个生活中的例子，我们把我们的系统类比一个大的园区，这个大的园区就需要有安保系统，来保护我们的园区和维持园区的秩序，不同的来源有不同的验证方式
车辆使用电子扫描卡，普通人使用通行证，为了保证小区的安全，那么这个安保系统有一些它必须要做的事情：
 
 1.对进入人员进行身份验证
 2.查询当前已经经过身份验证的用户
 3.确定进入的人员是临时人员还是常住人员
 4.对进入的人员进行记录登记
 ***
 
#### 2 带着以上的故事背景我们来看laravel的auth组件源码。
##### 我们来看Auth文件夹下都有哪些类， 我们基本上通过类名的结尾就能够猜到这个些类都是干什么的，providers基本上就是服务提供者用来实例化具体的类，绑定到容器中的，Guard就是可以理解为上述例子中的保安。用来对用户进行一系列的身份检查，放行登记的，在phpstorm里面可以看到有几个trait 接口通用的功能实现抽都放在了tarit里，在不同的实例中，可以引用trait 来避免重复写代码。
![](/images/auth组件/auth2.png)
laravel的这个代码结构是很清晰的，php为了实现类似C++的多继承引入了trait特质，auth的源码很好的使用了这一点，一个接口可能有不同的实现，但是接口定义的方法必须要实现，那么些相同的接口实现就就可以写在trait里，不同的实现类里需要写不同的实现，然后引用tarit里就可以了。比如我们来看TokenGuard这个类它实现了Guard这个接口，但是它的类里却没有接口Guard里面定义的check(),等方法，

![](/images/auth组件/auth3.png)
![](/images/auth组件/auth4.png)
但是我们再来看TokenGuard里面引用了,GuardHelper这个特质，GuardHelps这个特质里面实现了 Guard接口的几乎全部的方法，由此我们可以得到laravel的一个写代码的套路，定义接口->编写特质抽象公用的方法->编写具体的实现类->编写不同的providers来绑定具体的实现类到容器。
按照这个思路我们来看laravel的代码会不会觉得很清晰？
![](/images/auth组件/auth5.png)
#### 了解了这样的一个套路，我们再来看auth的文件夹。
我们就猜到Authenticatable 是接口的定义方法的抽象实现（其实是use Illuminate\Contracts\Auth\Authenticatable的具体实现，但是在auth文件夹下并没有这样使用，这个特质留给了用户使用，具体的实现类了可以看GenericUser类），AuthManager是具体的auth这个功能的抽象，authServiceProvider具体实现类的绑定一些实例类到容器里以方便使用
如registerAuthenticator()方法绑定auth实例类，registerUserResolver()注册用户实例类，registerAccessGate()，大门类，registerRequestRebindHandler(),请求注册处理类。
![](/images/auth组件/auth6.png)
我们看看AuthManager 都做了哪些事，AuthManager引用了CreatesUserProviderst特质（这个特质里提供了获取配置判断是使用eloquent还是使用database来存储user模型服务），

AuthManager的构造函数很简单，把实例化app容器，实例化使用的guard模型，如果有传入的自定义的guard 就使用自定义的，如果没有就使用框架默认的配置，
然后就是resolve方法解析容器里的guard实例，createSessionDriver(),createTokenDriver(),分别创建基于session 的驱动和给予token的驱动，两者都会调用CreatesUserProviderst 的createUserProvider（）方法存储用户信息，然后根据配置的类型选择创建SessionGuard 和TokenGuard，如果是session请求还会设置cookie和事件分发，如果创建了token的实例则会直接刷新容器的实例。
这里注意一下__call 方法不管你设置了哪个guard保安，你只需要AuthManager->methods 就可以直接调用SessionGuard 或者TokenGuard 里面的方法了，call 会先获取当前设置的实例，然后调用相应的放方法。（吐槽一下，laravel为了优雅，确实把代码搞得有点复杂了，其实我最喜欢的框架是YII2.0,YII2.0也很优雅，却没有laravel这么繁琐）
 
 



(如下图所示)
![](/images/auth组件/auth7.png)
我们都知道我们在使用php artisan auth 的时候很爽，框架直接帮我们生成好了代码，那生成好的代码是从哪里来的呢，我们打开auth->Console->stups->make,可以看到controllerwen文件夹和views文件夹，还有routes.stub这些就是写好的代码，然后框架使用了copy方法帮我们生成好了代码。
![](/images/auth组件/auth8.png)

### 3我们在安装完laravel的时候会有auth这个文件夹，里面帮我们写好了 登录注册，找回密码等功能，auth::route()里面包含了这些路由

打开Auth文件夹下的LoginController  我们会看到在这个类的构造方法里指定了'guest'这个中间件，这个其实是RedirectIfAuthenticated的别名
![](/images/auth组件/auth9.png)
我们在app文件夹下面的Kernel里面就可以看到这个guest对应中间件名字其实是在app->http->Middleware文件夹下面的RedirectIfAuthenticated类
![](/images/auth组件/auth10.png)
RedirectIfAuthenticated
我们来看这个中间的处理方法
```php

 public function handle($request, Closure $next, $guard = null)
    {
        if (Auth::guard($guard)->check()) {
            return redirect('/home');
        }

        return $next($request);
    }
```
这个中间件其实调用了我们上面讲的讲的guard里面的check方法来检查是否登录，这个check方法在哪里呢，因为RequestGuard，SessionGuard，TokenGuard其实都需要check方法的，按照laravel的套路

所以这个check方法被抽象了，写在了一个tarit特质里，没错就是GuardHelpers特质，这个我们可以看下RequestGuard，SessionGuard，TokenGuard都引用了这个特质，我们来看这个GuardHelpers特质的check方法
在check()方法里反而调用自己本类的user 方法; 这个user()方法就是在RequestGuard，SessionGuard，TokenGuard每个类里自己具体实现的,SessionGuard里面的user()方法回去检查session里面有没有这个用户，
TokenGuard会使用Eloquent或者Database服务去数据库查询用户所带的token是不是有效的token（是不是很绕哦？不要急，还有更绕的）

```php
public function check()
    {
        return ! is_null($this->user());
    }

```
值得一提的是如果你使用token 认证，那么
```php
 public function getTokenForRequest()
    {
        $token = $this->request->query($this->inputKey);

        if (empty($token)) {
            $token = $this->request->input($this->inputKey);
        }

        if (empty($token)) {
            $token = $this->request->bearerToken();
        }

        if (empty($token)) {
            $token = $this->request->getPassword();
        }

        return $token;
    }

```

上述代码会先从请求里查找token字段，如果不存在就在表单里查找token,字段如果表单里还是没有找到，则在header 头查找token 字段

在使用laravel框架的时候，我们只需要在路由里指定auth中间件larvel就会帮我们检查，用户是否登，没有登录的话，如果我们传递了登录的账号和密码那么laravel也会帮我们登录的，那么这些登录的代码是在哪里呢？
登录的逻辑其实是一个比较常用的逻辑，这里使用了AuthenticatesUsers特质，而AuthenticatesUsers又引用了RedirectsUsers，ThrottlesLogins 特质，基本上看这两个个特制的名字就已经猜到，一个特质是用于用户重定向的，另一个是用于显示用户登录频率的，
比如但用户连续登录三次错误的时候需要一些限制等等，说到ThrottlesLogins，我们来看看这个特质里有什么东西，ThrottlesLogins 特质里maxAttempts(),会判断有没有设置最大的错误登录次数maxAttempts属性，如果没有设置则，默认是5次，decayMinutes()方法则判断是否设置了decayMinutes等待登录时间属性，如果没有设置
则默认为1分钟。limiter() 方法获取锁的实例，这个锁其实是放在缓存里的，具体可以看RateLimiter 类，hit()方法进行加锁，设置首次枷锁的时间，以及每次设置key++，每次返回hits 就是尝试登陆的次数。

![](/images/auth组件/auth11.png)


```php
//RateLimiter 的加锁的代码。
public function hit($key, $decayMinutes = 1)
    {
        $this->cache->add(
            $key.':timer', $this->availableAt($decayMinutes * 60), $decayMinutes
        );

        $added = $this->cache->add($key, 0, $decayMinutes);

        $hits = (int) $this->cache->increment($key);

        if (! $added && $hits == 1) {
            $this->cache->put($key, 1, $decayMinutes);
        }

        return $hits;
    }

```
我们继续来看AuthenticatesUsers特质的登录login()方法，如下图所示，login方法先会验证登陆的参数，然后判断是否登录超出次数，如果没有超出次数，则调用attemptLogin()方法进行尝试登录，attemptLogin()方法继续调用credentials()
方法来获取username,和password的键和值，最后传给底层的保安（有得叫守卫都一个意思）guard()->attempt()去尝试登录，对没错就是SessionGuard，或者TokenGuard 得attempt方法，但是TokenGuard 并没有attempt()方法
![](/images/auth组件/auth12.png)



