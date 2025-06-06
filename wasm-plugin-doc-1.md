离线使用wasm插件的方法（需要将wasm插件下载下来）：

测试过程中发现，拉取插件的.wasm文件存在两种情况：

1、直接通过`oras pull higress-registry.cn-hangzhou.cr.aliyuncs.com/plugins/custom-response:1.0.0 -o ./custom-response`，会将对应.wasm文件下载到`custom-response/bazel-bin/extensions/custom_response/custom_response.wasm`

2、通过`oras pull`时提示如下：
```shell
# oras pull higress-registry.cn-hangzhou.cr.aliyuncs.com/plugins/request-validation:1.0.0 -o ./request-validation
✓ Skipped     application/vnd.docker.image.rootfs.diff.tar.gzip                                     894/894 kB 100.00%     0s
└─ sha256:49eab777f61fa3071169473a2698d53305d4369f183e6265a64375feb91ec4ac
✓ Skipped     application/vnd.docker.container.image.v1+json                                        469/469  B 100.00%     0s
└─ sha256:f83693145de5c9a543ffad5d0ad44dc15d725eaa0b706ac8f71c0d1f3e75b108
✓ Pulled      application/vnd.docker.distribution.manifest.v2+json                                  526/526  B 100.00%     0s
└─ sha256:d3ce7b3b6e3e6c591c8633f20819d92cd43591c81069d409ddd0e356e16cf232
Skipped pulling layers without file name in "org.opencontainers.image.title"
Use 'oras copy higress-registry.cn-hangzhou.cr.aliyuncs.com/plugins/request-validation:1.0.0 --to-oci-layout <layout-dir>' to pull all layers.
```
说明该镜像不是oci标准，使用`docker mainfest inspect`命令查看到镜像的layers:
```shell
# docker manifest inspect  higress-registry.cn-hangzhou.cr.aliyuncs.com/plugins/request-validation:1.0.0
{
        "schemaVersion": 2,
        "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
        "config": {
                "mediaType": "application/vnd.docker.container.image.v1+json",
                "size": 469,
                "digest": "sha256:f83693145de5c9a543ffad5d0ad44dc15d725eaa0b706ac8f71c0d1f3e75b108"
        },
        "layers": [
                {
                        "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
                        "size": 915704,
                        "digest": "sha256:49eab777f61fa3071169473a2698d53305d4369f183e6265a64375feb91ec4ac"
                }
        ]
}

```
这种情况需要使用`oras copy`命令，将镜像拉到本地，通过上面的输出确定layer的压缩类型(若是tar.gzip，则使用`tar -zxvf`命令解压即可。若为tar,则使用`tar -xvf`命令解压即可)，使用对应命令解压，即可解析出.wasm文件。
```shell
# oras copy higress-registry.cn-hangzhou.cr.aliyuncs.com/plugins/request-validation:1.0.0 --to-oci-layout ./request-validation
✓ Copied  application/vnd.docker.container.image.v1+json                                            469/469  B 100.00%  254µs
  └─ sha256:f83693145de5c9a543ffad5d0ad44dc15d725eaa0b706ac8f71c0d1f3e75b108
✓ Copied  application/vnd.docker.image.rootfs.diff.tar.gzip                                         894/894 kB 100.00%   39ms
  └─ sha256:49eab777f61fa3071169473a2698d53305d4369f183e6265a64375feb91ec4ac
✓ Copied  application/vnd.docker.distribution.manifest.v2+json                                      526/526  B 100.00%     0s
  └─ sha256:d3ce7b3b6e3e6c591c8633f20819d92cd43591c81069d409ddd0e356e16cf232
Copied [registry] higress-registry.cn-hangzhou.cr.aliyuncs.com/plugins/request-validation:1.0.0 => [oci-layout] ./request-validation
Digest: sha256:d3ce7b3b6e3e6c591c8633f20819d92cd43591c81069d409ddd0e356e16cf232
```
通过上面的`docker manifest inspect`命令，可以查看到镜像的压缩类型为tar.gzip，主层为`49eab777f61fa3071169473a2698d53305d4369f183e6265a64375feb91ec4ac`，找到该层：
```shell
# ll request-validation/blobs/sha256/
total 904
-r--r--r-- 1 root root 915704 Feb 28 10:11 49eab777f61fa3071169473a2698d53305d4369f183e6265a64375feb91ec4ac
-r--r--r-- 1 root root    526 Feb 28 10:11 d3ce7b3b6e3e6c591c8633f20819d92cd43591c81069d409ddd0e356e16cf232
-r--r--r-- 1 root root    469 Feb 28 10:11 f83693145de5c9a543ffad5d0ad44dc15d725eaa0b706ac8f71c0d1f3e75b108
```
通过`tar -zxvf`命令，可解析出.wasm文件(为了不污染当前目录，可以新建一个文件夹来存放解压内容):
```shell
# mkdir request-validation/wasm
# tar -xzf request-validation/blobs/sha256/49eab777f61fa3071169473a2698d53305d4369f183e6265a64375feb91ec4ac -C request-validation/wasm
# ll request-validation/wasm
total 2704
-rwxr-xr-x 1 root root 2767529 Aug 25  2024 plugin.wasm
```
将上述wasm文件打成镜像,Dockerfile内容如下：
```shell
FROM scratch as output
COPY plugin.wasm plugin.wasm
```
在Dockerfile所在目录下执行`docker build -t higress-registry.cn-hangzhou.cr.aliyuncs.com/plugins/custom-response:1.0.0 .`，即可生成镜像。

通过`docker save higress-registry.cn-hangzhou.cr.aliyuncs.com/plugins/custom-response:1.0.0 > custom-response.tar`将镜像保存为文件