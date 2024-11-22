---
title: "如何灵活地在 slurm 集群中启动并行任务，以 vllm server 为例"
date: 2024-11-21T00:00:00+08:00
toc: false
images:
tags:
  - slurm
  - sbatch
  - srun
  - caddy
  - vllm
  - gpt
  - 反向代理
---

## 简介

本文介绍如何在 slurm 集群中启动并行的任务，并以 vllm server 为例．这里例子具体地说，是要在集群中使用多个节点启动 vllm server 服务，提供 openai 兼容的 API 接口，然后使用高并发的 HTTP 请求去处理数据．

## 简单介绍一下 slurm 中的几个概念

先约定一下一些概念，slurm 中我们一般会写一个 sbatch 脚本，然后用 sbatch 命令提交作业（job）去排队，等待集群有满足的资源的时候自动执行，执行完毕就自动退出并释放资源．
在 sbatch 脚本中，我们一般会申请多个节点的资源，并使用 srun 命令将任务（或者称作作业步 job step）分配到各个节点去执行．这里的作业步就是可以并行运行的．

## vllm server

这里不对 vllm 本身做太多深入的介绍，只需要知道 vllm 是一个用于启动 openai 兼容接口的后端服务的工具．但是 vllm 要在集群上工作，仍有一些特别需要注意的地方．

