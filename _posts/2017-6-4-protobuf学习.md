---
layout: post
title: Protobuf简单介绍
category: blog
description: just to read it
---

protobuf开源协议序列化框架介绍
==========

## Protobuf介绍

protobuf是google提供的一个开源序列化框架，类似于XML，JSON这样的数据表示语言，其最大的特点是基于二进制，因此比传统的XML表示高效短小得多。虽然是二进制数据格式，但并没有因此变得复杂，开发人员通过按照一定的语法定义结构化的消息格式，然后送给命令行工具，工具将自动生成相关的类，可以支持PHP、Java、c++、Python等语言环境。通过将这些类包含在项目中，可以很轻松的调用相关方法来完成业务消息的序列化与反序列化工作.

## Protobuf使用
*   首先需要一个原始的proto 文件,定义需要做串行化的数据结构信息。每个结构化信息是一小段包含一系列键值对的记录。
*	利用protobuf的编译器将原始proto文件生成相应的可执行代码.里面提供了大量的内联接口,可以对结构化信息里面的键值进行get ,set之类的操作.
*	在工程里面引用上述生成的可执行代码.
*	接收方和发送方只要约定这份原始的proto文件即可.如果有改动,重新编译生成代码使用.

## Protobuf语法 
在protobuf中，协议是由一系列的消息组成的。消息由至少一个字段组合而成，类似于C语言中的结构。每个字段都有一定的格式。
字段格式：限定修饰符① | 数据类型② | 字段名称③ | = | 字段编码值④ | [字段默认值⑤]

###	限定修饰符
包含 required\optional\repeated
 
  * Required	表示是一个必须字段，必须相对于发送方，在发送消息之前必须设置该字段的值，对于接收方，必须能够识别该字段的意思。发送之前没有设置required字段或者无法识别required字段都会引发编解码异常，导致消息被丢弃
  * Optional	表示是一个可选字段，对于发送方可选，在发送消息时，可以有选择性的设置或者不设置该字段的值。对于接收方，如果能够识别可选字段就进行相应的处理，如果无法识别，则忽略该字段，消息中的其它字段正常处理
  * Repeated	表示该字段可以包含0~N个元素。其特性和optional一样，但是每一次可以包含多个值。可以看作是在传递一个数组的值


    注:因为optional字段的特性，很多接口在升级版本中都把后来添加的字段都统一的设置为optional字段，这样老的版本无需升级程序也可以正常的与新的软件进行通信，只不过新的字段无法识别而已。这样可以满足新版本兼容的问题.   
    
    建议：项目投入运营以后涉及到版本升级时的新增消息字段全部使用optional或者repeated，尽量不实用required。如果使用了required，需要全网统一升级，如果使用optional或者repeated可以平滑升级。



### 数据类型


    |protobuf 数据类| 描述| 打包| C++语言映射|

    | -------- | -----:   | :----: | ------|

    |bool|布尔类型|1字节|bool|

    |double|	64位浮点数|	N|	double|
    |float	|32为浮点数	|N	|float|
    |int32	|32位整数  |N	|int
    |uin32	|无符号32位整数	|N	|unsigned int
    |int64	|64位整数	|N	|__int64    
    |uint64	|64为无符号整	|N	|unsigned __int64
    |sint32	|32位整数，处理负数效率更高	|N	|int32
    |sing64	|64位整数 处理负数效率更高	|N	|__int64
    |fixed32	|32位无符号整数	|4	|unsigned int32
    |fixed64	|64位无符号整数	|8	|unsigned __int64
    |sfixed32	|32位整数、能以更高的效率处理负数	|4	|unsigned int32
    |sfixed64	|64为整数	|8	|unsigned __int64
    |string	|只能处理 ASCII字符	|N	|std::string
    |bytes	|用于处理多字节的语言字符、如中文	|N	|std::string
    |enum	|可以包含一个用户自定义的枚举类型uint32	|N(uint32)	|enum
    |message	|可以包含一个用户自定义的消息类型	|N	|object of class


注:

*	N 表示打包的字节,并不是固定的。而是根据数据的大小或者长度, 例如int32，如果数值比较小，在0~127时，使用一个字节打包。

*	关于枚举的打包方式和uint32相同。

*	关于message，类似于C语言中的结构包含另外一个结构作为数据成员一样。

*	关于 fixed32 和int32的区别。fixed32的打包效率比int32的效率高，但是使用的空间一般比int32多。因此一个属于时间效率高，一个属于空间效率高。根据项目的实际情况，一般选择fixed32，如果遇到对传输数据量要求比较苛刻的环境，可以选择int32.

### 字段名称

字段名称的命名与C、C++、Java等语言的变量命名方式几乎是相同的。
protobuf建议字段的命名采用以下划线分割的驼峰式。例如 first_name 而不是firstName.

### 字段编码值
有了该值，通信双方才能互相识别对方的字段。当然相同的编码值，其限定修饰符和数据类型必须相同。编码值的取值范围为 1~2^32（4294967296）。其中 1~15的编码时间和空间效率都是最高的，编码值越大，其编码的时间和空间效率就越低（相对于1-15），当然一般情况下相邻的2个值编码效率的是相同的，除非2个值恰好是在4字节，12字节，20字节等的临界区。比如15和16.
1900~2000编码值为Google protobuf 系统内部保留值，建议不要在自己的项目中使用。
protobuf 还建议把经常要传递的值把其字段编码设置为1-15之间的值。
消息中的字段的编码值无需连续，只要是合法的，并且不能在同一个消息中有字段包含相同的编码值。 

