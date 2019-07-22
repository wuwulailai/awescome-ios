iOS微信安装包瘦身

前提
微信经过多次版本迭代，产生不少冗余代码和无用资源。之前微信也没有很好的手段知道哪个模块增量多少。另外去年10月微信开始做ARC支持，目的是为了减少野指针带来的Crash，但代价是可执行文件增大20%左右。而苹果规定今年6月提交给Appstore的应用必须支持64位，32位和64位两个架构的存在使得可执行文件增加了一倍多。安装包大小优化迫在眉睫。

Appstore安装包是由资源和可执行文件两部分组成，安装包瘦身也是从这两部分进行。

资源瘦身
资源瘦身主要是去掉无用资源和压缩资源，资源包括图片、音视频文件、配置文件以及多语言wording。无用资源是指资源在工程文件里，但没有被代码引用。检查方法是，用资源关键字（通常是文件名，图片资源需要去掉@2x @3x），搜索代码，搜不到就是没有被引用。当然，有些资源在使用过程中是拼接而成的（如loading_xxx.png），需要手工过滤。

资源压缩主要对png进行无损压缩，用的是ImageOptim工具和compress命令（需要安装XQuartz-2.7.5.dm插件）。不建议对资源做有损压缩，有损压缩需要设计一个个检查，通常压缩后效果不尽人意。

Xcode's Link Map File
在讲可执行文件瘦身之前先介绍Xcode的LinkMap文件。LinkMap文件是Xcode产生可执行文件的同时生成的链接信息，用来描述可执行文件的构造成分，包括代码段（__TEXT）和数据段（__DATA）的分布情况。只要设置Project->Build Settings->Write Link Map File为YES，并设置Path to Link Map File，build完后就可以在设置的路径看到LinkMap文件了：



每个LinkMap由3个部分组成，以微信为例：

1. Object files:
[ 0] linker synthesized
[ 1] /xxxx/WCPayInfoItem.o
[ 2] /xxxx/GameCenterFriendRankCell.o
[ 3] /xxxx/WloginTlv_0x168.o
...

第一部分列举可执行文件里所有.obj文件，以及每个文件的编号。

2. Sections:


第二部分是可执行文件的段表，描述各个段在可执行文件中的偏移位置和大小。第一列是段的偏移量，第二列是段占用大小，Address(n)=Address(n-1)+Size(n-1)；第三列是段类型，代码段和数据段；第四列是段名字，如__text是可执行机器码，__cstring是字符串常量。有关段的概念可参考苹果官方文档《OS X ABI Mach-O File Format Reference》

3. Symbols:
# Address Size File Name
0x100005A50 0x00000074 [ 1] +[WCPayInfoItem initialize]
...
0x10231C120 0x00000018 [ 1] literal string: I16@?0@"WCPayInfoItem"8
...
0x10252A41A 0x0000000E [ 1] literal string: WCPayInfoItem
...

第三部分详细描述每个obj文件在每个段的分布情况，按第二部分Sections顺序展示。例如序号1的WCPayInfoItem.o文件，+[WCPayInfoItem initialize]方法在__TEXT.__text地址是0x100005A50，占用大小是116字节。根据序号累加每个obj文件在每个段的占用大小，从而计算出每个obj文件在可执行文件的占用大小，进而算出每个静态库、每个功能模块代码占用大小。这里要注意的地方是，由于__DATA.__bbs是代表未初始化的静态变量，Size表示应用运行时占用的堆大小，并不占用可执行文件，所以计算obj占用大小时，要排除这个段的Size。

可执行文件瘦身
回到我们的可执行文件瘦身问题，LinkMap文件可以帮助我们寻找优化点。

1. 查找无用selector
以往C++在链接时，没有被用到的类和方法是不会编进可执行文件里。但Objctive-C不同，由于它的动态性，它可以通过类名和方法名获取这个类和方法进行调用，所以编译器会把项目里所有OC源文件编进可执行文件里，哪怕该类和方法没有被使用到。

