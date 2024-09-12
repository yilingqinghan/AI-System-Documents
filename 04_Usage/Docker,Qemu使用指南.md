# Docker & QEMU & Ubuntu 使用指南

#### 从头开始

```shell
# 主机安装依赖
sudo yum update -y
sudo yum install -y yum-utils device-mapper-persistent-data lvm2

# 启用并开机自启 docker
sudo systemctl start docker
sudo systemctl enable docker

# 启动 docker 并挂载 /data 到主机 /data
docker run -it --name ubuntu18.04-codesize -v /data:/data ubuntu:18.04
```

#### 容器内部操作

```shell
# 更新和安装必要的工具
apt update -y
apt install -y build-essential gcc g++ cmake vim qemu qemu-user qemu-user-static 
apt install -y git libglib2.0-dev libfdt-dev libpixman-1-dev zlib1g-dev ninja-build 
apt install -y gcc-riscv64-linux-gnu g++-riscv64-linux-gnu autoconf automake autotools-dev 
apt install -y curl libmpc-dev libmpfr-dev libgmp-dev gawk build-essential bison flex 
apt install -y texinfo gperf libtool patchutils bc gcc-riscv64-unknown-elf g++-riscv64-unknown-elf 
apt install -y zip unzip linux-tools-common linux-tools-generic gdb

# 安装 qemu
git clone https://github.com/qemu/qemu.git
cd qemu
git checkout v6.2.0  # 可以选择最新的稳定版本

./configure --target-list=riscv32-softmmu,riscv64-softmmu,riscv32-linux-user,riscv64-linux-user
make -j$(nproc)
make install

# 检验安装是否正确
qemu-riscv32 -version

# 克隆 riscv-gnu-toolchain 仓库
git clone https://github.com/riscv/riscv-gnu-toolchain 
cd riscv-gnu-toolchain

# 重新初始化和更新子模块（重要）
git submodule sync
git submodule update --init --recursive

# 编译并安装 32 位工具链
./configure --prefix=/opt/riscv --with-arch=rv32im --with-abi=ilp32
make -j$(nproc)

# 测试程序
echo '#include <stdio.h>' > hello.c
echo 'int main() { printf("Hello, RISC-V!\\n"); return 0; }' >> hello.c
riscv32-unknown-elf-gcc hello.c -o hello
qemu-riscv32 hello
```

### 开始编译 LLVM

```shell
# 克隆 llvm-project 仓库
git clone https://gitee.com/openeuler/llvm-project.git
cd llvm-project && mkdir build && cd build

# 安装 CMake 3.20+ 版本
apt-get install -y apt-transport-https ca-certificates gnupg software-properties-common wget
wget -O - https://apt.kitware.com/keys/kitware-archive-latest.asc | apt-key add -
apt-add-repository 'deb https://apt.kitware.com/ubuntu/ bionic main'
apt-get update
apt-get install -y cmake

# 编译 LLVM
cmake -S ../llvm -B . -G "Ninja" -DLLVM_ENABLE_PROJECTS="clang;lld" -DCMAKE_BUILD_TYPE=Debug -DLLVM_TARGETS_TO_BUILD="RISCV"
ninja -j$(nproc)

# 编译为工具链
mkdir -p /data/compiler/llvm
cmake -S ../llvm -B . -G "Ninja" -DLLVM_ENABLE_PROJECTS="clang;lld;bolt" -DLLVM_TARGETS_TO_BUILD="RISCV" -DCMAKE_INSTALL_PREFIX="/data/yangz/llvm" -DCMAKE_BUILD_TYPE=Debug
cmake --build . --target install -j$(nproc)
```

### 安装 RISC-V Clang 运行时库

```shell
# 创建目录
mkdir -p /data/yangz/llvm/bin/../lib/clang-runtimes/riscv32/lib
mkdir -p /data/yangz/llvm/bin/../lib/clang-runtimes/riscv32/include

# 解压运行时库包
tar -zxf riscv32-lld.tar.gz

# 复制文件
cd /data/yangz
cp -r riscv32-lld/include/* /data/yangz/llvm/bin/../lib/clang-runtimes/riscv32/include/
cp -r riscv32-lld/rv32imc_ilp32/* /data/yangz/llvm/bin/../lib/clang-runtimes/riscv32/lib/
```

### 测试运行

