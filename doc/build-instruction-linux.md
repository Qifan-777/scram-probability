## vcpkg
* 下载 git clone https://github.com/microsoft/vcpkg.git
* 进入vcpkg目录，执行./bootstrap-vcpkg.sh

## 构建
* 进入scram_ext cmake --preset=debug-linux
* 进入build目录，ninja -j2
* ninja install