结合LinkMap文件的__TEXT.__text，通过正则表达式([+|-][.+\s(.+)])，我们可以提取当前可执行文件里所有objc类方法和实例方法（SelectorsAll）。再使用otool命令otool -v -s __DATA __objc_selrefs逆向__DATA.__objc_selrefs段，提取可执行文件里引用到的方法名（UsedSelectorsAll），我们可以大致分析出SelectorsAll里哪些方法是没有被引用的（SelectorsAll-UsedSelectorsAll）。注意，系统API的Protocol可能被列入无用方法名单里，如UITableViewDelegate的方法，我们只需要对这些Protocol里的方法加入白名单过滤即可。

另外第三方库的无用selector也可以这样扫出来的。

2. 查找无用oc类
查找无用oc类有两种方式，一种是类似于查找无用资源，通过搜索"[ClassName alloc/new"、"ClassName *"、"[ClassName class]"等关键字在代码里是否出现。另一种是通过otool命令逆向__DATA.__objc_classlist段和__DATA.__objc_classrefs段来获取当前所有oc类和被引用的oc类，两个集合相减就是无用oc类。

3. 扫描重复代码
可以利用第三方工具simian扫描。南非支付copy代码就是这样被发现的。但除此成果之外，扫描出来的结果过多，重构起来也不方便，不如砍功能需求效果好。

4. protobuf精简改造
protobuf是Google推出的一种轻量高效的结构化数据存储格式，在微信用于网络协议和本地文件序列化。但google默认工具生成的代码比较冗余，像序列化、反序列化、计算序列化大小等方法都生成在具体的pb类里，每个类的实现大同小异。通过代码分析以及结合protobuf原理，要想把这些方法抽象到基类，派生类提供每个字段相关信息就够了：

field number

field label, optional, required or repeated

wire type, double, float, int, etc

是否packed

repeated的数据类型

typedef struct {
    Byte _fieldNumber;
    Byte _fieldLabel;
    Byte _fieldType;
    BOOL _isPacked;
    int _enumInitValue;
    union {
        __unsafe_unretained NSString* _messageClassName;
        __unsafe_unretained Class _messageClass; // ClassName对应的Class
        IsEnumValidFunc _isEnumValidFunc; // 检测枚举值是否合法函数指针
    };
} PBFieldInfo;

另外通过无用selector列表，发现不少pb类属性的getter或setter没有被使用。原先的pb类属性是用@synthesize修饰，编译器会自动生成getter和setter。如果不想编译器生成，则要用@dynamic。甚至我们可以把pb类的成员变量去掉。做法如下：

基类增加id类型数组ivarValues（参考了objc_class结构体ivars做法），用于存放对象的属性值。对象属性值统一用oc对象表示，如果类型是基础类型（primitive，如int、float等），则用NSValue存

重载methodSignatureForSelector:方法，返回属性getter、setter的方法签名

重载forwardInvocation:方法，分析invocation.selector类型。如果是getter，从ivarValues获取属性值并设置为invocation的returnValue；如果是setter，从invocation第二个argument获取属性值，并存放到ivarValues里

重载setValue:forUndefinedKey:、valueForUndefinedKey:，防止通过KVO访问属性Crash

做下性能优化，如pb类在initialize做一次初始化，缓存属性名的hash值，属性的getter、setter方法的objcType等；属性值不用std::map（属性名->属性值），而是改用数组；MRC代替ARC（有些时候ARC自动添加的retain/release挺影响性能的）；等等

class PBClassInfo {
public:
    PBClassInfo(Class cls, PBFieldInfo* fieldInfo);
    ~PBClassInfo();

public:
    unsigned int _numberOfProperty;
    std::string* _propertyNames;
    size_t* _propertyNameHashes;
    std::string* _getterObjCTypes;
    std::string* _setterObjCTypes;

    PBFieldInfo* _fieldInfos;
};

@interface WXPBGeneratedMessage () {
    uint32_t _has_bits_[3]; // 最多96个属性，表示属性是否有赋值
    int32_t _serializedSize;
    PBClassInfo* _classInfo;
    id* _ivarValues;
}
- (NSMethodSignature*) methodSignatureForSelector:(SEL) aSelector;
- (void) forwardInvocation:(NSInvocation*) anInvocation;
- (void) setValue:(id) value forUndefinedKey:(NSString*) key;
- valueForUndefinedKey:(NSString*) key;
@end

