[阿里巴巴 Java 开发规范](https://github.com/alibaba/p3c/blob/master/%E9%98%BF%E9%87%8C%E5%B7%B4%E5%B7%B4Java%E5%BC%80%E5%8F%91%E6%89%8B%E5%86%8C%EF%BC%88%E5%8D%8E%E5%B1%B1%E7%89%88%EF%BC%89.pdf)

## 减少 if else …… 代码
### 表驱动
表驱动有三种方法：直接驱动法、索引驱动法、阶梯访问表
1. 直接驱动法
一维表查找、二维表、多维表查找
一维查找比如查找每个月多少天？
```
month_day[12] = {31,30,29,30,30,31,31,31,30,30,31,30};
```
二维查找比如根据是否学生、性别决定景区门票的价格
2. 索引驱动法
如果是按照年龄来分，每个年龄的景区门票价格不一样，如果每一个年龄都建一个下标，那会很多，所以可以针对一个范围建立一个索引，这个算是表驱动的优化了。
3. 阶梯访问表
很多问题，不需要一一对应，而是将其进行归类，比如学生的成绩，分为 A、B、C、D 档位。
### 前置 return
就是判断条件取方的方法，代码在逻辑表达上更加清晰，尽量判断条件在最上层或者最外层做掉，最后才是业务的核心代码，变得很干净。
### 策略模式
1. 多态
通过接口和实现接口实现多态，然后具体的策略放在一个 map 中，然后获取这个对象，运行对象中相对应的方法从而实现特定的业务。
2. 枚举
其实在枚举中可以定义属性（状态），还可以在枚举中定义实现一个 run 方法。
### 使用 Optional
Optional 主要用于非空判断，那么如何使用正确使用 Optional 呢。
1. 尽量避免在程序中直接调用Optional对象的get()和isPresent()方法；
2. 避免使用Optional类型声明实体类的属性；
第一条建议中直接调用get()方法是很危险的做法，如果Optional的值为空，那么毫无疑问会抛出NullPointerException异常，而为了调用get()方法而使用isPresent()方法作为空值检查，这种做法与传统的用if语句块做空值检查没有任何区别。

第二条建议避免使用Optional作为实体类的属性，它在设计的时候就没有考虑过用来作为类的属性，如果你查看Optional的源代码，你会发现它没有实现java.io.Serializable接口，这在某些情况下是很重要的（比如你的项目中使用了某些序列化框架），使用了Optional作为实体类的属性，意味着他们不能被序列化。
```
Integer i = null;
// 第一种使用方式，和普通的 if else 没有任何区别
Optional<Integer> integerOptional = Optional.ofNullable(i);
if (integerOptional.isPresent()) {
    System.out.println("not null.");
} else {
    System.out.println("is null.");
}

/**
 * (1)尽量在程序中避免直接调用 Optional 中的 get() 和 isPresent() 方法
 * (2)避免使用Optional类型声明实体类的属性
 */

// 第二种使用方式，和三目表达式差不多，简化了代码
Integer resultOne =  integerOptional.orElse(2);

// 第三种使用方式，可以简化 if else 方法
int resultTwo = integerOptional.map(Integer::intValue).orElse(0);

System.out.println("resultOne:" + resultOne + "resultTwo:" + resultTwo);
```
### if else 嵌套太多
if  else 表达式过于复杂，可以把校验、判断代码放在方法开头，核心逻辑放在后面处理。

### 代码整洁
命名服务，可读，简短，单复数
方法短小，单一职责，抽象层级，减小嵌套
if else 表驱动，枚举，策略，抽象工厂，optional，前置驱动
注释
格式，相同代码紧接着，不同代码空格
代码不需要过度设计
code review 勤于重构 ide自动重构技巧 自动化测试 
代码静态检查，sonar，阿里巴巴插件，测试单元测试覆盖率 sonarqube
系统架构设计，广度优先，而不是深度优先
模板方法，公共方法到父类，不同方法抽象起来，自定义实现


















