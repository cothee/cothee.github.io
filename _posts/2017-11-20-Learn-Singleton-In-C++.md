# C++单例模式学习 #

  首先，单例模式有以下两个要求：   
*（1）保证一个类之创建一个实例。   
*（2）保证该类的实例可以全局访问。   

## 1. Lazy方式 ##

  该模式在调用的时候返回一个实例，如果该实例已经创建，则直接返回已创建好的实例;如果该实例还未创建，则先new一个出来，然后再返回。代码如下：   

```cpp
//in header file

class Singleton {
public:

  Singleton& Instance() {
    if (single_instance == null) {
      single_instance = new Singleton();
    }
    return *single_instance;
  }
  
private:

  Singleton();
  ~Singleton();
  Singleton (const Singleton&);
  Singleton& operator=(const Singleton& s);

  static Singleton* single_instance;
}
```

```cpp
//in implementation file

Singleton* Singleton::single_instance = null;
```
  需要注意的是，该方式只需在第一次调用时生成实例。Instance()的返回为引用，这样可以避免该实例在外部被delete；同时，拷贝制构造函数和赋值操作符也被禁用。显然，该方式不是线程安全的，当出现竞争访问时，有可能会出现实例化两次的问题，当然，这种问题可以通过加锁解决。于是，出现了加锁的懒人模式。

## 2. 加锁的Lazy ##
  代码如下：
```cpp
//in header file

class Singleton {
  public:

    Singleton& Instance(){
      if (single_instance != null) {
        return *single_instance;
      }

      {
        Lock lock_varible;
        if (single_instance == null) {
          single_instance = new Singleton();
        }
        return single_instance;
      }
    }

  private:
    
    Singleton();
    ~Singleton();
    Singleton(const Singleton& s);
    Singleton& operator=(const Singleton& s);

    static Singleton* single_instance;
}
```

```cpp
\\in implementation file
  Singleton* Singleton::single_instance=null;
```

## 3. mayer方式 ##
  确实很优雅，在函数内部用local static方式声明变量,这样只需要一次初始化。

```cpp
\\in header file

  class Singleton{
    public：
      Singleton& Instance(){
      static Singleton single_instance;
      return single_instance;
    }

    private:
      Singleton();
      ~Singleton();
      Singleton(const Singleton& s);
      Singleton& operator=(const Singleton& s);
  }

```

## 4.Eager 方式 ##
  与Lazy方式有点类似，不过在外部完成初始化，不用指针;
```cpp
  class Singleton{
    public:
      Singleton& Instance(){
        return single_instance;
      }

    private:
      Singleton();
      ~Singleton();
      Singleton(const Singleton& s);
      Singleton& operator=(const Singleton& s);
      
      static Singleton single_instance;
  }
```
```cpp
  Singleton Singleton:singleton_instance;

```


  


















