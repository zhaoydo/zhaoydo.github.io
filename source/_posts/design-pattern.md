---
title: 设计模式总结
date: 2020-09-05 15:38:00
tags:
---
原则

# 创建型模式

## 简单工厂
最简单的创建型模式，工厂类提供静态方法，根据不同参数构造对象  

<!--more-->

```java
/**
 * 简单工厂模式
 * 理解：一个静态类根据不同参数派生出对象（同一父类或接口）
 * 职责单一原则，一个类只提供一种功能，FoodFactory只提供生产Food的功能
 * 定义一个Food工厂，实现根据参数foodName创建不同的Food对象
 * @param foodName
 * @return
 */
public static Food makeFood(String foodName){
	//LanzhouNoodle、HuangmenChicken是Food的子/实现类
    if("noodle".equals(foodName)){
        Food noodle = new LanzhouNoodle();
        noodle.ingredients = "拉面";
        return noodle;
    }else if("chicken".equals(foodName)){
        Food chicken = new HuangmenChicken();
        chicken.ingredients = "鸡肉";
        return chicken;
    }else{
        return null;
    }
}

```
与构造函数对比，对比优缺点  
优点：  
- 自定义方法名，可读性更好
- 允许根据条件决定是否实例化，可以返回null
- 可以实例化返回类型的派生类，实现根据参数返回不同的派生类
缺点：  
- 与构造器相比，静态工厂方法是普通静态类，只能通过方法名识别

## 工厂模式
有时候需要两种以上的工厂，这里就需要工厂模式了。  
与简单工厂模式相比，工厂分为工厂接口+具体的工厂类。不同工厂类生产不同类的产品。  
理解：

1. 选择合适的工厂类
2. 调用工厂类静态工厂方法(简单工厂模式)  

```java
/**
 * 工厂模式
 * FoodFactory分为ChineseFoodFactory、AmericanFoodFactory，根据业务需要调用不同的工厂类
 * @param args
 */
public static void main(String[] args) {
    //1、选择1个具体的工厂
    FoodFactory foodFactory = new ChineseFoodFactory();
    //2、由具体工厂对象，makeFood
    Food food = foodFactory.makeFood("A");
}

//工厂接口
interface FoodFactory{
    Food makeFood(String foodName);
}

//chineseFood 工厂
static class ChineseFoodFactory implements FoodFactory{
    @Override
    public Food makeFood(String foodName) {
        if("A".equals(foodName)){
            return new ChineseFoodA();
        }else if("B".equals(foodName)){
            return new ChineseFoodB();
        }else{
            return null;
        }
    }
}
//americanFood 工厂
static class AmericanFoodFactory implements FoodFactory{
    @Override
    public Food makeFood(String foodName) {
        if("A".equals(foodName)){
            return new AmericanFoodA();
        }else if("B".equals(foodName)){
            return new AmericanFoodB();
        }else{
            return null;
        }
    }
}

static class Food{
    /**原料*/
    private String ingredients;
}

static class ChineseFoodA extends Food {
}
static class ChineseFoodB extends Food {
}
static class AmericanFoodA extends Food {
}
static class AmericanFoodB extends Food {
}
```

## 抽象工厂模式
理解：
1. 工厂抽象（ComputerFactory）
2. 产品抽象（代码中用String省略了）
3. 产品间关系抽象（new Computer(cpu,mainBoard)）

