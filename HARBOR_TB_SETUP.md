# 本机 Harbor：TB2 / TB Pro 环境与代理配置

配置日期：2026-11-04。

本文记录本次已经完成的配置、复现命令、运行入口和实际验证结果。
Harbor 和 Docker 沙盒都运行在当前机器，模型通过 HTTP 调用独立部署的 vLLM 服务。
不再使用 144 作为沙盒宿主机。

## 1. 当前部署和文件位置

| 项目 | 实际配置 |
| --- | --- |
| Harbor 源码 | `/ceph_home/zhangkaiqi2024/harbor` |
| Conda 环境 | `harbor_tb` |
| 环境目录 | `/ceph_home/zhangkaiqi2024/miniconda3/envs/harbor_tb` |
| Python | 3.13.18 |
| Harbor | 当前仓库源码，包版本 0.25.0，可编辑安装 |
| Agent | `terminus-2` |
| Sandbox | 当前机器的 Docker，默认 context 为 `default` |
| 本机 Docker / Compose / Buildx | 23.0.3 / v2.17.2 / v0.10.4 |
| Docker 数据目录 / 默认 runtime | `/var/lib/docker` / `runc` |
| 正式评测输出 | 仓库下 `jobs/` |

Terminus-2 的控制器运行在宿主机 Harbor 进程中，调用 vLLM；模型生成的终端操作在任务容器中执行。
`--env docker` 选择 Docker 后端，具体机器由 Harbor 的运行位置和 Docker context/daemon 决定。

```text
harbor/
├── HARBOR_TB_SETUP.md                 # 本文
├── datasets/
│   ├── terminal-bench/                # TB2，89 个任务
│   └── terminal-bench-pro/            # Pro 公开集，200 个任务
├── .harbor/tb/
│   ├── proxy.env                     # 宿主机代理环境变量
│   ├── docker/config.json            # 评测专用 Docker 客户端配置
│   ├── compose-proxy.yaml            # Dockerfile 构建代理 overlay
│   ├── tb2.json                      # TB2 job 配置
│   ├── tbpro.json                    # Pro job 配置
│   ├── run.sh                        # 加载代理并启动评测
│   ├── pull-image.sh                 # 下载并缓存指定镜像
│   ├── tools/crane                   # Crane 0.22.1
│   ├── requirements.lock.txt         # 从 uv.lock 导出的基础依赖
│   ├── installed-packages.txt        # 实际 pip freeze
│   ├── datasets.json                 # 数据集版本、commit 和解析结果
│   ├── images.json                   # 任务引用的镜像列表
│   ├── tbpro-image-tasks.json         # Pro 基础镜像与任务的对应关系
│   ├── validation.json               # 环境和参考解验证摘要
│   ├── smoke-jobs/                   # 验证 job 的完整产物
│   └── logs/                         # 下载、构建和检查日志
└── jobs/
```

`.harbor/`、`datasets/` 和 `jobs/` 使用仓库已有的 Git 忽略规则。
本文可以单独纳入版本管理；本机配置与下载产物仍保留在本机。

## 2. 现在如何进入环境

```bash
source /ceph_home/zhangkaiqi2024/miniconda3/etc/profile.d/conda.sh
conda activate harbor_tb
cd /ceph_home/zhangkaiqi2024/harbor

which python
which harbor
python -m pip check
```

激活时 Conda 自动设置本次保存的代理变量，以及评测专用的 `DOCKER_CONFIG`。
环境没有与 Miles 的 Python 或虚拟环境共用。

运行使用 `harbor` 或本文的 `run.sh`。直接执行 `uv run harbor` 默认会管理仓库的 `.venv`，
不能据此保证使用 `harbor_tb`。

## 3. 环境是如何安装的

以下是首次安装步骤，当前机器已经完成，不需要再次创建同名环境。

```bash
source /ceph_home/zhangkaiqi2024/miniconda3/etc/profile.d/conda.sh
conda create -n harbor_tb python=3.13 pip -y \
  --override-channels -c conda-forge
conda activate harbor_tb

cd /ceph_home/zhangkaiqi2024/harbor
python -m pip install 'uv>=0.13.5'

uv export --locked --no-dev --no-emit-project \
  --format requirements-txt \
  --output-file .harbor/tb/requirements.lock.txt

uv pip install --python "$CONDA_PREFIX/bin/python" \
  --require-hashes -r .harbor/tb/requirements.lock.txt

uv pip install --python "$CONDA_PREFIX/bin/python" --no-deps -e .

uv pip install --python "$CONDA_PREFIX/bin/python" \
  socksio PySocks ruff==0.15.17 ty==0.0.49

python -m pip check
```

