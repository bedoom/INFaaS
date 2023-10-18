# # 前言

1. 创造一个IAM角色，管理该模型
2. 创造一个安全组，仅允许信任IP访问
3. 创造一个INFaaS模型仓库，推荐保持私有
4. 创造一个模型配置桶，当模型注册后，该桶包含已分析可用配置。INFaaS在注册一个模型时，需要进入该桶。

# Genearl Setup

1. 创造一个虚拟机实例作为主节点 虚拟机配置：8vCPU，32GiB
2. 下载 INfaaS 仓库：`git clone https://github.com/stanford-mast/INFaaS.git`.
3. 打开 `start_infaas.sh` ，配置启动脚本（由于本地部署，因此需要重写）。
4. 运行 `./start_infaas.sh`，这会启动所有 INFaaS 组件并初始化 workers，同时也会运行一些基础测试来检测是否运行成功。如果某个worker启动失败，你可以重新运行脚本。所有的可执行文件都可以在 `build/bin`中找到。

# Model Registration

目前，用户必须分析他们的模型并通过 `infaas_modelregistration` 生成一个配置文件，操作步骤如下：

* 找到 `src/profiler`文件夹
* 运行 `./profile_modle.sh <frozen-model-path> <accuracy>` `<dataset> <task> [cpus]`
  * 这个脚本是交互式的，并提示你分析模型所需要的信息，最后会生成一个配置文件（xxx.config）
  * 上传这个配置文件到配置桶里，AWS做法：`aws s3 cp mymodel.config s3://your-config-bucket/mymodel.config`
* 将 .config文件作为第二个参数传递给 `infaas_modelregistraion`

### Example

在 `infaas-sample-public`中，我们在单个 Inferentia 核心上提供了一个 CPU TensorFlow 模型、一个针对批次 4 优化的等效 TensorRT 模型以及一个针对批次 1 优化的等效 Inferentia 模型。我们也提供了他们的生成配置文件，你需要做的只是：

```
./infaas_modelregistration resnet_v1_50_4.config infaas-sample-public/resnet_v1_50_4/
./infaas_modelregistration resnet50_tensorflow-cpu_4.config infaas-sample-public/resnet50_tensorflow-cpu_4/
./infaas_modelregistration resnet50_inferentia_1_1.config infaas-sample-public/resnet50_inferentia_1_1/
```

当出现SUCCEEDED时，INFaaS成功启动。

# 推理查询

已注册模型信息：

* 对于一个给定的任务和数据集，查看可用模型架构，使用 `infaas_modarch`
* 获取有关模型架构的模型变体的更多信息，使用 `infaas_modinfo`

## Example

对于上面完成注册的例子，查看已注册模型，使用 `./infaas_modarch classification imagenet`。这时你会看到一个模型架构：*resnet50。*

运行 `./infaas_modinfo resnet50`，你会看到3个不同的模型变体：*resnet_v1_50_4, resnet50_tensorflow-cpu_4, resnet_inferentia_1_1。*

运行请求：

* 在线请求：`infaas_online_query`
* 离线请求：`infaas_offline_query`。INFaaS返回是否工作调度成功，如果成功，你可以在 `out_url`中监视工作。

**Example**

注意：运行这个例子，必须至少有一个GPU worker和一个Inferentia worker.

我们在线给INFaaS发送一个图片分类请求并指定模型架构和时延：

`./infaas_online_query -d 224 -i ../../data/mug_224.jpg -a resnet50 -l 300`

第一次你运行这个请求，延迟以秒为单位，这是由于模型需要加载。如果你重新运行这个请求，它应该完成地更快。INFaaS使用resnet50_tensorflow-cpu_4服务这个请求。

现在我们将指定时延降低：

`./infaas_online_query -d 224 -i ../../data/mug_224.jpg -a resnet50 -l 50`

这时，INFaaS使用resnet50_inferentia_1_1_4服务这个请求。

最后，我们发送一个batch-2的query并指定一个非常低的延迟：

`./infaas_online_query -d 224 -i ../../data/mug_224.jpg -i ../../data/mug_224.jpg -a resnet50 -l 20`

GPU模型会花很长的时间，因此可能初次使用时延会大于10s，如果你再重新请求，他会在毫秒内完成。INFaaS使用resnet_v1_50_4完成工作。

你也可以简单地使用INFaaS，通过时延和精度：

`./infaas_online_query -d 224 -i ../../data/mug_224.jpg -t classification -D imagenet -A 70 -l 50`

# Clean Up

在 `shutdown_infaas`中升级下面两个参数：

```
REGION='<REGION>'
ZONE = '<ZONE>'
```

然后运行 `./shutdown_infaas.sh`
