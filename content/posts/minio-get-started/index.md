---
title: "MinIO大杂烩"
author: "爱吃芒果"
description:
date: 2025-01-22T19:03:56+08:00
image:
math:
license:
hidden: false
comments: true
draft: false
tags:
  - "MinIO"
categories:
  - "MinIO"
---

## xl.meta 数据结构

当对象大小超过 128KiB 后，比如`a.txt`，数据和元数据分开存储

MinIO 提供了命令行工具`xl-meta`用来查看`xl.meta`文件

```json
{
  "Versions": [
    {
      "Header": {
        "EcM": 1,
        "EcN": 0,
        "Flags": 2,
        "ModTime": "2025-01-23T15:27:45.311572+08:00",
        "Signature": "d0c2b58b",
        "Type": 1,
        "VersionID": "00000000000000000000000000000000"
      },
      "Idx": 0,
      "Metadata": {
        "Type": 1,
        "V2Obj": {
          "CSumAlgo": 1,
          "DDir": "74hQxU7FTrq56ShK8pjqAA==",
          "EcAlgo": 1,
          "EcBSize": 1048576,
          "EcDist": [1],
          "EcIndex": 1,
          "EcM": 1,
          "EcN": 0,
          "ID": "AAAAAAAAAAAAAAAAAAAAAA==",
          "MTime": 1737617265311572000,
          "MetaSys": {},
          "MetaUsr": {
            "content-type": "text/plain",
            "etag": "90a1a2b65a4e40d55d758f2a59fe33b4"
          },
          "PartASizes": [2097152],
          "PartETags": null,
          "PartNums": [1],
          "PartSizes": [2097152],
          "Size": 2097152
        },
        "v": 1734527744
      }
    }
  ]
}
```

```bash
.
├── a.txt
│   ├── ef8850c5-4ec5-4eba-b9e9-284af298ea00
│   │   └── part.1
│   └── xl.meta
└── b.txt
    └── xl.meta
```

## minio 的启动流程

minio 启动核心的核心命令为 `minio server https://minio{1...4}.example.net:9000/mnt/disk{1...4}/minio`，表示 minio 服务分布部署在 4 台服务器上总共 16 块磁盘上，`...这种写法称之为拓展表达式，比如 `http://minio{1...4}.example.net:9000`实际上表示`http://minio1.example.net:9000`到`http://minio4.example.net:9000`的4台主机。

go 程序的入口为`main#main()`函数，直接调用了`cmd#Main`,其中做了一些命令行程序的相关操作，包括注册命令，其中`registerCommand(serverCmd)`注册服务相关命令，`cmd#ServerMain`是主要启动流程函数。

```go
// Run the app - exit on error.
if err := newApp(appName).Run(args); err != nil {
  os.Exit(1) //nolint:gocritic
}
```

```go
var serverCmd = cli.Command{
	Name:   "server",
	Usage:  "start object storage server",
	Flags:  append(ServerFlags, GlobalFlags...),
	Action: serverMain,
```

### ServerMain

```go
server
http://127.0.0.1:/Users/hanjing/mnt/minio0{1...3}
http://127.0.0.1:/Users/hanjing/mnt/minio0{4...6}
```

处理系统终止或者重启相关的信号等

```go
signal.Notify(globalOSSignalCh, os.Interrupt, syscall.SIGTERM, syscall.SIGQUIT)
go handleSignals()
```

`buildServerCtxt`决定磁盘布局以及是否使用 legacy 方式，调用函数`cmd#mergeDisksLayoutFromArgs`判断是否使用了拓展表达式，如果没有，`legacy = true`，否则`legacy =false`, `legacy`参数的作用我们在后面就能看到了。

`serverHandleCmdArgs`函数中调用 `createServerEndpoints`，

```go
	// Handle all server command args and build the disks layout
	bootstrapTrace("serverHandleCmdArgs", func() {
    // 这里确定了erasure set size的大小
		err := buildServerCtxt(ctx, &globalServerCtxt)
		logger.FatalIf(err, "Unable to prepare the list of endpoints")

		serverHandleCmdArgs(globalServerCtxt)
	})
```

## MinIO 的 DNS 缓存

MinIO 为了避免向外发送过多的 DNS 查询，所以实现了 DNS 缓存，默认使用`net.DefaultResolver`实际执行 DNS 查询，设置的 DNS 查询超时时间为`5s`，缓存的刷新时间在容器环境下默认为`30s`，在其他环境下为`10min`，可以通过`dns-cache-ttl`指定。

```go
type Resolver struct {
	// Timeout defines the maximum allowed time allowed for a lookup.
	Timeout time.Duration

	// Resolver is used to perform actual DNS lookup. If nil,
	// net.DefaultResolver is used instead.
	Resolver DNSResolver

	once  sync.Once
	mu    sync.RWMutex
	cache map[string]*cacheEntry
}

globalDNSCache = &dnscache.Resolver{
  Timeout: 5 * time.Second,
}
```

```go

func runDNSCache(ctx *cli.Context) {
	dnsTTL := ctx.Duration("dns-cache-ttl")
	// Check if we have configured a custom DNS cache TTL.
	if dnsTTL <= 0 {
		if orchestrated {
			dnsTTL = 30 * time.Second
		} else {
			dnsTTL = 10 * time.Minute
		}
	}

	// Call to refresh will refresh names in cache.
	go func() {
		// Baremetal setups set DNS refresh window up to dnsTTL duration.
		t := time.NewTicker(dnsTTL)
		defer t.Stop()
		for {
			select {
			case <-t.C:
				globalDNSCache.Refresh()

			case <-GlobalContext.Done():
				return
			}
		}
	}()
}
```

## 构造拓扑关系 (`buildServerCtxt`)

```go
// serverCtxt保存了磁盘布局
type disksLayout struct {
  // 是否使用拓展表达式
	legacy bool
  // server pool的集合
	pools  []poolDisksLayout
}
type poolDisksLayout struct {
  // server pool对应的命令行命令
	cmdline string
  // layout的第一位表示不同的erasure set，第二维表示同一个erasure set中不同的磁盘路径
	layout  [][]string
}
```

构造拓扑关系的主要函数实现是`mergeDisksLayoutFromArgs`，判断环境变量`MINIO_ERASURE_SET_DRIVE_COUNT`是否存在，环境变量`MINIO_ERASURE_SET_DRIVE_COUNT`表示 erasure set 中指定的磁盘数量，否则默认为 0，表示自动设置最优结果。根据是否使用拓展表达式会走不同的逻辑。这里我们主要关心使用拓展表达式的场景`GetAllSets(setDriveCount, arg)`。（顺带一提，legacy style 会走`GetAllSets(setDriveCount, args...)`，可以看到 legacy style 只能指定一个`server pool`）