```shell
# 编写测试程序
cat > hello.c <<EOF
#include <stdio.h>

int main() {
    printf("Hello, RISC-V!\\n");
    return 0;
}
EOF

# 编译并运行测试程序
clang --target=riscv32 -Os -static hello.c -o hello /data/yangz/llvm/lib/clang-runtimes/riscv32/lib/crt0.o
qemu-riscv32 hello
# Hello, RISC-V!

# 测量 RISC-V 二进制文件大小
root@b91536c742ac:/data/yangz# size hello
   text    data     bss     dec     hex filename
   3730     156    1348    5234    1472 hello
```

#### 平时运行：

```shell
docker start ubuntu18.04-codesize
docker exec -it ubuntu18.04-codesize /bin/bash
```

#### 调试往届代码(中科大为例)

```shell
# 前置步骤
cd /data/yangz
git clone https://gitlab.eduxiji.net/educg-group-17293-1896550/202310358211617-3797.git
mv 202310358211617-3797 llvm-lastyear && cd llvm-lastyear
mkdir -p build && cd build
cmake -S ../llvm -B . -G "Ninja" \
			-DLLVM_ENABLE_PROJECTS="clang;lld;bolt" \
			-DLLVM_TARGETS_TO_BUILD="RISCV" \
			-DCMAKE_INSTALL_PREFIX="/data/yangz/llvm-last" \
			-DCMAKE_BUILD_TYPE=Debug
# 编译并安装工具链到/data/yangz/llvm-last(注意勿覆盖⚠️)
cmake --build . --target install -j$(nproc)
# 补充链接器库
cd /data/yangz
mkdir -p /data/yangz/llvm-last/bin/../lib/clang-runtimes/riscv32/lib
mkdir -p /data/yangz/llvm-last/bin/../lib/clang-runtimes/riscv32/include
# 复制链接器库到安装目录
cp -r riscv32-lld/include/* /data/yangz/llvm-last/bin/../lib/clang-runtimes/riscv32/include/
cp -r riscv32-lld/rv32imc_ilp32/* /data/yangz/llvm-last/bin/../lib/clang-runtimes/riscv32/lib/
##############################################################################################
# 遵循规则:端到端运行(毕昇17.0.6)已配置为bashrc
cd /data/yangz/test && clang --target=riscv32 -Os -static hello.c -o hello /data/yangz/llvm/lib/clang-runtimes/riscv32/lib/crt0.o
size hello
#   text    data     bss     dec     hex filename
#   3730     156    1348    5234    1472 hello
##############################################################################################
# 遵循规则:端到端运行(中科大-毕昇15.0.3)
cd /data/yangz/test
/data/yangz/llvm-last/bin/clang --target=riscv32 -Os -static hello.c -o hello /data/yangz/llvm-last/lib/clang-runtimes/riscv32/lib/crt0.o
size hello
#   text    data     bss     dec     hex filename
#   3448     156    1348    4952    1358 hello
# 显著观察到text段占用空间减小，评测以text段大小为准
##############################################################################################
# 遵循规则:调试链接器(中科大-毕昇15.0.3)
/data/yangz/llvm-last/bin/clang --target=riscv32 -Os -static hello.c -c -o hello.o /data/yangz/llvm-last/lib/clang-runtimes/riscv32/lib/crt0.o
# 获取链接过程命令
/data/yangz/llvm-last/bin/clang --target=riscv32 -v -Os -static hello.c -o hello /data/yangz/llvm-last/lib/clang-runtimes/riscv32/lib/crt0.o
# 得到: "/data/yangz/llvm-last/bin/ld.lld" /tmp/hello-973392.o /data/yangz/llvm-last/lib/clang-runtimes/riscv32/lib/crt0.o -Bstatic -L/data/yangz/llvm-last/bin/../lib/clang-runtimes/riscv32/lib -L/data/yangz/llvm-last/lib/clang/15.0.3/lib/baremetal -lc -lm -lclang_rt.builtins-riscv32 -o hello
# 参考如上命令将tmp文件替换为实际hello.o
gdb --args /data/yangz/llvm-last/bin/ld.lld hello.o \
									/data/yangz/llvm-last/lib/clang-runtimes/riscv32/lib/crt0.o \
									-Bstatic -L/data/yangz/llvm-last/bin/../lib/clang-runtimes/riscv32/lib \
									-L/data/yangz/llvm-last/lib/clang/15.0.3/lib/baremetal \
									-lc -lm -lclang_rt.builtins-riscv32 -o hello
(gdb) break riscv32_optimize
(gdb) run
# Thread 1 "ld.lld" hit Breakpoint 1, riscv32_optimize (filename=0x7ffda57138ba "hello") at /data/yangz/llvm-lastyear/lld/ELF/riscv32_opt/optimize.c:12
# 12      void riscv32_optimize(const char *filename) { # 自此开始调试学习

# 使用自己的llvm-project
/data/yangz/llvm-project/build/bin/clang --target=riscv32 -Os -static /data/yangz/test/hello.c -o hello /data/yangz/llvm-project/build/lib/clang-runtimes/riscv32/lib/crt0.o
```

