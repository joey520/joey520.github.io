---
toc: true
title: 深入理解Bitcode
date: 2019-11-21 9:51:15
categories: iOS
tags: [Bitcode,LLVM]
---

## 前言

在最近的版本中，MSDK要支持bitcode了。于是开始对这个2015WWDC就提出老概念进行了新的学习。很多开发者都知道开启bitcode可以在archive时生成一份中间代码以提交给苹果，让其根据不同的架构生成不同的ipa，以减少安装包体积与应对未来出现新架构不兼容问题。也几乎都碰到过引用第三方库因其不支持bitcode而编译不过。但是Bitcode到底是什么呢？

我们知道编译器前端可以把不同的高级语言转换为中间代码（IR），再由编译器后端将中间代码转换成指定平台的目标代码，再通过链接器把目标代码和依赖的库文件link到一起即可变成可执行文件。因此理论上只需要中间代码即可生成任一平台的目标代码（只要编译器后端支持），因为中间代码已经包含了源程序所所要表达的全部意思。

再看下LLVM官方文档的描述：

```
The LLVM code representation is designed to be used in three different forms: as an in-memory compiler IR, as an on-disk bitcode representation (suitable for fast loading by a Just-In-Time compiler), and as a human readable assembly language representation.This allows LLVM to provide a powerful intermediate representation for efficient compiler transformations and analysis, while providing a natural means to debug and visualize the transformations. The three different forms of LLVM are all equivalent. 
```

重点是三种代码表现形式：编译过程中的中间代码IR；编译出的bitcode；可读的汇编代码。其实都是对源代码的一种描述，只是面向了不同的对象时的表现形态。由此可知bitcode其实只是IR的另一种表现形式。

<!--more-->

## bitcode可以做什么

1.首先我们写一个小demo:

```c
#include <stdio.h>
int main() {
    printf("hello\n");
    return 0;
}
```

2.把源文件转换为LLVM的中间表示bitcode文件：

```shell
$ clang -emit-llvm -c hello.c -o hello.bc
```

可以看到hello.bc文件格式如下：

```shell
$ file hello.bc
hello.bc: LLVM bitcode, wrapper x86_64
```

3.然后把bitcode转换成目标文件,输出为Mach-O文件：

```shell
$ clang -c hello.bc -o hello.bc.o
file hello.bc.o
hello.bc.o: Mach-O 64-bit object x86_64
```

4.直接把源代码编译成目标文件并和由bitcode生成目标文件进行对比：

```shell
$ clang -c hello.c -o hello.o
$ md5 hello.o hello.bc.o
MD5 (hello.o) = 92311036e62f4b3e4468b3c93c314960
MD5 (hello.bc.o) = 92311036e62f4b3e4468b3c93c314960
```

可以发现通过bitcode获取和直接编译出来的目标代码是一模一样的。因此只需要得到bicode文件就可以编译出一样的目标文件。

## Bitcode是什么：

通过LLVM官方对bitcode的描述，可以知道bitcode是一种以位为单位存取的二进制文件，它可以存在于包装结构中，如上一节中看到``LLVM bitcode, wrapper x86_64`` ,也可以存在于``Object``文件中，例如``Mach-O``文件等。**对于``Mach-O``文件并且必定存在于名为``__LLVM``或者``__bitcode``的section中，因此我们可以根据这两个字段来判断生成的Mach-O文件是否包含bitcode.**

### Bitcode的优化

1.首部

Bitcode利用首部简单明了的描述了Bitcode文件的信息。通过该首部编译器可以快速确定该如何编当前的Bitcode文件。

采用前4个字节是固定的``magic number``来标识，就我这个`Apple clang version 11.0.0 (clang-1100.0.33.12)`编出来的是`FEEDFACF`。当然bitcode文件格式仍然还在变化，这并不能作为唯一识别bitcode文件的依据。

然后4个字节描述了CPU架构，实际是为了表示Bitcode文件的字节序，以便编译器可以正确读取。

之后描述了`Load command`，文件类型信息等。

2.整形长度

