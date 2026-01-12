### 查看 cuda 版本
  - 使用 `nvidia-smi` 指令可以查看当前系统安装的 GPU 驱动版本：
    ```
    root@A100-01:~# nvidia-smi
    Tue Jan 13 00:09:03 2026       
    +-----------------------------------------------------------------------------------------+
    | NVIDIA-SMI 580.105.08             Driver Version: 580.105.08     CUDA Version: 13.0     |
    +-----------------------------------------+------------------------+----------------------+
    ```
    示例: `Driver Version` `580.105.08` 为当前安装的驱动版本，驱动只能同时安装一个版本。`CUDA Version` `13.0` 为当前驱动版本支持的最大 cuda 版本，并支持向下兼容，可以同时安装多个 cuda 版本，`13.0` 并不代表当前安装的 cuda 版本

  - 查看当前系统默认 cuda 版本
    ```bash
    echo $CUDA_HOME
    ```
    会有类似如下显示
    
    `/usr/local/cuda-13.1`

    或者

    `/usr/local/cuda`

    若只显示 `/usr/local/cuda`，可运行
    ```bash
    ll /usr/local/cuda
    ```
    会有类似如下显示

    `lrwxrwxrwx 1 root root 21 Nov 24 12:36 /usr/local/cuda -> /usr/local/cuda-13.1//`

    则代表当前首选 cuda 版本为 `13.1`

  - 查看当前系统所有已安装 cuda 版本
    ```bash
    ll /usr/local/ | grep cuda
    ```
    可以看到类似如下输出
    
    `drwxr-xr-x 17 root root 4096 Jan 12 23:14 cuda-12.4/`
    
    `drwxr-xr-x 17 root root 4096 Nov 24 12:37 cuda-13.0/`

    `...`

    所有列出版本均已安装并可单独使用
    

### conda 更改 cuda 版本
  在 conda 的环境中可以单独配置该环境使用的 cuda 版本
  - `conda activate <env_name>` 激活要更改的环境后，创建激活脚本目录
    ```bash
    mkdir -p $CONDA_PREFIX/etc/conda/activate.d
    mkdir -p $CONDA_PREFIX/etc/conda/deactivate.d
    ```

  - 创建环境启动覆盖脚本
    ```bash
    nano $CONDA_PREFIX/etc/conda/activate.d/set_cuda.sh
    ```
    写入如下内容：
    ```text
    #!/bin/sh
    export OLD_CUDA_HOME=$CUDA_HOME
    export CUDA_HOME=/usr/local/cuda-12.4  # 指向需要的 cuda 版本，以 12.4 为例
    export PATH=$CUDA_HOME/bin:$PATH
    export LD_LIBRARY_PATH=$CUDA_HOME/lib64:$LD_LIBRARY_PATH
    ```

  - 创建环境退出覆盖脚本
    ```bash
    nano $CONDA_PREFIX/etc/conda/deactivate.d/unset_cuda.sh
    ```
    写入如下内容：
    ```text
    #!/bin/sh
    export CUDA_HOME=$OLD_CUDA_HOME
    ```

  - 在一个新的 bash 窗口中激活环境或者重新激活环境即可生效
    ```bash
    conda deactivate
    conda activate your_env_name
    ```
    在激活环境的 bash 中查看验证新的 cuda 版本
    ```bash
    echo $CUDA_HOME
    ```
    输出版本应为配置的新版本 `/usr/local/cuda-12.4`