```go
// mergeDisksLayoutFromArgs supports with and without ellipses transparently.
// 构造网络拓扑
func mergeDisksLayoutFromArgs(args []string, ctxt *serverCtxt) (err error) {
	if len(args) == 0 {
		return errInvalidArgument
	}

	ok := true
	// ok 表示是否使用拓展表达式，true表示不使用拓展表达式
	// 只要在其中一个arg中使用拓展表达式，结果均为false
	for _, arg := range args {
		ok = ok && !ellipses.HasEllipses(arg)
	}

	var setArgs [][]string

	// 通过环境变量得到erasure set的大小，默认为0
	v, err := env.GetInt(EnvErasureSetDriveCount, 0)
	if err != nil {
		return err
	}
	setDriveCount := uint64(v)

	// None of the args have ellipses use the old style.
	if ok {
		setArgs, err = GetAllSets(setDriveCount, args...)
		if err != nil {
			return err
		}
		// 所有的参数组成一个server pool
		ctxt.Layout = disksLayout{
			legacy: true,
			pools:  []poolDisksLayout{{layout: setArgs, cmdline: strings.Join(args, " ")}},
		}
		return
	}

	for _, arg := range args {
		if !ellipses.HasEllipses(arg) && len(args) > 1 {
			// TODO: support SNSD deployments to be decommissioned in future
			return fmt.Errorf("all args must have ellipses for pool expansion (%w) args: %s", errInvalidArgument, args)
		}
		setArgs, err = GetAllSets(setDriveCount, arg)
		if err != nil {
			return err
		}
		ctxt.Layout.pools = append(ctxt.Layout.pools, poolDisksLayout{cmdline: arg, layout: setArgs})
	}
	return
}
```

`GetAllSets`主要调用了`parseEndpointSet`，通过正则表达式解析带有拓展表达式的输入参数，并返回一个`[][]string`，表示不同 erasure set 中的磁盘路径。这里主要对应的数据结构是`endpointSet`，主要实现两件事情，第一确定 setSize，第二确定如何将 endpoints 分布到不同的 erasure set 中。

```go
// Endpoint set represents parsed ellipses values, also provides
// methods to get the sets of endpoints.
type endpointSet struct {
  // 解析终端字符串得到的arg pattern，如果有多个ellipses，对应多个`Pattern`
	argPatterns []ellipses.ArgPattern
	endpoints   []string   // Endpoints saved from previous GetEndpoints().
  // 对于ellipses-style的参数
  // setIndexes对应一行，记录了server pool size /setSize 个 setSize值
	setIndexes  [][]uint64 // All the sets.
}
type ArgPattern []Pattern
// Pattern - ellipses pattern, describes the range and also the
// associated prefix and suffixes.
type Pattern struct {
	Prefix string
	Suffix string
	Seq    []string
}
```

函数`getSetIndexes`的目的是找到合适的`setSize`，MinIO 规定分布式部署 setSize 的取值必须属于`var setSizes = []uint64{2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16}`，首先从`SetSizes`中找到能够被`server pool size`整除的`setCounts`集合，如果自定义了`setSize`则判断自定义的`setSize`是否属于`setCounts`集合，如果属于则`setSize`设置成功，否则返回错误。如果没有设置自定义的`setSize`，函数`possibleSetCountsWithSymmetry`从`setCounts`集合中找到具有`symmetry`属性的值，MinIO 中输入带拓展表达式的参数对应的 pattern 列表和参数中的顺序是相反的，`symmetry`过滤出能够被 pattern 中最后一个 pattern 对应的数量整除或者被整除的`setCounts`中的值，这里举一个例子`http://127.0.0.{1...4}:9000/Users/hanjing/mnt/minio{1...32}`，显然`symmetry`函数会判断 4 和`setCounts`中值的关系，而不是 32 和`setCounts`中值的关系，这可能与 MinIO 希望尽可能将 erasure set 的中不同磁盘分布到不同的节点上有关。最后取出剩余候选值中最大的值作为最终的`setSize`。

```go
func (s endpointSet) Get() (sets [][]string) {
	k := uint64(0)
	endpoints := s.getEndpoints()
	for i := range s.setIndexes {
		for j := range s.setIndexes[i] {
			sets = append(sets, endpoints[k:s.setIndexes[i][j]+k])
			k = s.setIndexes[i][j] + k
		}
	}

	return sets
}
```

`endpointSet#Get`方法返回一个二维数据，第一维表示 不同的 erasure set，第二位表示 erasure set 中的不同磁盘。这里`getEndpoints`多重循环迭代 ellipses-style 对应的 pattern，如果还记得的话，pattern 的顺序和实际在参数中出现的顺序相反，这样得到的`endpoints`列表将不同节点上的磁盘均匀分布，后面连续取列表中的一段组成`erasure set`时，得到的`erasure set`中的磁盘也分布在不同的节点上。

### serverHandleCmdArgs 函数

```go
	globalEndpoints, setupType, err = createServerEndpoints(globalMinioAddr, ctxt.Layout.pools, ctxt.Layout.legacy)
	logger.FatalIf(err, "Invalid command line arguments")
	globalNodes = globalEndpoints.GetNodes()

	globalIsErasure = (setupType == ErasureSetupType)
	globalIsDistErasure = (setupType == DistErasureSetupType)
	if globalIsDistErasure {
		globalIsErasure = true
	}
	globalIsErasureSD = (setupType == ErasureSDSetupType)
	if globalDynamicAPIPort && globalIsDistErasure {
		logger.FatalIf(errInvalidArgument, "Invalid --address=\"%s\", port '0' is not allowed in a distributed erasure coded setup", ctxt.Addr)
	}

	globalLocalNodeName = GetLocalPeer(globalEndpoints, globalMinioHost, globalMinioPort)
	nodeNameSum := sha256.Sum256([]byte(globalLocalNodeName))
	globalLocalNodeNameHex = hex.EncodeToString(nodeNameSum[:])

	// Initialize, see which NIC the service is running on, and save it as global value
	setGlobalInternodeInterface(ctxt.Interface)
```

里面有一个比较重要的工具函数`isLocalHost`，通过 DNS 查询 host 对应的 ip，和所有网卡对应的所有本地 ip 取交集,如果交集为空，说明不是本地服务器，否则是本地服务器。