bitcode为了减少体积，充分利用了按位存取的特定，根据数据类型更精确的分配数据长度，例如``boolean``只需要一个``bit``即可。而且利用变长整形来encode数据来减少表示值很小数时的内存浪费，它按4个bit这样为一个feild来表示数据，最高位表示是否连续（即和后面的数据是否是组合的），后面的3个bit表示0-7的数，对于连续的数字，后面的接的feiled通过移位``<<3``后和前面的feild的数据组合到一起。

例如数字27在bitcode中表示为``1011 0011``,前一个feild值为3，由于最高位1表示连续，则后一个feild的``011``需要左移3个bit即24，而后一个feild最高位为0表示不再连续。则数据为：``24+3 = 27``.

咋一看浪费了两个bit来表示是否连续，但是首先它只占用了一个字节，，而且对于比``1111 0111``略大一点的数时，只需要在连续一个feild即4个bit即可，而整形恰好又是最基本的数据。可见这里的可以减少非常多的内存空间。

3.6个bit的字符

在bitcode中，只用6个bit来表示字符。因为它只需要``a-z``,``A-Z``，``0-9``和``.-``共64个字符。所以所谓优化，最重要的就是把不需要的东西都去掉。。。

上面我们知道bitcode文件是一种LLVM bitcode文件，实际是一些字节因此无法阅读，不过llvm-dis是一个可以可以把bitcode文件转换为LLVM表示的反汇编的可视化的汇编语言(注意与Clang下的表示不一致):

```shell
$ llvm-dis < hello.bc
```

同样的-S选项可以把源文件转换为LLVM表示的汇编语言(注意与Clang下的表示不一致):

```shell
$ clang -emit-llvm -S hello.c -o hello.ll
```

打开hello.ll可以看到如下，我加了一些comment来简单的描述：

```assembly
<---描述bitcode文件的基本信息,数据对其--->
; ModuleID = 'hello.c'
source_filename = "hello.c"
target datalayout = "e-m:o-i64:64-f80:128-n8:16:32:64-S128"
target triple = "x86_64-apple-macosx10.14.0"
<---程序中输出的全局常量，由于是字符串，一字节对齐--->
@.str = private unnamed_addr constant [7 x i8] c"hello\0A\00", align 1

; Function Attrs: noinline nounwind optnone ssp uwtable
<---定义函数主体部分，int已经转换成了内部的表现i32，同事对该函数进行下面的attribute0的编译描述--->
define i32 @main() #0 {
<---分配一个局部int变量并赋值为0，不过貌似没有用到，应该会在编译优化时被干掉--->
  %1 = alloca i32, align 4
  store i32 0, i32* %1, align 4
  <---调用prinrt函数，并把上面str传入,此处会由汇编器生成跳转指令--->
  %2 = call i32 (i8*, ...) @printf(i8* getelementptr inbounds ([7 x i8], [7 x i8]* @.str, i32 0, i32 0))
  <---函数返回--->
  ret i32 0
}
<---声明了printf函数，同时对该函数进行下面的attribute1的编译描述--->
declare i32 @printf(i8*, ...) #1

attributes #0 = { noinline nounwind optnone ssp uwtable "correctly-rounded-divide-sqrt-fp-math"="false" "disable-tail-calls"="false" "less-precise-fpmad"="false" "min-legal-vector-width"="0" "no-frame-pointer-elim"="true" "no-frame-pointer-elim-non-leaf" "no-infs-fp-math"="false" "no-jump-tables"="false" "no-nans-fp-math"="false" "no-signed-zeros-fp-math"="false" "no-trapping-math"="false" "stack-protector-buffer-size"="8" "target-cpu"="penryn" "target-features"="+cx16,+fxsr,+mmx,+sahf,+sse,+sse2,+sse3,+sse4.1,+ssse3,+x87" "unsafe-fp-math"="false" "use-soft-float"="false" }
attributes #1 = { "correctly-rounded-divide-sqrt-fp-math"="false" "disable-tail-calls"="false" "less-precise-fpmad"="false" "no-frame-pointer-elim"="true" "no-frame-pointer-elim-non-leaf" "no-infs-fp-math"="false" "no-nans-fp-math"="false" "no-signed-zeros-fp-math"="false" "no-trapping-math"="false" "stack-protector-buffer-size"="8" "target-cpu"="penryn" "target-features"="+cx16,+fxsr,+mmx,+sahf,+sse,+sse2,+sse3,+sse4.1,+ssse3,+x87" "unsafe-fp-math"="false" "use-soft-float"="false" }
<---记录产生bitcode文件的编译器的版本等信息，用于指定编译器进行后续转换--->
!llvm.module.flags = !{!0, !1, !2}
!llvm.ident = !{!3}

!0 = !{i32 2, !"SDK Version", [2 x i32] [i32 10, i32 15]}
!1 = !{i32 1, !"wchar_size", i32 4}
!2 = !{i32 7, !"PIC Level", i32 2}
!3 = !{!"Apple clang version 11.0.0 (clang-1100.0.33.8)"}
```

