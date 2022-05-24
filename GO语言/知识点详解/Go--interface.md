

# GO接口

### GO语言与鸭子类型的关系

##### 静态语言和动态语言，强类型语言和弱类型语言

如果类型检查发生在编译阶段(compile time)，那么是静态类型语言(statically typed languages)中，相反的，如果类型检查发生在运行阶段(run time)，那么是动态类型语言(dynamically typed languages)

<img src="assets/aHR0cHM6Ly9hc2sucWNsb3VkaW1nLmNvbS9odHRwLXNhdmUvNjQzMDE4NC8yM2V2aG42NXJoLmpwZWc" alt="在这里插入图片描述" style="zoom:50%;" />

##### 鸭子类型和多态

多态的概念是应用于Java和C#这一类强类型语言中，而Python崇尚"鸭子类型"

**<u>*鸭子类型：*</u>**动态语言调用实例方法时**不检查类型，只要方法存在，参数正确，就可以调用**。这就是动态语言的“鸭子类型”，它并不要求严格的继承体系，一个对象只要“看起来像鸭子，走起路来像鸭子”，那它就可以被看做是鸭子。

在动态语言 python 中，定义一个这样的函数：

```python
def hello_world(coder):
    coder.say_hello()
```

当调用此函数的时候，可以传入任意类型，只要它实现了 `say_hello()` 函数就可以。如果没有实现，运行过程中会出现错误，这就是**鸭子类型**。而在java中必须要显示的声明实现某个接口，**这就是多态**

**<u>*多态：*</u>**定义时的类型和运行时的类型不一样，就称为多态。

多态性的好处在于增强了程序的灵活性和可扩展性，对于鸭子类型只需要制造出外观和行为相同对象，同样可以实现不考虑对象类型而使用对象，这正是Python崇尚的“鸭子类型”(duck typing)比起继承的方式，**鸭子类型在某种程度上实现了程序的松耦合度**

而Go语言作为一门静态语言，它引入了动态语言的便利，同时又会进行静态语言的类型检查，GO语言不要求类型显示地声明实现了某个接口，只要实现了相关的方法即可，编译器就能检测到。

##### 举个例子

先定义一个接口，和使用此接口作为参数的函数：

```go
type IGreeting interface {
	sayHello()
}

func sayHello(i IGreeting) {
	i.sayHello()
}
```

再来定义两个结构体:

```
type Go struct {}
func (g Go) sayHello() {
	fmt.Println("Hi, I am GO!")
}

type PHP struct {}
func (p PHP) sayHello() {
	fmt.Println("Hi, I am PHP!")
}
```

最后，在 main 函数里调用 sayHello() 函数

```go
func main() {
	golang := Go{}
	php := PHP{}

	sayHello(golang)
	sayHello(php)
}
```

可以看到Go和PHP没有继承IGreeting，知识实现了其sayHello方法，就可以传入，调用

### 值传递和指针传递