函数`createServerEndpoints`将数据结构`[]poolDisksLayout`转换成`EndpointServerPools`，并指定对应的`SetupType`

对于单磁盘部署，要求使用目录路径指定输入参数，`IsLocal`一定为`true`，`SetupType`为`ErasureSDSetupType`。其他情况下根据，根据本地 ip 和给定的 host，判断`IsLocal`，如果 host 为空（MinIO 称为`PathEndpointType`)，则`setupType = ErasureSetupType`，否则为`URLEndpointType`情况，如果不同`host:port`的数量等于 1，则是`ErasureSetupType`，否则对应`DistErasureSetupType`，根据得到的`SetType`设置全局参数。

`EndpointServerPools`实际上是`[][]EndPoint`，第一位

```go
// EndpointServerPools是 PoolEndpoints的集合，实际上描述整个部署的拓扑结构
type EndpointServerPools []PoolEndpoints
// PoolEndpoints represent endpoints in a given pool
// along with its setCount and setDriveCount.
// PoolEndpoints表示一个server pool的结构
type PoolEndpoints struct {
	// indicates if endpoints are provided in non-ellipses style
  // legacy 表示 是否使用遗留的方法表示终端，而不使用省略号表达式
	Legacy       bool
  // SetCount表示 server pool中的 erasure set的数量
	SetCount     int
  // DrivesPerSet 表示一个erasure set中的磁盘数量
	DrivesPerSet int
  // type Endpoints []Endpoint
  // 表示一个server pool中的所有disk
	Endpoints    Endpoints
  // server pool对应的命令行指令
	CmdLine      string
  // 操作系统信息
	Platform     string
}

type Endpoint struct {
	*url.URL
  // 如果是单个目录的输入，则 IsLocal为true
  // 如果输入参数ip是本地ip，IsLocal也为true
  // 其他情况下为false
	IsLocal bool

	PoolIdx, SetIdx, DiskIdx int
}
```

```go
// SetupType - enum for setup type.
type SetupType int

const (
	// UnknownSetupType - starts with unknown setup type.
	UnknownSetupType SetupType = iota

	// FSSetupType - FS setup type enum.
	FSSetupType

	// ErasureSDSetupType - Erasure single drive setup enum.
	ErasureSDSetupType

	// ErasureSetupType - Erasure setup type enum.
	ErasureSetupType

	// DistErasureSetupType - Distributed Erasure setup type enum.
	DistErasureSetupType
)
```

以下函数列出了 Minio 支持的不同模式，和上面的`SetType`之间存在对应关系。

```go
// Returns the mode in which MinIO is running
func getMinioMode() string {
	switch {
	case globalIsDistErasure:
		return globalMinioModeDistErasure
	case globalIsErasure:
		return globalMinioModeErasure
	case globalIsErasureSD:
		return globalMinioModeErasureSD
	default:
		return globalMinioModeFS
	}
}
```

## HTTP 服务器注册 API

- 注册分布式命名空间锁
- `registerAPIRouter`注册 s3 相关的主要 api

```go
	// Configure server.
	bootstrapTrace("configureServer", func() {
		handler, err := configureServerHandler(globalEndpoints)
		if err != nil {
			logger.Fatal(config.ErrUnexpectedError(err), "Unable to configure one of server's RPC services")
		}
		// Allow grid to start after registering all services.
		close(globalGridStart)
		close(globalLockGridStart)

		httpServer := xhttp.NewServer(getServerListenAddrs()).
			UseHandler(setCriticalErrorHandler(corsHandler(handler))).
			UseTLSConfig(newTLSConfig(getCert)).
			UseIdleTimeout(globalServerCtxt.IdleTimeout).
			UseReadTimeout(globalServerCtxt.IdleTimeout).
			UseWriteTimeout(globalServerCtxt.IdleTimeout).
			UseReadHeaderTimeout(globalServerCtxt.ReadHeaderTimeout).
			UseBaseContext(GlobalContext).
			UseCustomLogger(log.New(io.Discard, "", 0)). // Turn-off random logging by Go stdlib
			UseTCPOptions(globalTCPOptions)

		httpServer.TCPOptions.Trace = bootstrapTraceMsg
		go func() {
			serveFn, err := httpServer.Init(GlobalContext, func(listenAddr string, err error) {
				bootLogIf(GlobalContext, fmt.Errorf("Unable to listen on `%s`: %v", listenAddr, err))
			})
			if err != nil {
				globalHTTPServerErrorCh <- err
				return
			}
			globalHTTPServerErrorCh <- serveFn()
		}()

		setHTTPServer(httpServer)
	})
```

```go
// configureServer handler returns final handler for the http server.
func configureServerHandler(endpointServerPools EndpointServerPools) (http.Handler, error) {
	// Initialize router. `SkipClean(true)` stops minio/mux from
	// normalizing URL path minio/minio#3256
	router := mux.NewRouter().SkipClean(true).UseEncodedPath()

	// Initialize distributed NS lock.
	if globalIsDistErasure {
		registerDistErasureRouters(router, endpointServerPools)
	}

	// Add Admin router, all APIs are enabled in server mode.
	registerAdminRouter(router, true)

	// Add healthCheck router
	registerHealthCheckRouter(router)

	// Add server metrics router
	registerMetricsRouter(router)

	// Add STS router always.
	registerSTSRouter(router)

	// Add KMS router
	registerKMSRouter(router)

	// Add API router
	registerAPIRouter(router)

	router.Use(globalMiddlewares...)

	return router, nil
}
```

`registerAPIRouter`会注册主要的 s3 API，这里举`GetObject`操作为例进行说明，当 http method 为`GET`时，如果没有命中其他的路由，则认为是`GetObject`操作，从 Path 中获取`object`名字，并使用`api.GetObjectHandler`进行处理和响应，`s3APIMiddleware`作为中间件，可以做一些额外的操作，比如监控和记录日志。

`api`对象中保存了一个函数引用，通过这个函数引用，能够得到全局的`ObjectLayer`对象，`ObjectLayer`实现了对象 API 层的基本操作。

```go
// GetObject
router.Methods(http.MethodGet).Path("/{object:.+}").
  HandlerFunc(s3APIMiddleware(api.GetObjectHandler, traceHdrsS3HFlag))

// Initialize API.
api := objectAPIHandlers{
  ObjectAPI: newObjectLayerFn,
}
// objectAPIHandlers implements and provides http handlers for S3 API.
type objectAPIHandlers struct {
	ObjectAPI func() ObjectLayer
}
func newObjectLayerFn() ObjectLayer {
	globalObjLayerMutex.RLock()
	defer globalObjLayerMutex.RUnlock()
	return globalObjectAPI
}
```