本次解析得到 Python 3.13.18、uv 0.13.7。基础运行依赖来自仓库锁文件，
没有安装所有云平台、训练后端和开发可选依赖。
`socksio` 和 `PySocks` 分别补充 HTTPX 和 Requests 使用 SOCKS 代理时需要的依赖。

## 4. 数据集是如何下载的

使用仓库自带的 `registry.json`，将数据固定到注册表记录的版本：

```bash
harbor datasets download terminal-bench@2.0 \
  --registry-path registry.json --output-dir datasets

harbor datasets download terminal-bench-pro@1.0 \
  --registry-path registry.json --output-dir datasets
```

| 数据集 | 本地目录 | 任务数 | 上游 commit |
| --- | --- | ---: | --- |
| TB2 2.0 | `datasets/terminal-bench` | 89 | `69671fbaac6d67a7ef0dfec016cc38a64ef7a77c` |
| TB Pro 1.0 公开集 | `datasets/terminal-bench-pro` | 200 | `82bc46de337f48d57a5d78f4900ce7eabc485101` |

289 个任务均通过 Harbor 的任务对象解析。
这里下载的是任务指令、环境定义、测试和参考解等文件；没有预拉全部任务镜像。
正式 job 配置直接读取上述本地目录，不需要每次重新解析远端数据集。

## 5. 代理需要分别覆盖哪些位置

Harbor 没有一个能覆盖所有网络阶段的统一代理开关，使用的是以下现有配置入口：

| 网络阶段 | 本次采用的入口 | 配置文件 |
| --- | --- | --- |
| Harbor 进程、模型客户端、Git、Python 下载 | 宿主机环境变量 | Conda 环境变量、`.harbor/tb/proxy.env` |
| Docker daemon 拉取镜像 | Docker systemd 服务环境 | 机器原有 `http-proxy.conf` |
| Dockerfile 的 `RUN` 命令 | `environment.extra_docker_compose` / `--extra-docker-compose` | `.harbor/tb/compose-proxy.yaml` |
| 模型操作的任务容器、生命周期命令 | `environment.env` | `.harbor/tb/tb2.json`、`tbpro.json` |
| verifier 执行命令 | `verifier.env` | 同上 |
| Docker 客户端代理默认值 | `DOCKER_CONFIG` 下的 `proxies.default` | `.harbor/tb/docker/config.json` |

只在宿主机 `export` 代理，不会自动将所有变量传给任务容器或 verifier。
Docker 客户端配置也不等于 Docker daemon 的代理配置。

### 5.1 宿主机进程代理

`.harbor/tb/proxy.env` 保存如下变量：

```bash
export http_proxy=http://124.16.138.150:7890
export https_proxy=http://124.16.138.150:7890
export all_proxy=socks5://124.16.138.150:7890
export HTTP_PROXY=http://124.16.138.150:7890
export HTTPS_PROXY=http://124.16.138.150:7890
export ALL_PROXY=socks5://124.16.138.150:7890
export NO_PROXY_LIST=localhost,127.0.0.1,127.0.1.1,::1,0.0.0.0,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16,169.254.0.0/16,.local,.lan,.internal,.cluster.local,kubernetes.default.svc
export no_proxy="$NO_PROXY_LIST"
export NO_PROXY="$NO_PROXY_LIST"
export DOCKER_CONFIG=/ceph_home/zhangkaiqi2024/harbor/.harbor/tb/docker
```

这些值也持久化在 Conda 环境中。需要重新保存时，在加载该文件后执行：

```bash
source .harbor/tb/proxy.env
conda env config vars set -n harbor_tb \
  "http_proxy=$http_proxy" "https_proxy=$https_proxy" "all_proxy=$all_proxy" \
  "HTTP_PROXY=$HTTP_PROXY" "HTTPS_PROXY=$HTTPS_PROXY" "ALL_PROXY=$ALL_PROXY" \
  "NO_PROXY_LIST=$NO_PROXY_LIST" "no_proxy=$no_proxy" "NO_PROXY=$NO_PROXY" \
  "DOCKER_CONFIG=$DOCKER_CONFIG"
```

`run.sh` 每次启动也会加载此文件，并将命令行 `--base-url` 的主机追加到宿主机进程的
`no_proxy` / `NO_PROXY`，让 vLLM 请求直接到达模型服务。

### 5.2 Docker daemon 拉镜像的代理

当前机器原本已经存在：

```text
/etc/systemd/system/docker.service.d/http-proxy.conf
```

本次读取到的内容是：

