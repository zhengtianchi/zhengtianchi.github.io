# docker - Docker容器运行时引擎-runC分析




## Docker、containerd、runC、Kubernetes

### 从Docker到containerd、runC

Docker 是一个开源工具，它可以将你的应用打包成一个标准格式的镜像，并且以容器的方式运行。Docker 容器将一系列软件包装在一个完整的文件系统中（代码、运行时工具、系统工具、系统依赖…)，保证了容器内应用程序运行环境的稳定性，不会被容器外的系统环境所影响。

2013年Docker宣布开源，初始阶段的Docker容器化基于LXC实现，镜像分层文件系统基于AUFS，核心功能都是基于当时的现成技术组装实现，但由于其工具生态完善、公共镜像仓库、组建重用、多版本支持、以应用为中心的容器封装等特性，这也使得Docker火速地流行了起来。

> 可以参考Docker相关的FAQ：[https://docs.docker.com/engine/faq](https://docs.docker.com/engine/faq/#what-does-docker-technology-add-to-just-plain-lxc)

在2014年开源了基于Golang开发的libcontainer，越过LXC直接操作系统内核模块，不必依赖LXC来提供容器化隔离能力了，自身架构也不断发展拆分成了如下几个模块：

<img src="https://user-images.githubusercontent.com/19829495/118020412-a0d46700-b38c-11eb-8dad-3be7e8c0bb71.png" alt="DeepinScreenshot_select-area_20210513013836" style="zoom:200%;" />

2015年，Docker联合多家公司制定了开放容器交互标准（Open Container Initiative），即OCI；制定了关于容器相关的规范，包含运行时标准（[runtime-spec](https://github.com/opencontainers/runtime-spec/blob/master/spec.md)[ ](https://github.com/opencontainers/runtime-spec/blob/master/spec.md)）、容器镜像标准（[image-spec](https://github.com/opencontainers/image-spec/blob/master/spec.md)[ ](https://github.com/opencontainers/image-spec/blob/master/spec.md)）和镜像分发标准（[distribution-spec](https://github.com/opencontainers/distribution-spec/blob/master/spec.md)[ ](https://github.com/opencontainers/distribution-spec/blob/master/spec.md)）。

2016年，Docker为了符合OCI标准，将原本libcontainer模块独立出来，封装重构成[runC项目](https://github.com/opencontainers/runc)[ ](https://github.com/opencontainers/runc)，捐献给了Linux基金会管理，而后为了能够兼容所有符合标准的OCI Runtime实现，Docker重构了Docker Daemon子系统，将其中与运行时交互的部分抽象为[containerd项目](https://containerd.io/)[ ](https://containerd.io/)捐献给了CNCF基金会管理，内部为每个容器运行时创建一个containerd-shim适配进程，与runC搭配工作。

<img src="https://user-images.githubusercontent.com/19829495/118022236-b8145400-b38e-11eb-942c-57e68e255ae7.png" alt="图片" style="zoom:200%;" />

### Docker Daemon、containerd与runC

containerd提供了镜像管理（镜像、元信息等）、容器运行（调用最终运行时组件执行）的功能，向上为 Docker Daemon 提供了 gRPC 接口，使得 Docker Daemon 屏蔽下面的结构变化，确保原有接口向下兼容；向下运行容器由符合OCI规范的容器支持，如runC，使得引擎可以独立升级。

### Kubernetes与Docker

Kubernetes 是一个自动化部署，缩放，以及容器化管理应用程序的开源系统，前身是Google内部运行多年的集群管理系统Borg。

2014年Google使用Golang完全重写后开源，诞生后备受云计算相关的业界巨头追捧，成为云原生时代的操作系统，让复杂软件在云计算下获得韧性（Resilience）、弹性（Elasticity）、可观测性（Observability）的最佳路径。

2015年Kubernetes发布了第一个正式版本，Google宣布与Linux基金会共同筹建[云原生基金会](https://www.cncf.io/)[ ](https://www.cncf.io/)（Cloud Native Computing Foundation，CNCF），并且将Kubernetes托管到CNCF，成为其第一个项目。

随着Kubernetes的火速流行，自身的架构也在渐进演化，演进路线如下：

Kubernetes 1.5之前与Docker是强绑定的关系，Kubernetes 管理容器的方式都是通过内部的DockerManager向Docker Engine发送指令（HTTP方式），通过Docker来操作镜像；调用链路如下所示

```
    Kubernetes Master → kubelet → DockerManager → Docker Engine → containerd → runC
```

Kubernetes 1.5版本开始引入容器运行时接口（Container Runtime Interface，CRI），CRI为定义容器运行时应该如何接入到kubelet的规范标准，内部的DockerManager也被KubeGenericRuntimeManager替代，已提供更通用的实现；kubelet与KubeGenericRuntimeManager之间通过gRPC协议通信。由于Docker不能直接支持CRI，所以Kubernetes提供了DockerShim服务作为Docker与CRI的适配层，实现了DockerManager的功能，**演化到这里Docker Engine对Kubernetes来说只是一项默认依赖**；调用链路如下所示：

```
    Kubernetes Master → kubelet → KubeGenericRuntimeManager → DockerShim → Docker Engine → containerd → runC
```

2017年，由Google、RedHat、Intel、SUSE、IBM联合发起的[CRI-O](https://github.com/cri-o/cri-o)[ ](https://github.com/cri-o/cri-o)（Container Runtime Interface Orchestrator）项目发布了首个正式版本。CRI-O是完全遵循CRI规范进行实现的，而且可以支持所有符合OCI运行时标准的容器引擎，默认使用runC搭配工作也可换成其他OCI运行时。此时Kubernetes可以选择使用CRI-O、cri-containerd、DockerShim作为CRI实现，**演化到这里Docker Engine对Kubernetes来说只是一个可选项**；调用链路如下所示：

```
    Kubernetes Master → kubelet → KubeGenericRuntimeManager → CRI-O→ runC
```

2018年，由Docker捐献给CNCF的containerd发布了1.1版，与1.0的区别是完全支持CRI标准，意味着原本用作CRI适配器的cri-containerd不再需要了，Kubernetes也在1.10版本宣布开始支持containerd 1.1，**演化到这里Kubernetes已经可以完全脱开Docker Engine**；调用链路如下所示：

```
    Kubernetes Master → kubelet → KubeGenericRuntimeManager → containerd → runC
```

2020年，Kubernetes官方宣布从1.20版本起放弃对Docker的支持，即不在维护DockerShim这一Docker与CRI的适配层；往后CRI适配器可以选择CRI-O或者containerd。

![118159973-47ce0700-b450-11eb-84aa-a54af0d9adb2](https://user-images.githubusercontent.com/19829495/154808848-2c154bed-128d-4a7b-85ca-5ee48b5cd672.jpg)

## 容器运行时-runC分析

runc是一个CLI工具，用于根据OCI规范生成和运行容器，这里会通过runC源码简单分析其运行原理与流程。runC可以说是Docker容器中最为核心的部分，容器的创建，运行，销毁等等操作最终都将通过调用runC完成。

### OCI标准

**image spec**

OCI 容器镜像标准包含如下几个模块：

1. config.json：包含容器必需的元信息
2. rootfs：容器root文件系统，保存在一个文件夹里

config.json包含的元素如下所示，具体可以参考[runtime-spec相关文档](https://github.com/opencontainers/runtime-spec/blob/master/config.md)

```
ociVersion：指定OCI容器的版本号
root：配置容器的root文件系统
process：配置容器进程信息
hostname：配置容器的主机名
mounts：配置额外的挂载点
hook：配置容器生命周期触发钩子
```

**runtime spec**

OCI 容器运行时标准主要是指定容器的运行状态，和 runtime 需要提供的命令，如下状态流转图所示：

<img src="https://user-images.githubusercontent.com/19829495/118305620-cd1dee00-b51a-11eb-8fa5-814f86ca0f42.png" alt="DeepinScreenshot_select-area_20210515011303" style="zoom:200%;" />

**基于busybox创建OCI runtime bundle相关文件**

```bash
docker pull busybox
mkdir mycontainer && cd mycontainer
mkdir rootfs
# 创建rootfs，基于busybox镜像
docker export $(docker create busybox) | tar -C rootfs -xvf -
# 根据OCI规范来生成配置文件
runc spec
```

文件目录如下所示

```
└── mycontainer
    ├── config.json
    └── rootfs
        ├── bin
        ├── dev
        ├── etc
        ├── home
        ├── proc
        ├── root
        ├── sys
        ├── tmp
        ├── usr
        └── var
```

### runC相关命令

通过runc -h可以看到runc支持的命令如下所示：

```
checkpoint  checkpoint正在运行的容器，配合restore使用
create      创建容器
delete      删除容器及容器所拥有的任何资源
events      显示容器事件，如OOM通知，cpu，内存和IO使用情况统计信息
exec        在容器内执行新的进程
init        初始化namespace并启动进程
kill        发送指定的kill信号（默认值：SIGTERM）到容器的init进程
list        列出容器
pause       暂停容器内的所有进程
ps          显示在容器内运行的进程
restore     从之前的checkpoint还原容器
resume      恢复容器内暂停的所有进程
run         创建并运行容器
spec        创建specification文件
start       在创建的容器中执行用户定义的进程
state       输出容器的状态
update      更新容器资源约束，如cgroup参数
```

实践操作：

```bash
# 1. 修改config文件中的process元素的terminal与args
"process": {
		"terminal": false,
		"args": [
			"/bin/sleep", "3600"
		]
# 2. runc create创建容器（此时状态为created,容器内进程为init进程）
sudo runc create mycontainer
# 3. runc start启动容器（此时状态为running，容器内进程为我们指定的进程）
sudo runc start mycontainer
# 4. runc 暂停/恢复容器（暂停时状态为pause）
sudo runc pause mycontainer
sudo runc resume mycontainer
# 5. runc kill关闭容器（此时状态为stopped）
sudo runc kill mycontainer
# 6. runc delete删除容器
sudo runc delete mycontainer
# 7. runc run组合命令，组合了runc create创建，runc start启动以及在退出之后runc delete的删除命令
sudo runc run mycontainer
```

### runC-run

runC源码在github [runc项目](https://github.com/opencontainers/runc)可以获取到，项目结构如下所示，其中如`create.go`、`delete.go`等文件为runC命令的实现；`libcontainer`目录为核心包（参考上文runC的由来），可见runC本质上是对`libcontainer`的一层封装，在`libcontainer`上加了一层OCI的适配操作与hook；`main.go`是入口文件，使用 `github.com/urfave/cli` 库进行命令行解析与执行函数：

```bash
$ tree -L 1 -F --dirsfirst
.
├── contrib/
├── docs/
├── libcontainer/
├── man/
├── script/
├── tests/
├── types/
├── vendor/
├── checkpoint.go
├── create.go
├── delete.go
├── events.go
├── exec.go
├── init.go
├── kill.go
├── list.go
├── main.go
├── pause.go
├── ps.go
├── restore.go
├── run.go
├── spec.go
├── start.go
├── state.go
...
```

通过`run.go`我们可以看到runCommand相关的执行逻辑：

```go
Action: func(context *cli.Context) error {
		if err := checkArgs(context, 1, exactArgs); err != nil {
			return err
		}
		if err := revisePidFile(context); err != nil {
			return err
		}
		spec, err := setupSpec(context)
		if err != nil {
			return err
		}
		status, err := startContainer(context, spec, CT_ACT_RUN, nil)
		if err == nil {
			os.Exit(status)
		}
		return err
	}
```

这里主要涉及四步操作：

1. 参数校验
2. 若指定了pidFile，进行路径转换
3. 读取 `config.json` 文件转换成 spec 结构对象
4. 根据配置启动容器

第四步是操作调用了`utils_linux.go`下`startContainer`方法进行容器创建、运行，具体实现如下：

```go
func startContainer(context *cli.Context, spec *specs.Spec, action CtAct, criuOpts *libcontainer.CriuOpts) (int, error) {
	id := context.Args().First()
	...
	container, err := createContainer(context, id, spec)
	if err != nil {
		return -1, err
	}
	...
	r := &runner{
		enableSubreaper: !context.Bool("no-subreaper"),
		shouldDestroy:   true,
		container:       container,
		listenFDs:       listenFDs,
		notifySocket:    notifySocket,
		consoleSocket:   context.String("console-socket"),
		detach:          context.Bool("detach"),
		pidFile:         context.String("pid-file"),
		preserveFDs:     context.Int("preserve-fds"),
		action:          action,
		criuOpts:        criuOpts,
		init:            true,
		logLevel:        logLevel,
	}
	return r.run(spec.Process)
}
```

1. 通过调用`createContainer`创建出逻辑容器（包含保存了namespace、cgroups、mounts…容器配置）
2. 创建`runner`对象，通过调用`runner`的run方法运行容器进程，并进行对应的设置

**Factory创建逻辑容器**

```go
func createContainer(context *cli.Context, id string, spec *specs.Spec) (libcontainer.Container, error) {
	...
    config, err := specconv.CreateLibcontainerConfig(&specconv.CreateOpts{
		CgroupName:       id,
		UseSystemdCgroup: context.GlobalBool("systemd-cgroup"),
		NoPivotRoot:      context.Bool("no-pivot"),
		NoNewKeyring:     context.Bool("no-new-keyring"),
		Spec:             spec,
		RootlessEUID:     os.Geteuid() != 0,
		RootlessCgroups:  rootlessCg,
	})
	if err != nil {
		return nil, err
	}

	factory, err := loadFactory(context)
	if err != nil {
		return nil, err
	}
	return factory.Create(id, config)
}
```

1. 通过`specconv.CreateLibcontainerConfig`把spec转换成libcontainer内部config对象
2. 通过factory 模式获取对应平台的factory实现（目前仅linux平台支持）,如`libcontainer/factory_linux.go`，并调用其Create方法构建返回`libcontainer.Container` 对象（包含容器命令的实现，如Run、Start、State…）
3. 注意`libcontainer/factory_linux.go`New方法的InitPath为`/proc/self/exe`,InitArgs为init

```go
func (l *LinuxFactory) Create(id string, config *configs.Config) (Container, error) {
	...
	c := &linuxContainer{
		id:            id,
		root:          containerRoot,
		config:        config,
		initPath:      l.InitPath,
		initArgs:      l.InitArgs,
		criuPath:      l.CriuPath,
		newuidmapPath: l.NewuidmapPath,
		newgidmapPath: l.NewgidmapPath,
		cgroupManager: l.NewCgroupsManager(config.Cgroups, nil),
	}
	if l.NewIntelRdtManager != nil {
		c.intelRdtManager = l.NewIntelRdtManager(config, id, "")
	}
	c.state = &stoppedState{c: c}
	return c, nil
}

func New(root string, options ...func(*LinuxFactory) error) (Factory, error) {
	...
	l := &LinuxFactory{
		Root:      root,
		InitPath:  "/proc/self/exe",
		InitArgs:  []string{os.Args[0], "init"},
		Validator: validate.New(),
		CriuPath:  "criu",
	}
	Cgroupfs(l)
	for _, opt := range options {
		if opt == nil {
			continue
		}
		if err := opt(l); err != nil {
			return nil, err
		}
	}
	return l, nil
}
```

**Runner运行容器-container_linux.start**

runner的run方法会根据OCI specs.Process生成 `libcontainer.Process`逻辑process对象（包含进程的命令、参数、环境变量、用户…）；根据对应的command如run会调用container的Run方法，在linux平台下最终会调用`container_linux`的start方法与exec方法，start方法如下所示：

```go
func (c *linuxContainer) start(process *Process) (retErr error) {
	parent, err := c.newParentProcess(process)
	if err != nil {
		return newSystemErrorWithCause(err, "creating new parent process")
	}
    //
	if err := parent.start(); err != nil {
		return newSystemErrorWithCause(err, "starting container process")
	}
	...
	//更新容器状态，PostStart Hook回调
	return nil
}

func (c *linuxContainer) newParentProcess(p *Process) (parentProcess, error) {
	parentInitPipe, childInitPipe, err := utils.NewSockPair("init")
	if err != nil {
		return nil, newSystemErrorWithCause(err, "creating new init pipe")
	}
	messageSockPair := filePair{parentInitPipe, childInitPipe}

	parentLogPipe, childLogPipe, err := os.Pipe()
	if err != nil {
		return nil, fmt.Errorf("Unable to create the log pipe:  %s", err)
	}
	logFilePair := filePair{parentLogPipe, childLogPipe}

	cmd := c.commandTemplate(p, childInitPipe, childLogPipe)
	if !p.Init {
		return c.newSetnsProcess(p, cmd, messageSockPair, logFilePair)
	}
	if err := c.includeExecFifo(cmd); err != nil {
		return nil, newSystemErrorWithCause(err, "including execfifo in cmd.Exec setup")
	}
	return c.newInitProcess(p, cmd, messageSockPair, logFilePair)
}
```

`newParentProcess`方法执行如下操作：

1. 创建`parentPipe` 和 `childPipe` 进程通信管道用于父进程与容器内父进程进行通信
2. cmd封装容器父进程运行模板`/proc/self/exe - init`
3. 创建initProcess，即容器内父进程逻辑对象，包含（cmd、逻辑process对象、paerentPipe`和`childPipe等）

`newParentProcess`返回initProcess后`container_linux`的start方法会调用initProcess的start方法

```go
func (p *initProcess) start() (retErr error) {
	defer p.messageSockPair.parent.Close()
    //启动容器init进程
	err := p.cmd.Start()
	p.process.ops = p
	p.messageSockPair.child.Close()
	p.logFilePair.child.Close()
    waitInit := initWaiter(p.messageSockPair.parent)
	...
    //将容器pid加入到cgroup 中
	if err := p.manager.Apply(p.pid()); err != nil {
		return newSystemErrorWithCause(err, "applying cgroup configuration for process")
	}
	if p.intelRdtManager != nil {
		if err := p.intelRdtManager.Apply(p.pid()); err != nil {
			return newSystemErrorWithCause(err, "applying Intel RDT configuration for process")
		}
	}
    //向容器init进程发送bootstrapData，即初始化数据
	if _, err := io.Copy(p.messageSockPair.parent, p.bootstrapData); err != nil {
		return newSystemErrorWithCause(err, "copying bootstrap data to pipe")
	}
    //等待容器init进程初始化
	err = <-waitInit
	if err != nil {
		return err
	}
	...
	//创建网络 interface
	if err := p.createNetworkInterfaces(); err != nil {
		return newSystemErrorWithCause(err, "creating network interfaces")
	}
    //更新Spec状态
	if err := p.updateSpecState(); err != nil {
		return newSystemErrorWithCause(err, "updating the spec state")
	}
    // 给容器init进程发送进程配置信息
	if err := p.sendConfig(); err != nil {
		return newSystemErrorWithCause(err, "sending config to init process")
	}
	//和容器init进程进行同步,待其初始化完成之后，执行Hook回调操作，设置cgroup，最后，关闭init管道，容器创建完成
	ierr := parseSync(p.messageSockPair.parent, func(sync *syncT) error {
    ...
    }
	...
}
```

initProcess的start操作主要涉及如下几个步骤：

1. 执行cmd命令，通过`/proc/self/exe - init`启动容器init进程执行init command操作
2. 向容器init进程发送bootstrapData初始化数据
3. 创建网络 interface
4. 和容器init进程进行同步,待其初始化完成之后，执行Hook回调操作，设置cgroup，最后，关闭init管道，容器创建完成

**Runner运行容器-container_linux.exec**

具体实现如下

```go
func (c *linuxContainer) exec() error {
	path := filepath.Join(c.root, execFifoFilename)
	pid := c.initProcess.pid()
	blockingFifoOpenCh := awaitFifoOpen(path)
	for {
		select {
		case result := <-blockingFifoOpenCh:
			return handleFifoResult(result)

		case <-time.After(time.Millisecond * 100):
			stat, err := system.Stat(pid)
			if err != nil || stat.State == system.Zombie {
				if err := handleFifoResult(fifoOpen(path, false)); err != nil {
					return errors.New("container process is already dead")
				}
				return nil
			}
		}
	}
}
```

通过打开路径`exec.fifo`的管道，调用`readFromExecFifo`从管道中将容器init进程从写入的信息读出。一旦管道中的数据被读出，容器init进程将不再被阻塞，完成之后的`Exec`系统调用，这时候容器init进程切换为command指定的进程。

### runC-init

上文分析runc run流程，在最后通过`/proc/self/exe - init`启动容器init进程执行init command操作，这里直接追溯到init操作的核心方法`standard_init_linux.go`下的Init方法，如下所示：

```go
func (l *linuxStandardInit) Init() error {

	//配置容器网络与路由
	if err := setupNetwork(l.config); err != nil {
		return err
	}
	if err := setupRoute(l.config.Config); err != nil {
		return err
	}

	//设置rootfs，挂载文件系统，chroot
	if err := prepareRootfs(l.pipe, l.config); err != nil {
		return err
	}
	//配置console, hostname, apparmor, sysctl，seccomp，user namespace...
	...
	//同步父进程 init进程准备执行exec
	if err := syncParentReady(l.pipe); err != nil {
		return errors.Wrap(err, "sync ready")
	}
	...
	//已完成初始化，关闭管道
	l.pipe.Close()
	...
	//向exec.fifo管道写入数据，阻塞到exec执行，读取管道中的数据
	fd, err := unix.Open("/proc/self/fd/"+strconv.Itoa(l.fifoFd), unix.O_WRONLY|unix.O_CLOEXEC, 0)
	if err != nil {
		return newSystemErrorWithCause(err, "open exec fifo")
	}
	if _, err := unix.Write(fd, []byte("0")); err != nil {
		return newSystemErrorWithCause(err, "write 0 exec fifo")
	}
	//调用Exec系统命令，执行用户进程
	if err := unix.Exec(name, l.config.Args[0:], os.Environ()); err != nil {
		return newSystemErrorWithCause(err, "exec user process")
	}
	return nil
}
```

这里涉及如下几步操作：

1. 配置容器网络与路由
2. 设置rootfs，挂载文件系统，chroot
3. 配置console, hostname, apparmor, sysctl，seccomp，user namespace
4. 同步父进程，让父进程执行hook等操作
5. 调用Exec系统命令，执行用户进程，此时容器的init进程会替换成用户进程

最后，在Init执行之前，会被`/runc/libcontainer/nsenter`劫持，在Go的runtime之前执行对应的C代码，从init管道中读取容器的配置，设置namespace，调用`setns`系统调用，将容器init进程加入到对应的namespace中。

### runC-run-init-运行流程

**runC-run运行流程**

![runc-run](https://user-images.githubusercontent.com/19829495/118503688-6f2d1880-b75d-11eb-9942-6e277f249012.jpg)

**runC-init运行流程**

![runc-init](https://user-images.githubusercontent.com/19829495/118503773-81a75200-b75d-11eb-8d0d-85b912864e9b.jpg)