## ObjectLayer 的初始化流程

```go
	var newObject ObjectLayer
	bootstrapTrace("newObjectLayer", func() {
		var err error
		newObject, err = newObjectLayer(GlobalContext, globalEndpoints)
		if err != nil {
			logFatalErrs(err, Endpoint{}, true)
		}
	})
```

`storageclass.LookupConfig`函数根据环境变量等初始化`Standard Storage Class`、`Reduced Redundancy Storage Class`以及`Optimized Storage Class`、以及 `inline data`的大小

`Standard Storage Class`：通过环境变量`MINIO_STORAGE_CLASS_STANDARD`指定，否则会根据`erasure set`的大小指定

```go
// DefaultParityBlocks returns default parity blocks for 'drive' count
func DefaultParityBlocks(drive int) int {
	switch drive {
	case 1:
		return 0
	case 3, 2:
		return 1
	case 4, 5:
		return 2
	case 6, 7:
		return 3
	default:
		return 4
	}
}
```

`Reduced Redundancy Storage Class`: 通过环境变量`MINIO_STORAGE_CLASS_RRS`指定，否则默认为 1

`Optimized Storage Class`：通过环境变量`MINIO_STORAGE_CLASS_OPTIMIZE`指定，默认为`""`

`inline block size`: 通过环境变量`MINIO_STORAGE_CLASS_INLINE_BLOCK`指定，默认为`128KiB`,如果 shard 数据的大小小于`inline block size`，则会直接将数据和元数据写到同一个文件，即`xl.meta`

### MinIO 的存储分层

#### erasureServerPools

```go
// erasureServerPools
// minio 服务可以由多个server pool 组成，用来水平拓展
// erasureServerPools是server pool的集合
type erasureServerPools struct {
	poolMetaMutex sync.RWMutex
	poolMeta      poolMeta

	rebalMu   sync.RWMutex
	rebalMeta *rebalanceMeta

	deploymentID     [16]byte
	distributionAlgo string

	// server pool 由多个erasure set 组成
	// 这里的erasureSets结构实际上指单个 server pool
	serverPools []*erasureSets

	// Active decommission canceler
	decommissionCancelers []context.CancelFunc

	s3Peer *S3PeerSys

	mpCache *xsync.MapOf[string, MultipartInfo]
}
```

#### erasureSets

```go
// erasureSets implements ObjectLayer combining a static list of erasure coded
// object sets. NOTE: There is no dynamic scaling allowed or intended in
// current design.
// server pool 由多个erasure set 组成
// 这里的erasureSets结构实际上指单个 server pool
// 上面这段话的意思是不能动态扩展server pool，初始指定后就不能再修改了
type erasureSets struct {
	sets []*erasureObjects

	// Reference format.
	format *formatErasureV3

	// erasureDisks mutex to lock erasureDisks.
	erasureDisksMu sync.RWMutex

	// Re-ordered list of disks per set.
	erasureDisks [][]StorageAPI

	// Distributed locker clients.
	erasureLockers setsDsyncLockers

	// Distributed lock owner (constant per running instance).
	erasureLockOwner string

	// List of endpoints provided on the command line.
	endpoints PoolEndpoints

	// String version of all the endpoints, an optimization
	// to avoid url.String() conversion taking CPU on
	// large disk setups.
	endpointStrings []string

	// Total number of sets and the number of disks per set.
	setCount, setDriveCount int
	defaultParityCount      int

	poolIndex int

	// Distribution algorithm of choice.
	distributionAlgo string
	deploymentID     [16]byte

	lastConnectDisksOpTime time.Time
}
```

#### erasureObjects

```go
// erasureObjects - Implements ER object layer.
// 表示一个erasure Set
type erasureObjects struct {
	setDriveCount      int
	defaultParityCount int

	setIndex  int
	poolIndex int

	// getDisks returns list of storageAPIs.
	getDisks func() []StorageAPI

	// getLockers returns list of remote and local lockers.
	getLockers func() ([]dsync.NetLocker, string)

	// getEndpoints returns list of endpoint belonging this set.
	// some may be local and some remote.
	getEndpoints func() []Endpoint

	// getEndpoints returns list of endpoint strings belonging this set.
	// some may be local and some remote.
	getEndpointStrings func() []string

	// Locker mutex map.
	nsMutex *nsLockMap
}
```

### StorageAPI

`StorageAPI`主要有两个实现

- `xlStorage`表示本地存储
- `storageRESTClient`表示远程主机上的存储

```go
// Depending on the disk type network or local, initialize storage API.
func newStorageAPI(endpoint Endpoint, opts storageOpts) (storage StorageAPI, err error) {
	if endpoint.IsLocal {
		storage, err := newXLStorage(endpoint, opts.cleanUp)
		if err != nil {
			return nil, err
		}
		return newXLStorageDiskIDCheck(storage, opts.healthCheck), nil
	}

	return newStorageRESTClient(endpoint, opts.healthCheck, globalGrid.Load())
}
```

`newXLStorage`函数调用了`getDiskInfo`函数，并要求路径不能在`rootDrive`上。判断磁盘是否支持`O_DIRECT`，在分布式部署下，如果不支持`O_DIRECT`，则直接报错。

```go
	// Return an error if ODirect is not supported. Single disk will have
	// oDirect off.
	// 在类似unix的平台上 disk.ODirectPlatform应该为true
	if globalIsErasureSD || !disk.ODirectPlatform {
		s.oDirect = false
	} else if err := s.checkODirectDiskSupport(info.FSType); err == nil {
		s.oDirect = true
	} else {
		return s, err
	}
```

```go
// getDiskInfo returns given disk information.
func getDiskInfo(drivePath string) (di disk.Info, rootDrive bool, err error) {
	if err = checkPathLength(drivePath); err == nil {
		di, err = disk.GetInfo(drivePath, false)

		if !globalIsCICD && !globalIsErasureSD {
			if globalRootDiskThreshold > 0 {
				// Use MINIO_ROOTDISK_THRESHOLD_SIZE to figure out if
				// this disk is a root disk. treat those disks with
				// size less than or equal to the threshold as rootDrives.
				rootDrive = di.Total <= globalRootDiskThreshold
			} else {
				rootDrive, err = disk.IsRootDisk(drivePath, SlashSeparator)
			}
		}
	}
```