大概可以了解到bitcode文件记录了源文件的一些基本信息如ModuleID(参考XXXX)，文件名，ABI。然后包含源代码的解析。最后保留了构建bitcode文件的编译器的版本。

### Bitcode的结构

我们分别编译一个不带bitcode的Object文件和一个带有bitcode的object文件，然后进行对比：

```shell
$ otool -l main_bitcode.o >> main_bitcode.o.txt
$ otool -l main.o >> main.o.txt
$ vimdiff main.o.txt main_bitcode.o.txt
```

![image-20191119111851211](https://github.com/joey520/joey520.github.io/blob/hexo/post_images/bitcode_bundle.png?raw=true)

可以看到开启bitcode后几乎每个section都会变大，并且多了名为`__bitcode`的section和名为`__cmdline`的section。

通过`segedit`可以提取出指定section.

```shell
$ segedit -extract __LLVM __bitcode main_bitcode.o.bc \
-extract __LLVM __cmdline main_bitcode.o.cmdline \
main_bitcode.o
```

对提取出的bitcode和直接编译出的bitcode文件进行MD5校验：

```shell
$ md5 main.bc main_bitcode.o.bc
MD5 (main.bc) = e9e80622c4830574dc2a6ed209bb662c
MD5 (main_bitcode.o.bc) = 2dee4301533eef0be4d91670d5de3187
```

并且查看文件大小可以看到:

```shell
$ ls -al main.o main.bc main_bitcode.o.bc
-rw-r--r--  1 joey.cao  staff  3184 11 19 12:47 main.bc
-rw-r--r--  1 joey.cao  staff   896 11 19 12:41 main.o
-rw-r--r--  1 joey.cao  staff  3168 11 19 12:41 main_bitcode.o.bc
```

可以看到导出的bitcode和直接编出的bitcode并不完全一致，提取出的bitcode比直接编译出的bc文件大很多。

`__cmdline`section存储了一些用于bitcode重建object文件是的clang编译选项，只有会影响代码生成和没有存在bitcode section的属性才会存在这里。

链接时，链接器把Object文件链接到一起生成Mach-O文件，把不同的object文件中的`__bitcode`section中的数据取出来并放到不同的文件中，这里文件会连同`__cmdline`中的编译选项整合为[xar-archive](https://en.wikipedia.org/wiki/Xar_%28archiver%29)，这是macOS上的一种归档格式，类似我们archive出来的文件。产生的archive文件会保存在`__LLVM`segment中的`__bundle`section中。

直接查看含有bitcode的可执行文件中的`__LLVM`segment下的`__bundle`section如下, otool -v可以打印出可视化的符号:

```shell
$ otool -v -s __LLVM __bundle a.out
a.out:
Contents of (__LLVM,__bundle) section
For (__LLVM,__bundle) section: xar table of contents:
<?xml version="1.0" encoding="UTF-8"?>
<xar>
 <subdoc subdoc_name="Ld">
  <version>1.0</version>
  <architecture>x86_64</architecture>
  <platform>macOS</platform>
  <sdkversion>10.14.0</sdkversion>
  <dylibs>
   <lib>{SDKPATH}/usr/lib/libSystem.B.dylib</lib>
  </dylibs>
  <link-options>
   <option>-execute</option>
   <option>-platform_version</option>
   <option>macOS</option>
   <option>10.14.0</option>
   <option>10.14.0</option>
   <option>-e</option>
   <option>_main</option>
   <option>-executable_path</option>
   <option>a.out</option>
  </link-options>
 </subdoc>
 <toc>
  <checksum style="sha1">
   <size>20</size>
   <offset>0</offset>
  </checksum>
  <creation-time>2019-11-19T05:00:11</creation-time>
  <file id="1">
   <name>1</name>
   <type>file</type>
   <data>
    <archived-checksum style="sha1">a16348f7b5300db507ec381a097348633c443272</archived-checksum>
    <extracted-checksum style="sha1">a16348f7b5300db507ec381a097348633c443272</extracted-checksum>
    <size>3168</size>
    <offset>20</offset>
    <encoding style="application/octet-stream"/>
    <length>3168</length>
   </data>
   <file-type>Bitcode</file-type>
   <clang>
    <cmd>-triple</cmd>
    <cmd>x86_64-apple-macosx10.14.0</cmd>
    <cmd>-emit-obj</cmd>
    <cmd>-disable-llvm-passes</cmd>
   </clang>
  </file>
 </toc>
</xar>
```

可以看到，里面`__bundule`section就是一个xml配置文件，包含了link选项配置，数据长度，和通过`__cmdline`重建object文件的clang选项等等。因此通过它就可以创建可执行文件。

我们尝试从中提取出bitcode文件:

```shell
//1.提取出bitcode相关的bundle
$ segedit -extract __LLVM __bundle bundle main_bitcode.out
//2.解压出头文件
$ xar -d toc.xml -f bundle
```

我们对比bundle描述头文件和通过otool打印出的bundle描述文件一模一样。

```shell
//1.加压出bitcode文件，自动生成在当前文件夹下名为1
$ xar -x -f bundle
//2.比较提取出的bitcode文件和编译出的bitcode文件
$ md5 main_bitcode.o.bc 1
MD5 (main_bitcode.o.bc) = 5d6b72f817fb8fb92550cc9aae6340ab
MD5 (1) = 5d6b72f817fb8fb92550cc9aae6340ab
```

比较从`__bitcode`section和 链接后从`__bundle`中提取出的bitcode文件发现是完全一致的，说明链接时只是单纯的对bitcode进行拷贝。

### 小结

经过本章的分析，对Bitcode文件是什么，所做的优化，文件结构，以及怎么使用有了比较深的认识。其中可以学习到的是Bitcode优化的部分，其中的思想其实也可以应用再我们的业务中。同时再次遇到Bitcode时也不在会是一脸懵逼。同时还更熟悉了一些系统工具的使用，可以有效的帮助我们在遇到一些难解问题时进行分析。

## Bitcode使用

#### 通过Bitcode文件生成Mach-O文件

通过Clang的`-fembed-bitcode`描述是生成的Object文件内嵌bitcode:

```shell
$ clang -fembed-bitcode -c main.cc -o main.o
```

利用objdump或者otool工具可以查看Object文件，根据官方文档说明，内嵌bitcode文件后，会出现__LLVM字段，因此可以依据此来判断bitcode是否生效：

```shell
$ otool -l main.o | grep __LLVM
```

或是：

```shell
$ objdump -all-headers mian_bitcode.out | grep __LLVM
```

需要注意的一点是，如果单纯在buildSetting中`Enable Bitcode`编出来的products是不带bitcode。例如，现在需要编译一个动态库，设置`Enable Bitcode`为YES。 然后选择模拟器，编译一个`x86_64`版本的动态库。通过查找`__LLVM`section发现并没有bitcode。

我们查看一下编译输出：

```shell
/Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin/clang -x objective-c -target x86_64-apple-ios9.0-simulator -fmessage-length=0 -fdiagnostics-show-note-include-stack -fmacro-backtrace-limit=0 -std=gnu11 -fobjc-arc -fobjc-weak -fmodules -gmodules -fmodules-cache-path=/Users/joey.cao/Desktop/DJINetwork/djinetworkrtkhelper/DJINetworkRTKHelper/DerivedData/ModuleCache.noindex -fmodules-prune-interval=86400 -fmodules-prune-after=345600 
...
-c /Users/joey.cao/Desktop/DJINetwork/djinetworkrtkhelper/DJINetworkRTKHelper/DJINetworkRTKHelper/DJINRTKManager.m -o /Users/joey.cao/Desktop/DJINetwork/djinetworkrtkhelper/DJINetworkRTKHelper/DerivedData/DJINetworkRTKHelper/Build/Intermediates.noindex/DJINetworkRTKHelper.build/Release-iphonesimulator/DJINetworkRTKHelper.build/Objects-normal/x86_64/DJINRTKManager.o

```

为了好看，把中间的option选项省略了，可以看到其实就是利用clang，然后把build setting里的编译配置作为option来编译出了一个Object文件到build文件夹下面。 但是build option中并没有bitcode，所以编出来的产物自然不带bitcode。

#### 脚本构建带有Bitcode的Mach-O文件

因此我们应该利用脚本主动调用`-fembed-bitcode`来生成带有bitcode的产物。正好，我们编出的动态库还需要同时支持多个架构，如果完全依赖xcode，我们得真机下编一次，模拟器编一次，再手动合并成`fat file`。而这都可以通过脚本来处理：

```shell
xcodebuild -project "${SRCROOT}/${PROJECT_NAME}.xcodeproj" -configuration "${CONFIGURATION}" -scheme "${PROJECT_NAME}" -sdk iphoneos VALID_ARCHS="x86_64 arm64 arm64e" -derivedDataPath "${SRCROOT}" clean
xcodebuild -project "${SRCROOT}/${PROJECT_NAME}.xcodeproj" -configuration "${CONFIGURATION}" -scheme "${PROJECT_NAME}" -sdk iphonesimulator VALID_ARCHS="x86_64 arm64 arm64e" -derivedDataPath "${SRCROOT}" clean

xcodebuild OTHER_CFLAGS="-fembed-bitcode" -project "${SRCROOT}/${PROJECT_NAME}.xcodeproj" -configuration "${CONFIGURATION}" -scheme "${PROJECT_NAME}" -sdk iphoneos VALID_ARCHS="x86_64 arm64 arm64e" -derivedDataPath "${SRCROOT}"
xcodebuild OTHER_CFLAGS="-fembed-bitcode" -project "${SRCROOT}/${PROJECT_NAME}.xcodeproj" -configuration "${CONFIGURATION}" -scheme "${PROJECT_NAME}" -sdk iphonesimulator VALID_ARCHS="x86_64 arm64 arm64e" -derivedDataPath "${SRCROOT}"

MT_DEVICE_DIR=iphoneos
MT_SIMUL_DIR=iphonesimulator
MT_OUTPUT_DIR="${PROJECT_DIR}/Output/"
echo "{SRCROOT}/build/Products/${CONFIGURATION}-${MT_DEVICE_DIR}/${PROJECT_NAME}.framework"
rm -rf "${MT_OUTPUT_DIR}"
mkdir -p "${MT_OUTPUT_DIR}"
cp -a -f "${SRCROOT}/Build/Products/${CONFIGURATION}-${MT_DEVICE_DIR}/${PROJECT_NAME}.framework" "${MT_OUTPUT_DIR}"
cp -a -f "${SRCROOT}/Build/Products/${CONFIGURATION}-${MT_DEVICE_DIR}/${PROJECT_NAME}.framework.dSYM" "${MT_OUTPUT_DIR}"
lipo -create "${SRCROOT}/Build/Products/${CONFIGURATION}-${MT_SIMUL_DIR}/${PROJECT_NAME}.framework/${PROJECT_NAME}" "${MT_OUTPUT_DIR}/${PROJECT_NAME}.framework/${PROJECT_NAME}" -output "${MT_OUTPUT_DIR}/${PROJECT_NAME}.framework/${PROJECT_NAME}"
```

我们利用xcodebuild传入`OTHER_CFLAGS`，`OTHER_CFLAGS`的选项会直接传递给编译器，因此clang就具有了`-fembed-bitcode`选项：

```shell
/Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin/clang -x objective-c -arch x86_64 -fmessage-length=0  
-fembed-bitcode 
-c 
/Users/joey.cao/Desktop/DJINetwork/djinetworkrtkhelper/DJINetworkRTKHelper/DJINetworkRTKHelper/DJINRTKManager.m 
-o 
/Users/joey.cao/Desktop/DJINetwork/djinetworkrtkhelper/DJINetworkRTKHelper/Build/Intermediates.noindex/DJINetworkRTKHelper.build/Release-iphonesimulator/DJINetworkRTKHelper.build/Objects-normal/x86_64/DJINRTKManager.o
```

通过刚才的测试可以看到如果没有`-fembed-bitcode`编译的产物是不包含`__LLVM`和`__bitcode`字段的，对编出来的`fat file`进行检查如下，可以看到已经包含了bitcode文件：

```shell
$ otool -arch arm64 -l /Users/joey.cao/Desktop/DJINetwork/djinetworkrtkhelper/DJINetworkRTKHelper/Output/DJINetworkRTKHelper.framework/DJINetworkRTKHelper | grep __LLVM
  segname __LLVM
   segname __LLVM
```

#### Bitcode应用上架

使用bitcode嘛最终目的还是为了应用的上架，但是比较坑爹是所有依赖的库文件必须支持bitcode。否则构建的mach-O文件就不能支持bitcode。假设我们目前所有依赖的库文件都已经支持bitcode了，我们使用一个MSDK预上架的工程archive一个支持bitcode的和一个不支持bitcode。可以看到以下区别

1.惊人发现开启bitcode之前归档出的文件只有118.5MB,而开启bitcode之后直接增加到254MB。这下是不是要超过Apple的文件大小限制了啊。根据前面的测试我们知道bitcode放在`__LLVM`section，而苹果二进制文件大小限制只是针对`__Text`段，即代码段的，因此bitcode文件并不会影响苹果的限制。

2.当我们需要上传App时，若果开启了bitcode的App会提示选择是否上传bitcode内容。如果勾选了bitcode，则同时会上传bitcode到苹果服务器，并且用户下载的ipa是苹果针对bitcode再次编译出来的。而如果不勾选，则用户下载的是我们上传的ipa，并且会剔除bitcode。 如果选择的Development发布方式，则会出现选择`Rebuild from bitcode`，此时如果勾选则会发现会利用`bitcode-build-tool`来构建ipa文件，而不是直接利用归档出的ipa。这样就跟苹果在服务器归档后发布给用户的ipa一致了：

```shell
$ /Applications/Xcode.app/Contents/Developer/usr/bin/bitcode-build-tool -v -t /Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin --sdk /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/SDKs/iPhoneOS13.2.sdk -o /var/folders/64/2_nlwt_s6kq250l7_t5mfdzwrh9_pg/T/ipatool20191119-81739-om7ou4/thinned-out/arm64/Payload/DJI\ MSDK\ Preview.app/DJI\ MSDK\ Preview --generate-dsym /var/folders/64/2_nlwt_s6kq250l7_t5mfdzwrh9_pg/T/ipatool20191119-81739-om7ou4/thinned-out/arm64/Payload/DJI\ MSDK\ Preview.app/DJI\ MSDK\ Preview.dSYM --strip-swift-symbols /var/folders/64/2_nlwt_s6kq250l7_t5mfdzwrh9_pg/T/ipatool20191119-81739-om7ou4/thinned-in/arm64/Payload/DJI\ MSDK\ Preview.app/DJI\ MSDK\ Preview
```

3.开启bitcode的归档文件下面多了个`BCSymbolMaps`的文件夹。这也可以作为一个判定开启bitcode成功的标志。`BCSymbolMaps`是一个类似dSYM的文件，可以辅助定位crash，因为我们知道苹果会根据bitcode再次构建ipa，那么原有的dSYM已经不能用了。因此当出现crash，必须通过App Store Connect下载新的dSYM来进行定位。

4.Cocoapods也是支持bitcode的，可以通过配置podfile快速为所有target设置:

```ruby
post_install do |installer|
    installer.pods_project.targets.each do |target|
        target.build_configurations.each do |config|
            config.build_settings['ENABLE_BITCODE'] = 'YES'
        end
    end
end
```

## 参考资料

1. http://llvm.org/docs/BitCodeFormat.html#metadata-attachment
2. https://jonasdevlieghere.com/embedded-bitcode/
3. https://blog.csdn.net/sinat_31177681/article/details/87355711