#### 其他命令

```shell
# 删除容器
docker rm <容器名字>/<容器id>

# 删除镜像
docker rmi <镜像名字>/<镜像id>

# 查看所有镜像
docker images

# 查看所有容器
docker ps -a

# 停止容器
docker stop <容器名字>/<容器id>

# 查看容器日志
docker logs <容器名字>/<容器id>

# 保存镜像
docker save -o <保存路径>/<镜像名字>.tar <镜像名字>

# 加载镜像
docker load -i <镜像路径>/<镜像名字>.tar

# 运行新容器并挂载目录
docker run -it --name <容器名字> -v /path/on/host:/path/in/container <镜像名字>
```

#### 测试结果

```shell
#!/bin/bash

# 定义一个函数来编译、运行和获取大小
build_run_size() {
    local project_dir=$1
    local compiler_path=$2
    local crt0_path=$3
    local output_file=$4
    local run_script=$5

    cd "$project_dir" || exit
    make clean
    make CC="$compiler_path" CFLAGS="--target=riscv32 -static -march=rv32im $crt0_path"
    size "$output_file"
    ./"$run_script"
    cd - || exit
}
# 原始
# 中科大
echo "Building and running tests for Origin"
build_run_size "/data/yangz/llvm-project-new/BM/bm1" "/home/yangz/llvm/build/bin/clang" "/data/yangz/llvm-last/lib/clang-runtimes/riscv32/lib/crt0.o" "bm1.out" "run_bm1.sh"
build_run_size "/data/yangz/llvm-project-new/BM/bm2" "/home/yangz/llvm/build/bin/clang" "/data/yangz/llvm-last/lib/clang-runtimes/riscv32/lib/crt0.o" "bm2.out" "run_bm2.sh"
build_run_size "/data/yangz/llvm-project-new/BM/bm3" "/home/yangz/llvm/build/bin/clang" "/data/yangz/llvm-last/lib/clang-runtimes/riscv32/lib/crt0.o" "qrduino" "run_bm3.sh"


# 中科大
echo "Building and running tests for USTC"
build_run_size "/data/yangz/llvm-project-new/BM/bm1" "/data/yangz/llvm-last/bin/clang" "/data/yangz/llvm-last/lib/clang-runtimes/riscv32/lib/crt0.o" "bm1.out" "run_bm1.sh"
build_run_size "/data/yangz/llvm-project-new/BM/bm2" "/data/yangz/llvm-last/bin/clang" "/data/yangz/llvm-last/lib/clang-runtimes/riscv32/lib/crt0.o" "bm2.out" "run_bm2.sh"
build_run_size "/data/yangz/llvm-project-new/BM/bm3" "/data/yangz/llvm-last/bin/clang" "/data/yangz/llvm-last/lib/clang-runtimes/riscv32/lib/crt0.o" "qrduino" "run_bm3.sh"

# 国防科大1
echo "Building and running tests for NUDT1"
build_run_size "/data/yangz/llvm-project-new/BM/bm1" "/data/yangz/nudt1_install/bin/clang" "/data/yangz/nudt1_install/lib/clang-runtimes/riscv32/lib/crt0.o" "bm1.out" "run_bm1.sh"
build_run_size "/data/yangz/llvm-project-new/BM/bm2" "/data/yangz/nudt1_install/bin/clang" "/data/yangz/nudt1_install/lib/clang-runtimes/riscv32/lib/crt0.o" "bm2.out" "run_bm2.sh"
build_run_size "/data/yangz/llvm-project-new/BM/bm3" "/data/yangz/nudt1_install/bin/clang" "/data/yangz/nudt1_install/lib/clang-runtimes/riscv32/lib/crt0.o" "qrduino" "run_bm3.sh"

# 国防科大2
echo "Building and running tests for NUDT2"
build_run_size "/data/yangz/llvm-project-new/BM/bm1" "/data/yangz/nudt2_install/bin/clang" "/data/yangz/nudt2_install/lib/clang-runtimes/riscv32/lib/crt0.o" "bm1.out" "run_bm1.sh"
build_run_size "/data/yangz/llvm-project-new/BM/bm2" "/data/yangz/nudt2_install/bin/clang" "/data/yangz/nudt2_install/lib/clang-runtimes/riscv32/lib/crt0.o" "bm2.out" "run_bm2.sh"
build_run_size "/data/yangz/llvm-project-new/BM/bm3" "/data/yangz/nudt2_install/bin/clang" "/data/yangz/nudt2_install/lib/clang-runtimes/riscv32/lib/crt0.o" "qrduino" "run_bm3.sh"

# 湖南大学
echo "Building and running tests for HNU"
build_run_size "/data/yangz/llvm-project-new/BM/bm1" "/data/yangz/hnu_install/bin/clang" "/data/yangz/nudt2_install/lib/clang-runtimes/riscv32/lib/crt0.o" "bm1.out" "run_bm1.sh"
build_run_size "/data/yangz/llvm-project-new/BM/bm2" "/data/yangz/hnu_install/bin/clang" "/data/yangz/nudt2_install/lib/clang-runtimes/riscv32/lib/crt0.o" "bm2.out" "run_bm2.sh"
build_run_size "/data/yangz/llvm-project-new/BM/bm3" "/data/yangz/hnu_install/bin/clang" "/data/yangz/nudt2_install/lib/clang-runtimes/riscv32/lib/crt0.o" "qrduino" "run_bm3.sh"
```

