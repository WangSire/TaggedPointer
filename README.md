# TaggedPointer

内存优化之TaggedPointer

###什么是Tagged Pointer？
在2013年9月，苹果推出了[iPhone5s](https://link.jianshu.com?t=http://en.wikipedia.org/wiki/IPhone_5S)，与此同时，iPhone5s配备了首个采用64位架构的[A7双核处理器](https://link.jianshu.com?t=http://en.wikipedia.org/wiki/Apple_A7)，为了节省内存和提高执行效率，苹果提出了Tagged Pointer的概念。对于64位程序，引入Tagged Pointer后，相关逻辑能减少一半的内存占用！

####那么Tagged Pointer是如何节省内存的呢？
- 我们先看看原有的对象为什么会浪费内存。假设我们要存储一个 NSNumber 对象，其值是一个整数。正常情况下，如果这个整数只是一个 NSInteger 的普通变量，那么它所占用的内存是与 CPU 的位数有关，在 32 位 CPU 下占 4 个字节，在 64 位 CPU 下是占 8 个字节的。而指针类型的大小通常也是与 CPU 位数相关，一个指针所占用的内存在 32 位 CPU 下为 4 个字节，在 64 位 CPU 下也是 8 个字节。
所以一个普通的 iOS 程序，如果没有Tagged Pointer对象，从 32 位机器迁移到 64 位机器中后，虽然逻辑没有任何变化，但这种 NSNumber、NSDate 一类的对象所占用的内存会翻倍。如下图所示：
![未使用Tagged Pointer内存图.png](https://upload-images.jianshu.io/upload_images/4053175-6de9061e5b74d97f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
从上图中可以看到内存的多余的浪费，以及查找该值的繁琐逻辑！我们再来看看效率上的问题，为了存储和访问一个 NSNumber 对象，我们需要在堆上为其分配内存，另外还要维护它的引用计数，管理它的生命期。这些都给程序增加了额外的逻辑，造成运行效率以及内存的损失。

- 为了改进上面提到的内存占用和效率问题，所以苹果提出了Tagged Pointer对象。对于某些占用内存很小的数据实例，不再单独开辟空间去存储，而是将实际的实例值存储在对象的指针中，同时对该指针进行标记，用于区分正常的指针指向！由于NSNumber、NSDate类的变量本身的值需要占用的内存大小常常不需要8个字节，拿整数来说，4个字节所能表示的有符号整数就可以达到20多亿（注：2^31=2147483648，另外1位作为符号位)，对于绝大多数情况都是可以处理的。所以苹果将一个对象的指针拆成两部分，一部分直接保存数据，另一部分作为特殊标记，表示这是一个特别的指针，不指向任何一个地址。所以，引入了Tagged Pointer对象之后，64位CPU下NSNumber的内存图变成了以下这样：
![64位使用Tagged Pointer内存图.png](https://upload-images.jianshu.io/upload_images/4053175-1fbc3fd88c1c8a95.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#####下面进入大家最喜欢的环节，没错，就是验证！

1.我先来看一下Tagged Pointer的初始化过程
![createTaggedPointer@2x.png](https://upload-images.jianshu.io/upload_images/4053175-a8647df855f86ce1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

2.在上述的代码中我们可以看到对真实的结果进行了一次encode操作，接下来我们再来看看_objc_encodeTaggedPointer函数里做了什么操作

![encodeTaggedPointer@2x.png](https://upload-images.jianshu.io/upload_images/4053175-0add73b4bbeaa996.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
将实际结果与objc_debug_taggedpointer_obfuscator进行异或操作！

3. 那么objc_debug_taggedpointer_obfuscator是什么呢？我们再来看看
![objc_debug_taggedpointer_obfuscator@2x.png](https://upload-images.jianshu.io/upload_images/4053175-46abb6e05137392a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

    可以看到在iOS 12系统版本之前objc_debug_taggedpointer_obfuscator是0，之后是一个无符号长整型的随机数！那么为什么要改为随机数呢？是为了一个简便的加密！大家可以试想一下，用一个值去^一个0，那么结果是多少，可以自己尝试一下！（可以往密码加盐这方面理解）

4. 我们所看见的的地址，实际上是被编码过的，如果想要看到真实的结果则需要自己手动去解码！
![objc_decodeTaggedPointer@2x.png](https://upload-images.jianshu.io/upload_images/4053175-8e4fde0e406d25cf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
通过上述代码可以得知，就是把已编码后的地址再次进行异或得到原来的值！


**我们来一起看下这特殊地址的庐山真面目，是不是还有点小激动 😁**
![声明](https://upload-images.jianshu.io/upload_images/4053175-408abad57f9af701.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![log](https://upload-images.jianshu.io/upload_images/4053175-9e2f216c509dd915.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



通过log我们可以看出，0xa以及0xb实际为特殊标识位，以0xb开头为例，将其转换为二进制就是1011， 首位1表示这是一个tagged Pointer，而011转换为十进制是3，参考下图中tagged Pointer的类型枚举，可以看出这是一个NSNumber类型！
![tagged Pointer类型枚举](https://upload-images.jianshu.io/upload_images/4053175-711297369f9abadd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

而末尾的数字仿佛在代表着类型标识符！经过实践可以总结为：0表示char类型，1表示short类型，2表示整形，3表示长整型，4表示单精度类型，5表示双精度类型！而中间这一部分才是真正的值，可以看到A为char类型，返回的是ASCII码（该值为16进制的，需要进行十进制转换）
需要注意的是：TaggedPointerString类型的指针与基本类型的指针是不一样的，末尾的数字为字符串的长度！
![ASCII对照表.png](https://upload-images.jianshu.io/upload_images/4053175-c68c22beb0628950.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**声明： 以上只是一些初探，系统是如何区分以及划分的需要大家继续去探索，例如：Tagged Pointer最大可以存储多大值**


#####总结
1.TaggedPointer：并不是一个类，它是适用于 64位处理器的一个内存优化机制，专门用来存储小对象，当存储不下时，则转为对象！例如 NSString、NSNumber 和 NSDate等对象进行优化。
2.指针不再是地址了，而是经过标识过的的值。它不再是一个对象了，只是普通变量而已。所以，它的内存并不存储在堆中，也不需要malloc和free。
3.在内存读取上有着3倍的效率，创建时比以前快106倍。

**注意**：Tagged Pointer并不是真正的对象，而是一个伪对象，对象都有 isa指针，而Tagged Pointer是没有的，因为它不是真正的对象，不能直接访问Tagged Pointer的isa！