把冗余代码去掉后，整个类清爽多了。像GameResourceReq只有3个属性的proto结构体，类方法代码行数由以前的127行变成现在的8行。protobuf精简改造中，精简类方法减少了可执行文件8.8M，去掉类成员变量和类属性改用@dynamic减少了2.5M。

message GameResourceReq {
    required BaseRequest BaseRequest = 1;
    required int32 PropsCount = 2;
    repeated uint32 PropsIdList = 3[packed=true];
}

// 老实现
@implementation GameResourceReq

@synthesize hasBaseRequest;
@synthesize baseRequest;
@synthesize hasPropsCount;
@synthesize propsCount;
@synthesize mutablePropsIdListList;
@dynamic propsIdList;

- (id) init {...}
- (void) SetBaseRequest:(BaseRequest*) value {...}
- (void) SetPropsCount:(int32_t) value {...}
- (NSArray*) propsIdListList {...}
- (NSMutableArray*)propsIdList {...}
- (void)setPropsIdList:(NSMutableArray*) values {...}
- (BOOL) isInitialized {...}
- (void) writeToCodedOutputStream:(PBCodedOutputStream*) output {...}
- (int32_t) serializedSize {...}
+ (GameResourceReq*) parseFromData:(NSData*) data {...}
- (GameResourceReq*) mergeFromCodedInputStream:(PBCodedInputStream*) input {...}
- (void) addPropsIdList:(uint32_t) value {...}
- (void) addPropsIdListFromArray:(NSArray*) values {...}
@end

// 新实现
@implementation GameResourceReq

PB_PROPERTY_TYPE baseRequest;
PB_PROPERTY_TYPE opType;
PB_PROPERTY_TYPE brandUserName;

+ (void) initialize {
  static PBFieldInfo _fieldInfoArray[] = {
    {1, FieldLabelRequired, FieldTypeMessage, NO, 0, ._messageClassName = STRING_FROM(BaseRequest)},
    {2, FieldLabelRequired, FieldTypeInt32, NO, 0, 0},
    {3, FieldLabelRepeated, FieldTypeUint32, NO, 0, 0},
  };
  initializePBClassInfo(self, _fieldInfoArray);
}
@end

5. 编译选项优化
Strip Link Product设成YES，WeChatWatch可执行文件减少0.3M

Make Strings Read-Only设为YES，也许是因为微信工程从低版本Xcode升级过来，这个编译选项之前一直为NO，设为YES后可执行文件减少了3M

去掉异常支持，Enable C++ Exceptions和Enable Objective-C Exceptions设为NO，并且Other C Flags添加-fno-exceptions，可执行文件减少了27M，其中__gcc_except_tab段减少了17.3M，__text减少了9.7M，效果特别明显。可以对某些文件单独支持异常，编译选项加上-fexceptions即可。但有个问题，假如ABC三个文件，AC文件支持了异常，B不支持，如果C抛了异常，在模拟器下A还是能捕获异常不至于Crash，但真机下捕获不了（有知道原因可以在下面留言：）。去掉异常后，Appstore后续几个版本Crash率没有明显上升。个人认为关键路径支持异常处理就好，像启动时NSCoder读取setting配置文件得要支持捕获异常，等等

6. 其他可探索途径
iOS8 Embed-Framework：提取WeChatWatch、ShareExtention和微信主工程的公共代码，可执行文件可以减少5M+，不过这特性需要最低版本iOS8才能用，iOS7设备启动会crash

iOS9 App Thinning：严格来说App Thinning不会让安装包变小，但用户安装应用时，苹果会根据用户的机型自动选择合适的资源和对应CPU架构的二进制执行文件（也就是说用户本地可执行文件不会同时存在armv7和arm64），安装后空间占用更小

7. 建立监控
通过对LinkMap文件的分析，可以得知每个模块可执行文件占用大小。再对比两个版本，就知道业务模块的增量大小。参考如下：

