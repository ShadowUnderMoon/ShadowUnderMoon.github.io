+++
date = '2025-01-22T19:03:56+08:00'
draft = false
title = 'Minio Get Started'
tags = [

"minio"

] 
categories = [
    "minio"
]

+++

https://blog.min.io/minio-versioning-metadata-deep-dive/

http://127.0.0.{1...4}:9000/Users/hanjing/mnt/minio{1...32}

## xl.meta数据结构

当对象大小超过128KiB后，比如`a.txt`，数据和元数据分开存储

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
          "EcDist": [
            1
          ],
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
          "PartASizes": [
            2097152
          ],
          "PartETags": null,
          "PartNums": [
            1
          ],
          "PartSizes": [
            2097152
          ],
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



## minio的启动流程

minio启动核心的核心命令为 `minio server https://minio{1...4}.example.net:9000/mnt/disk{1...4}/minio`，表示minio服务分布部署在4台服务器上总共16块磁盘上，`...`用来简写，比如 `http://minio{1...4}.example.net:9000`实际上表示`http://minio1.example.net:9000`到`http://minio4.example.net:9000`的4台主机。

go程序的入口为`main#main()`函数，直接调用了`cmd#Main`,其中做了一些命令行程序的相关操作，包括注册命令，其中`registerCommand(serverCmd)`注册服务相关命令，`cmd#ServerMain`是主要启动流程函数。

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

`buildServerCtxt`决定磁盘布局以及是否使用legacy方式，调用函数`cmd#mergeDisksLayoutFromArgs`判断是否使用拓展表达式，如果没有，`legacy = true`，否则`legacy =false`, `legacy`参数的作用我们在后面就能看到了。

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

## MinIO的DNS缓存

MinIO为了避免向外发送过多的DNS查询，所以实现了DNS缓存，默认使用`net.DefaultResolver`实际执行DNS查询，设置的DNS查询超时时间为`5s`，缓存的刷新时间在容器环境下默认为`30s`，在其他环境下为`10min`，可以通过`dns-cache-ttl`指定。

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

构造拓扑关系的主要函数实现是`mergeDisksLayoutFromArgs`，判断环境变量`MINIO_ERASURE_SET_DRIVE_COUNT`是否存在，环境变量`MINIO_ERASURE_SET_DRIVE_COUNT`表示erasure set中指定的磁盘数量，否则默认为0，表示自动设置最优结果。根据是否使用拓展表达式会走不同的逻辑。这里我们主要关心使用拓展表达式的场景`GetAllSets(setDriveCount, arg)`。（顺带一提，legacy style会走`GetAllSets(setDriveCount, args...)`，可以看到legacy style只能指定一个`server pool`）

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

`GetAllSets`主要调用了`parseEndpointSet`，通过正则表达式解析带有拓展表达式的输入参数，并返回一个`[][]string`，表示不同erasure set中的磁盘路径。这里主要对应的数据结构是`endpointSet`，主要实现两件事情，第一确定setSize，第二确定如何将endpoints分布到不同的erasure set中。

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

函数`getSetIndexes`的目的是找到合适的`setSize`，MinIO规定分布式部署setSize的取值必须属于`var setSizes = []uint64{2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16}`，首先从`SetSizes`中找到能够被`server pool size`整除的`setCounts`集合，如果自定义了`setSize`则判断自定义的`setSize`是否属于`setCounts`集合，如果属于则`setSize`设置成功，否则返回错误。如果没有设置自定义的`setSize`，函数`possibleSetCountsWithSymmetry`从`setCounts`集合中找到具有`symmetry`属性的值，MinIO中输入带拓展表达式的参数对应的pattern列表和参数中的顺序是相反的，`symmetry`过滤出能够被pattern中最后一个pattern对应的数量整除或者被整除的`setCounts`中的值，这里举一个例子`http://127.0.0.{1...4}:9000/Users/hanjing/mnt/minio{1...32}`，显然`symmetry`函数会判断4和`setCounts`中值的关系，而不是32和`setCounts`中值的关系，这可能与MinIO希望尽可能将erasure set的中不同磁盘分布到不同的节点上有关。最后取出剩余候选值中最大的值作为最终的`setSize`。

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