```ini
[Service]
Environment="HTTP_PROXY=http://124.16.138.150:7890"
Environment="HTTPS_PROXY=http://124.16.138.150:7890"
Environment="NO_PROXY=localhost,127.0.0.1"
```

本次复用了这份配置，没有修改 systemd unit、Docker daemon 配置或重启 Docker。
daemon 原有的部分镜像站响应较慢，原生 pull 可能先等待镜像站再回退。

### 5.3 Docker 客户端的独立配置

`.harbor/tb/docker/config.json` 的内容是：

```json
{
  "proxies": {
    "default": {
      "httpProxy": "http://124.16.138.150:7890",
      "httpsProxy": "http://124.16.138.150:7890",
      "allProxy": "socks5://124.16.138.150:7890",
      "noProxy": "localhost,127.0.0.1,127.0.1.1,::1,0.0.0.0,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16,169.254.0.0/16,.local,.lan,.internal,.cluster.local,kubernetes.default.svc"
    }
  }
}
```

`DOCKER_CONFIG` 将本次评测指向这个目录，不修改用户原来的 `~/.docker/config.json`。
Harbor 的构建和运行代理还通过下面两层显式传递，不单独依赖客户端自动继承。

### 5.4 镜像构建阶段的代理

`compose-proxy.yaml` 为 `main` 服务补充 `build.args`，等价 YAML 如下：

```yaml
services:
  main:
    build:
      args:
        http_proxy: ${http_proxy}
        https_proxy: ${https_proxy}
        all_proxy: ${all_proxy}
        HTTP_PROXY: ${HTTP_PROXY}
        HTTPS_PROXY: ${HTTPS_PROXY}
        ALL_PROXY: ${ALL_PROXY}
        no_proxy: ${no_proxy}
        NO_PROXY: ${NO_PROXY}
```

两份 job JSON 都引用了它：

```json
{
  "environment": {
    "type": "docker",
    "extra_docker_compose": [
      "/ceph_home/zhangkaiqi2024/harbor/.harbor/tb/compose-proxy.yaml"
    ]
  }
}
```

也可以通过 CLI 的 `--extra-docker-compose /path/to/overlay.yaml` 传入。
这层用于 Dockerfile 的 `RUN apt-get`、`RUN pip install` 等命令，
不负责让 Docker daemon 拉取 `FROM` 镜像。

### 5.5 任务容器和 verifier 的代理

`tb2.json` / `tbpro.json` 在 `environment.env` 和 `verifier.env` 中都设置了：

- 小写 `http_proxy`、`https_proxy`、`all_proxy`、`no_proxy`。
- 大写 `HTTP_PROXY`、`HTTPS_PROXY`、`ALL_PROXY`、`NO_PROXY`。

使用的是具体代理值，完整内容见这两份实际配置文件。
这样模型执行下载命令、安装依赖，以及 verifier 运行测试时都能取得代理变量。

实测此版本不能在运行级 `environment.env` 中用
`"https_proxy": "${https_proxy}"` 这样的同名自引用替代具体值：
该值会覆盖 Compose 子进程的同名变量，构建工具最终尝试解析字面量 `${https_proxy}`。
当前配置已经改为具体值，真实 Pro 构建和验证通过。

### 5.6 镜像下载备用入口

```bash
bash .harbor/tb/pull-image.sh alexgshaw/fix-git:20251031
```

脚本依次执行：

1. 检查本地镜像，存在则直接复用。
2. 使用 Crane 0.22.1，通过当前进程代理从原始 registry 下载 `linux/amd64` 镜像，再 `docker load`。
3. 失败后，对普通 Docker Hub 标签尝试 DaoCloud，并 tag 回原始引用。
4. 再失败则使用原生 `docker pull`。

DaoCloud 对部分镜像实行白名单限制，不能保证任意 TB 镜像都可下载。
Crane 路径不依赖 daemon 的镜像站设置。辅助脚本只在显式调用时运行，
不会自动拦截 Harbor 内部的每一次拉取。

## 6. 如何启动自己的 vLLM 评测

实际 vLLM 地址、served-model-name、鉴权设置和上下文长度尚未提供。
下面是需要替换这些值的运行示例：

```bash
source /ceph_home/zhangkaiqi2024/miniconda3/etc/profile.d/conda.sh
conda activate harbor_tb
cd /ceph_home/zhangkaiqi2024/harbor

export EVAL_VLLM_API_KEY=dummy

bash .harbor/tb/run.sh tb2 \
  --model openai/my-model \
  --base-url http://VLLM_IP:8000/v1 \
  --api-format openai_chat_completions \
  --api-key-name EVAL_VLLM_API_KEY \
  --ak 'model_info={"max_input_tokens":32768,"max_output_tokens":4096,"input_cost_per_token":0.0,"output_cost_per_token":0.0}' \
  --n-concurrent 1 \
  --n-tasks 1
```

