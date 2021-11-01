# AFLGo:  定向灰盒Fuzzing 
> 📖: 对AFLGo的源码进行一个梳理，翻译或者理解不对的地方请指正。


<a href="https://mboehme.github.io/paper/CCS17.pdf" target="_blank"><img src="https://github.com/mboehme/mboehme.github.io/blob/master/paper/CCS17.png" align="right" width="250"></a>

AFLGo 是 <a href="https://lcamtuf.coredump.cx/afl/" target="_blank">American Fuzzy Lop (AFL)</a>的一个拓展.
给一个指定地址的集合（如`folder/file.c:582`)生成测试这些地址的输入。

与AFL不同的是，AFLGo将大部分时间精力用于到达特定的目标地点（强调目的性），而不会浪费资源去强调不相关的项目内容（并不是覆盖率导向）。定向Fuzzing可以用于以下的一些场景：

* **补丁测试** 通过将打补丁后的语句设置为目标。如果当一个关键组件被改变时，可以检查这个打补丁操作是否引入了新的漏洞。AFLGo可以暴露出这些打补丁的问题。
* **静态分析报告验证** 通过将静态分析后认为存在潜在危险或者引起漏洞的语句设置为目标。在评估程序的安全性时，静态分析工具可能会识别危险的位置，例如关键的系统调用。AFLGo可以测试这些语句是不是真的危险。
* **信息流检测** 通过设置敏感源和暴露点设置为目标。为了暴露数据泄漏的漏洞，安全研究人员希望产生能泄漏私人信息的具体执行。AFLGo可以帮助生成特定执行流程的输入。
* **crash 复现**  通过将堆栈追踪中的函数调用设置为目标。当Crash被报告时候，只有堆栈追踪和一些环境参数被发送到内部开发团队。为了保护用户的隐私，crash的输入往往是不公开的。AFLGo可以帮助内部团队复现这些crash。

AFLGo is 基于 Michał Zaleski \<lcamtuf@coredump.cx\>的<a href="http://lcamtuf.coredump.cx/afl/" target="_blank">AFL</a>. 此项目 [awesome-directed-fuzzing](https://github.com/strongcourage/awesome-directed-fuzzing) 可以查看相关的灰盒/白盒定向Fuzzing。

# 整合到OSS-Fuzz中
使用AFLGo最简单的方法是作为OSS-Fuzz中的补丁测试工具。集成如下:
* https://github.com/aflgo/oss-fuzz

# 环境变量
* **AFLGO_INST_RATIO** -- 插桩了距离值基本块的占比  (默认值: 100).
* **AFLGO_SELECTIVE** -- 只对有距离值的基本块添加AFL-蹦床？ (default: off).
* **AFLGO_PROFILING_FILE** -- 当CFG-tracing启用的时候，数据存储的地方

# 如何使用AFLGo插桩一个二进制
1) 安装带有<a href="http://llvm.org/docs/GoldPlugin.html" target="_blank">Gold</a>-plugin的 <a href="https://llvm.org/docs/CMake.html" target="_blank">LLVM</a> **11.0.0**。 你可以按照 <a href="https://github.com/aflgo/oss-fuzz/blob/master/infra/base-images/base-clang/checkout_build_install_llvm.sh" target="_blank">these</a> 中的指令或者执行 [AFLGo building script](./scripts/build/aflgo-build.sh)脚本。

> - LLVM: （Low Level Virtual Machine 底层虚拟机） 提供了与编译器相关的支持（编译期优化，链接优化等），是大多数编译器的后端。
>   - LLVM将各种语言转化为中间语言（IR），然后经过各种pass的优化，最后经过生成代码产出。
> - Gold: 一种新的，快速的，ELF专属的连接器
> - llvm-gold插件：在libLTO之上实现了gold插件的接口
>   - libLTO: 一个共享对象，是LLVM工具的一部分。libLTO提供了一个抽象的C接口来使用LLVM程序间优化器，而不暴露LLVM的内部细节。