```java
/**
 * 抽象工厂模式
 * @author zhaoyd
 * @date 2020-06-05
 */
public class AbstractFactory {
   
    public static void main(String[] args) {
        //选定某个品牌的大厂， 例如AMD
        ComputerFactory computerFactory = new AmdFactory();
        //生产amd cpu
        String cpu = computerFactory.makeCPU();
        //生产amd主板
        String mainBoard = computerFactory.makeMainBoard();
        //amd cpu 主板 组装成电脑
        Computer computer = new Computer(cpu, mainBoard);
    }

    /**
     * 定义一个抽象类
     */
    interface ComputerFactory{
        //不想定义CPU实体类了，用string偷个懒
        String makeCPU();
        String makeMainBoard();
    }

    static class IntelFactory implements ComputerFactory{
        @Override
        public String makeCPU() {
            return "intel cpu";
        }
        @Override
        public String makeMainBoard() { return "intel mainBoard"; }
    }
    static class AmdFactory implements ComputerFactory{
        @Override
        public String makeCPU() {
            return "amd cpu";
        }
        @Override
        public String makeMainBoard() { return "amd mainBoard"; }
    }
    /**
     * 产品
     */
    static class Computer{
        Computer(String cpu, String board){
            //省略...
        }
    }
}
```
缺点：如果需要增加工厂的产品，例如ComputerFactory中增加makeMonitor，需要给所有的工厂增加实现类。这一点违反了“对修改关闭，对扩展开放”的原则。

## 单例模式
四种：饿汉式、饱汉式、嵌套式、枚举
### 饿汉式
通过静态方法生成对象实例
```java
/**
 * 单例模式-饿汉（提前生成实例，线程安全，缺点是不必要的时候也生成了单例示例）
 * 1、私有化构造方法
 * 2、创建私有静态实例
 * 3、提供公共方法获取实例
 * @author zhaoyd
 * @date 2020-06-20
 */
public class Singleton1 {
    /**1、私有化构造方法，将new Singleton1()屏蔽*/
    private Singleton1(){
        System.out.println("执行构造方法");
    };
    /**2、创建私有静态实例，类第一次使用（第一次加载）会创建*/
    private static Singleton1 instance = new Singleton1();

    /**
     * 3、公共方法获取实例
     * @return
     */
    public static Singleton1 getInstance(){
        return instance;
    }
    /**
     * 其他静态方法， 当调用Singleton1.getDate()，时也会生成Singleton1实例，这算是饿汉式的一个缺点
     * @return
     */
    public static Date getDate(){
        return new Date();
    }
}
```
### 饱汉式
饱汉式复杂一些
1. 静态示例变量用volatile修饰，因为Singleton instance = new Singleton();非原子操作，可能先赋值空对象，再初始化对象。 用volatile禁止指令重排序
2. 使用双重检查锁， 第一重锁保证性能， 第二重锁保证安全  

```java
/**
 * 单例模式-饱汉（不预先生成实例，提供公共方法生成实例，使用双重检查锁保证线程安全问题）
 * @author zhaoyd
 * @date 2020-06-20
 */
public class Singleton2 {
    /**1、私有化构造方法*/
    private Singleton2(){}
    /**2、静态变量，不预先创建实例，默认为null*/
    //volatile 防止jvm对instance的修改操作（new Singleton2()）重排 参考https://blog.csdn.net/u011248395/article/details/71087379
    private static volatile Singleton2 instance = null;

    /**
     * 创建实例
     * synchronized锁住整个方法，会导致每次请求都走锁，性能不高
     * 使用双重检查锁， 第一重锁保证性能， 第二重锁保证安全
     */
    public Singleton2 getInstance(){
        //第一重检查，线程不安全，但是可以过滤掉绝大部分请求，
        if(instance == null){
            //第二重检查，加锁。 线程安全，牺牲性能（因为第一重检查过滤掉绝大部分了），保证单例
            synchronized(Singleton2.class){
                if(instance == null){
                    instance = new Singleton2();
                }
            }
        }
        return instance;
    }
}
```
### 嵌套类
避免了饿汉式模式，调用其他不相关的静态方法导致类加载时生成实例（例如上面代码中调用Singleton1.getDate()，时也会生成Singleton1实例）
```java
/**
 * 单例模式-嵌套类
 * 和饿汉式类似， 加了一层嵌套类保证调用其他静态方法时，不额外生成实例
 * @author zhaoyd
 * @date 2020-06-20
 */
public class Singleton3 {

    private Singleton3(){
        System.out.println("构造方法");
    }

    private static class Holder{
        private static Singleton3 instance = new Singleton3();
    }

    public static Singleton3 getInstance(){
        return Holder.instance;
    }
    public static Date getDate(){
        return new Date();
    }

    public static void main(String[] args) {
        Singleton3.getDate();
    }
}
```
### 枚举
《Effective Java》上推荐的，枚举天然是单例，并且能方式反序列化破坏单例
```java
/**
 * 单例模式-枚举
 * @author zhaoyd
 * @date 2020-06-20
 */
public enum Singleton4 {

    INSTANCE();

    public void doSomething() {
        System.out.println("doSomething");
    }
}
```

