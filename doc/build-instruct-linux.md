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
* 进入vcpkg目录下的versions目录，找到versions/b-/boost.json文件
* 查看boost.json文件的commit log，找到[boost] update to 1.79.0 (#24210) 类型的提交，其对应的commit ID就可以作为vcpkg-configuration.json的baseline使用

## 构建
* 进入scram_ext cmake --preset=debug-linux
* 进入build目录，ninja -j2
* ninja install