2) 安装其它先行软件
```bash
sudo apt-get update
sudo apt-get install python3
sudo apt-get install python3-dev
sudo apt-get install python3-pip
sudo apt-get install libboost-all-dev  # 如果你在步骤7中使用genDistance.sh，则不需要boost(C++语言标准库提供扩展的一些C++程序库的总称)
sudo pip3 install --upgrade pip
sudo pip3 install networkx    #以下包都是为了CG或者CFG分析使用的
sudo pip3 install pydot
sudo pip3 install pydotplus
```
3) 编译AFLGo模糊测试器, LLVM 插桩pass 和距离计算器
```bash
# 获得源码
git clone https://github.com/aflgo/aflgo.git
export AFLGO=$PWD/aflgo

# 编译源码
pushd $AFLGO
make clean all #清除
cd llvm_mode
make clean all #清除
cd ..
cd distance_calculator/    # distance_calculator：cpp文件 用于计算距离
cmake -G Ninja ./  #cmake: 跨平台自动化构建系统 -G 指定构建系统生成器 Ninja:专注于速度的小型构建器
cmake --build ./   #--build: 建立一个项目  （cmake的命令内容查看 ./distance_calculator/CMakeLists.txt
popd
```
4) 下载项目源码，例如 (e.g., <a href="http://xmlsoft.org/" target="_blank">libxml2</a>) 或者直接执行 [libxml2 fuzzing script](./scripts/fuzz/libxml2-ef709ce2.sh)脚本。
```bash
# 克隆仓库
git clone https://gitlab.gnome.org/GNOME/libxml2
export SUBJECT=$PWD/libxml2
```
5) 设置目标 (例如提交中打的补丁 <a href="https://git.gnome.org/browse/libxml2/commit/?id=ef709ce2" target="_blank">ef709ce2</a>). 写文件 BBtargets.txt.
```bash
# 设置一个包含所有临时文件的文件夹
mkdir temp
export TMP_DIR=$PWD/temp

# 下载提交分析工具 
wget https://raw.githubusercontent.com/jay/showlinenum/develop/showlinenum.awk  #该脚本输入commit中有差别的行，格式为[path:]<line number>:<diff line>
chmod +x showlinenum.awk
mv showlinenum.awk $TMP_DIR

# 生成提交的 ef709ce2 基本块目标（BBtargets）
pushd $SUBJECT
  git checkout ef709ce2 #切换有漏洞的分支
  git diff -U0 HEAD^ HEAD > $TMP_DIR/commit.diff  # 比较当前版本与先前版本的差异，不包括上下文，输出到了commit.diff文件中
popd

# 获得有差异的行数（即目标）并提取输出到BBtargets.txt中
cat $TMP_DIR/commit.diff |  $TMP_DIR/showlinenum.awk show_header=0 path=1 | grep -e "\.[ch]:[0-9]*:+" -e "\.cpp:[0-9]*:+" -e "\.cc:[0-9]*:+" | cut -d+ -f1 | rev | cut -c2- | rev > $TMP_DIR/BBtargets.txt

# 打印提取到的结果
echo "Targets:"
cat $TMP_DIR/BBtargets.txt
```
6) **注意**: 如果没有目标，就没有必要插桩了。
7) 从目标(例如libxml2)中生成CG或者程序内CFG
```bash
# 设置 AFLgo 插桩
export CC=$AFLGO/afl-clang-fast
export CXX=$AFLGO/afl-clang-fast++

# 设置 AFLGo 插桩 标志
export COPY_CFLAGS=$CFLAGS
export COPY_CXXFLAGS=$CXXFLAGS
#  添加的标志主要是这里
export ADDITIONAL="-targets=$TMP_DIR/BBtargets.txt -outdir=$TMP_DIR -flto -fuse-ld=gold -Wl,-plugin-opt=save-temps"
# -targets 目标点位置文件  例：valid.c:2640
# -outdir  图导出的目录
# -flto    连接时间优化
# -fuse-ld=gold  使用gold连接器
# -Wl,-plugin-pot=save-temp   (wl)将后面的参数传给连接器,整体是保存子啊整个程序的.bc文件 为了是进一步生成CG
export CFLAGS="$CFLAGS $ADDITIONAL"
export CXXFLAGS="$CXXFLAGS $ADDITIONAL"

# 建立 libxml2 (为了生成 CG和CFGs).
#    这里就是具体地编译某个东西了,插桩的逻辑主要在afl-llvm-pass.so.cc,见源码
export LDFLAGS=-lpthread
pushd $SUBJECT
  ./autogen.sh
  ./configure --disable-shared
  make clean
  make xmllint
popd
# * 如果链接器（CCLD）抱怨说你应该运行ranlib，请确保在/usr/lib/bfd-plugins中可以找到libLTO.so和LLVMgold.so（用Gold构建LLVM）
# * 如果编译器崩溃了，可能是LLVM不支持我们的插桩（afl-llvm-pass.so.cc:540-577）。LLVM经常改变插桩的API。
# * 你可以通过并行构建来加快编译的速度。但是，这可能会影响哪些BB被识别为目标。见 https://github.com/aflgo/aflgo/issues/41.


# 测试 CG/CFG 是否被成功提取
$SUBJECT/xmllint --valid --recover $SUBJECT/test/dtd3 #程序是否正常运行
ls $TMP_DIR/dot-files                                 #列出生成的文件（dot文件）
echo "Function targets"
cat $TMP_DIR/Ftargets.txt                             #目标函数  例如：xmlAddID

# 清理
cat $TMP_DIR/BBnames.txt | rev | cut -d: -f2- | rev | sort | uniq > $TMP_DIR/BBnames2.txt && mv $TMP_DIR/BBnames2.txt $TMP_DIR/BBnames.txt  #提取出基本块的名字 例：HTMLparser.c:109 HTMLparser.c:112
cat $TMP_DIR/BBcalls.txt | sort | uniq > $TMP_DIR/BBcalls2.txt && mv $TMP_DIR/BBcalls2.txt $TMP_DIR/BBcalls.txt                             #提取出基本块的调用关系 例 HTMLparser.c:117,__xmlRaiseError

# 生成距离 ☕️
# AFLGO/scripts/genDistance.sh是原始版本，但速度明显较慢
$AFLGO/scripts/gen_distance_fast.py $SUBJECT $TMP_DIR xmllint  

# 检查距离文件
echo "Distance values:"
head -n5 $TMP_DIR/distance.cfg.txt
echo "..."
tail -n5 $TMP_DIR/distance.cfg.txt
```
8) 注意: 如果没有`distance.cfg.txt`文件, 是因为计算CG级别和BB级别时候出错了. See `$TMP_DIR/step*`.
9) 插桩目标文件 (i.e., libxml2)    具体逻辑查看`afl-llvm-pass.so.cc`文件
```bash
export CFLAGS="$COPY_CFLAGS -distance=$TMP_DIR/distance.cfg.txt"  #将距离文件传给编译器
export CXXFLAGS="$COPY_CXXFLAGS -distance=$TMP_DIR/distance.cfg.txt"

# 清楚并插桩距离
pushd $SUBJECT
  make clean
  ./configure --disable-shared    #禁止创建共享库
  make xmllint
popd
```

如果你的编译在这一步崩溃了，请看Issue [#4](https://github.com/aflgo/aflgo/issues/4#issuecomment-333947041).

# 如何fuzz插桩了的二进制
> AFLGo 增加的逻辑看   `afl-fuzz.c` 文件
* 我们设置了基于**指数**的退火法功率调度 (-z exp).
* 我们将探索时间设置为45分钟（-c 45m），假设模糊器运行了大约一个小时
```bash
# 构建种子语料库
mkdir in
cp $SUBJECT/test/dtd* in
cp $SUBJECT/test/dtds/* in

$AFLGO/afl-fuzz -S ef709ce2 -z exp -c 45m -i in -o out $SUBJECT/xmllint --valid --recover @@
```
* **小技巧**: 同时用经典的AFL将最新的版本作为主版本进行模糊处理 :)
```bash
$AFL/afl-fuzz -M master -i in -o out $MASTER/xmllint --valid --recover @@
```
* 查看[fuzzing scripts](./scripts/fuzz) 更多的真实程序案例，例如Binutils, jasper, lrzip, libming 和 DARPA CGC。 