总结：
1. 饿汉式，可能会创建必须要的实例
2. 饱汉式，代码复杂
3. 嵌套类，推荐，但是可能会通过反序列化破坏单例，不过一般不考虑这种情况
4. 枚举，推荐

## 建造者模式
很常见,链式调用一堆方法，最后调用build()方法构造对象。类名一般为xxxBuilder
```java
User user = User.builder().name("Tom").password("***").age(10).build();
```
```java
/**
 * @author zhaoyd
 * @date 2020-06-20
 */
public class UserBuilder{
    private String  name;
    private String password;
    private int age;
    
    public UserBuilder name(String name){
        this.name = name;
        return this;
    }
    public UserBuilder password(String password){
        this.password = password;
        return this;
    }
    public UserBuilder age(int age){
        this.age = age;
        return this;
    }
    public User build(){
        //这里可以加一些校验的方法
        if(this.age<=0){
            throw new RuntimeException("年龄不合法");
        }
        //建造User对象
        User user = new User();
        user.setName(this.name);
        user.setPassword(this.password);
        user.setAge(this.age);
        return user;
    }
}
```
## 原型模式
定义：根据已有的一个原型实例，基于这个实例产生新的实例。也就是克隆对象  
Object类中的clone方法，要调用这个方法需要实现Cloneable接口，否则会抛出CloneNotSupportedException异常
```java
protected native Object clone() throws CloneNotSupportedException;
```
注意，java的克隆是浅克隆，如果需要深克隆，将对象序列化再反序列化

## 创建型模式总结
创建型模式的特点是将创建对象与使用对象分离。封装创建对象的逻辑，使用时能够简单的通过参数、创建类来轻松创建所需的对象
# 结构型模式
## 代理模式
实现类、代理类都实现同一接口。代理类的实现方法中调用实现类的方法，又在调用前后额外加了点方法。  
spring中的aop用到了动态代理，核心思想是一样的
```java
/**
 * 代理模式
 * @author zhaoyd
 * @date 2020-06-22
 */
public class FoodServiceProxy implements FoodService {

    private FoodServiceImpl foodService = new FoodServiceImpl();
    @Override
    public void makeNoodle() {
        //执行方法前做一些事
        System.out.println("准备制作面条");
        foodService.makeNoodle();
        //执行方法后做一些事
        System.out.println("面条制作完成啦");
    }

    @Override
    public void makeChicken() {
        System.out.println("准备制作鸡肉");
        foodService.makeChicken();
        System.out.println("鸡肉制作完成啦");
    }


    public static void main(String[] args) {
        FoodService foodService = new FoodServiceProxy();
        foodService.makeChicken();
    }
}
```
## 适配器模式
### 默认适配器模式
例如一个接口，定义了很多方法，我们只需要实现其中1个方法。 如果直接实现接口的话，要定义不少空方法，代码多。   
这种情况下定义一个默认实现类（适配器），实现所有方法。 接口=>适配器=>实现类， 这样实现类只需要重写需要用到的方法即可
> jdk1.8中，接口允许定义一个默认实现，和默认适配器的道理是一样的
```java
/**
 * 用适配器来构造默认的实现
 * @author zhaoyd
 * @date 2020-06-23
 */
public class FileAlterationListenerAdaptor implements FileAlterationListener {
    @Override
    public void onCreate() {
    }

    @Override
    public void onDelete() {
    }

    @Override
    public void onChange() {
    }
}
```
### 对象适配器模式
将一个对象适配成另一个对象。 例如通过适配器将鸡适配成鸭
```java
//部分代码
public class CockAdapter implements Duck {
    private Cock cock;
    public CockAdapter(Cock cock){
        this.cock = cock;
    }
    @Override
    public void quack() {
        //公鸡不会鸭子的呱呱叫，用咕咕叫实现
        cock.gobble();
    }
    @Override
    public void fly() {
        cock.fly();
    }

    public static void main(String[] args) {
        //定义一只野鸡
        Cock wildCock = new WildCock();
        //将野鸡适配成鸭
        Duck duck = new CockAdapter(wildCock);

        duck.quack();
        duck.fly();
    }
}
```
### 类适配器模式
比较常用  
适配器继承、实现的方式来获得父类和多个接口的特性（不再代码演示了）
> Adapter extends BaseServiceImpl implements BaseService1,BaseService2

