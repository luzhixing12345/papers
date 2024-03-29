
# Hello bytes, bye blocks- PCIe storage meets compute express link for memory expansion (CXL-SSD)

## 摘要

- 三种CXL协议
- 讨论了最优的PCIe存储设备选择(Type3)
- 做对比实现讨论CXL带来了的性能提升
- 探索网络拓扑和管理内存扩展的方式

## 术语表

|名词|释义|
|:--:|:--:|
|BARs|PCIe base address registers|
|FlexBus|CXL.io创建的高速IO通道|
|CXL RP|CXL root port|
|HDM|host-managed device memory|
|CXL flit|主机设备通过CXL RP发送给type3设备用于同步的信息|
|USP|upstream ports|
|DSP|downstream ports|
|fabric manager|swicth's crossbar,交换机连接USP,DSP的部分|
|VH|virtual hierarchy|
|MLD|multiple logical device|
|GPF|global persistent flush|
|DT|deterministic|
|ND|non-deterministic|
|BF|bufferable|
|NB|non-bufferable|

## 为什么使用CXL扩展PCIe存储?

### 字节寻址能力的需要

一直以来都在探索对于PCIe存储如何像内存设备一样可以实现字节寻址, NVMe标准提供了通过向BARs暴露内部SSD内存/缓存以实现字节寻址的方式.既然BARs可以被直接映射到系统的内存空间,那么主机段的应用就可以向访问本地内存一样访问扩展的内存资源了,直接使用load/store指令而不是复制块,并且可以将内部的内存看作写回的cache用于减小SSD设备的高延迟

### PCIe协议的不可缓存的限制

尽管PCIe带宽很大,尽管存储设备有能力通过PCIe BARs处理load/store指令,但是PCIe规定将其看作CPU管理和通信的外围设备,被限制不能作为内存工作

作为内存映射的BARs是主机使下层设备了解控制指令的唯一接口,CPU必须保证load/store指令uncached & accessible, 这种不能被缓存的特性严重影响了内存访问目标的性能

但是如果CPU可以缓存这些内存请求,那么PCIe存储就不能捕获他们的到来,这会导致系统故障,存储失去连接等问题,因此X86指令集不允许PCIe相关内存请求在CPU处被缓存,这种性质促使存储集成内存扩展器被排除在传统内存层次结构之外,并禁止它们利用 CPU 缓存

### CXL的优势

CXL作为一种缓存一致性连接协议,设计之初就是为了支持不同加速器/内存设备,CXL可以维持若干内存地址空间在PCIe网络域中的一致性,使得不同的处理器和硬件加速器通过对应的协议持续访问. CXL协议可以覆盖PCIe的IO接口,使得不同CXL设备兼容PCIe

尽管CXL建立在PCIe基础之上,但所有缓存内容可以在同一个CXL层次保持一致性,这使得对于PCIe存储地址的请求可以被缓存,尽管CXL目前只支持DRAM或PMEM作为内存池,但是我们认为CXL可以改变PCIe存储的块接口,变成一个像内存一样的比特接口. 基于CXL协议可以融合不同的PCIe存储设备为一个缓存一致的内存空间,CXL可以创建一个比基于DRAM和基于PMEM的内存扩展大得多的内存池

## CXL的三种协议和三种设备类型

CXL共有三种协议

- CXL.io: CXL设备和主机通信的最基本的协议, CXL.io修补了PCIe层次中通信层的不足之处,并且创建了一个高速IO通道, FlexBus, 其将接收到的CXL数据转换成一个更易于PCIe物理层的格式
- CXL.cache:为FlexBus提供缓存一致性能力
- CXL.mem:为FlexBus提供内存访问能力

CXL协议覆盖了PCIe,CXL RP允许若干设备的内存地址映射到同一个主机地址的可缓存的内存空间中,这种设计的目的在于统一不同域内存设备,映射到单个的缓存一致的内存池,以便与内存扩展器针对使用不同的存储技术利用它们

CXL共有三种设备,它们是基于如何组合使用CXL协议来区分的

