---
title: "使用NCCL作为PyTorch多节点分布式训练后端时候的经验总结"
date: 2024-06-29T00:00:00+08:00
toc: false
images:
tags:
  - DDP
  - InfiniBand
  - NCCL
  - PyTorch
  - RDMA
  - RoCE
  - accelerate
  - slurm
  - 分布式训练
---

## 闲言

最近开始做 LLM/GPT 了，单机多卡训练只能勉强满足需求，能够利用上多机多卡，配合上 InifinBand，会很有大的提升的．这中间的测试来来回回也踩了不少坑，这里做一些简单的记录．

## 对多机多卡训练的极简介绍

这里简单介绍一下如何启动多机多卡训练．

### 使用 torchrun

这个是 PyTorch 推荐使用的新方式，支持动态增减节点，自动给节点分配 rank 等．比传统的方式更加灵活且简洁．直接使用 `torchrun` 命令等价于 `python -m torch.distributed.run`．

```bash
torchrun \
    --nnodes $SLURM_NNODES \
    --nproc_per_node $SLURM_GPUS_PER_NODE \
    --rdzv_backend c10d \
    --rdzv_endpoint $MASTER_ADDR:$MASTER_PORT \
    --rdzv_id $SLURM_JOB_ID \
    train.py args...
```

参数解释：

1. `--nnodes`：并行的机器数量．
2. `--nproc-per-node`：每个节点上的任务数量，一个 GPU 一个任务，实际上也就是每个节点上的 GPU 数量．
3. `--rdzv_backend`：rdzv 后端，一般用 c10d．
4. `--rdzv_endpoint`：master 节点的 rdzv 服务地址．分布式训练的每个计算节点会通过这个地址进行通信，并且根据 `rdzv_id` 来标识他们属于同一组作业．
5. `--rdzv_id`：`rdzv_id` 用于标识一个唯一的分布式训练，你可以在相同的机器上启动多个分布式训练实验，只要他们的 ID 不同．所以一般在 SLURM 集群上，我们会使用 SLURM_JOB_ID 作为这个 ID．

### accelerate

```bash
accelerate launch \
    --dynamo_backend no \
    --machine_rank $SLURM_NODEID \
    --main_process_ip $MASTER_ADDR \
    --main_process_port $MASTER_PORT \
    --mixed_precision no \
    --multi_gpu \
    --num_machines $SLURM_NNODES \
    --num_processes $num_processes \
    --rdzv_backend c10d \
    train.py args...
```

参数解释：

1. `dynamo_backend`：我们不需要这个，指定为 `no` 只是不想看到 warning．
2. `--machine_rank`：节点的 rank，一般设置为环境变量 `SLURM_NODEID` 即可．
3. `--main_process_ip` 和 `--main_process_port`：主节点的地址和端口，组成 `IP:PORT` 作为 `rdzv_endpoint`．
4. `--mixed_precision`：混合精度，根据需要设置即可．
5. `--multi_gpu`：启用多卡并行训练．
6. `--num_machines`：并行训练的节点数量．
7. `--num_processes`：accelerate 这里要的是总的任务数，每个 GPU 算一个，也就是总的 GPU 数量．所以这里对应到 torchrun 的方式，应该是 `SLURM_GPUS_PER_NODE x SLURM_NNODES`
8. `--rdzv_backend`：这个与前面 torchrun 命令的选项一样．经过测试，仅设置为 `c10d` 是成功的，且不要设置 `--same_network`．
9. `--config_file`： accelerate 的选项可以通过 `accelerate config` 交互式生成，保存到配置文件中，使用的时候此选项指定．但是命令行选项应该优先级更高．你可以把通用的部分写到配置文件，而其他的在命令行指定．

### 多节点启动

如果不使用别的工具，一般我们就需要分别在每一台机器上执行上面的命令．但是大多数时候我们都是在集群环境下使用的，这个时候可以使用集群工具来自动帮我们完成这个操作．比较常用工具的是 slurm，也有部分集群使用 PBS，两者区别不大．