三种适配器总结：
1. 默认适配器：实现接口的空方法
2. 对象适配器：将一个对象适配成另一个对象
3. 类适配器： 适配器继承、实现的方式来获得父类和多个接口的特性

## 桥梁模式
直接看代码，就是类中注入不同的接口实现类来实现不同的功能。在这里接口作为桥梁  
```java
/**
 * 桥梁模式
 * 形状+颜色，
 * 定义形状，注入画笔颜色接口的不同实现，连接形状和画笔颜色
 * @author zhaoyd
 * @date 2020-07-02
 */
public class Bridge {
    // 这个接口是桥梁,连接形状与画笔，Shape-（DrawAPI）-具体的画笔
    public interface DrawAPI{
        public void draw(int radius, int x, int y);
    }

    public static class RedPen implements DrawAPI{
        @Override
        public void draw(int radius, int x, int y) {
            System.out.println("用红色笔画图，radius:" + radius + ", x:" + x + ", y:" + y);
        }
    }
    public static class GreenPen implements DrawAPI{
        @Override
        public void draw(int radius, int x, int y) {
            System.out.println("用绿色笔画图，radius:" + radius + ", x:" + x + ", y:" + y);
        }
    }

    public static abstract class Shape{
        protected DrawAPI drawAPI;

        public Shape(DrawAPI drawAPI){
            this.drawAPI = drawAPI;
        }
        public abstract void draw();
    }

    public static class Circle extends Shape{
        private int radius;
        public Circle(int radius,DrawAPI drawAPI) {
            super(drawAPI);
            this.radius = radius;
        }

        @Override
        public void draw() {
            super.drawAPI.draw(radius, 0 ,0);
        }
    }
    public static class Rectangle  extends Shape{
        private int x;
        private int y;
        public Rectangle(int x, int y,DrawAPI drawAPI) {
            super(drawAPI);
            this.x = x;
            this.y = y;
        }

        @Override
        public void draw() {
            super.drawAPI.draw(0, x ,y);
        }
    }

    public static void main(String[] args) {
        System.out.println("绿色的圆");
        Shape greenCircle = new Circle(10, new GreenPen());
        System.out.println("红色的长方形");
        Shape redRectangle = new Rectangle(4, 8, new RedPen());
        greenCircle.draw();
        redRectangle.draw();
    }
}
```