![20230109214757](https://raw.githubusercontent.com/learner-lu/picbed/master/20230109214757.png)

- Type1: 运算加速设备, CXL.io + CXL.cache

  Type1设备具有直接访问主机内存的能力,例如向量加速单元,DPU等. 十分珍贵

- Type2: 带内存的分离加速设备, CXL.io + CXL.cache + CXL.mem

  Type2设备自带内存,这些内存被称为HDM. 通过三种CXL协议,主机和HDM可以互相通信

  值得注意的是这种设备与带私有内存模块的传统加速设备(比如GPU)不同,尽管主机也可以访问GPU的内存(GDDR),但它仅支持内存拷贝.

  Type2类型的设备是指支持主机通过缓存一致的load/store指令管理HDM,并且也支持HDM充分利用CXL三种协议的特性访问主机CPU内存

- Type3: 没有加速设备,没有处理单元的HDM内存扩展, CXL.io + CXL.mem

  Type3设备用于扩展主机处的内存,Type3设备不具备向主机发送请求的功能,CPU只能通过CXL.mem完成对HDM的读写

  CXL还允许Type3设备使用CXL.io去适应多样灵活的IO指令

## 使用CXL整合PCIe存储设备

### 设备类型的选择

我们并不能将PCIe存储设备看作一个简单的,被动的设备.除了后端模块,实际上PCIe存储设备也通过管理内部DRAM的存储和缓存去处理请求和响应数据,它同时也具备处理多种数据处理任务的计算能力,例如地址转换.尽管Type2类型的设备看起来是作为存储集成的内存扩展器的不错选择,即可以充分利用HDM,又可以在被主机CPU感知的情况下将数据处理能力整合到存储中.但是我们认为将Type3类型的数据作为CXL存储集成的内存扩展器,有以下三个原因

1. 尽管Type2允许主机直接处理存储处的HDM,但是Type2类型设备是被设计服务于计算密集型应用(任务)的,因此每个CXL RP只允许一台设备连接到主机,这使得Type2设备并不像Type3设备那样易于扩展
2. Type2设备同时具备三种CXL协议,但是CXL.cache和CXL.mem会造成通信负担进而降低性能

   具体来说所有load/store请求都需要检查 PCIe 存储计算复合体的缓存状态,每次IO服务都会执行多种CXL事务.尽管这对PCIe存储高效的管理内部DRAM至关重要,但并不会同步的管理主机CPU处的缓存

3. 因为Type 2的CXL.cache同时管理着主机的本地内存和HDM,因此每次计算资源到达设备时都必须向主机请求许以实现内存/缓存一致,这使得设备级的I/O性能甚至比以前更差

> 不用Type1是因为太贵,而且type1本身要求必须直接访问主机的内存和cache

### 存储方面的修改

因为PCIe设备通常采用PCIe端点和NVMe控制器来解析传入的请求,并在主机和SSD的内部DRAM之间传输数据,因此想要支持type3在PCIe存储侧的硬件改动非常简单

例如我们可以利用现有的PCIe端点逻辑,组成一个CXL存储控制器来处理CXL事务包格式化和CXL.io控制.现有的现有的NVMe控制器的能力,如命令解析和页面内存拷贝等功能,也可以简化为实现CXL.mem的读写接口

NVMe规范允许PCIe存储在固件或硬件中实现其控制器. 然而我们认为最好是在硬件上实现CXL.mem的读写服务程序的自动化,而让固件负责管理内部DRAM和后端模块

### 系统整合

![20230110123706](https://raw.githubusercontent.com/learner-lu/picbed/master/20230110123706.png)

上图展示了PCIe设备如何连接到主机以及主机用于如何直接通过load/store指令访问存储设备

系统总线提供了一个CXL RP连接PCIe设备,并识别为type3.当主机启动时,它枚举连接到其RP的CXL设备,并通过将它们的内部内存空间映射系统内存来初始化这些设备上来初始化.具体来说,主机从PCIe存储设备中检索CXL BAR和HDM的大小然后将它们映射到系统内存空间(保留CXL RP).其中HDM会被映射到一个可缓存的内存地址空间,以便用于通过load/store直接访问.

因为CXL BAR和HDM都被映射到一个新地址了,所以CXL RP需要让底层CXL控制器指导它们被映射到哪里了,这种地址空间同步是通过向目标存储的CXL配置区写入相应的信息(例如重新映射的地址偏移量)来实现的

当应用使用系统内存读取/写入数据,CXL RP会通过CXL.mem发送一条信息(CXL flit)给CXL设备控制器,然后底层和CXL控制器解析flit并提取请求信息(例如指令和目的地址),接着控制器就可以和底层固件配合处理数据了.

## 性能测试

鉴于目前没有处理器支持CXL.mem CXL.io,作者团队制作了一个CXL存储集成的内存扩展器并进行了一系列仿真测试(具体参数见论文),结果如下所示

![20230110130921](https://raw.githubusercontent.com/learner-lu/picbed/master/20230110130921.png)

> 其中alpha表示局部性占比,0.001表示具有最高的局部性(最佳情况),1表示具有最差的局部性(最坏情况).横轴对比了PCIe,CXL,DRAM三类设备,纵轴为延迟(越低越好)

在最佳情况下说明了CXL要比基于PCIe的内存扩展性能好得多(129倍的优势),几乎所有的内存请求都击中了缓存,PCIe不能从主机处缓存处获得任何优势(注意上文提及了PCIe规定PCIe相关内存请求不能被CPU缓存),而且CXL充分利用缓存,其低延迟甚至比肩DRAM

在平均情况下CXL也有相较PCIe 3倍的延迟优势,然而在最坏情况下CXL表现不佳,因为benchmark的测试是完全随机,Z-NAND延迟不可避免,CXL只有1.6倍的优势,相较DRAM差距极大.

## 存储分解

本节讨论了一个系统如何将CXL控制器和存储设备从其计算资源中分离出来,同时保持其字节寻址能力

### 通过字节接口汇集存储

CXL2.0规定FlexBus可以接入若干个CXL switches, 每一个交换机可以有若干个USP和DSP,这使得互联的网络易于扩展.尽管现在CXL还没有确定switch内部的具体实现和内部组成,但USP和DSP可以通过一个可重构的横梁开关简单地互连起来

![20230110160711](https://raw.githubusercontent.com/learner-lu/picbed/master/20230110160711.png)

- USP可以通过FlexBus连接到CXL RP或另一个交换机的DSP,并且它在内部将传入的信息返回给一个或多个底层DSP
- DSP连接一个较低级别的硬件模块,如存储设备的CXL端点或下层的一个交换机的USP

上图展示了一种扩展方式,每一个DSP对应连接一台PCIe设备,一个USP连接所有DSP然后将其暴露给主机的CXL RP.对于这种类型的存储集成的内存扩展,主机应该将每个HDM映射到其物理内存的不同地方,但是每个交换机的通道数有限(64-128),每个存储设备需要16条通道,所以每个交换机总计只有4-8个端口可以用于设备扩展

![20230110161453](https://raw.githubusercontent.com/learner-lu/picbed/master/20230110161453.png)

所以我们可以设计出如上的网络结构,顶层的交换机负责作为主机和低一层交换机的通信桥梁,最下层的交换机负责连接PCIe设备,这种网络能处理的设备数量取决于设备的大小和CXL可以处理的内存容量(目前是4PB,PB=1024TB)

### 多主机连接管理

为了更好的利用存储资源,我们也可以将任意数量的主机设备连接到CXL网络,鉴于fabric manager可以记录每一个USP和DSP的连接,我们可以为每一个主机到每一个设备构造一条独一无二的路由路径,被称为VH.每一个VH保证在CXL网络的任何一个位置,每一个存储设备都可以对应到一台主机

VHs允许系统将许多PCIe存储设备从多主机计算资源中完全分离出来,用于内存扩展

![20230110162606](https://raw.githubusercontent.com/learner-lu/picbed/master/20230110162606.png)

虽然这些可重新配置的VHs可以实现完全的扩展架构,但存储设备扩展的内存资源在精细控制方面是很棘手的,因为存储设备只应与一台主机相关联,但它可能在不同的CPU上利用不足或利用不平衡

### 存储设备虚拟化

为了解决上述问题,我们可以将每个存储设备虚拟化,由不同的主机共享.CXL允许一个系统在逻辑上将每个端点分割成多个type3设备(最多16个),每一个虚拟逻辑设备称为MLD

![20230110224711](https://raw.githubusercontent.com/learner-lu/picbed/master/20230110224711.png)

我们可以为每个MLD定义它自己的HDM,它们可以被映射到其他主机的内存中,类似一个物理存储设备.由于与同一存储设备相关联的每个MLD可以是不同VH的一部分,它有望通过利用基础存储资源以细粒度的方式分配内存扩展器

多主机的VH的**缺点是带宽共享和通信阻塞**,为了支持MLD,PCIe存储可能需要对底层后端和内部DRAM进行分区,降低了平行度,导致降低每个MLD的带宽.而且由于单个存储设备可以被多个主机共享,端点的处理设备可能会比往常更加拥堵,这需要更为精细的网络设计和存储设计

## 存储控制的扩展

由于CXL的Type 3是为内存池设计的,而不是为块存储设计的,我们需要考虑以下两个问题:**延迟波动和数据持久化**

- CXL.mem和CXL.io并没有严格规定load/store指令的响应时间因为CXL的内存请求可以以异步的方式提供服务,我们在性能预测中假定PCIe存储设备没有内部任务,然而内部任务的延迟根据固件的操作方式而变化.固件的操作方式不同,都会影响响应速度.
- 如果主机处的一些库(例如PMDK)已经做了数据持久化,目前CXL的冲洗机制可能不足以处理底层PCIe存储,为此CXL提供了GPF寄存器用于强制将所有CXL网络和SSD内部DRAM中的持久数据立即写回后端的方式,这也会使得存储集成的内存扩展器服务延迟

为了解决以上问题,作者提出添加两个特征: **确定性和缓存性**, 用于为CXL信息添加注释并将主机语义提示给底层的CXL控制器.这种方案基于CXL协议允许type3设备利用CXL.io处理多种IO指令,并且CXL.mem中含有可以被用于添加注释信息的保留字段

### 延迟和持久性控制

确定性有两种:

- DT: 主机希望第三类设备无内部任务参与的为其提供服务标记的请求
- ND: 主机发送的请求发出之后无需干预,交由第三类设备自行处理

如果指定了DT,目标存储可以安排一个或多个内部任务来操作后续的由ND注释请求,或在空闲时操作

缓存性有两种:

- BF: 对应的请求可以被缓存到SSD的内部DRAM中
- NB: 将持久性视为其服务的第一等公民

### 使用场景

确定性和缓存性的DT/ND + BF/NB在不同的场景下可以使用不同的组合

1. 数据库和事务性存储器

   它们在事务开始的时候记录数据,但在事务提交之前日志不需要被持久化存储,主机可以使用 ND + BF/NB 的组合, 存储设备可以利用这段时间执行内部任务,缓冲所有进入的写操作

   当事务被提交之后,主机匹配GPF刷新掉所有缓存数据并且将提交信息通过 DT+NB写入

2. load指令

   大多数指令都在等待其操作数的到达,DT对于load指令很有帮助.但如果并没有后续的指令需要利用前一条指令的结果,那么load的操作就不需要被同步

   我们可以精确的使用 DT/ND + BF, 将预取的数据暂时存放到存储内部的DRAM中

   例如数据与循环代码段的空间/时间有关系(矩阵运算),我们可以让存储设备知道,这些数据迟早会再次被内部DRAM击中

3. 锁和同步管理

   它们的机制大多情况下不需要持久性,延迟才是关键.例如自旋锁使用例如compare-and-swap的原子指令,是读写指令的组合.自旋锁的参数并不会在CPU缓存,相应的原子指令不断迭代以访问相同的内存地址

   这种情况下使用 DT + BF 就会好很多