- `my-model` 必须对应 vLLM 的实际模型服务名；启用鉴权时替换 `dummy`。
- token 限制为示例，按实际模型和 vLLM 上下文配置填写。
- 默认 Terminus-2 使用 JSON parser；需要 Miles 训练时的 XML 交互格式，可增加 `--ak parser_name=xml`。
- `run.sh` 透传后续 Harbor CLI 参数，并选择 `tb2.json` 或 `tbpro.json`。
- 改为 `tbpro` 可选择 Pro；完整 Pro 目前还受第 8 节的镜像问题阻碍。
- 小规模验证后，去掉 `--n-tasks 1`，按资源和模型吞吐调整并发。配置文件默认并发为 4、每任务 1 次尝试。

查看正式运行结果：

```bash
harbor view ./jobs
```

## 7. 本次实际验证结果

| 检查 | 结果 |
| --- | --- |
| Conda 中导入 Harbor | 指向当前仓库源码，版本 0.25.0 |
| `python -m pip check` | 通过 |
| 数据集任务解析 | TB2 89/89，Pro 200/200 |
| Docker 容器内经代理访问 GitHub | HTTP 200 |
| Compose 构建阶段经代理访问 GitHub | HTTP 200 |
| 原生 Docker 拉取 TB2 `fix-git` 镜像 | 成功 |
| Crane 下载并导入 `busybox:1.37.0` | 成功 |
| TB2 `fix-git`，官方参考解 oracle | 2 个测试通过，reward 1 |
| Pro `python-prime-http-server`，官方参考解 oracle | 94 个测试通过，reward 1 |

最终验证产物：

```text
.harbor/tb/smoke-jobs/tb2-fix-git-oracle-fixed/
.harbor/tb/smoke-jobs/tbpro-prime-oracle-fixed/
.harbor/tb/validation.json
```

这些是环境和参考解检查，不是自建模型的评测成绩。
早期 hello-world 检查出现过 Ubuntu 源 502 和 verifier 超时，失败日志也保留在本地。

按仓库要求执行了 Ruff 检查、格式化检查和 ty 检查。
Ruff 检查通过，1646 个文件格式未变；全仓 ty 报告 256 项诊断，
包含当前未安装的可选依赖等问题，没有报告全仓类型检查通过。
日志位于 `.harbor/tb/logs/`。本次配置没有修改框架源码或下载的任务定义。

## 8. TB Pro 完整运行的现有阻碍

固定版本的 200 个公开任务中，197 个任务使用以下两个基础镜像：

| 基础镜像 | 任务数 |
| --- | ---: |
| `skylensage-registry.cn-hangzhou.cr.aliyuncs.com/instances/1124:t-bench-python-3-13` | 129 |
| `skylensage-registry.cn-hangzhou.cr.aliyuncs.com/instances/1124:t-bench-ubuntu-24-04` | 68 |

当前机器直连该 registry 超时，经指定的 HTTP 或 SOCKS 代理访问也出现 EOF/502。
这次检查没有确定它是服务状态、网络路径还是权限要求所致。

任务文件已完整下载，但完整 Pro 评测还需要可访问的原始镜像副本、可用网络路径，
或维护方提供的访问方式。没有将这些镜像替换成普通 Python/Ubuntu 镜像。
受影响任务清单见 `.harbor/tb/tbpro-image-tasks.json`。

## 9. 后续修改配置的位置

| 要修改的内容 | 修改位置 |
| --- | --- |
| 模型地址、模型名、密钥来源、token 限制 | 启动命令中的相应参数 |
| 并发、尝试次数、数据集和结果目录 | `tb2.json` / `tbpro.json`，或 CLI 覆盖 |
| 宿主机代理 | `proxy.env`，以及 Conda 保存的环境变量 |
| 任务与 verifier 代理 | `tb2.json` / `tbpro.json` 的两个 `env` 字段 |
| Docker 客户端代理 | `docker/config.json` |
| 构建代理变量的传递方式 | `compose-proxy.yaml` |

代理地址变化时，需要同步更新宿主机、两份 job JSON 和 Docker 客户端配置。
`run.sh` 只会自动更新宿主机进程对指定模型主机的 NO_PROXY，
不会自动重写 job JSON 或系统 Docker daemon 配置。
