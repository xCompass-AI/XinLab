## Podman (推荐)
Podman 已在系统中安装，指令功能和 docker 完全相同

使用示例
- 拉取镜像。Podman 默认不会只在 Docker Hub 找镜像，它会询问你。建议带上完整的域名。
  ```bash
  podman pull docker.io/library/ubuntu
  ```

- 运行容器
  ```bash
  podman run -it --name test_os docker.io/library/ubuntu /bin/bash
  ```

- 查看容器
  ```bash
  podman ps -a
  ```

- 停止与删除
  ```bash
  podman stop test_os
  podman rm test_os
  ```

- 使用 CDI 配置文件调用 GPU 示例，`/etc/cdi/nvidia.yaml`
  ```bash
  # 将所有 GPU 挂载入容器内
  podman run --rm --device nvidia.com/gpu=all ubuntu nvidia-smi
  ```
  ```bash
  # 将第一张和第二张 GPU 挂载入容器内
  podman run --rm \
    --device nvidia.com/gpu=0 \
    --device nvidia.com/gpu=1 \
    ubuntu nvidia-smi
  ```

- (可选) 别名伪装，bash 中输入 docker 指向 podman
  ```bash
  nano ~/.bashrc
  ```
  在 .bashrc 后面添加
  ```bash
  alias docker=podman
  ```

## Rootless docker (不推荐)
- 用户层面单独进行配置安装
  ```bash
  # 运行安装脚本（不需要 sudo）
  curl -fsSL https://get.docker.com/rootless | sh
  ```

- 安装完成后，终端会提示你设置环境变量。将其添加到该用户的 ~/.bashrc 中
  ```bash
  nano ~/.bashrc
  ```
  将下列内容添加到 .bashrc 文件
  ```bash
  export PATH=/home/testuser/bin:$PATH
  export DOCKER_HOST=unix:///run/user/$(id -u)/docker.sock
  ```

- 使用 CDI 配置文件调用 GPU 示例，`/etc/cdi/nvidia.yaml`
  ```bash
  # 将所有 GPU 挂载入容器内
  docker run --rm -it --device nvidia.com/gpu=all ubuntu nvidia-smi
  ```
  ```bash
  # 将第一张和第二张 GPU 挂载入容器内
  docker run --rm \
    --gpus '"device=0,1"' \
    ubuntu nvidia-smi
  ```

## Podman 配置系统级共享镜像池
  Podman 默认镜像池路径存放在个人 home 目录下 `~/.local/share/containers/storage`
- 用户层面单独添加共享仓库
  ```bash
  mkdir -p ~/.config/containers
  nano ~/.config/containers/storage.conf
  ```
  在 storage.conf 中添加
  ```TOML
  [storage]
  driver = "overlay"
    
  [storage.options]
  additionalimagestores = [
      "/var/lib/shared-containers/storage"
  ]
  ```

- 使用共享镜像池拉取脚本，直接在 bash 中使用 share-image 指令
  ```bash
  # sudo 运行该指令不会触发报错
  sudo share-image <image name>
  # 例如
  sudo share-image docker.io/library/nvidia-cuda-image
  ```

## 所有用户容器 uid 映射表
  ```
  wuhuanhuan:296608:65536
  wanghaoran:362144:65536
  maxusheng:427680:65536
  changshaole:493216:65536
  qixiaoning:558752:65536
  hujiaxin:624288:65536
  fangchen:689824:65536
  liuwenhao:755360:65536
  guoshuyu:820896:65536
  dingyali:886432:65536
  chenqicheng:951968:65536
  houting:1017504:65536
  huyanping:1083040:65536
  wangyunlong:1148576:65536
  renxianwen:1214112:65536
  ```
  以 wuhuanhuan:296608:65536 为例，容器内 uid=0 的用户身份映射到外部宿主机的 uid 为 296608，该用户可用的映射空间以 296608 为起点，空间大小为 65536，也就是 296608 ~ 296608+65536