`endpointSet#Get`方法返回一个二维数据，第一维表示 不同的 erasure set，第二位表示erasure set中的不同磁盘。这里`getEndpoints`多重循环迭代ellipses-style对应的pattern，如果还记得的话，pattern的顺序和实际在参数中出现的顺序相反，这样得到的`endpoints`列表将不同节点上的磁盘均匀分布，后面连续取列表中的一段组成`erasure set`时，得到的`erasure set`中的磁盘也分布在不同的节点上。

### serverHandleCmdArgs函数

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

里面有一个比较重要的工具函数`isLocalHost`，通过DNS查询host对应的ip，和所有网卡对应的所有本地ip取交集,如果交集为空，说明不是本地服务器，否则是本地服务器。

函数`createServerEndpoints`将数据结构`[]poolDisksLayout`转换成`EndpointServerPools`，并指定对应的`SetupType`

对于单磁盘部署，要求使用目录路径指定输入参数，`IsLocal`一定为`true`，`SetupType`为`ErasureSDSetupType`。其他情况下根据，根据本地ip和给定的host，判断`IsLocal`，如果 host为空（MinIO称为`PathEndpointType`)，则`setupType = ErasureSetupType`，否则为`URLEndpointType`情况，如果不同`host:port`的数量等于1，则是`ErasureSetupType`，否则对应`DistErasureSetupType`，根据得到的`SetType`设置全局参数。

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

以下函数列出了Minio支持的不同模式，和上面的`SetType`之间存在对应关系。

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

## 分布式锁

```go
	// Initialize grid
	bootstrapTrace("initGrid", func() {
		logger.FatalIf(initGlobalGrid(GlobalContext, globalEndpoints), "Unable to configure server grid RPC services")
	})

	// Initialize lock grid
	bootstrapTrace("initLockGrid", func() {
		logger.FatalIf(initGlobalLockGrid(GlobalContext, globalEndpoints), "Unable to configure server lock grid RPC services")
	})
```

```go
	// Initialize distributed NS lock.
	if globalIsDistErasure {
		registerDistErasureRouters(router, endpointServerPools)
	}
```



```go
// Node holds information about a node in this cluster
type Node struct {
	*url.URL
	Pools    []int
	IsLocal  bool
	GridHost string
}
```

## HTTP服务器注册API 

- 注册分布式命名空间锁
- `registerAPIRouter`注册 s3相关的主要api

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

`registerAPIRouter`会注册主要的s3 API，这里举`GetObject`操作为例进行说明，当http method为`GET`时，如果没有命中其他的路由，则认为是`GetObject`操作，从Path中获取`object`名字，并使用`api.GetObjectHandler`进行处理和响应，`s3APIMiddleware`作为中间件，可以做一些额外的操作，比如监控和记录日志。

`api`对象中保存了一个函数引用，通过这个函数引用，能够得到全局的`ObjectLayer`对象，`ObjectLayer`实现了对象API层的基本操作。

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

## ObjectLayer的初始化流程

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

`Reduced Redundancy Storage Class`: 通过环境变量`MINIO_STORAGE_CLASS_RRS`指定，否则默认为1

`Optimized Storage Class`：通过环境变量`MINIO_STORAGE_CLASS_OPTIMIZE`指定，默认为`""`

`inline block size`: 通过环境变量`MINIO_STORAGE_CLASS_INLINE_BLOCK`指定，默认为`128KiB`,如果shard数据的大小小于`inline block size`，则会直接将数据和元数据写到同一个文件，即`xl.meta`





### MinIO的存储分层

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

ObjectLayer就是Minio提供的面向Object的接口，而`StorageAPI`则是具体的本地或者远程存储磁盘。

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

同一个对象对应到的`erasure set`总是同一个，这是通过确定性的hash算法得到的，所以server pool不能被修改，否则hash映射关系可能发生变化。

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