```go
// StorageAPI interface.
// 对应磁盘
type StorageAPI interface {
	// Stringified version of disk.
	String() string

	// Storage operations.

	// Returns true if disk is online and its valid i.e valid format.json.
	// This has nothing to do with if the drive is hung or not responding.
	// For that individual storage API calls will fail properly. The purpose
	// of this function is to know if the "drive" has "format.json" or not
	// if it has a "format.json" then is it correct "format.json" or not.
	IsOnline() bool

	// Returns the last time this disk (re)-connected
	LastConn() time.Time

	// Indicates if disk is local or not.
	IsLocal() bool

	// Returns hostname if disk is remote.
	Hostname() string

	// Returns the entire endpoint.
	Endpoint() Endpoint

	// Close the disk, mark it purposefully closed, only implemented for remote disks.
	Close() error

	// Returns the unique 'uuid' of this disk.
	GetDiskID() (string, error)

	// Set a unique 'uuid' for this disk, only used when
	// disk is replaced and formatted.
	SetDiskID(id string)

	// Returns healing information for a newly replaced disk,
	// returns 'nil' once healing is complete or if the disk
	// has never been replaced.
	Healing() *healingTracker
	DiskInfo(ctx context.Context, opts DiskInfoOptions) (info DiskInfo, err error)
	NSScanner(ctx context.Context, cache dataUsageCache, updates chan<- dataUsageEntry, scanMode madmin.HealScanMode, shouldSleep func() bool) (dataUsageCache, error)

	// Volume operations.
	MakeVol(ctx context.Context, volume string) (err error)
	MakeVolBulk(ctx context.Context, volumes ...string) (err error)
	ListVols(ctx context.Context) (vols []VolInfo, err error)
	StatVol(ctx context.Context, volume string) (vol VolInfo, err error)
	DeleteVol(ctx context.Context, volume string, forceDelete bool) (err error)

	// WalkDir will walk a directory on disk and return a metacache stream on wr.
	WalkDir(ctx context.Context, opts WalkDirOptions, wr io.Writer) error

	// Metadata operations
	DeleteVersion(ctx context.Context, volume, path string, fi FileInfo, forceDelMarker bool, opts DeleteOptions) error
	DeleteVersions(ctx context.Context, volume string, versions []FileInfoVersions, opts DeleteOptions) []error
	DeleteBulk(ctx context.Context, volume string, paths ...string) error
	WriteMetadata(ctx context.Context, origvolume, volume, path string, fi FileInfo) error
	UpdateMetadata(ctx context.Context, volume, path string, fi FileInfo, opts UpdateMetadataOpts) error
	ReadVersion(ctx context.Context, origvolume, volume, path, versionID string, opts ReadOptions) (FileInfo, error)
	ReadXL(ctx context.Context, volume, path string, readData bool) (RawFileInfo, error)
	RenameData(ctx context.Context, srcVolume, srcPath string, fi FileInfo, dstVolume, dstPath string, opts RenameOptions) (RenameDataResp, error)

	// File operations.
	ListDir(ctx context.Context, origvolume, volume, dirPath string, count int) ([]string, error)
	ReadFile(ctx context.Context, volume string, path string, offset int64, buf []byte, verifier *BitrotVerifier) (n int64, err error)
	AppendFile(ctx context.Context, volume string, path string, buf []byte) (err error)
	CreateFile(ctx context.Context, origvolume, olume, path string, size int64, reader io.Reader) error
	ReadFileStream(ctx context.Context, volume, path string, offset, length int64) (io.ReadCloser, error)
	RenameFile(ctx context.Context, srcVolume, srcPath, dstVolume, dstPath string) error
	RenamePart(ctx context.Context, srcVolume, srcPath, dstVolume, dstPath string, meta []byte) error
	CheckParts(ctx context.Context, volume string, path string, fi FileInfo) (*CheckPartsResp, error)
	Delete(ctx context.Context, volume string, path string, opts DeleteOptions) (err error)
	VerifyFile(ctx context.Context, volume, path string, fi FileInfo) (*CheckPartsResp, error)
	StatInfoFile(ctx context.Context, volume, path string, glob bool) (stat []StatInfo, err error)
	ReadParts(ctx context.Context, bucket string, partMetaPaths ...string) ([]*ObjectPartInfo, error)
	ReadMultiple(ctx context.Context, req ReadMultipleReq, resp chan<- ReadMultipleResp) error
	CleanAbandonedData(ctx context.Context, volume string, path string) error

	// Write all data, syncs the data to disk.
	// Should be used for smaller payloads.
	WriteAll(ctx context.Context, volume string, path string, b []byte) (err error)

	// Read all.
	ReadAll(ctx context.Context, volume string, path string) (buf []byte, err error)
	GetDiskLoc() (poolIdx, setIdx, diskIdx int) // Retrieve location indexes.
}
```

### ObjectLayer

唯一一个实现就是`erasureServerPools`

ObjectLayer 就是 Minio 提供的面向 Object 的接口，而`StorageAPI`则是具体的本地或者远程存储磁盘。