## 装饰者模式
javaIO中很多地方用到了装饰者模式
```java
/**
 * 装饰模式
 * 子类装饰父类
 * 1、饮料：红茶，绿茶，奶茶
 * 2、小料：焦糖，牛奶，布丁
 * 小料装饰饮料
 * @author zhaoyd
 * @date 2020-07-02
 */
public class Decorator {

    //1、定义饮料
    public static abstract class Drink{
        /**饮料描述*/
        public abstract String description();
        /**饮料价格*/
        public abstract BigDecimal price();
    }

    //2、定义红茶、绿茶
    public static class RedTeaDrink extends Drink{

        @Override
        public String description() {
            return "红茶";
        }

        @Override
        public BigDecimal price() {
            return new BigDecimal(10);
        }
    }
    public static class GreenTeaDrink extends Drink{

        @Override
        public String description() {
            return "绿茶";
        }

        @Override
        public BigDecimal price() {
            return new BigDecimal(12);
        }
    }
    //2、定义调料 为了装饰drink， 继承它(其实不用Codiment，调料直接继承Drink也可以)
    public static abstract class Condiment extends Drink {

    }

    //2、加料
    public static class CaramelCondiment extends Condiment{
        private Drink drink;
        public CaramelCondiment(Drink drink){
            this.drink = drink;
        }
        @Override
        public String description() {
            return drink.description()+"+焦糖";
        }

        @Override
        public BigDecimal price() {
            return drink.price().add(new BigDecimal(2.50));
        }
    }
    public static class MilkCondiment extends Condiment{
        private Drink drink;
        public MilkCondiment(Drink drink){
            this.drink = drink;
        }
        @Override
        public String description() {
            return drink.description()+"+牛奶";
        }

        @Override
        public BigDecimal price() {
            return drink.price().add(new BigDecimal(3.00));
        }
    }
}
```
调用  
```java
public static void main(String[] args) {
    //绿茶
    Drink greenTea = new GreenTeaDrink();
    //加奶
    greenTea = new MilkCondiment(greenTea);
    //焦糖
    greenTea = new CaramelCondiment(greenTea);
    System.out.println(greenTea.description() + "价格：" + greenTea.price());
    //红茶
    Drink redTea = new RedTeaDrink();
    //加奶
    redTea = new MilkCondiment(redTea);
    System.out.println(redTea.description() + "价格：" + redTea.price());

    //或者可以直接这样，这种很像java IO中的一些操作了  
	//InputStream inputStream = new LineNumberInputStream(new BufferedInputStream(new FileInputStream("")));
    Drink drink = new CaramelCondiment(new MilkCondiment(new GreenTeaDrink()));
}
```
理解装饰者模式：  
- 与继承相比，装饰者模式更加灵活
- 与组合相比，装饰者模式着重于对目标对象增强，组合模式着重于对一系列目标集成

## 门面模式
门面模式也叫外观模式，通过门面类的不同方法调用不同的实现类
```java
/**
 * 门面模式
 * 用门面类通过不同的方法调用不同的实现类，对调用者来说，更加清晰简洁
 * @author zhaoyd
 * @date 2020-07-09
 */
public class FacadeMode {

    //定义一个接口
    public static interface Shape{
        void draw();
    }
    //定义接口不同的实现（不同的形状）
    public static class Circle implements Shape{
        @Override
        public void draw() {
            System.out.println("画一个圆");
        }
    }
    //定义接口不同的实现（不同的形状）
    public static class Square implements Shape{
        @Override
        public void draw() {
            System.out.println("画一个正方形");
        }
    }
    //门面
    public static class ShapeMaker{
        private Shape circle;
        private Shape square;
        public ShapeMaker(){
            circle = new Circle();
            square = new Square();
        }

        public void drawCircle(){
            circle.draw();
        }
        public void drawSquare(){
            square.draw();
        }
    }
}
```
客户端调用  
```java
public static void main(String[] args) {
    ShapeMaker shapeMaker = new ShapeMaker();
    //画一个圆
    shapeMaker.drawCircle();
    //画一个正方形
    shapeMaker.drawSquare();
}
```

## 组合模式
理解：
1. 组合数据结构，一般是上下级关系，聚合关系
2. 组合功能  

