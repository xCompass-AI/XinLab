# Podman 容器
Podman 是一个容器化工具，和 docker 的功能一致且更完善，其指令功能以及使用方法和 docker 完全相同，只需将 `docker` 指令替换为 `podman`

- (可选) 别名伪装，bash 中输入 `docker` 指向 `podman`，方便使用
  ```bash
  nano ~/.bashrc
  ```
  在 .bashrc 文件后面追加内容：
  ```bash
  alias docker=podman
  ```

### 使用示例
- 拉取镜像。拉取方式与 docker 相同，需要注意的是 Podman 默认不会只在 Docker Hub 找镜像，它会询问你。建议带上完整的域名。
  ```bash
  podman pull docker.io/library/ubuntu
  ```

- 运行容器，运行 ubuntu 镜像并启动容器内 bash
  ```bash
  podman run -it --name <test-name> docker.io/library/ubuntu /bin/bash
  ```

- 查看所有已创建容器
  ```bash
  podman ps -a
  ```

- 停止与删除
  ```bash
  podman stop <test-name>
  podman rm <test-name>
  ```

### 容器内使用 GPU
容器内使用 GPU 需要手动将 nvidia 设备文件挂载入容器内（nvidia 基础设备 `/dev/nvidiactl`、`/dev/nvidia-uvm`、`/dev/nvidia-uvm-tools`，以及 nvidia 设备节点 `/dev/nvidia0`、`/dev/nvidia1`、...），并设置 `NVIDIA_VISIBLE_DEVICES` 环境变量，下面为具体示例

- 容器内使用所有 GPU，基于 nvidia/cuda:12.2.0-base 镜像创建容器并进入容器内 bash 交互界面
  ```bash
  # --rm 参数建议调试时使用，可以在容器意外退出时自动清理，长期运行不需要添加
  podman run -it \
    --rm \
    --device /dev/nvidia0 \
    --device /dev/nvidia1 \
    --device /dev/nvidia2 \
    --device /dev/nvidia3 \
    --device /dev/nvidia4 \
    --device /dev/nvidia5 \
    --device /dev/nvidia6 \
    --device /dev/nvidia7 \
    --device /dev/nvidiactl \
    --device /dev/nvidia-uvm \
    --device /dev/nvidia-uvm-tools \
    nvidia/cuda:12.2.0-base /bin/bash
  ```

- 容器内使用部分 GPU，以 GPU0 和 GPU2 为例，基于 nvidia/cuda:12.2.0-base 镜像创建容器并进入容器内 bash
  ```bash
  # --rm 参数建议调试时使用，可以在容器意外退出时自动清理，长期运行不需要添加
  # -e NVIDIA_VISIBLE_DEVICES=0,2 在使用部分 GPU 时必须要设置
  podman run -it \
    --rm \
    --device /dev/nvidia0 \
    --device /dev/nvidia2 \
    --device /dev/nvidiactl \
    --device /dev/nvidia-uvm \
    --device /dev/nvidia-uvm-tools \
    -e NVIDIA_VISIBLE_DEVICES=0,2 \
    nvidia/cuda:12.2.0-base /bin/bash
  ```

- 使用 bash 脚本启动，示例如下

  创建脚本
    ```bash
    nano podman-gpu-run.sh
    ```

  写入如下内容并保存
    ```bash
    #!/usr/bin/env bash
    set -euo pipefail
  
    # IMGAE 变量设置镜像名
    IMAGE=${IMAGE:-"nvidia/cuda:12.2.0-base"}
  
    # CMD 变量设置启动容器后执行的指令
    CMD=${CMD:-"/bin/bash"}
    # CMD=${CMD:-"nvidia-smi"}
    
    GPU_REQ=${1:-all}
  
    BASE_DEVICES=(
      /dev/nvidiactl
      /dev/nvidia-uvm
      /dev/nvidia-uvm-tools
    )
  
    mapfile -t GPU_NODES < <(ls /dev/nvidia[0-9]* 2>/dev/null || true)
    
    if [ ${#GPU_NODES[@]} -eq 0 ]; then
      echo "ERROR: No /dev/nvidiaX devices found"
      exit 1
    fi
  
    DEVICE_ARGS=()
    for dev in "${BASE_DEVICES[@]}" "${GPU_NODES[@]}"; do
      DEVICE_ARGS+=(--device "$dev")
    done
    
    if [ "$GPU_REQ" = "all" ]; then
      NV_VISIBLE="all"
    else
      NV_VISIBLE="$GPU_REQ"
    fi
    
    echo "[INFO] GPUs requested: $NV_VISIBLE"
    echo "[INFO] Image: $IMAGE"
    echo "[INFO] Command: $CMD"
    
    exec podman run -it \
      --rm \
      "${DEVICE_ARGS[@]}" \
      -e NVIDIA_VISIBLE_DEVICES="$NV_VISIBLE" \
      "$IMAGE" $CMD
    ```

  为脚本添加执行权限
    ```bash
    chmod +x podman-gpu-run.sh
    ```

  使用方式：
    ```bash
    ./podman-gpu-run.sh all # 指定使用全部 GPU
    ./podman-gpu-run.sh 0,2 # 指定使用 0 和 2 GPU
    ./podman-gpu-run.sh GPU-xxxx,GPU-yyyy # 使用 GPU PCIe 地址指定（不推荐）
    ```

### Podman 配置系统级共享镜像池
  Podman 默认镜像池路径存放在个人 home 目录下 `~/.local/share/containers/storage`，可添加系统级共享镜像池方便多用户共享镜像
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
  也可自行设置为其他可用路径

- 使用共享镜像池拉取脚本，直接在 bash 中使用 share-image 指令
  ```bash
  # sudo 运行该指令不会触发报错
  sudo share-image <image name>
  # 例如
  sudo share-image docker.io/library/nvidia-cuda-image
  ```

# Rootless docker (不推荐)
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

# 所有用户容器 uid 映射表
  若容器启用 user namespace，则按照下表的规则与宿主机的 uid 命名空间进行映射，若未启用 user namespace，则和宿主机共享同一个命名空间，共享同样的文件权限
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
  以 wuhuanhuan:296608:65536 为例，容器内 uid=0 的用户为容器内 root 用户，其身份映射到外部宿主机时的 uid 为 296608，该用户可用的映射空间以 296608 为起点，空间大小为 65536，也就是 296608 ~ 296608+65536
