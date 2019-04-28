---
title: laravel 源码
date: 2019-01-25 15:17:16
tags: laravel源码剖析
---

### 1 我们首先来看下laravel源码的目录结构
##### 打开vendor->laravel->framework->src->llluminate 即可看到如下图所示的文件夹结构
![](/images/laravel源码/laravel1.png)
<!--more-->
#####  在这个文件夹可以看到我们常用的功能组件都在这个文件夹下如cookie、auth 等组件。一个文件夹就是一个组件的实现，很清晰，我们站到框架设计者的角度来想几个个问题，
 1.作为一个面向对象的框架，每次用到这些组件的都需要new 这个对象，如何避免大量的new实例，使代码变得简洁优雅呢
 2.当框架初始化的时候，我们其实只需要加载一些常用的必须的实例，非必须的实例可以用到的时候再实例化，这样可以提高框架的性能
 3.框架设计的第一原则就是面向接口编程，而不是面向实例编程，我们如何面向接口编程，保持模块的独立而又可以优雅的调用呢
 4.框架的运行应当在不同的时候留下锚点（钩子），以便我们在不同的阶段根据业务的需要，做一些特殊的处理
 ***
#### 2 带着以上的问题，我们会随着对源码的解读，逐渐揭开laravel神秘的面纱。
##### 在生活中，无论我们做什么事，都会有一定的标准和规范，有了这些规范和标准，使得我们的操作变得有迹可循。编程亦是如此，在完成某个业务逻辑的时候，如果我们能够把业务抽象出来，制定统一的规范，不仅会我们的业务逻辑变得清晰，也会使得代码可以服用，减少工作量，避免重复的造轮子。那么，对于优秀的代码，我们在编写业务逻辑的时候通常是先写接口，制定规范和协议。
##### 在laravel的源码中有这样的一个文件夹，Contracts(如下图所示)
![](/images/laravel源码/laravel2.png)
##### Contracts的英文翻译有契约，协议的意思。因此在此文件夹下，我们可以看到里面有很多和上级llluminate文件夹有很多同名的文件夹，Contracts文件夹下放的都是llluminate文件夹下的组件的接口规范，里面都是interface类，定义了实现这些接口必须要实现的方法，因此，如果我们想知道某些组件是如何使用的，我们可以通过查看这个组件在llluminate->Contracts->组件接口，基本上也能够猜到这个接口所具有的功能。
![](/images/laravel源码/laravel3.png)
### 3再讲laravel核心的实现之前，为了方便理解，我们先看一个例子，就拿一个常见的工厂设计模式来举例：
故事背景：
小明有一天要从北京到上海新天地找他女朋友，他怎么到上海呢?计划-------------->高铁->打车，
小红有一天要从北京到上海五角场找他男朋友，他怎么到上海呢？计划------------->飞机->公交车->摩拜
为了方便理解，假如如果每种交通工具都是一个类，小明可以乘坐多种交通工具，每一个类都有一个go方法可以从某地到某地，我们来编写这个案例：
```php

interface Transport{     
public function go($address);
}
class Bus implements Transport{   
public function go($address){       
echo "----乘坐公交车到达{$address}----\n";
    }
}
class Car implements Transport{   
public function go($address){       
echo "----乘坐出租车到达{$address}----\n";
    }
}
class Bike implements Transport{   
public function go($address){       
echo "----骑着摩拜到达{$address}----\n";
    }
}
class Airplane implements Transport{   
    public function go($address){       
    echo "----乘坐飞机到达{$address}----\n";
        }
    }
class Highiron implements Transport{   
        public function go($address){       
        echo "----乘坐高铁到达{$address}-----\n";
            }
        }
class transFactory{   
public static function factoryTool($name,$transport)
    {     
      echo "{$name}";
        switch ($transport) {           
        case 'bus':               
        return new Bus();               
        break;           
        case 'car':               
        return new Car();               
        break;           
        case 'bike':               
        return new Bike(); 
        case 'airplane':               
        return new Airplane();              
        break;
        case 'highiron':               
        return new Highiron();              
        break;
        default:
        echo "-----没有{$transport}交通工具,请先创建";
        }
    }
}
//小红-----------------------
$xiaoMing=transFactory::factoryTool('小明','highiron');
$xiaoMing->go("上海");
$xiaoMing=transFactory::factoryTool('小明','car');
$xiaoMing->go("新天地");
echo "\n";
//------------------------------- 小红
$xiaoHong=transFactory::factoryTool('小红','airplane');
$xiaoHong->go("上海");
$xiaoHong=transFactory::factoryTool('小红','bus');
$xiaoHong->go("邯郸路");
$xiaoHong=transFactory::factoryTool('小红','bike');
$xiaoHong->go("五角场");
//--------------------------------小红想坐飞碟去上海
$xiaoHong=transFactory::factoryTool('小红','feidie');
//$xiaoHong->go('上海');

```
在这个案例中我们使用工厂模式优雅的获取了获取了，交通工具类的实例，实例的new 都是在工厂模式中，一定程度上解耦了业务代码和交通工具类的耦合。当我们修改了某个交通工具，只需要修改工厂模式即可，代码结构也变得清晰，但是我们思考一个问题，如果有越来越多的交通工具，每次添加，都需要修改工厂类，增加new的实例，交通工具工厂类会变得越来越难以为维护，我们需要不停的修改工厂类。
有没有什么方法可以解决这个问题，思考5秒钟···
***
#### 我们需要下面这样的代码：我们希望类的创建动态的完成
```php
//我们需要下面这样的代码：类的创建动态的完成
class transFactory{   
public static function factoryTool($name,$transport)
    {     
      echo "{$name}";
        switch ($transport) {           
        case 'transport':               
        return new $transport();               
        default:
        echo "-----没有{$transport}交通工具,请先创建";
        }
    }
}
```
 当然是有的我们可以通过php反射里的反射来动态的获取类的实例
 我们的代码可能就编程这个样子了，