```java
/**
 * 组合模式  上下级关系，聚合关系
 * @author zhaoyd
 * @date 2020-07-09
 */
public class Combination {
    //定义学校接口
    public static abstract class SchoolPart{
        public List<SchoolPart> subParts = new ArrayList<>();
        //添加部门
        public abstract void addPart(SchoolPart p);
        //移除部门
        public abstract void removePart(SchoolPart p);
        //获取子部门
        public abstract List<SchoolPart> getSubParts();
        //部门action
        public abstract void action();
        //部门action
        public abstract void actionAll();
    }

    /**
     * 校委会
     */
    public static class SchoolCommittee extends SchoolPart{

        @Override
        public void addPart(SchoolPart p) {
            subParts.add(p);
        }

        @Override
        public void removePart(SchoolPart p) {
            subParts.remove(p);
        }

        @Override
        public List<SchoolPart> getSubParts() {
            return subParts;
        }

        @Override
        public void action() {
            System.out.println("我是校委会");
        }

        @Override
        public void actionAll() {
            //调用自己
            this.action();
            //调用子类
            if(this.subParts != null){
                for (SchoolPart subPart : this.subParts) {
                    subPart.actionAll();
                }
            }
        }
    }
    /**
     * 招生办 无子节点
     */
    public static class SchoolAdmissions  extends SchoolPart{
        @Override
        public void addPart(SchoolPart p) {
        }

        @Override
        public void removePart(SchoolPart p) {
        }

        @Override
        public List<SchoolPart> getSubParts() {
            return subParts;
        }

        @Override
        public void action() {
            System.out.println("我是招生办");
        }
        @Override
        public void actionAll() {
            //调用自己
            this.action();
            //调用子类
            if(this.subParts != null){
                for (SchoolPart subPart : this.subParts) {
                    subPart.actionAll();
                }
            }
        }
    }
}

```
客户端调用  
```java
public static void main(String[] args) {
    //校委会
    SchoolPart schoolCommittee = new SchoolCommittee();
    SchoolPart schoolAdmissions = new SchoolAdmissions();
    schoolCommittee.addPart(schoolAdmissions);
	//调用自身
    schoolCommittee.action();
	//调用所有，包括下级
    schoolCommittee.actionAll();
}
```

## 享元模式
理解：实际是共享元器件， 例如在新建一个对象之前，查找有无存在的对象（缓存已创建的对象），如果有则可以复用 。
> 例如对象池的实现

## 结构型模式总结
1. 代理模式：方法增强
2. 适配器模式： 默认适配器：填充默认空方法；对象适配器：A对象适配为B对象；类适配器：适配父类和多个接口
3. 桥梁模式：各种颜色画笔和形状解耦
4. 装饰模式：装饰类和增强类
5. 门面模式：封装一层可读性强的方法作为门面，客户端不需要关心实例化的对象
6. 组合模式：层次的数据结构和功能
7. 享元模式：缓存已创建的对象

# 行为型模式
行为型模式关注各个类之间调用，将职责划分清楚

## 策略模式
跟桥梁模式很像，策略模式比桥梁模式少了一层抽象。 主要区别是桥梁模式着重于结构，策略模式着重于行为，废话哈哈。
```java
/**
 * 策略模式
 * 策略类Context通过选择不同策略（例如new BlackPen()），运行时体现不同逻辑
 * @author zhaoyd
 * @date 2020-07-23
 */
public class Strategy {
    //定义策略接口
    static interface IStrategy{
        void draw(int radius, int x, int y);
    }

    static class RedPen implements IStrategy{
        @Override
        public void draw(int radius, int x, int y) {
            System.out.println("用红笔画图");
        }
    }
    static class BlackPen implements IStrategy{
        @Override
        public void draw(int radius, int x, int y) {
            System.out.println("用黑笔画图");
        }
    }
    //
    static class Context{
        private IStrategy strategy;

        public Context(IStrategy strategy){
            this.strategy = strategy;
        }

        public void executeDraw(int radius, int x, int y){
            strategy.draw(radius, x, y);
        }
    }
}
```
客户端调用  
```java
public static void main(String[] args) {
	//传入BlackPen策略
    Context context = new Context(new BlackPen());
	//用黑笔画图
    context.executeDraw(10,0,0);
}
```
## 观察者模式
观察者模式在项目中很常用
1. 主题（Subject）中定义一系列观察者（Observer）
2. 给主题注册观察者（Subject.observers.add(Observer)）
3. 主题变更时，遍历观察者集合，触发动作（例如更新时，遍历执行Observer.update()）