1. vllm 的张量并行可以把一个模型的不同部分放到不同的显卡中去执行，适合一个显卡的显存不足矣支持一个模型运行的时候使用．当然，即使一张显卡足矣塞进整个模型，我们也依然可以用张量并行来使用多个显卡．但是张量并行一般仅限单个节点，且会要求模型的某些中间层的尺寸能够被显卡数量和分块大小给整除，否则无法使用．
2. vllm 还有一个流水线并行的特性，一般会使用 ray 来支持多显卡，但是会不支持异步生成，性能会降低．更多关于 vllm 分布式推理的内容可以参考[官方文档](https://docs.vllm.ai/en/latest/serving/distributed_serving.html)．
3. vllm 的流水线并行一般是用于多节点场景的．但是在单节点也是可以用的，并且张量并行和流水线并行是可以同时使用的．但是问题还是异步生成无法工作，性能有所降低．
4. `--enable-chunked-prefill` 和 `--enable-prefix-caching` 在实际使用的时候可能会遇到内存越界的错误，如果实际测试不能用，建议不要启用这两个选项．见 [issues#9918](https://github.com/vllm-project/vllm/issues/9918)．
5. `--guided-decoding-backend=outlines` 在使用共享的存储的时候会出错，而 slurm 一般都是用的共享存储．建议设置为 `--guided-decoding-backend=lm-format-enforcer`．

## 具体做法

具体的思路是这样子的：

1. 使用 srun 命令在每个节点的每个显卡启动一个 vllm server，并记录这些节点的主机名和使用的端口．
2. 等待一定的时间，预估 vllm server 服务启动成功之后，检查一下这些进程是否正常．
3. 根据记录的主机名和端口，使用 caddy 启动一个反向代理服务器，作为对外提供服务的统一入口，并进行简单的负载均衡．
4. 启动请求 openai 接口的脚本，进行数据处理．

sbatch 脚本大致如下：

```bash
#!/bin/bash
# 计费账号
#SBATCH --account account-name
# 任务名称
#SBATCH --job-name myscript
# 申请的分区
#SBATCH --partition partition-name
# 使用预留的资源
#SBATCH --reservation=reservation_name
# 申请的节点数量
# 多节点并行就修改这个值大于 1
#SBATCH --nodes=1
# 每一个节点申请的显卡数量
#SBATCH --gpus-per-node=8
# 每个 GPU 搭配申请的 CPU 数量
#SBATCH --cpus-per-gpu=32
# 输出日志文件
#SBATCH --output log.jobid.%j.txt

# 激活 conda 环境
module load cuda12.1/toolkit/12.1.1
module load gcc-10.3.0/10.3.0
source /path/to/anaconda3/etc/profile.d/conda.sh
conda activate vllm
# 环境变量设置
export OMP_NUM_THREADS=$SLURM_CPUS_ON_NODE
# 脚本每卡并发数量
# 请根据使用的 GPU 型号来设置
# 这里推荐 A100 为 32 或者 64
num_worker_per_gpu=32
# 我们使用这个脚本来生成对应的 vllm 命令，并用 srun 分发到每个节点去启动 vllm server
# 同时记录了对应的服务端口，再用 caddy 启动反向代理服务
# 最后启动脚本去执行请求
# 请求全部处理完毕停止掉 vllm 和 caddy 等进程，关闭日志文件，最后退出
python start.py --model-name your-model-name --model-path model-path --concurrency-per-gpu $num_worker_per_gpu --eval-script-file request.py --sleep 180
```

**注意：**sbatch 的参数设置建议就采用上面提供的，特别是不要指定 task 相关的选项，如 `--ntasks`, `--ntasks-per-gpu`, `--ntasks-per-node` 等．

`request.py` 脚本应该是根据你自己的需要来写，一般使用 `concurrent.futures.ThreadPoolExecutor` 或者 asyncio 实现并发请求，差距都不太大．

`start.py` 文件是一个用来启动 vllm server，caddy 反向代理服务和 `request.py` 的脚本．大致的内容如下：

```python
#!/usr/bin/env python3
#
# 启动脚本
# 基本思路，使用 srun 命令分配任务到每个计算节点的 GPU 上，每个 GPU 一个 vllm server 服务
# 然后使用 caddy 反向代理提供统一的请求入口
# 睡眠一段时间，等待 vllm server 就绪
# 然后启动脚本发起请求
# 最后脚本执行完毕退出
#
import argparse
import os
import random
import socket
import subprocess
import time
import traceback

from loguru import logger


def is_port_free(host: str, port: int) -> bool:
    """
    检查指定的端口是否未被占用．

    :param host: 主机地址
    :param port: 端口号
    :return: 如果端口未被占用，返回True；否则返回False
    """
    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as sock:
        return sock.connect_ex((host, port)) != 0


def get_random_ports(
    n: int, host: str = "localhost", start: int = 1024, end: int = 65535
):
    """
    获取n个随机的不被占用的端口．

    :param n: 需要获取的端口数量
    :param host: 主机地址，默认为'localhost'
    :param start: 端口范围的起始值，默认1024
    :param end: 端口范围的结束值，默认65535
    :return: 一个包含n个随机端口的列表
    """
    ports = []
    while len(ports) < n:
        port = random.randint(start, end)
        if is_port_free(host, port):
            ports.append(port)
    return ports


def get_args():
    parser = argparse.ArgumentParser()
    parser.add_argument("--model-name", type=str, required=True, help="model name")
    parser.add_argument("--model-path", type=str, required=True, help="model path")
    parser.add_argument(
        "--sleep",
        type=int,
        default=180,
        help="等待若干秒直到vllm server就绪，默认值：%(default)s",
    )
    parser.add_argument(
        "--concurrency-per-gpu",
        type=int,
        help="每卡并发数，与--concurrency互斥，只需要指定其中一个",
    )
    # 下面是 request.py 额外需要的参数，请根据需要写
    parser.add_argument(
        "--script-file", type=str, required=True, help="需要启动的脚本，比如request.py"
    )

    return parser.parse_args()


if __name__ == "__main__":
    args = get_args()
    logger.info(args)

    # 获取一些 slurm 相关的环境变量的值
    slurm_job_id = os.environ["SLURM_JOB_ID"]
    gpus_per_node = int(os.environ["SLURM_GPUS_PER_NODE"])
    cpus_per_gpu = int(os.environ["SLURM_CPUS_PER_GPU"])
    slurm_job_nodelist = os.environ["SLURM_JOB_NODELIST"]
    logger.info(f"节点列表SLURM_JOB_NODELIST：{slurm_job_nodelist}")
    cmd = ["scontrol", "show", "hostnames", f"{slurm_job_nodelist}"]
    result = subprocess.run(cmd, capture_output=True, text=True, check=True)
    if result.returncode:
        raise RuntimeError(f"使用命令{cmd}获取节点列表失败：{result.stderr}")
    else:
        nodes = result.stdout.splitlines()
    logger.info(f"使用的节点为：{nodes}")

    # 写个循环，遍历所有的节点和GPU，每个GPU提交一个任务
    # 保存打开的文件对象，方便最后关闭文件
    fps = []
    # 保存子进程对象，方便最后关闭子进程
    processes = []
    # caddy 反向代理需要的参数
    caddy_reverse_proxy_conifgs = []
    # 套一个 try，在抛出异常之后好收尾
    try:
        for node in nodes:
            ports = get_random_ports(gpus_per_node, host=f"{node}")
            for idx in range(gpus_per_node):
                # 使用 srun 命令来给每个节点的每个显卡发送对应的命令去启动任务
                port = ports[idx]
                vllm_cmd = [
                    "vllm",
                    "serve",
                    f"{args.model_path}",
                    "--host",
                    "0.0.0.0",
                    "--port",
                    f"{port}",
                    "--gpu-memory-utilization",
                    "0.9",
                    "--served-model-name",
                    f"{args.model_name}",
                    "--seed",
                    "42",
                    "--guided-decoding-backend",
                    "lm-format-enforcer",
                    "--disable-log-requests",
                ]
                # 注意：
                # 1. 这里要使用 --overlap 选项，否则不会成功启动并行的 vllm server
                # 2. 这里指定要使用的节点数量为 1，即 --nodes=1
                # 3. 这里要指定要使用的节点，即 --nodelist 选项
                # 4. 这里指定启动的任务数量为 1，即 --tasks 1
                # 5. 这里指定要使用的 gpu 数量，即 --gpus-per-node。这样就保证启动的任务会分配到一个 GPU，不需要我们自己设置环境变量 CUDA_VISIBLE_DEVICES
                # 6. --cpus-per-gpu 还是继承 sbatch 的设置即可
                cmd = [
                    "srun",
                    "--label",
                    "--overlap",
                    "--gpus-per-node",
                    "1",
                    "--cpus-per-gpu",
                    f"{cpus_per_gpu}",
                    "--nodes",
                    "1",
                    "--tasks",
                    "1",
                    "--nodelist",
                    f"{node}",
                ] + vllm_cmd

                logger.info(f"使用命令：{cmd}启动vllm server")
                log_filename = (
                    f"log.vllm.server.{slurm_job_id}.{node}.{port}.gpu{idx}.txt"
                )
                f = open(log_filename, "w")
                fps.append(f)
                # 使用 subprocess.Popen 启动子进程，不会阻塞
                proc = subprocess.Popen(
                    cmd,
                    stdout=f,
                    stderr=subprocess.STDOUT,
                    close_fds=True,
                )
                processes.append(proc)
                caddy_reverse_proxy_conifgs.append(
                    {"proc": proc, "host": node, "port": port}
                )
        logger.info(f"睡眠{args.sleep}秒，等待vllm server启动就绪")
        time.sleep(args.sleep)
        # 检查 vllm server 是否正常启动了，启动成功的才加入 caddy 的反向代理列表
        caddy_reverse_proxy_args = []
        for item in caddy_reverse_proxy_conifgs:
            if item["proc"].poll() is None:
                caddy_reverse_proxy_args.append("--to")
                host = item["host"]
                caddy_reverse_proxy_args.append(f'http://{item["host"]}:{item["port"]}')
            else:
                logger.warning(f"子进程：{proc}启动的vllm失败")
        if len(caddy_reverse_proxy_args) == 0:
            raise RuntimeError(f"没有一个成功启动的vllm server．")
        # 启动 caddy 反向代理
        proxy_port = get_random_ports(1)[0]
        hostname = socket.gethostname()
        cmd = [
            "caddy",
            "reverse-proxy",
            "--access-log",
            "--from",
            f"http://{hostname}:{proxy_port}",
        ] + caddy_reverse_proxy_args
        log_filename = (
            f"log.{slurm_job_id}.caddy.reverse.proxy.{hostname}.port{proxy_port}.txt"
        )
        f = open(log_filename, "w")
        fps.append(f)
        logger.info(f"使用命令：{cmd}启动caddy反向代理")
        # 同样，启动子进程服务，这里也不能阻塞
        proc = subprocess.Popen(cmd, stdout=f, stderr=subprocess.STDOUT, close_fds=True)
        processes.append(proc)
        # 启动评测脚本
        # 按每卡并发设置的时候，实际卡数量为启动成功的 vllm server 数量
        # 不出意外应该是这个
        # max_workers = args.concurrency_per_gpu * gpus_per_node * len(nodes)
        # 不过这里用这个更加稳妥
        num_gpus = len(caddy_reverse_proxy_args) / 2
        max_workers = int(args.concurrency_per_gpu * num_gpus)
        url = f"http://{hostname}:{proxy_port}/v1"
        # 这里启动request.py脚本，额外的参数根据需要添加
        # 这里指定了常用的model_name, url, max_workers等
        cmd = [
            "python",
            f"{args.script_file}",
            "--model_name",
            f"{args.model_name}",
            "--url",
            f"{url}",
            "--max_workers",
            f"{max_workers}",
        ]
        logger.info(f"使用命令：{cmd}启动脚本")
        # 这里使用 subprocess.run 启动，需要阻塞，等待子进程运行结束退出
        # 我们的作业也就可以正常退出
        subprocess.run(cmd)
    except Exception as e:
        traceback.print_exception(e)
        traceback.print_exc()
    finally:
        logger.info(f"收尾工作")
        for proc in processes:
            logger.info(f"停止子进程：{proc}，并等待5秒")
            proc.terminate()
            proc.wait(5)
            try:
                proc.wait(5)
            except subprocess.TimeoutExpired:
                logger.info(f"超时5s，强制杀死子进程：{proc}")
                proc.kill()
                proc.wait(5)
        for f in fps:
            if not f.closed:
                logger.info(f"关闭文件{f}")
                f.close()
```

最后，使用 sbatch 命令提交作业即可．

## 总结

本文介绍了如何使用 srun 把任务分发到各个节点去运行，并以启动 vllm server，提供 openai 兼容的 API，使用 HTTP 请求进行数据处理作为例子，进行详细的介绍．