结果(text大小)

| 题目 | 中科大15.0.3 |      | 1国防科大15.0.3 |      | 2国防科大15.0.3 |      | 湖南大学15.0.3 |      | 原始17.0.6 |      | ai-system17.0.6(HN,NDT,opt,启用imc) |      |
| ---- | ------------ | ---- | --------------- | ---- | --------------- | ---- | -------------- | ---- | ---------- | ---- | ----------------------------------- | ---- |
| bm1  | 196439       | ×    | 99215           | √    | 220659          | √    | 123379         | √    | 224527     | √    | 148323/121419/97899/==85019==/74659 | √    |
| bm2  | 37583        | √    | 23183           | √    | 37799           | √    | 37799          | √    | 44887      | √    | 44887/30079/23187/==22987==         | √    |
| bm3  | 14160        | √    | 12450           | √    | 14194           | √    | 14252          | √    | 14878      | √    | 15108/13482/13298/==10770==         | √    |

```c++
 "/data/yangz/llvm-project-new/build/bin/ld.lld" bm2_1.o bm2_2.o bm2_3.o bm2_4.o /data/yangz/llvm-last/lib/clang-runtimes/riscv32/lib/crt0.o -Bstatic -L/data/yangz/llvm-project-new/build/bin/../lib/clang-runtimes/riscv32/lib -L/data/yangz/llvm-project-new/build/bin/../lib/clang-runtimes/riscv32/lib -L/data/yangz/llvm-project-new/build/lib/clang/17/lib/baremetal -lc -lm -lclang_rt.builtins-riscv32 -X -o a.out
```

#### GDB结合Qemu跨架构调试

```bash
# 启动服务端
qemu-system-riscv32 -nographic -machine sifive_e -kernel /data/yangz/BM/bm1/bm1.out -s -S
# 新建一个客户端
riscv32-unknown-elf-gdb /data/yangz/BM/bm1/bm1.out
(gdb) target remote localhost:1234
```

调研失败：
①https://stackoverflow.com/questions/55189463/how-to-debug-cross-compiled-qemu-program-with-gdb

②https://stackoverflow.com/questions/76455941/how-to-run-and-debug-a-simple-riscv32-bare-metal-assembly-compiled-into-elf-us

③https://blog.csdn.net/ctbinzi/article/details/134697407

④https://stackoverflow.com/questions/3082570/debugging-with-bochs-gdb-cannot-find-bounds-of-current-function

⑤https://stackoverflow.com/questions/8741493/why-i-do-get-cannot-find-bound-of-current-function-when-i-overwrite-the-ret-ad

⑥https://stackoverflow.com/questions/2420813/using-gdb-to-single-step-assembly-code-outside-specified-executable-causes-error