```go
// ObjectLayer implements primitives for object API layer.
// 重要接口
type ObjectLayer interface {
	// Locking operations on object.
	NewNSLock(bucket string, objects ...string) RWLocker

	// Storage operations.
	Shutdown(context.Context) error
	NSScanner(ctx context.Context, updates chan<- DataUsageInfo, wantCycle uint32, scanMode madmin.HealScanMode) error
	BackendInfo() madmin.BackendInfo
	Legacy() bool // Only returns true for deployments which use CRCMOD as its object distribution algorithm.
	StorageInfo(ctx context.Context, metrics bool) StorageInfo
	LocalStorageInfo(ctx context.Context, metrics bool) StorageInfo

	// Bucket operations.
	MakeBucket(ctx context.Context, bucket string, opts MakeBucketOptions) error
	GetBucketInfo(ctx context.Context, bucket string, opts BucketOptions) (bucketInfo BucketInfo, err error)
	ListBuckets(ctx context.Context, opts BucketOptions) (buckets []BucketInfo, err error)
	DeleteBucket(ctx context.Context, bucket string, opts DeleteBucketOptions) error
	ListObjects(ctx context.Context, bucket, prefix, marker, delimiter string, maxKeys int) (result ListObjectsInfo, err error)
	ListObjectsV2(ctx context.Context, bucket, prefix, continuationToken, delimiter string, maxKeys int, fetchOwner bool, startAfter string) (result ListObjectsV2Info, err error)
	ListObjectVersions(ctx context.Context, bucket, prefix, marker, versionMarker, delimiter string, maxKeys int) (result ListObjectVersionsInfo, err error)
	// Walk lists all objects including versions, delete markers.
	Walk(ctx context.Context, bucket, prefix string, results chan<- itemOrErr[ObjectInfo], opts WalkOptions) error

	// Object operations.

	// GetObjectNInfo returns a GetObjectReader that satisfies the
	// ReadCloser interface. The Close method runs any cleanup
	// functions, so it must always be called after reading till EOF
	//
	// IMPORTANTLY, when implementations return err != nil, this
	// function MUST NOT return a non-nil ReadCloser.
	GetObjectNInfo(ctx context.Context, bucket, object string, rs *HTTPRangeSpec, h http.Header, opts ObjectOptions) (reader *GetObjectReader, err error)
	GetObjectInfo(ctx context.Context, bucket, object string, opts ObjectOptions) (objInfo ObjectInfo, err error)
	PutObject(ctx context.Context, bucket, object string, data *PutObjReader, opts ObjectOptions) (objInfo ObjectInfo, err error)
	CopyObject(ctx context.Context, srcBucket, srcObject, destBucket, destObject string, srcInfo ObjectInfo, srcOpts, dstOpts ObjectOptions) (objInfo ObjectInfo, err error)
	DeleteObject(ctx context.Context, bucket, object string, opts ObjectOptions) (ObjectInfo, error)
	DeleteObjects(ctx context.Context, bucket string, objects []ObjectToDelete, opts ObjectOptions) ([]DeletedObject, []error)
	TransitionObject(ctx context.Context, bucket, object string, opts ObjectOptions) error
	RestoreTransitionedObject(ctx context.Context, bucket, object string, opts ObjectOptions) error

	// Multipart operations.
	ListMultipartUploads(ctx context.Context, bucket, prefix, keyMarker, uploadIDMarker, delimiter string, maxUploads int) (result ListMultipartsInfo, err error)
	NewMultipartUpload(ctx context.Context, bucket, object string, opts ObjectOptions) (result *NewMultipartUploadResult, err error)
	CopyObjectPart(ctx context.Context, srcBucket, srcObject, destBucket, destObject string, uploadID string, partID int,
		startOffset int64, length int64, srcInfo ObjectInfo, srcOpts, dstOpts ObjectOptions) (info PartInfo, err error)
	PutObjectPart(ctx context.Context, bucket, object, uploadID string, partID int, data *PutObjReader, opts ObjectOptions) (info PartInfo, err error)
	GetMultipartInfo(ctx context.Context, bucket, object, uploadID string, opts ObjectOptions) (info MultipartInfo, err error)
	ListObjectParts(ctx context.Context, bucket, object, uploadID string, partNumberMarker int, maxParts int, opts ObjectOptions) (result ListPartsInfo, err error)
	AbortMultipartUpload(ctx context.Context, bucket, object, uploadID string, opts ObjectOptions) error
	CompleteMultipartUpload(ctx context.Context, bucket, object, uploadID string, uploadedParts []CompletePart, opts ObjectOptions) (objInfo ObjectInfo, err error)

	GetDisks(poolIdx, setIdx int) ([]StorageAPI, error) // return the disks belonging to pool and set.
	SetDriveCounts() []int                              // list of erasure stripe size for each pool in order.

	// Healing operations.
	HealFormat(ctx context.Context, dryRun bool) (madmin.HealResultItem, error)
	HealBucket(ctx context.Context, bucket string, opts madmin.HealOpts) (madmin.HealResultItem, error)
	HealObject(ctx context.Context, bucket, object, versionID string, opts madmin.HealOpts) (madmin.HealResultItem, error)
	HealObjects(ctx context.Context, bucket, prefix string, opts madmin.HealOpts, fn HealObjectFn) error
	CheckAbandonedParts(ctx context.Context, bucket, object string, opts madmin.HealOpts) error

	// Returns health of the backend
	Health(ctx context.Context, opts HealthOptions) HealthResult

	// Metadata operations
	PutObjectMetadata(context.Context, string, string, ObjectOptions) (ObjectInfo, error)
	DecomTieredObject(context.Context, string, string, FileInfo, ObjectOptions) error

	// ObjectTagging operations
	PutObjectTags(context.Context, string, string, string, ObjectOptions) (ObjectInfo, error)
	GetObjectTags(context.Context, string, string, ObjectOptions) (*tags.Tags, error)
	DeleteObjectTags(context.Context, string, string, ObjectOptions) (ObjectInfo, error)
}
```

## 重要常量

```go
const (
	// Represents Erasure backend.
	formatBackendErasure = "xl"

	// Represents Erasure backend - single drive
	formatBackendErasureSingle = "xl-single"

	// formatErasureV1.Erasure.Version - version '1'.
	formatErasureVersionV1 = "1"

	// formatErasureV2.Erasure.Version - version '2'.
	formatErasureVersionV2 = "2"

	// formatErasureV3.Erasure.Version - version '3'.
	formatErasureVersionV3 = "3"

	// Distribution algorithm used, legacy
	formatErasureVersionV2DistributionAlgoV1 = "CRCMOD"

	// Distributed algorithm used, with N/2 default parity
	formatErasureVersionV3DistributionAlgoV2 = "SIPMOD"

	// Distributed algorithm used, with EC:4 default parity
	formatErasureVersionV3DistributionAlgoV3 = "SIPMOD+PARITY"
)

```

## s3 API: PutObject

```go
// PutObject
// http处理函数
router.Methods(http.MethodPut).Path("/{object:.+}").
  HandlerFunc(s3APIMiddleware(api.PutObjectHandler, traceHdrsS3HFlag))
// objectLayer层接口
putObject = objectAPI.PutObject
```

```go
// Validate storage class metadata if present
// x-amz-storage-class minio 只支持 REDUCED_REDUNDANCY 和 Standard
if sc := r.Header.Get(xhttp.AmzStorageClass); sc != "" {
    if !storageclass.IsValid(sc) {
       writeErrorResponse(ctx, w, errorCodes.ToAPIErr(ErrInvalidStorageClass), r.URL)
       return
    }
}
```

```go
// maximum Upload size for objects in a single operation
// 5TiB
if isMaxObjectSize(size) {
    writeErrorResponse(ctx, w, errorCodes.ToAPIErr(ErrEntityTooLarge), r.URL)
    return
}
```

