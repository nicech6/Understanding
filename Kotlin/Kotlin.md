## Kotlin语法糖
> Kotlin 语言是一种新的静态类型编程语言，可运行于 JVM 环境同时也能用来开发 Android 应用

### 准备工作
#### 1.安装Kotlin插件 (本篇文章基于AS，IDE类似)
![Alt text](./1-3-1.png)
#### 2.新建.kt文件
![Alt text](./1-4-3.png)
### 撸码
> 变量声明 
``` java
 private var unicode: String? = null
 private var studentId: Int? = null
```
> 类方法 
``` java
class MainActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
    }
}
```
> 静态方法
``` java
 companion object {
        fun start(id: Int) {}
    }
```
* 方法如果没有方法体，{} 可省略

> for循环(0到10++)
``` java
// 升序
 for (i in 0..10){
            等于java中 for (int i = 0; i < 10; i++) {
        }
        }
// 升序还有一种写法: 
 for (i in list.indices) {
   print(list[i])
}
// 降序
 for (i in 10 downTo 2) {
同理 
        }
```
> if 表达式
``` java
fun max(a: Int, b: Int): Int {
    if (a > b)
        return a
    else
        return b
}
```
> when表达式 (替代switch表达式 )
``` java
    when (id) {
            0 -> {
            }
            1 -> {
            }
        }
```
> while表达式 (与java一致 )
``` java
while (a < 10) {
            a += b
        }
```
> Lambda表达式 
``` java
val Lambda: (Int, Int) -> Int = {x,y -> x+y}
```
> 主构造方法 ( 如果主构造哦函数没有注解或可见性说明，则 constructor 关键字是可以省略：)
``` java
class Student constructor(name: String) {
}
```
>get set方法
``` java
 var id: Int = 0 // 初始化后默认已经实现了get.set方法
```
>接口定义
``` java
interface MyInterface {
    fun success()
    fun failed() {
      // optional body
    }
```
> 实现接口
``` java
class test: MyInterface {
   override fun failed() {
      // body
   }
}
```
> 继承
``` java
 open class a{
       父类必须是 使用 open关键字来描述  
    }
    class b: a() {
       
    }
```
> 创建类的实例 
``` java
var test = test()
// Kotlin没有new关键字
```
> list 
``` java
var list=ArrayList<String>()
//遍历
list.forEach {
    print(it)
}
```
> 万能的冒号
> 在Kotlin中，冒号可以说是无处不在,那么到底什么时候能用到冒号呢？
``` java
val表示常量var表示变量声明	 val name: String = "kotlin" 
类继承					 class MyActivity : Activity()
方法参数					 fun a (id : Int){}
```
> 空判断
``` java
var name: String? = "kotlin" 	表示这个变量可以为空
val name = names!!.toInt()	表示为null的时候报空值针
val name = names?!.toInt()	安全，为null不报空值针
val id= ids?.toInt() ?: -1	为空 则返回-1
```
*  [官方文档](https://kotlinlang.org/docs/reference/)
*  [Kotlin中文论坛](https://www.kotliner.cn/)



