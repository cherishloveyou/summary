



**iOS启动流程**
APP启动主要分为两个阶段：pre-main和main之后，而APP的启动优化也主要是在这两个阶段进行的。
main之后的优化：1. 减少不必要的任务，2.必要的任务[延迟执行](https://so.csdn.net/so/search?q=延迟执行&spm=1001.2101.3001.7020)，例如放在控制器界面等等。

APP启动的大致过程：
**APP启动 -> 加载libSystem -> Runtime注册回调函数 -> 加载image(镜像文件) -> 执行map_images和load_images方法 -> 调用main函数。**

查看pre-main耗时，添加DYLD_PRINT_STATISTICS到（Edit Scheme -> Run -> Arguments -> Environment Variables）就可以在控制台看到耗时