```go
	objInfo, err := putObject(ctx, bucket, object, pReader, opts)
```

同一个对象对应到的`erasure set`总是同一个，这是通过确定性的 hash 算法得到的，所以 server pool 不能被修改，否则 hash 映射关系可能发生变化。

```go
// Returns always a same erasure coded set for a given input.
func (s *erasureSets) getHashedSetIndex(input string) int {
	// 通过hash得到对应erasure set，以下要素可能影响hash结果
	return hashKey(s.distributionAlgo, input, len(s.sets), s.deploymentID)
}
```

```go

// PutObject - creates an object upon reading from the input stream
// until EOF, erasure codes the data across all disk and additionally
// writes `xl.meta` which carries the necessary metadata for future
// object operations.
func (er erasureObjects) PutObject(ctx context.Context, bucket string, object string, data *PutObjReader, opts ObjectOptions) (objInfo ObjectInfo, err error) {
	return er.putObject(ctx, bucket, object, data, opts)
}
```

### block 和 shard

block （块）

`blockSize` 代表原始数据在存储时被切分的最小单位。

- 在 MinIO 中，数据在存储前被分割成多个 `block`。
- 这些 `block` 经过 **纠删码（Erasure Coding）** 计算后，生成 **数据块（data blocks）** 和 **校验块（parity blocks）**。

Shard (分片)

`shard` 是 MinIO **存储在磁盘上的物理单位**，包含 **数据块** 和 **校验块**。

- 在 **N+M 纠删码**（N 个数据块 + M 个校验块）中，每个 `shard` 对应 **一个数据块或一个校验块**。

- 例如，

  ```
  EC: 4+2
  ```

  （4 个数据块 + 2 个校验块）表示：

  - **数据被分成 4 个 `block`**。
  - **计算出 2 个额外的 `parity block`（用于恢复数据）**。
  - **最终存储 6 个 `shard`**，每个 `shard` 分别存放在不同的磁盘上。

假设数据为 300MiB，blocksize 为 10MiB, 遵循 EC: 4 + 2, shardsize = ceil(10MiB / 4) =2.5MiB，最终每个 blocksize 存储在磁盘上为 6 个 shard，4 个 data shard，6 个 parity shard

### putObject 的主要流程

1. 创建临时目录，写入分片数据
2. 如果没有加锁，获取名字空间锁，实现原子操作，避免数据竞争
3. rename 操作，包含将分片移动到目标目录以及写入 `xl.meta`元数据
4. 最后好像有提交操作，没有看懂

## 名字空间锁的实现原理 （TODO)

```go
// func (er erasureObjects) putObject
if !opts.NoLock {
		lk := er.NewNSLock(bucket, object)
		lkctx, err := lk.GetLock(ctx, globalOperationTimeout)
		if err != nil {
			return ObjectInfo{}, err
		}
		ctx = lkctx.Context()
		defer lk.Unlock(lkctx)
	}
```

`erasureObjects`中有两个关键的字段`getLockers`和`nsMutex`用于名字空间加锁。

```go
type erasureObjects struct {
	// getLockers returns list of remote and local lockers.
	getLockers func() ([]dsync.NetLocker, string)

	// Locker mutex map.
	nsMutex *nsLockMap
}
```

```go
type erasureSets struct {
	sets []*erasureObjects

	// Distributed locker clients.
	erasureLockers setsDsyncLockers

  // Distributed lock owner (constant per running instance).
	erasureLockOwner string
  // setsDsyncLockers is encapsulated type for Close()
	type setsDsyncLockers [][]dsync.NetLocker
```

```go
func (s *erasureSets) GetLockers(setIndex int) func() ([]dsync.NetLocker, string) {
	return func() ([]dsync.NetLocker, string) {
		lockers := make([]dsync.NetLocker, len(s.erasureLockers[setIndex]))
		copy(lockers, s.erasureLockers[setIndex])
    // erasureLockerOwner实际上是globalLocalNodeName
    // The name of this local node, fetched from arguments
	  // globalLocalNodeName    string
		return lockers, s.erasureLockOwner
	}
}
```

```go
// nsLockMap - namespace lock map, provides primitives to Lock,
// Unlock, RLock and RUnlock.
type nsLockMap struct {
	// Indicates if namespace is part of a distributed setup.
	isDistErasure bool
	lockMap       map[string]*nsLock
	lockMapMutex  sync.Mutex
}
// newNSLock - return a new name space lock map.
func newNSLock(isDistErasure bool) *nsLockMap {
	nsMutex := nsLockMap{
		isDistErasure: isDistErasure,
	}
	if isDistErasure {
		return &nsMutex
	}
	nsMutex.lockMap = make(map[string]*nsLock)
	return &nsMutex
}
```

### **什么是 dsync？**

`dsync` 是 **MinIO** 实现的**分布式锁（distributed locking）库**，用于在多节点环境下进行同步锁定，确保数据一致性。

**主要作用：**

- 在 **MinIO 集群** 中，确保多个 MinIO 服务器节点在 **并发访问同一资源** 时，正确管理**读/写锁**。
- 提供 **类似 `sync.Mutex` 和 `sync.RWMutex` 的分布式版本**，但适用于**分布式系统**，而不是单机环境。
- **避免数据竞争和不一致性**，保证多个 MinIO 服务器不会发生并发冲突。

#### **`NetLocker` 接口**

你提供的 `NetLocker` 接口定义了一种**分布式锁管理机制**，与 `dsync` 兼容，核心方法包括：

- `Lock()` / `Unlock()` —— **写锁**
- `RLock()` / `RUnlock()` —— **读锁**
- `Refresh()` —— **续约锁，防止锁过期**
- `ForceUnlock()` —— **强制解锁**
- `IsOnline()` / `IsLocal()` —— **检查锁服务是否在线，本地还是远程**
- `String()` / `Close()` —— **返回锁的标识 & 关闭连接**

这套机制允许 MinIO 在**多个服务器节点**间进行**分布式锁管理**，确保一致性。

### **dsync 是如何工作的？**

`dsync` **采用基于 `n/2+1` 多数决机制的分布式锁**，适用于 MinIO **分布式对象存储集群**。

**核心特点：**

1. **分布式锁（类似 `sync.Mutex`）**

   - `Lock()` / `Unlock()` 实现互斥锁，确保**多个 MinIO 节点不会同时写入相同数据**。
   - `RLock()` / `RUnlock()` 允许**多个读取者并发访问**，但不能同时有写入者。

