# 依赖环境
## llvm(不推荐)
* https://apt.llvm.org/，下载llvm.sh脚本
* 使用如下命令安装18版本的llvm
```
./llvm.sh 18 all
```
* 安装完成后使用如下命令建立软连接
```
sudo ln -s /usr/bin/clang-18 /usr/bin/clang
```

## llvm
* apt-get install clang

## cmake
* apt-get install cmake

## ninja
* apt-get install ninja-build

## vcpkg
* 下载 git clone https://github.com/microsoft/vcpkg.git
* 进入vcpkg目录，执行./bootstrap-vcpkg.sh

## 构建
* 进入scram_ext cmake --preset=debug-linux
* 进入build目录，ninja -j2
* ninja install
