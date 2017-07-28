# Objective-C 小记

## 1.内存管理语义

属性用于封装数据，而数据则要有“具体的所有权语义”。下面这一组特质仅会影响“设置方法”。例如，用“设置方法”设定一个新值时，它是应该“保留（retain）”此值呢，还是只将其赋给底层实例变量就好？编译器在合成存取方法时，要根据此特质来决定所生成的代码。

* **assign** 只会执行针对“纯量类型”（例如,CGFloat、NSInteger等）的简单赋值操作
* **strong** 此特质表明该属性定义了一种“拥有关系”。为这种属性设置新值时，设置方法会先保留新值，并释放旧值，然后再将新值设置上去。
* **weak** “非拥有关系”，为这种属性设置新值时，既不保留新值，也不释放旧值。此特质同“assign”类似，然而在属性所指的对象遭到摧毁时，属性值也会清空。
* **unsafe_unretained** 此特质的语义和assgin相同，但它适用于“对象类型”，该特质表达一种“非拥有关系”（不保留），当目标对象遭到摧毁时，属性值不会自动清空，这一点与weak有区别。
* **copy** 此特质所表达的所属关系与strong类似。然而设置方法并不保留新值，而是将其“拷贝”(copy)。

## 2.理解objc_msgSend的作用

Objective-C是C的超集，所以最好先理解C语言的函数调用方式。C语言使用“静态绑定”，也就是说，在编译期就能决定运行时所应调用的函数。例如：

```
#import<stdo.h>

void printHello() {
	printf("Hello, world!\n");
}

void printGoodbye() {
	printf("Goodbye, world!\n");
}

void doTheThing(int type) {
	if(type == 0){
		printHello();
	} else {
		printGoodbye();
	}
	return 0;
}

```
如果不考虑“内联”，那么编译器在编译代码的时候已经知道程序中有printHello与printGoodbye这两个函数了，于是会直接生成调用这些函数的指令。而函数地址实际上是硬编码在指令中的。若是将刚才的代码写成下面这样，会如何呢？

```
#import<stdo.h>

void printHello() {
	printf("Hello, world!\n");
}

void printGoodbye() {
	printf("Goodbye, world!\n");
}

void doTheThing(int type) {
	void (*fnc)();
	if(type == 0){
		fnc = printHello;
	} else {
		fnc = printGoodbye;
	}
	fnc();
	return 0;
}

```
这时就得使用“动态绑定”了，因为所要调用的函数直到运行期才能确定。编译器在这种情况下，只有一个函数调用指令，不过待调用的函数地址无法硬编码在指令中，而是要在运行期读取出来。

在Objective-C中，如果向某对象传递消息，就会使用动态绑定机制来决定需要调用的方法。在底层，所有方法都是普通的C语言函数，然而对象收到消息后，究竟该调用哪个方法则完全于运行期决定，甚至可以在程序运行时改变。

给对象发送消息可以这样来写：

```
id returnValue = [someObject messageName:parameter];
```
在本例中，**someObject叫做“接受者”(receiver)，messageName叫做“选择子”(selector)。选择子与参数合起来称为“消息”(message)**。编译器看到此消息后，将其转换为一条标准的C语言函数调用，所调用的函数乃是消息传递机制中的核心函数，叫做objc_msgSend，其“原型”(protoype)如下：

```
void objc_msgSend(id self, SEL cmd, ...)
```
这是个参数个数可变的函数，能接受两个或两个以上的参数。第一个参数代表接收者，第二个参数代表选择子，后续参数就是消息中的那些参数，其顺序不变。选择子就是方法的名字。

编译器会把刚才那个例子中的消息转换为如下函数：

```
id returnValue = objc_msgSend(someObject, @selector(messageName:), parameter);
```
**objc_msgSend***函数会依据接收者与选择子的类型来调用适当的方法。为了完成此操作，该方法需要在接收者所属的类中搜寻其“方法列表”，如果能找到与选择子相符的方法，就跳至其实现代码。若是找不到，那就沿着继承体系继续向上查找，等找到合适的方法之后再跳转。如果最终还是找不到相符的方法，那就执行“消息转发”操作*。