2. **基于 Raft 的一致性算法**

   不存储锁的持久化状态，而是采用 n/2+1 机制

   - **如果大多数（n/2+1）MinIO 节点同意加锁，则锁成功。**
   - 如果未达到多数决（如部分节点宕机），加锁失败，防止数据不一致。

   - 这类似于 **Paxos/Raft** 选举机制，保证**数据一致性**。

3. **超时 & 续约（避免死锁）**

   - **锁会自动超时**，防止死锁问题。
   - `Refresh()` 允许持有锁的进程 **续约**，防止锁过期被其他进程获取。

4. **支持本地 & 远程锁**

   - **单机模式**：类似 `sync.Mutex`，锁是**本地的**。
   - **分布式模式**（`dsync`）：锁请求会被发送到**多个 MinIO 服务器**，确保整个集群同步加锁。

**MinIO 为什么需要 `dsync`？**

在 MinIO **分布式对象存储** 中，多个节点可能同时操作同一个对象（如 PUT/DELETE 操作）。
如果没有锁，可能会出现 **数据覆盖、损坏或不一致** 的问题。

**使用 `dsync` 进行分布式锁管理，MinIO 解决了这些问题**：

- **确保多个节点不会同时写入同一对象**，防止数据损坏。
- **允许多个节点同时读取数据**，提高并发性能。
- **防止死锁 & 允许锁续约**，确保锁不会永久占用资源。

### **总结**

- **`dsync` 是 MinIO 的分布式锁库**，用于**多节点同步**，确保一致性。
- 采用 **n/2+1 多数决机制**，防止数据竞争 & 保证锁安全。
- 提供 **读/写锁、强制解锁、锁续约等功能**，适用于高并发场景。
- **MinIO 通过 `dsync` 确保多个服务器不会并发写入相同对象**，保证数据一致性。

## O_DIRECT 的实际用途

## s3 API: GetObject

```go
// GetObject
router.Methods(http.MethodGet).Path("/{object:.+}").
  HandlerFunc(s3APIMiddleware(api.GetObjectHandler, traceHdrsS3HFlag))
```

- 首先加分布式读锁
- 通过读取`xl.meta`获取对象的元数据信息，`xl.meta`保存了`part`和`verison`的全部信息，注意可能存在某些磁盘上的`xl.meta`由于故障而修改落后，所以依然需要读取法定人数的磁盘，从而确定实际的元数据
- 如果 http 请求通过`part`或者`range`要求读取部分数据，最终都会转换成对多个 part 的读取，每个 part 都会划分成不同的`block`进行操作。

## 纠删码的基本原理

https://p0kt65jtu2p.feishu.cn/docx/LZ36dMN3LoZCuUxFadccNOXGnKb

假设将数据分成 4 块，采用 EC:2 冗余比例，可以将原来的数据组合成一个输入矩阵 `P = [][]byte`， 第一维表示不同的数据块，第二维表示数据块的数据，所以这里的 P 的大小为 `4 * n`，n 为每个数据块的大小

编码矩阵 E 的大小为 `6 * 4`，要求 编码矩阵的前 4 行组成的矩阵为单位矩阵，保持原来数据块数据不变，后两行用来生成冗余数据。

E \* P = C （C 表示生成的数据块和冗余块）

假设有两行数据不慎丢失，此时去掉那两行对应的数据后依然有关系 $E' * P = C'$ 成立，此时通过求逆可以得到原先的 P，也就从数据丢失中恢复了原来的数据。

## 分片上传和断点续传

分片下载可以通过前面说过的 http 请求中的`range`或者`partnumber`实现。

主要涉及的 s3 API（客户端）:

- InitiateMultipartUpload
- UploadPart
- AbortMultipartUpload
- CompleteMultipartUpload

在 minio 的客户端代码中实现了分片上传，并且支持并发上传

```go

// PutObject creates an object in a bucket.
//
// You must have WRITE permissions on a bucket to create an object.
//
//   - For size smaller than 16MiB PutObject automatically does a
//     single atomic PUT operation.
//
//   - For size larger than 16MiB PutObject automatically does a
//     multipart upload operation.
//
//   - For size input as -1 PutObject does a multipart Put operation
//     until input stream reaches EOF. Maximum object size that can
//     be uploaded through this operation will be 5TiB.
//
//     WARNING: Passing down '-1' will use memory and these cannot
//     be reused for best outcomes for PutObject(), pass the size always.
//
// NOTE: Upon errors during upload multipart operation is entirely aborted.
func (c *Client) PutObject(ctx context.Context, bucketName, objectName string, reader io.Reader, objectSize int64,
	opts PutObjectOptions,
)
```

```go
// putObjectMultipartStreamParallel uploads opts.NumThreads parts in parallel.
// This is expected to take opts.PartSize * opts.NumThreads * (GOGC / 100) bytes of buffer.
func (c *Client) putObjectMultipartStreamParallel(ctx context.Context, bucketName, objectName string,
	reader io.Reader, opts PutObjectOptions,
) (info UploadInfo, err error) {
```

```go
  // PutObjectPart
  router.Methods(http.MethodPut).Path("/{object:.+}").
    HandlerFunc(s3APIMiddleware(api.PutObjectPartHandler, traceHdrsS3HFlag)).
    Queries("partNumber", "{partNumber:.*}", "uploadId", "{uploadId:.*}")
  // ListObjectParts
  router.Methods(http.MethodGet).Path("/{object:.+}").
    HandlerFunc(s3APIMiddleware(api.ListObjectPartsHandler)).
    Queries("uploadId", "{uploadId:.*}")
  // CompleteMultipartUpload
  router.Methods(http.MethodPost).Path("/{object:.+}").
    HandlerFunc(s3APIMiddleware(api.CompleteMultipartUploadHandler)).
    Queries("uploadId", "{uploadId:.*}")
  // NewMultipartUpload
  router.Methods(http.MethodPost).Path("/{object:.+}").
    HandlerFunc(s3APIMiddleware(api.NewMultipartUploadHandler)).
    Queries("uploads", "")
  // AbortMultipartUpload
  router.Methods(http.MethodDelete).Path("/{object:.+}").
    HandlerFunc(s3APIMiddleware(api.AbortMultipartUploadHandler)).
    Queries("uploadId", "{uploadId:.*}")
```

**NewMultipartUpload**

- 生成 uuid 作为 uploadId
- 将元数据写入 `.minio.sys/multipart` uploadId 路径下

**PutObjectPart**

- 类似于 PutObejct

**CompleteMultipartUpload**

- 并没有合并 part，仍然保留每个 part
