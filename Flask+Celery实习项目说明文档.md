# Flask+Celery实习项目说明文档



## Background

### Flask

基于python的小型web框架，支持自由组合扩展。使用WSGI即python的标准web接口

初始化项目-编列路由-编列jinja2的渲染模板（包含条件语句）

### SQLAlchemy

*创建数据库模型*，继承自`db.Model`

### celery

一个简单、灵活且可靠的，处理消息的分布式MQ系统，支持搭配redis消息代理系统

支持异步、延迟、定时任务

三种任务：

@celery.task def get_stable_rates_html:获取稳定率 并渲染html.

@celery.task verify_all_rules: 校验所有规则的定时任 务,并考虑发送告警邮件

@celery.task verify:校验指定任务,直接调用_do_verify.

#### 基本概念

任务队列（Task Queue）系统的基本概念主要包括以下几个方面，这些概念在使用像 Celery 这样的分布式任务队列时非常重要：

1. **任务（Task）**

   任务是指需要被执行的工作单元。通常是某种函数或方法，代表一个逻辑操作或一段代码。例如，发送邮件、处理图像、生成报告等都可以定义为一个任务。

   **任务的属性**：
   - **异步执行**：任务被放入队列后可以异步执行，不会阻塞主线程。
   - **重试机制**：某些任务如果执行失败，可以配置自动重试。
   - **优先级**：任务可以根据优先级进行调度，优先级高的任务会先被处理。

2. **队列（Queue）**

   队列是任务等待执行的地方。任务在被工作节点（Worker）执行之前，都会先放入队列中，队列通常按照**先进先出（FIFO）**的顺序进行任务的调度。

   **队列的特性**：
   - **多队列**：系统中可以存在多个队列，每个队列可以处理不同类型的任务。
   - **分布式队列**：队列可以被分布在不同的物理节点上，以实现分布式处理。

3. **工作节点（Worker）**

   Worker 是负责从队列中取出任务并执行它的后台进程。每个 Worker 可以处理一个或多个队列中的任务，并异步执行这些任务。Worker 可以在多个服务器上运行，以提高并行处理能力。

   **Worker的特性**：
   - **并发执行**：可以配置 Worker 的并发度，以实现同时处理多个任务。
   - **可扩展性**：随着工作量的增加，新的 Worker 可以动态添加到系统中。

4. **消息代理（Message Broker）**

   消息代理是负责传递任务的消息中间件，任务从应用程序发送到消息代理，Worker 从消息代理中获取任务进行处理。常见的消息代理包括：
   - **RabbitMQ**：高级的消息队列协议，支持复杂的消息路由、持久化等特性。
   - **Redis**：高性能的键值存储，也可以作为轻量级的消息队列。

5. **任务结果（Result/Backends）**

   当任务执行完毕后，系统可以选择存储任务的结果。结果存储（Backend）用于保存任务执行的状态和返回的结果，以便后续查询。常见的结果存储包括：
   - **内存存储**：用于小规模系统的临时存储。
   - **数据库**：如 MySQL、PostgreSQL。
   - **NoSQL 数据库**：如 Redis、MongoDB。

6. **调度器（Scheduler）**

   调度器是负责定时或周期性地将任务放入队列的组件。它允许你配置任务的执行时间，比如每天午夜执行某个任务。Celery 提供了 `celery beat` 来实现定时调度功能。

7. **中继任务（Chaining Tasks）**

   中继任务允许多个任务按顺序依次执行，一个任务的输出可以作为下一个任务的输入。常见的有两种形式：
   - **链（Chains）**：顺序执行多个任务，任务 A 完成后调用任务 B，再调用任务 C。
   - **组（Groups）**：并行执行多个任务，并收集所有任务的结果。

8. **重试机制（Retries）**

   如果某个任务因为某种错误（如网络问题、服务器故障）而失败，Celery 支持对失败的任务进行自动重试，直到任务成功或达到最大重试次数。

9. **路由（Routing）**

   路由是决定任务分配到哪个队列和哪个 Worker 的机制。你可以配置不同类型的任务被路由到不同的队列，甚至不同的服务器。

10. **任务的生命周期**

   一个任务通常经历以下几个阶段：
   1. **任务定义**：定义任务的逻辑操作。
   2. **任务调度**：任务被放入队列中，等待 Worker 获取。
   3. **任务执行**：Worker 从队列中获取任务并执行它。
   4. **任务结果存储**：任务完成后，结果可以选择性地存储。
   5. **任务查询**：可以通过任务 ID 查询任务状态或结果。

通过这些概念，任务队列系统可以高效地处理大量并发任务，解耦系统中的各个部分，并提高系统的可扩展性和容错性。

