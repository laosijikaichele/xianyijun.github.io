---

title: Java虚拟机之Class文件结构
date: 2016-06-08 10:57:49
tags: JVM

---


# **Java虚拟机之Class文件结构** #
在Java虚拟机规范中，class文件是一个重要接口。只要语言的源文件能编译为正确的class文件，就可以在Java虚拟机上正确的运行。
Class文件统一使用无符号整数作为基本类型，由u1、u2、u4、u8来表示无符号单字节、２字节、4字节、８字节整数。如果是数组的话，
则使用u1数组来表示。
根据Java虚拟机规范，一个class文件可以描述为:

```java
ClassFile{
	u4		magic;
    u2		minor_version;
    u2		major_version;
    u2		constant_pool_count;
    cp_info		constant_pool[constant_pool_count-1];
    u2		access_flag;
    u2		this_class;
    u2		super_class;
    u2		interfaces_count;
    u2		interfaces[interfaces_count];
    u2		fields_count;
    field_info		fields[fidlds_count];
    u2		method_count;
    method_info		method[method_count];
    u2		attributes_count;
    attribute_info		attributes[attributes_count];
}
```

Class文件以一个４字节的魔数开头，紧跟着两个大小版本号，然后就是常量池，常量池的个数为constant_pool_count，而常量池的项只有constant_pool_size-1个；常量池之后就是类的访问修饰符、类的自身引用、父类引用和接口个数和实现的接口引用;然后就是类的字段个数、字段描述和方法个数、方法描述；最后存放着类文件的属性信息。

## **魔数** ##
魔数是Class文件的标志，它是用来告诉虚拟机这个文件是Class文件，魔数是一个无符号的4字节整数，固定是0xCAFEBABE，
如果class文件不是以0xCAFEBABE开头的话，虚拟机在进行文件验证的时候会抛出ClassFormatErroe错误:
Incompatiable magic value 184466110 in class file
## **Class文件版本** ##
在Class文件魔数之后跟着两个2字节整数：大版本号和小版本号，表明Class文件是由哪个版本的编译器编译器产生的。
首先出现的是小版本号，是一个两字节的无符号整数，之后为大版本号，也是两个字节表示。
高版本的Java虚拟机可以执行由低版本编译生成的class文件，但是低版本的虚拟机不能执行高版本编译生成的class文件，会抛出UnsupportedClassValueVersionError错误：Unsupported major.minor version 。
## **常量池** ##
常量池是Class文件的核心内容，它对Class文件的字段和方法解析有着重要的作用。
常量池的表项的类型如下，不同类型的表项结构也不一样。
数字为类型的tag，表示该常量的整数枚举值。

- CONSTANT_Class 		7
- CONSTANT_Method		10
- CONSTANT_String		8
- CONSTANT_Float		4
- CONSTANT_Double		6
- CONSTANT_Utf8			1
- CONSTANT_MethodType		16
- CONSTANT_Fieldref		9
- CONSTANT_InterfaceMethodRef		11
- CONSTANT_Integer		3
- CONSTANT_Long		5
- CONSTANT_NameAndType		12
- CONSTANT_MethodHandle		15
- CONSTANT_InvokeDynamic		18

### **CONTSTANT_UTF8** ###

```c
CONSTANT_Utf8_info{
	u1	tag;
    u2	length;
    u1	bytes[length];
}
```

CONSTANT_Utf8_info表示Utf8的字符串，utf8的tag值为１，然后length表示字符串的长度,最后的是字符串的内容。
CONSTANT_UTF8字符串经常被其他常量引用。

### **CONSTANT_Class** ###

```java
CONSTANT_Class_info{
	u1	tag;
    u2	name_index;
}
```
CONSTANT_Class_info的tag为７，表示一个CONSTANT_Class常量，它的name_index是一个两字节的整数，他表示常量池的索引，指向的是一个Constant_Utf8的字符串。
### **CONSTANT_Integer、CONSTANT_Long、CONSTANT_Double、CONSTANT_Float** ###
当我们使用final去定义一个数字常量的时候，Class文件就会生成一个数字的常量。
```java
CONSTANT_Integer_info{
	u1	tag;
    u4	bytes;
}

CONSTANT_Float_info{
	u1	tag;
    u4	bytes;
}
CONSTANT_Double_info{
	u1	tag;
    u4	high_bytes;
    u4	low_bytes
}
CONSTANT_Long_info{
	u1	tag;
    u4	high_bytes;
    u4	low_bytes
}
```
### **CONSTANT_String**　###
CONSTANT_String是一个字符串常量。
```java
CONSTANT_String_info{
	u1 tag;
    u2 String_index;
}
```
CONSTANT_String_info的tag是８，string_index无符号整数指向常量池的索引，表示该字符串对应的UTF8内容。
### **CONSTANT_NameAndType** ###
```java
CONSTANT_NameAndType_info{
	u1 tag;
    u2 name_index;
    u2 descriptor_index;
}
```
CONSTANT_NameAndType_info的tag是12,name_index表示名称，值为常量池的索引，用来表示字段或者方法的名称，descriptor_index表示方法的签名或者字段的类型。
CONSTANT_NameAndType的discriptor_index使用特殊的字符串来表示类型.