## 一个快速简单的例子

以 PyTorch 官方提供了一个 minGPT-ddp 例子，这个例子足够小，可以在 RTX 2080 显卡上跑起来．完整的源码在 [pytorch-examples](https://github.com/hubutui/pytorch-examples)．将其 clone 下来之后，修改 `distributed/minGPT-ddp/mingpt/slurm/job.sh` 文件里的设置，然后使用 `sbatch job.sh` 命令提交即可．这里只需要根据具体使用的 slurm 集群的情况，修改其中的计费账户 account 和使用的分区 partition，应该就可以了．其他的脚本也可以参考此模板来创建 slurm sbatch 脚本．

我在一个 slurm 集群上的 2 个 A800 节点进行了测试，完整的测试命令见 [run.sh](https://github.com/hubutui/pytorch-examples/blob/main/distributed/minGPT-ddp/mingpt/slurm/run.sh)，输出日志见 [logs](https://github.com/hubutui/pytorch-examples/tree/main/distributed/minGPT-ddp/mingpt/slurm/logs) 下的文件．

即使不使用 slurm 集群，上面的脚本本身也是有效的 bash 脚本，简单修改一下，也可以直接执行：

1. 修改 `SLURM` 开头的环境变量，指定为合适的值．
2. 删掉 `srun --label`，即直接执行对应的命令，而无需通过 srun 启动．

### 补充

就个人的实际测试而言，这个并行训练也不是每次都成功，可能会遇到训练无法启动，就是卡住了．我不确定这个问题是不是我申请资源的时候每个节点值申请了 1 个 GPU，节点上还有其他用户的任务，导致影响到了我的训练，还是说这个集群的网络配置存在一些什么问题．不过好在单机多卡训练一般都是没有问题的，且上面提到的 slurm sbatch 脚本也都可以正常使用的，仅需要指定申请的节点数量为 1 即可．

## 关于 InfiniBand，RDMA，RoCE

我们这里不对这些概念做特别详细深入的介绍．只是单纯为了对比哪个更快更好，我们应该优先选哪个而已．

1. RDMA (remote direct memory access)，也就是远程直接内存访问，是一种绕过远程主机操作系统内核访问其内存中数据的技术．由于不经过 CPU、CPU 缓存或者上下文切换的参与，速度很快．
2. InfiniBand 即指他的物理设备（包括网卡、交换机），也指一种协议．这里我们更多的关心他的物理层面．一般地，计算机集群的计算节点会配备 IB 网卡，但是由于机房不一定．IB 提供了一种 RDMA 的实现．
3. 由于 IB 需要特定的交换机，更换就需要增加成本．一种解决方式是 RoCE (RDMA over Converged Ethernet)．简单的理解就是结合了 IB，但是用是以太网交换机，实现了 RDMA．

从速度和效率上来说：

1. 原生的 IB 是最优先考的，但是它需要 IB 交换机，多了一笔费用，部分机房可能没有配备．
2. RoCE 是其次的，他只需要用普通的以太网交换机．
3. 最次的就是完全没有 IB 设备，直接使用以太网网卡．

一般来说，NCCL 会自动选择合适的设备，但是如果出错，我们需要了解一下如何查看相关信息．

### 查看节点上 IB 设备

我们可以使用 `ibstatus` 命令查看 IB 设备的状态．例如：

```
Infiniband device 'mlx5_0' port 1 status:
        default gid:     fe80:0000:0000:0000:0ac0:ebff:fefe:428c
        base lid:        0x0
        sm lid:          0x0
        state:           4: ACTIVE
        phys state:      5: LinkUp
        rate:            100 Gb/sec (4X EDR)
        link_layer:      Ethernet

Infiniband device 'mlx5_1' port 1 status:
        default gid:     fe80:0000:0000:0000:946d:ae03:00fc:e51c
        base lid:        0x9
        sm lid:          0x6
        state:           4: ACTIVE
        phys state:      5: LinkUp
        rate:            200 Gb/sec (4X HDR)
        link_layer:      InfiniBand

Infiniband device 'mlx5_2' port 1 status:
        default gid:     fe80:0000:0000:0000:0ac0:ebff:fefe:547c
        base lid:        0x0
        sm lid:          0x0
        state:           1: DOWN
        phys state:      3: Disabled
        rate:            100 Gb/sec (4X EDR)
        link_layer:      Ethernet

Infiniband device 'mlx5_3' port 1 status:
        default gid:     fe80:0000:0000:0000:946d:ae03:00fc:e548
        base lid:        0x8
        sm lid:          0x6
        state:           4: ACTIVE
        phys state:      5: LinkUp
        rate:            200 Gb/sec (4X HDR)
        link_layer:      InfiniBand
```

这里一般只关注状态为 `ACTIVE` 的设备．可以看到 `mlx5_0` 的 `link_layer` 为 `Ethernet`，而 `mlx5_1` 的 `link_layer` 为 `InfiniBand`，且 `mlx5_1` 的速率更高．这里的设备可以用于设置环境变量 `NCCL_IB_HCA`．

简单的理解：

1. `link_layer` 为 `Ethernet`，说明这个 IB 设备的链路层协议为 Ethernet，一般就是使用了 RoCE 协议．
2. `link_layer` 为 `InfiniBand`，说明这个 IB 设备的链路层协议为 InfiniBand，也就是原生的 InfiniBand．

再看 `ibdev2netdev` 的输出：

```
mlx5_0 port 1 ==> ens13np0 (Up)
mlx5_1 port 1 ==> ib0 (Up)
mlx5_2 port 1 ==> ens17np0 (Down)
mlx5_3 port 1 ==> ib1 (Up)
```

可以看到这几个 IB 设备映射到了网络设备．

## 使用 NCCL 作为 PyTorch 分布式训练后端的时候，IB 设备的使用情况分析

下面我们测试一下各个情况的组合．这里我们确认了要申请的 2 个节点的 IB 设备名字都是前面输出的内容，也就是两个节点一致．我们测试的组合包括：

1. `NCCL_IB_DISABLE` 设置为 0 或者 1
2. `NCCL_IB_HCA` 设置为不同的设备．
3. `NCCL_SOCKET_IFNAME` 设置为不同的设备．

正如上文提到的，本次测试使用的 slurm sbatch 脚本见 [job.sh](https://github.com/hubutui/pytorch-examples/blob/main/distributed/minGPT-ddp/mingpt/slurm/job.sh)，实际执行命令见 [run.sh](https://github.com/hubutui/pytorch-examples/blob/main/distributed/minGPT-ddp/mingpt/slurm/run.sh)，输出日志见 [slurm](https://github.com/hubutui/pytorch-examples/blob/main/distributed/minGPT-ddp/mingpt/slurm/logs) 下的文件．

### 测试结果

1. 当 `NCCL_IB_DISABLE=0` 的时候，`NCCL_IB_HCA` 设置的值如果不是 `rdma link` 显示的 IB 设备，则 NCCL 会提示找不到 IB 设备，然后回落到 NET/Socket，识别到可用的网络设备，并且实际使用的是 ib0（见日志中的 `[send] via NET/Socket/1` 和 `NCCL INFO Using` 给出的设备列表）．
2. 当 `NCCL_IB_DISABLE=0` 的时候，`NCCL_IB_HCA` 设置的值是 `rdma link` 显示的 IB 设备，则 NCCL 会按照环境变量的设置去使用该设备，但是如果该设备状态为 Down，则还是会自动回落到 NET/Socket．
3. 当 `NCCL_IB_DISABLE=0`，其他的环境变量不指定的时候，NCCL 自动识别到可用的 IB 设备，并自动选择使用了速率最高的原生 IB．
4. 当 `NCCL_IB_DISABLE=1` 的时候，NCCL 将使用 `NCCL_SOCKET_IFNAME` 指定的设备，如果这设备不可用，则会直接报错．
5. 当 `NCCL_IB_DISABLE=1`，其他的环境变量不指定的时候，NCCL 自动识别到可用的设备 ib0 和 ib1，并自动选择了 ib0．

## 关于 NCCL 环境变量的小结

我们来小结一下 NCCL 环境变量的设置．

1. 需要查看一些详细的调试信息的话，可以设置 `NCCL_DEBUG=INFO`．
2. 确认需要使用 IB 设备和 IB 协议的话，首先要确认 `NCCL_IB_DISABLE` 不能设置为 `1`，然后设置 `NCCL_IB_HCA` 为 `rdma link` 输出的的设备名字，且优先选择 `ibstatus` 里显示的 link layer 为 InfiniBand 的设备，其次是 link layer 为 Ethernet 的设备．
3. 如果不用 IB 设备，则需要设置 `NCCL_IB_DISABLE=1`，然后设置 `NCCL_SOCKET_IFNAME` 为 `ibdev2netdev` 显示的 IB 映射到的以太网设备名字．
4. 最后，如果根本没有 IB，则只能设置 `NCCL_IB_DISABLE=1`，并设置 `NCCL_SOCKET_IFNAME` 为 `ip link show` 显示的以太网设备．

## 使用 LLaMa Factory 训练 LLM

这里其实没有什么好说的了，直接参看上文提到的 slurm sbatch 脚本即可，只需修改对应的 python 脚本和对应的选项参数即可．简单快捷的说明，大概改成：

```bash
torchrun \
    --nnodes $SLURM_NNODES \
    --nproc_per_node $SLURM_GPUS_PER_NODE \
    --rdzv_backend c10d \
    --rdzv_endpoint $MASTER_ADDR:$MASTER_PORT \
    --rdzv_id $SLURM_JOB_ID \
    llama_factory_dir/src/train.py args...
```

旧版本的 llama factory 的 `train.py` 脚本为 `train_bash.py`，请根据情况修改．另外，新版本的 llama factory 建议使用 `llamafactory-cli` 命令来启动训练，具体的多节点训练命令如下：

```bash
FORCE_TORCHRUN=1 NNODES=2 RANK=0 MASTER_ADDR=192.168.0.1 MASTER_PORT=29500 llamafactory-cli train examples/train_lora/llama3_lora_sft.yaml
FORCE_TORCHRUN=1 NNODES=2 RANK=1 MASTER_ADDR=192.168.0.1 MASTER_PORT=29500 llamafactory-cli train examples/train_lora/llama3_lora_sft.yaml
```

其中的 yaml 文件仅仅是把 `train.py` 的选项参数写到配置文件而已．详细见[此处](https://github.com/hiyouga/LLaMA-Factory/tree/v0.9.0/examples#supervised-fine-tuning-on-multiple-nodes)．个人实际测试多卡训练并不能启动，会一直卡住，分析原因在于 llama factory 没有使用 rdzv，详见此处[代码](https://github.com/hiyouga/LLaMA-Factory/blob/90d6df622252c6fad985f68b97771c979357e2fc/src/llamafactory/cli.py#L94-L108)．只需要修改此处的代码为使用 rdzv 的形式即可．

## 参考

以下是本文的一些重要参考资料来源，有兴趣的可以看看．

1. [sbatch doc](https://slurm.schedmd.com/sbatch.html)：需要用到的 SLURM 相关的环境变量这里都有．
2. [NCCL doc](https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/env.html)：NCCL 调试相关的环境变量可以参考这里．
3. [Fine-tuning Llama 2 70B using PyTorch FSDP](https://huggingface.co/blog/ram-efficient-pytorch-fsdp)：这里提供了一个使用 accelerate 训练的参考，他的 slurm sbatch 脚本也可以参考．
4. [如何理解 Nvidia 英伟达的 Multi-GPU 多卡通信框架 NCCL](https://www.zhihu.com/answer/3487108775)：这个知乎回答也值得看看．