## Project details

[![O4x5Nx.png](https://ooo.0x0.ooo/2024/09/13/O4x5Nx.png)](https://img.tg/image/O4x5Nx)

[![O4xBLj.png](https://ooo.0x0.ooo/2024/09/13/O4xBLj.png)](https://img.tg/image/O4xBLj)

## 增补八股

### 面向对象主要特性

面向对象编程（OOP）的四大主要特性是：

1. **封装（Encapsulation）**：封装是将对象的属性和行为（方法）打包在一起，并通过定义公共接口控制外界对其内部状态的访问。封装有助于隐藏对象的内部细节，只暴露必要的功能，从而提高安全性和代码可维护性。
2. **继承（Inheritance）**：承允许一个类从另一个类（父类或基类）继承属性和方法，从而复用已有的代码。子类不仅能够继承父类的行为，还可以扩展或修改这些行为，促进代码的复用和扩展。
3. **多态（Polymorphism）**：多态允许不同的类以统一的接口执行操作。即使对象属于不同的类，它们可以通过相同的方法调用进行不同的行为表现。常见的形式包括方法重写（Override）和方法重载（Overload），通过不同的实现方式提供灵活性。
4. **抽象（Abstraction）**：抽象是对复杂现实世界的简化表示，专注于对象的必要特征而忽略细节。通过定义接口或抽象类，能够规范一组对象的行为，并确保子类实现这些行为，而不关心具体的实现细节。

这些特性共同作用，使得面向对象编程在组织复杂系统、提高代码复用性和可扩展性方面具有很大优势。

### 设计模式

设计模式（Design Patterns）是软件开发中的最佳实践和模板，旨在解决常见设计问题。设计模式主要分为三大类：**创建型模式**、**结构型模式**和**行为型模式**。每种模式都有其特定的使用场景和目标，下面列举常见的设计模式并介绍如何在 Python 中实现这些模式。

### 创建型模式（Creational Patterns）
1. **单例模式（Singleton）**
   - **目的**: 保证一个类只有一个实例，并提供一个全局访问点。
   - **Python 实现**:
     ```python
     class Singleton:
         _instance = None
     
         def __new__(cls):
             if cls._instance is None:
                 cls._instance = super().__new__(cls)
             return cls._instance
     ```

2. **工厂模式（Factory）**
   - **目的**: 定义一个创建对象的接口，但让子类决定实例化哪个类。
   - **Python 实现**:
     ```python
     class AnimalFactory:
         def create_animal(self, animal_type):
             if animal_type == 'dog':
                 return Dog()
             elif animal_type == 'cat':
                 return Cat()
     
     class Dog:
         def speak(self):
             return "Woof!"
     
     class Cat:
         def speak(self):
             return "Meow!"
     ```

3. **抽象工厂模式（Abstract Factory）**
   - **目的**: 提供一个接口，用于创建一组相关或依赖的对象，而无需指定具体类。
   - **Python 实现**:
     ```python
     class DogFactory:
         def create_animal(self):
             return Dog()
     
     class CatFactory:
         def create_animal(self):
             return Cat()
     
     def get_factory(factory_type):
         if factory_type == 'dog':
             return DogFactory()
         elif factory_type == 'cat':
             return CatFactory()
     ```

4. **建造者模式（Builder）**
   - **目的**: 将对象的构建过程与表示分离，使得同样的构建过程可以创建不同的对象。
   - **Python 实现**:
     ```python
     class House:
         def __init__(self):
             self.walls = None
             self.roof = None
     
     class HouseBuilder:
         def __init__(self):
             self.house = House()
     
         def build_walls(self):
             self.house.walls = 'Walls built'
             return self
     
         def build_roof(self):
             self.house.roof = 'Roof built'
             return self
     
         def get_house(self):
             return self.house
     
     builder = HouseBuilder()
     house = builder.build_walls().build_roof().get_house()
     ```

### 结构型模式（Structural Patterns）
1. **适配器模式（Adapter）**
   - **目的**: 将一个类的接口转换为客户端期望的另一个接口。
   - **Python 实现**:
     ```python
     class EuropeanSocket:
         def voltage(self):
             return 230
     
     class USASocketAdapter:
         def __init__(self, european_socket):
             self.european_socket = european_socket
     
         def voltage(self):
             return 110
     
     european_socket = EuropeanSocket()
     adapter = USASocketAdapter(european_socket)
     print(adapter.voltage())  # 输出 110
     ```

2. **装饰器模式（Decorator）**
   - **目的**: 动态地给对象添加职责，而不改变其接口。
   - **Python 实现**:
     ```python
     class Coffee:
         def cost(self):
             return 5
     
     class MilkDecorator:
         def __init__(self, coffee):
             self.coffee = coffee
     
         def cost(self):
             return self.coffee.cost() + 2
     
     coffee = Coffee()
     milk_coffee = MilkDecorator(coffee)
     print(milk_coffee.cost())  # 输出 7
     ```

3. **外观模式（Facade）**
   - **目的**: 提供一个统一的接口来访问子系统中的一群接口。
   - **Python 实现**:
     ```python
     class CPU:
         def start(self):
             print("CPU starting")
     
     class Memory:
         def load(self):
             print("Memory loading")
     
     class ComputerFacade:
         def __init__(self):
             self.cpu = CPU()
             self.memory = Memory()
     
         def start(self):
             self.cpu.start()
             self.memory.load()
     
     computer = ComputerFacade()
     computer.start()
     ```

### 行为型模式（Behavioral Patterns）
1. **观察者模式（Observer）**
   - **目的**: 定义对象间的依赖关系，当一个对象改变时通知其他对象。
   - **Python 实现**:
     ```python
     class Subject:
         def __init__(self):
             self._observers = []
     
         def register(self, observer):
             self._observers.append(observer)
     
         def notify(self):
             for observer in self._observers:
                 observer.update()
     
     class Observer:
         def update(self):
             print("Observer notified")
     
     subject = Subject()
     observer = Observer()
     subject.register(observer)
     subject.notify()
     ```

2. **策略模式（Strategy）**
   - **目的**: 定义一系列算法，并使它们可以互换使用。
   - **Python 实现**:
     ```python
     class StrategyA:
         def execute(self):
             return "Executing strategy A"
     
     class StrategyB:
         def execute(self):
             return "Executing strategy B"
     
     class Context:
         def __init__(self, strategy):
             self.strategy = strategy
     
         def set_strategy(self, strategy):
             self.strategy = strategy
     
         def execute_strategy(self):
             return self.strategy.execute()
     
     context = Context(StrategyA())
     print(context.execute_strategy())  # 输出: "Executing strategy A"
     context.set_strategy(StrategyB())
     print(context.execute_strategy())  # 输出: "Executing strategy B"
     ```

3. **命令模式（Command）**
   - **目的**: 将请求封装成对象，从而使我们可以用不同的请求对客户进行参数化。
   - **Python 实现**:
     ```python
     class Command:
         def execute(self):
             pass
     
     class LightOnCommand(Command):
         def execute(self):
             print("Light is On")
     
     class LightOffCommand(Command):
         def execute(self):
             print("Light is Off")
     
     class RemoteControl:
         def __init__(self):
             self._command = None
     
         def set_command(self, command):
             self._command = command
     
         def press_button(self):
             self._command.execute()
     
     remote = RemoteControl()
     remote.set_command(LightOnCommand())
     remote.press_button()
     remote.set_command(LightOffCommand())
     remote.press_button()
     ```

这些是设计模式的一些经典实现，Python 可以通过其简洁的语法和动态特性简化许多设计模式的实现。



### Python多线程/多进程

python多线程用的是threading，底层上受到GIL的限制，因为 Python 内部的某些对象 (如内置的整数和列表) 的内存管理并不是线程安全的。GIL 确保了多线程访问 Python 对象的安全性，但代价是它阻止了多线程 Python 代码的并行执行。

GIL 的存在意味着即便在多核 CPU 上运行，多个线程也无法同时执行 Python 字节码。线程之间会轮流获得 GIL，这使得多线程在 CPU 密集型任务上无法有效利用多核优势。

结论：python的多线程没用

Python 的多进程是通过 multiprocessing 模块实现的，底层使用了操作系统提供的进程管理机制。不同于线程，进程是操作系统分配资源的独立单元，每个进程都有独立的内存空间和资源，**不共享全局解释器锁**。

由于每个进程都有独立的内存空间，Python 提供了几种方式进行进程间通信 (IPC)：

- **管道 (Pipes)**：使用操作系统的管道机制进行数据传输。
- **队列 (Queue)**：实现了线程和进程安全的队列，用于进程间数据共享。
- **共享内存 (Shared Memory)**：通过共享内存区域，允许多个进程共享数据而无需复制。
- **Manager 对象**：multiprocessing.Manager 提供了进程间共享 Python 对象的方式，例如列表、字典等。

对于大多数使用 CPython 的程序，GIL 限制了线程的并行执行能力。在 I/O 密集型任务中，线程仍然有助于提高性能，但在计算密集型任务中，多线程的效果非常有限。