先定义一个观察者抽象类
```java
/**
 * 观察者抽象类
 */
public abstract class AbstractObserver {
     protected Subject subject;
     abstract void update();
}
```
观察者的两个实现类  
二进制观察者
```java
/**
 * 二进制观察者
 * @author zhaoyd
 * @date 2020-06-10
 */
public class BinaryObserver extends AbstractObserver{

    BinaryObserver(Subject subject){
        this.subject = subject;
        //注册到subject的观察者
        subject.addObserver(this);
    }

    @Override
    public void update() {
        String result = Integer.toBinaryString(subject.getState());
        System.out.println("订阅的数据发生变化，新的数据处理为二进制值为：" + result);
    }
}

```
十六进制观察者
```java
/**
 * 十六进制观察者
 * @author zhaoyd
 * @date 2020-06-10
 */
public class HexaObserver extends AbstractObserver{

    HexaObserver(Subject subject){
        this.subject = subject;
        //注册到subject的观察者
        subject.addObserver(this);
    }

    @Override
    public void update() {
        String result = Integer.toHexString(subject.getState()).toUpperCase();
        System.out.println("订阅的数据发生变化，新的数据处理为十六值为：" + result);
    }
}
```
定义主题  
重点是修改state时，调用了notifyObserver方法通知所有观察者  
```java
/**
 * 定义一个主题，被观察者观察
 * @author zhaoyd
 * @date 2020-06-10
 */
public class Subject {
    private Vector<AbstractObserver> obs = new Vector<>();
    private int state;

    public int getState() {
        return state;
    }

    /**
     * state变化
     * @param state
     */
    public void setState(int state) {
        this.state = state;
        System.out.println("state变化，state="+state);
        //通知观察者
        this.notifyObserver();
    }

    /**
     * 添加一个观察者(注册观察者)
     * @param o
     */
    public synchronized void addObserver(AbstractObserver o) {
        if (o == null){
            throw new NullPointerException();
        }
        if (!obs.contains(o)) {
            obs.addElement(o);
        }
    }

    /**
     * 通知观察者
     */
    public void notifyObserver(){
        obs.forEach(observer -> {
            observer.update();
        });
    }
}
```
客户端调用  
```java
public static void main(String[] args) {
    Subject subject = new Subject();
    //二进制观察者
    BinaryObserver binaryObserver = new BinaryObserver(subject);
    //十六进制观察者
    HexaObserver hexaObserver = new HexaObserver(subject);
    //变化
    subject.setState(10);
}
```
## 责任链模式
理解：处理类形成一个链表（责任链），处理完成之后交给下一个节点处理  
```java
/**
 * 责任链模式
 * 处理类形成一个链表（责任链），处理完成之后交给下一个节点处理
 * @author zhaoyd
 * @date 2020-07-23
 */
public class Chain {
    //定义需要校验的目标
    public static class Context{
        boolean isNewUser = true;
        boolean isLocalUser = true;
    }
    //首先定义一个节点基类， 包含后续节点、当前节点的校验方法
    public static abstract class RuleHandler{
        protected RuleHandler nextHandler;
        public abstract void apply(Context context);
    }

    public static class NewUserRuleHandler extends RuleHandler{

        @Override
        public void apply(Context context) {
            if(context.isNewUser){
                System.out.println("新用户校验ok");
                if(this.nextHandler != null){
                    this.nextHandler.apply(context);
                }
            }else{
                throw new RuntimeException("非新用户");
            }
        }
    }
    public static class LocalRuleHandler extends RuleHandler{

        @Override
        public void apply(Context context) {
            if(context.isLocalUser){
                System.out.println("本地用户校验ok");
                if(this.nextHandler != null){
                    this.nextHandler.apply(context);
                }
            }else{
                throw new RuntimeException("非当地用户");
            }
        }
    }
    public static class LimitRuleHandler extends RuleHandler{
        int limit = 1;
        @Override
        public void apply(Context context) {
            if(limit-->0){
                System.out.println("活动名额校验ok");
                if(this.nextHandler != null){
                    this.nextHandler.apply(context);
                }
            }else{
                throw new RuntimeException("活动名额已满");
            }
        }
    }
}

```
客户端调用  
```java
public static void main(String[] args) {
    //构建责任链
    NewUserRuleHandler newUserRuleHandler = new NewUserRuleHandler();
    LocalRuleHandler localRuleHandler = new LocalRuleHandler();
    LimitRuleHandler limitRuleHandler = new LimitRuleHandler();
    newUserRuleHandler.nextHandler=localRuleHandler;
    localRuleHandler.nextHandler=limitRuleHandler;

    Context context = new Context();
    //使用责任链校验
    newUserRuleHandler.apply(context);
}
```
## 模板方法模式
抽象方法中定义模板方法，定义一组方法。具体实现逻辑是子类决定的 
```java
/**
 * 模板方法模式
 * 抽象方法中定义模板方法，定义一组方法。具体实现逻辑是子类决定的 
 * @author zhaoyd
 * @date 2020-07-23
 */
public class TempleteMethodPattern {
    //定义一个模板抽象类
    public static abstract class AbstractTemplete{
        public void templeteMethod(){
            //初始化方法
            init();
            //执行apply
            apply();
            //end方法，相当于回调函数
            end();
        }
        protected void init(){
            System.out.println("init");
        }

        protected abstract void apply();

        protected void end(){
            System.out.println("end");
        }
    }

    /**
     * 定义具体的模板类
     * 1、实现抽象的apply方法
     * 2、重写end方法
     */
    public static class MyTemplete extends AbstractTemplete{

        @Override
        protected void apply() {
            System.out.println("执行apply");
        }
        @Override
        protected void end(){
            System.out.println("end看做钩子方法，复写它");
        }
    }
}
```
客户端调用，很简单 
```java
public static void main(String[] args) {
    AbstractTemplete templete = new MyTemplete();
    templete.templeteMethod();
}
```