**objc_msgSend**等函数一旦找到应该调用的方法实现之后，就会“跳转过去”。之所以能这样做，是因为Objective-C对象的每个方法都可以视为简单的C函数，其原型如下：

```
<return_type> Class_selector(id self, SEL cmd, ...)
```
真正的函数名和上面写的可能不太一样，笔者用“类”和“选择子”来命名是想解释其工作原理。每个类里都有一张表格，其中的指针都会指向这种函数，而选择子的名称则是查表时所用的“键”。**objc_msgSend**等函数正是通过这张表格来寻找应该执行的方法并跳至其实现的。原型的样子和**objc_msgSend**函数很像，这不是巧合，而是为了利用“尾调用优化”（又称“尾递归优化”）技术，令“跳至方法实现”这一操场变得更简单些。

如果某函数的最后一项操作是调用另外一个函数，那么就可以运用”尾调用优化“技术。编译器会生成转至另一个函数所需的指令码，而且不会向调用堆栈推入新的”栈帧“。只有当某函数的最后一个操作仅仅是调用其他函数而不会将其返回值另作他用时，才能执行“尾调用优化”。

## 3.理解消息转发机制

消息转发分为两大阶段。第一阶段先征询接收者，所属的类，看其是否能动态添加方法，以处理当前这个“未知的选择子”，这叫做“动态方法解析”。第二阶段涉及“完整的消息转发机制”。

### 动态方法解析

对象在收到无法解读的消息后，首先将调用其所属类的下列类方法：

```
+ (BOOL)resolveInstanceMethod:(SEL)selector
```
该方法的参数就是那个未知的选择子，其返回值为Boolean类型，表示这个类是否能新增一个实例方法用以处理此选择子。假如尚未实现的方法不是实例方法而是类方法，那么运行期系统就会调用另外一个方法，该方法与“resolveInstanceMethod:”类似，叫做“resolveClassMethod:”。

使用该方法的前提是：**相关方法的实现代码已经写好，只等着运行的时候动态插在类里面就可以了。此方案常用来实现@dynamic属性。**

### 备援接收者

当前接收者还有第二次机会能处理未知的选择子，在这一步中，运行期系统会问它：能不能把这条消息转给其他接收者来处理。其对应的处理方法如下：

```
- (id)forwardingTargetForSelector:(SEL)selector
```
若当前接收者能找到备援对象，则将其返回，否则就返回nil。在一个对象内部，可能还有一系列其他对象，该对象可经由此方法将能够处理某选择子的相关内部对象返回，这样的话，在外界看来，好像是该对象亲自处理了这些消息似的。

特别注意，我们无法操作经由这一步所转发的消息。若是想在发送给备援接收者之前先修改消息内容，那就得通过完整的消息转发机制来做了。

### 完整的消息转发

如果转发算法已经来到这一步的话，那么唯一能做的就是启用完整的消息转发机制了。首先创建NSInvocation对象，把与尚未处理的那条消息有关的全部细节都封住其中。此对象包含选择子、目标及参数。

在触发NSInvocation对象时，“消息派发系统”将亲自出马，把消息指派给目标对象。此步骤会调用下来方法来转发消息：

```
- (void)forwardingInvocation:(NSInvocation*)invocation
```
这个方法可以实现的很简单：只需改变调用目标，使消息在新目标上得以调用即可。然而这样实现出来的方法与“备援接收者”方案所实现放的方法等校，所以很少有人采用这么简单的实现方式。

### 消息转发全流程

