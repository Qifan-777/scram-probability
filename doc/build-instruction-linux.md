## vcpkg
* 以http方式下载内网的vcpkg repo(已提前从https://github.com/microsoft/vcpkg.git拉取)，由于scram_ext的build脚本已经指定了VCPKG_ROOT，所以需要将该repo放置在/root下
```
git clone http://jihu.devops.org/huchenpeng/smartree-engine-build-vcpkg
```
### 如何处理vcpkg的baseline
* vcpkg-configuration.json中的baseline对应的是boost 1.76版本
* 在vcpkg的repo中，可以查看这个文件的提交历史：versions/b-/boost.json；找到"[boost] update to 1.76.0 (#17335)"类似字段的提交记录，则可以将该提交的commit id作为baseline的id
* 通过这种方式就可以控制vcpkg内各种库的版本了

## 构建
* 进入scram_ext cmake --preset=debug-linux
* 进入build目录，ninja -j2
* ninja install