## 状态模式
状态模式与策略模式代码结构很相似，UML图一样。  
先看代码主要理解两者的区别于应用场景  
理解：Context的状态变化的逻辑，委托给State类来实现
```java
/**
 * 状态模式
 * 理解：定义State接口(状态)，Context类(事物)
 * 不关心Context会进行什么操作（进行什么操作由State决定），关心Context提供什么操作
 * @author zhaoyd
 * @date 2020-07-25
 */
public class StatePattern {

    //定义Context
    public static class Context{
        private State state;
        private String name;

        Context(String name){
            this.name = name;
        }

        public State getState() {
            return state;
        }

        public void setState(State state) {
            this.state = state;
        }

        public String getName() {
            return name;
        }

        public void setName(String name) {
            this.name = name;
        }
    }
    //定义状态接口
    public interface State{
        void doAction(Context context);

    }

    public static class DeductState implements State{

        @Override
        public void doAction(Context context) {
            System.out.println("商品卖出，准备扣减库存");
            context.setState(this);
            //TODO 扣减库存的具体操作
        }
        @Override
        public String toString(){
            return "扣减库存状态";
        }
    }
    public static class RevertState implements State{

        @Override
        public void doAction(Context context) {
            System.out.println("商品采购到货，准备补库存");
            context.setState(this);
            //TODO 补库存的具体操作
        }
        @Override
        public String toString(){
            return "补库存状态";
        }
    }
}
```
客户端操作  
```java
public static void main(String[] args) {
    Context context = new Context("SKU:FES123");
    //扣减库存
    State deductState = new DeductState();
    deductState.doAction(context);
    System.out.println(context.getState().toString());
    //补库存
    State revertState = new RevertState();
    revertState.doAction(context);
    System.out.println(context.getState().toString());
}
```

策略模式：Strategy类中定义了当前类型执行逻辑的封装，不同业务规则封装不同的Strategy类。 客户端使用时，必须理解要使用哪种策略，以及选择的策略带来的结果
状态模式：对象将状态发生变化时操作的动作，委托给State类。 如果对象要在运行时根据状态来改变行为，考虑用状态模式

## 行为模式总结
不同的行为模式适合不同的场景。总的来说，目标都是让代码更加解耦、易维护

# 设计模式总结
使用设计模式目的是为了让代码更易扩展、易维护、更优雅  
使用设计模式要权衡，很多情况下代码扩展性、可读性、效率并不能同时满足，靠个人取舍了。