|字符串|类型|字符串|类型|
|:-|:-|:-|:-|:-|
|B|byte|C|char|
|D|double|F|float|
|I|int|J|long|
|S|short|Z|boolean|
|V|void|L|表示对象|
|[|数组|||

对于对象类型来说，以L开头，接着类的全限定名，用分号结尾。数组的话就以[作为标记。
### **CONSTANT_Methodref和CONSTANT_Fieldref** ###
```java
CONSTANT_Methodref_info{
	u1 tag;
    u2 class_index;
    u2 name_and_type_index;
}
```
```java
CONSTANT_Fieldref_info{
	u1 tag;
    u2 class_index;
    u2 name_and_type_index;
}
```
CONSTANT_Methodref_info的tag为10,CONSTANT_Fieldref_info的tag为９,表明它们是一个CONSTANT_Methodref或者CONSTANT_Fieldref_info,它的class_index是常量池的索引，它指向一个CONSTANT_Class;name_and_type_index也是指向常量池的索引，它表示的是一个CONSTANT_NameAndType结构，定义一个类的方法签名或者字段的类型和名字。

### **CONSTANT_InterfaceMethodref** ###
```java
CONSTANT_InterfaceMethodref_info{
	u1 tag;
    u2	class_index;
    u2	name_and_type_index;
}
```
CONSTANT_InterfaceMethodref表示一个接口的方法,如果Java程序中出现了对接口方法的调用，就会在常量池中生成一个对接口方法的引用。
### **CONSTANT_MethodHandle、CONSTANT_MethodType、CONSTANT_InvokeDynamic** ###
CONSTANT_MethodHandle、CONSTANT_MethodType、CONSTANT_InvokeDynamic表示函数句柄、函数类型签名和动态调用。
```java
CONSTANT_MethodType_info{
	u1 tag;
    u2 descriptor_index;
}
```
CONSTANT_MethodType_info的tag为16,descriptor_index指向常量池中UTF8字符串的索引，主要是用来描述一个方法签名。
```java
CONSTANT_MethodHandle_info{
	u1 tag;
    u2 reference_kind;
    u2 reference_index;
}
```
CONSTANT_MethodHandle_info的tag为15,reference_kind表示方法句柄的类型，reference_index指向常量池的索引，reference_index指向具体的类型由reference_kind决定。
```java
CONSTANT_InvokeDynamic_info{
	u1 tag;
    u2 bootstrap_method_attr_index;
    uw name_and_type_index;
}
```
CONSTANT_InvokeDynamic_info的tag为18,bootstrap_method_attr_index是指向引导方法的索引，定位到一个引导方法。引导方法用在于动态调用时进行运行时函数查找和绑定，引导方法表是Class的一个属性，name_and_type_index指向常量池的索引，指向的表项必须是CONSTANT_NameAndType，用来表示方法的名字和签名。
## **Class的访问标记** ##
访问标志用2字节来表示，用来表示类的访问信息，每一种类型都是设置访问标志中的特定位来实现的。

- ACC_PUBLIC 0x0001	表示public类
- ACC_FINAL	0x0010	是否为final类
- ACC_SUPER	0x0020	使用增强方法调用父类方法
- ACC_INTERFACE	0x0200	是否为接口
- ACC_ABSTRACT	0x0400	是否是抽象类
- ACC_SYNTHETIC	0x1000	由编译器产生的类，没有源码
- ACC_ANNOTATION	0x2000	是否是注解类
- ACC_ENUM	0x4000	是否是枚举

## **类、父类和接口** ##
```java
u2	this_class
u2	super_class
u2	interfaces_count;
u2	interfaces[interfaces_count]
```
this_class和super_class都是2字节无符号整数,指向常量池中的CONSTANT_Class,表示当前的类和父类，super_class指向的父类不可以是final的。
如果类没有实现接口的话,interfaces_count为０,interfaces指向为空,否则的话以数组的形式保存多个接口的索引，每一个索引都指向一个常量池中的CONSTANT_Class。
## **Class的字段** ##
在接口描述之后，由于一个类可以有多个字段,需要指明类的个数：
```java
u2	fields_count;
field_info fields[fields_count]
```
字段的数目是２字节无符号整数。字段的具体信息为field_info的结构。
```
field_info{
	u2	access_flags;
    u2	name_index;
    u2	descriptor_index;
    u2	attributes_count;
    attribute_info	attributes[attributes_count]
}
```
access_flags是字段的访问属性，然后是name_index指向常量池中的一个CONSTANT_Utf8结构表示字段的名称，descriptor_index也是指向常量池的CONSTANT_Utf8结构表示字段的类型。
attributes用来存储字段的额外属性信息，例如初始化值和注释信息等，attributes_count表示属性的个数，attributes表示属性的具体内容，每个属性都用attribute_info来表示。

## **Class方法的基本结构** ##
Class方法信息跟字段信息差不多，都是由：
```java
u2	methods_count;
method_info	methods[methods_count];
```
methods_count是一个2字节整数，表示这个Class有多少个方法，具体的方法信息是在method_info结构中存储。
```java
method_info{
	u2	access_flags;
    u2	name_index;
    u2 	descriptor_index;
    u2	attributes_count;
    attribute_info	attributes[attributes_count];
}
```
access_flags表示类的访问标志，name_index指向一个常量池的CONSTANT_Utf8索引，表示方法的名称，descriptor_index指向常量池的索引，是方法的描述符，表示方法的签名，方法的参数类型写在小括号中，并在括号的右边给出方法的返回值，eg : (IDLjava/lang/Thread;)Ljava/lang/Object。
方法也可以附带额外信息，attributes_count表示属性的个数，然后有attribute_count个属性attribute_info的描述。
```java
attribute_info{
	u2	attribute_name_index;
    u4	attribute_length;
    u1	info[attribute_length];
}
```
attribute_name_index指向常量池中的CONSTANT_Utf字符串，表示属性的名称，attribute_length为当前attribute的剩余长度的,然后是attribute_length个字节的byte数组。
### **方法执行主体－Code属性**　####
```java
Code_attribute{
	u2	attribute_name_index;
    u4	attribute_length;
    u2	max_stack;
    u2	max_locals;
    u4	code_length;
    u1	code[code_length];
    u2	exception_table_length;
    {
    	u2	start_pc;
        u2	end_pc;
        u2	handler_pc;
        u2	catch_type;
    }	exception_table[exception_table_length];
    u2	attributes_count;
    attribute_info attributes[attributes_count];
}
```
Code属性的attribute_name_index指向了常量池的索引，类型为CONSTANT_Utf8，表示属性的名称，Code属性的值恒为Code，attribute_length的值为Code属性除了前六个字节的长度。
在方法执行的过程，方法的操作数栈和局部变量表在不断地改变，max_stack表示方法的最大操作数栈，max_local表示局部变量表的最大值。
方法的字节码由code_length和code[code_length]两部分组成，code_length表示字节码的长度，code[code_length]表示字节码内容的本身。
在字节码之后，就是方法的异常处理表，异常处理表由两部分组成，表项数量和内容，exception_table_length表示表项的数目，exception_table[exception_table_length]结构表示异常表。异常表项由四部份组成，start_pc、end_pc、handler_pc、catch_type，它表示在方法字节码start_pc到end_pc只要遇到catch_type所指向的常量池索引的CONSTANT_Class异常类型，就会跳到handler_pc的位置执行。
Code属性可能包含更多的信息attribute_info，他们通过attribute属性嵌套在Code属性中。
#### **记录行号－LineNumberTable** ####
LineNumberTable是Code属性的属性，用来描述Code属性的，它主要是用来记录字节码偏移量和行号的对应关系。
```java
LineNumberTable_attribute{
	u2	attribute_name_index;
    u4	attribute_length;
    u2	line_number_table_length;{
    	u2 start_pc;
        u2 line_number;
    }line_number_table[line_number_table_length];
}
```
attribute_name_index指向常量池的索引，是一个CONSTANT_Utf8的字符串，表示属性的名称，在LineNumberTable_attribute中是LineNumberTable，attribute_length是一个无符号整数，表示属性的长度，line_number_table_length表示表项有多少条记录,line_number_table表示表项的内容，有line_number_table_length项，其中start_pc为字节码偏移量，line_number为对应的行号。
#### **局部变量和参数－LocalVariableTable** ####
LocalVariableTable为局部变量表：
```java
LocalVariableTable{
	u2	attribute_name_index;
    u4	attribute_length;
    u2	local_variable_table_length{
    	u2 start_pc;
        u2 length;
        u2 name_index;
        u2 descriptor_index;
        u2 index;
    }local_variable_table[local_variable_table_length];
}
```
attribute_name_index指向常量池索引，表示当前属性的名称，局部变量表的名称为LocalVariableTable,attribute_length为属性的长度，local_variable_table_length句局部变量表的表项数量。
局部变量表中由start_pc、length、name_index、descriptor_index和index组成。
start_pc、length表示当前局部变量的开始位置start_pc和结束位置start_pc+length、name_index指向一个常量池索引,表示局部变量的名称、descriptor_index指向常量池的索引，表示局部变量的类型描述,index表示局部变量在当前栈帧的局部变量表中的槽位,long和double的数据占据局部变量表的两个槽位。
#### **字节码类型校验－StackMapTable属性** ####
#### **异常－Exceptions属性** ####
Method除了Code属性之外，每一个方法还可以有一个Exceptions属性，用于保存该方法可能会抛出的异常信息,它跟Code属性的exception_table不一样，它表示方法可能抛出的异常，一般是throws指定的。
```java
Exceptions_attribute{
	u2 attribute_name_index;
    u4 attribute_length;
    u2 number_of_exceptions;
    u2 exception_index_table[number_of_exceptions];
}
```
attribute_name_index指向常量池中索引，指定了属性的名称，恒为Exceptions，attribute_length表示属性的长度，number_of_exceptions表示表项的数目。exception_index_table列出了所有的异常，每一项都指向了常量池中的索引，对应的结构是CONSTANT_Class，指向异常类类型。
## **Class来源－SourceFile**
SourceFile是Class文件的属性，表示当前这个Class文件是由哪个源文件编译出来的。
```java
SourceFile_attribute{
	u2	attribute_name_index;
    u4	attribute_length;
    u2	sourcefile_index;
}
```
attribute_name_index指向常量池的索引，表示属性的名称，为SourceFile，attribute_length为属性长度，对与SourceFile来说为２,sourcefile_index指向常量池索引表示源文件文件名。
## **动态调用－BootstrapMethod属性** ##
BootstrapMethod用来保存和描述引导方法的，invokeDyamic能够在运行时根据实际情况返回合适的方法调用，而根据那种策略来查找所需要的方法，则由引导方法来决定。
```java
BootstrapMethods_attribute{
	u2	attribute_name_index;
    u4	attribute_length;
    u2	num_bootstrap_methods;
    {
    	u2 bootstrap_method_ref;
        u2 num_bootstrap_arguments;
        u2 boostrap_arguments[num_bootstrap_arguments];
    } bootstrap_methods[num_bootstrap_methods];
}
```
attribute_name_index指向常量池索引，表示属性的名称，恒为BootstrapMethods，attribute_length表示属性的长度，num_bootstrap—methods表示这个类中包含的引导方法的个数，bootstrap_methods是所有引导方法的内容，每一个引导方法包含bootstrap_method_ref、num_bootstrap_arguments、bootstrap_argumnets[num_bootstrap_arguments],bootstrap_method_ref指向常量池的索引，类型结构是CONSTANT_MethodHandle,表示指定函数，num_bootstrap_arguments是引导方法的参数个数，boostrap_arguments[num_bootstrap_arguments]是引导方法的参数类型列表，每一个都指向常量池索引，指定特定的参数类型。
## **内部类－InnerClasses属性** ##
InnerClasses是Class文件的属性，用来表明外部类与内部类之间的关系。
```java
InnerClasses_attribute{
	u2	attribute_name_index;
    u4	attribute_length;
    u2	number_of_classes;
    {
    	u2 inner_class_info_idnex;
        u2 other_class_info_index;
        u2 inner_name_index;
        u2 inner_class_access_flags;
    }
    classes[number_of_classes];
}
```
attribute_name_index指向常量池的索引，表示属性的名称，这里恒为"InnerClasses",attribute_length是属性的长度，number_of_classes是指内部类的个数，每一个内部类记录都包含四个字段，inner_class_info_index指向常量池的索引，指向的是一个CONSTAANT_Class类型，表示内部类的类型，other_class_info_index也是指向CONSTANT_Class类型的常量池索引，不过它表示的是外部类的类型，inner_name_index指向常量池索引，表示内部类的名称，inner_class_access_flags表示内部类的访问标志。
## **Depercated属性** ##
Depreated属性可以用在类、字段、方法上，表示这些方法、字段、方法可能会在未来的版本中废弃.
```java
Depreated_attribute{
	u2	attribute_name_index;
    u4	attribute_length;
}
```
attribute_name_index指向常量池索引，表示属性的名称，恒为Depreated，attribute_length为０,当一个类、字段、方法被注解为Depreated的话，就会有这个属性。