![NSInvocation invocation](https://raw.githubusercontent.com/lxbboy326/Deep-learning-Objective-C/8f8faaa1f0b666ebf4b531e15145f61f15c35f8a/resource/invocation.png)

步骤越往后，处理消息的代价就越大。最好能在第一步就处理完，这样的话，运行期系统就可以将此方法缓存起来了。如果这个类的实例稍后还收到同名选择子，那么根本无须启动消息转发流程。若想在第三步把消息转给备援的接收者，那还不如把转发操作提前到第二步。因为第三步之时修改了调用目标，这项改动放在第二步会更为简单，不然后的话，还得闯将并处理完整的NSInvocation。

### 示例演示动态方法解析

本例编写一个类似于“字典”的对象，它里面可以容纳其他对象，只不过开发者要直接通过属性来存取其中的数据。这个类的设计思路是：由开发者来添加属性定义，并将其声明为@dynamic，而类则会自动处理相关属性值的存放与获取操作。
类的接口可以写成：

```
#import <Foundation/Foundation.h>

@interface LXBAutoDictionary : NSObject

@property(nonatomic, strong) NSString   *string;
@property(nonatomic, strong) NSNumber   *number;
@property(nonatomic, strong) NSDate *date;
@property(nonatomic, strong) id opaqueObject;

@end
```
在类的内部，每个属性的值还是会存放在字典里，所以我们先在类中编写如下代码，并将属性声明为**@dynamic**，这样的话，编译器就不会为其自动生成实例变量及存取方法了：

```
#import "LXBAutoDictionary.h"
#import <objc/runtime.h>

@interface LXBAutoDictionary()
@property(nonatomic, strong) NSMutableDictionary *backingStore;
@end

@implementation LXBAutoDictionary

@dynamic string, number, date, opaqueObject;

- (id)init {
    if (self = [super init]) {
        _backingStore = [NSMutableDictionary new];
    }
    return self;
}

+ (BOOL)resolveInstanceMethod:(SEL)sel {
    NSString *selectorString = NSStringFromSelector(sel);
    if ([selectorString hasPrefix:@"set"]) {
        class_addMethod(self, sel, (IMP)autoDictionarySetter, "v@:@");
    } else {
        class_addMethod(self, sel, (IMP)autoDictionaryGetter, "@@:");
    }
    return YES;
}

id autoDictionaryGetter(id self, SEL _cmd) {
    LXBAutoDictionary *typedSelf = (LXBAutoDictionary*)self;
    NSMutableDictionary *backingStore = typedSelf.backingStore;
    
    NSString *key = NSStringFromSelector(_cmd);
    return [backingStore objectForKey:key];
}

void autoDictionarySetter(id self, SEL _cmd, id value) {
    LXBAutoDictionary *typedSelf = (LXBAutoDictionary*)self;
    NSMutableDictionary *backingStore = typedSelf.backingStore;
    
    NSString *selectorString = NSStringFromSelector(_cmd);
    NSMutableString *key = [selectorString mutableCopy];
    
    //remove the ':' at the end
    [key deleteCharactersInRange:NSMakeRange(key.length - 1, 1)];
    
    //remove the 'set' prefix
    [key deleteCharactersInRange:NSMakeRange(0, 3)];
    
    //lowercase the first character
    NSString *lowercaseFirstChar = [[key substringToIndex:1] lowercaseString];
    [key replaceCharactersInRange:NSMakeRange(0, 1) withString:lowercaseFirstChar];
    
    if (value) {
        [backingStore setObject:value forKey:key];
    } else {
        [backingStore removeObjectForKey:key];
    }
}

@end

```
LXBAutoDictionary的用法很简单：

```
    LXBAutoDictionary *dict = [LXBAutoDictionary new];
    dict.date = [NSDate dateWithTimeIntervalSince1970:475372800];
    dict.string = @"Hello Objective-C";
    dict.number = @100;

    NSLog(@"dict.date = %@",dict.date);
    NSLog(@"dict.string = %@",dict.string);
    NSLog(@"dict.number = %@",dict.number);
```
输出结果：

```
dict.date = 1985-01-24 00:00:00 +0000 
dict.string = Hello Objective-C 
dict.number = 100
```