### 默认
当在传递数据时，对于required数据类型，如果用户没有设置值，则使用默认值传递到对端。当接受数据是，对于optional字段，如果没有接收到optional字段，则设置为默认值。


## Protobuf关键字

* import

 protobuf 接口文件可以像C语言的h文件一个，分离为多个，在需要的时候通过 import导入需要对文件。其行为和C语言的#include或者java的import的行为大致相同。

*	package

 避免名称冲突，可以给每个文件指定一个package名称，对于java解析为java中的包。对于C++则解析为名称空间。 

*	message

 支持嵌套消息，消息可以包含另一个消息作为其字段。也可以在消息内定义一个新的消息。

*	enum

 枚举的定义和C++相同，但是有一些限制。枚举值必须大于等于0的整数。使用分号(;)分隔枚举变量而不是C++语言中的逗号(,)。


## 序列化和反序列化

在使用时,常用的几个接口如下:

    Bool SerializeToString(string* output) const;	序列化消息，将存储字节的以string方式输出。注意字节是二进制，而非文本；
    bool ParseFromString(const string& data);	    解析给定的string
    bool SerializeToOstream(ostream* output) const;	写消息给定的c++ ostream中
    bool ParseFromIstream(istream* input);	        从给定的c++ istream中解析出消息

## Protobuf例子
下面以项目中经常出现的结构体协议为例子,看看如何使用protobuf.

(1)	首先定义proto文件

```C
package base_structure;

message StripeInfo {
	enum ObjectType {
	NORMAL_OBJ = 1;
	STREAM_OBJ = 2;
	INDEX_OBJ = 3;
	PIC_OBJ = 4;
	ATTACH_OBJ = 5;
	APPEND_OBJ = 6;
	}

	required int32 ec_n = 1;
	required int32 ec_m = 2;
	required int32 ec_k = 3;
	optional string bucket_name = 4;
	optional string stripe_id = 5;
	optional int32 start_time = 6;
	optional int32 end_time = 7;
	optional ObjectType obj_type = 8;
	optional int32 unit_count = 9;
   
    message UnitInfo {
        required string unit_key = 1;
		optional string object_key = 2;
        optional int64 offset_in_obj = 3;
		optional int32 index_in_stripe = 4;
		optional string osd_ip_port = 5;
		optional string wwn = 6;
   }

   repeated UnitInfo unit_info = 10;
}

message ClusterParams {
    optional string time_server_ip = 1;
    optional int32 time_server_port = 2;
    optional int32 interval = 3;
}
```

(2)	生成代码
 
生成的文件如下:
 
(3)	使用代码测试
```C
#include <iostream>
#include "data_structure.pb.h"
#pragma comment(lib, "libprotobuf.lib")  
#pragma comment(lib, "libprotoc.lib")
int main(int argc, char *argv[])
{
	base_structure::StripeInfo stripe;
	int max_unit_count = 30;
	stripe.set_stripe_id("test_stripe");
	stripe.set_bucket_name("test_bucket");
	stripe.set_start_time(1000000);
	stripe.set_end_time(2000000);
	stripe.set_ec_n(4);
	stripe.set_ec_m(2);
	stripe.set_ec_k(1);
	stripe.set_obj_type(base_structure::StripeInfo_ObjectType_NORMAL_OBJ);
	stripe.set_unit_count(max_unit_count);  
	
	for (int i=0; i<max_unit_count; i++)
	{
		base_structure::StripeInfo_UnitInfo* unit_list = stripe.add_unit_info();
		unit_list->set_unit_key("abc");
		unit_list->set_object_key("key1");
        unit_list->set_offset_in_obj(100);
		unit_list->set_index_in_stripe(i);
		unit_list->set_osd_ip_port("127.0.0.1");
		unit_list->set_wwn("abcdefg1234567890");
	}

	std::cout << "unit key count:" << stripe.unit_info_size() << std::endl;
	std::cout << "stripe size:" << stripe.ByteSize() << std::endl; 
    system("pause");  
    return 0;  
}
```

这里我定义了一个条带信息,类型为24+6,包含了30个unit_key的信息.运行结果如下:
 
序列化以后,protobuf给出的大小为1454字节.而如果采用现在的结构体方式定义,其中object_key最大为1024字节,unit_key最大为1536字节,那么30个unit_key至少要占据:30*(1024+1536)=76800字节,大概为75K的空间.而且不管实际使用了多少长度的unit_key和object_key,占用的空间不会变,这在条带数量增多时,空间占用更加明显.虽然目前不会在网络传输中出现丢包或失败,但是也增加了网络带宽消耗.相比protobuf,只需要1K左右的数据即可完成一个条带的协议数据传输,效率将大大提升.

## 总结
Protobuf的优缺点总结一下:

* 优点:

 *	压缩效率高,且反序列化速度快,通常在ns级别.
 *	兼容性强.协议改动时,可以做到平滑升级
 *	使用方便,协议修改时,只需要修改proto文件,重新编译即可,对业务逻辑的改动比直接修改可能要少一些
 *	可靠性高

* 缺点:
 * 由于内部采用二进制进行传输,相比其他文本协议,可读性不强.
