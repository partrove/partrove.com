+++
title = 'Flannel 绑定网卡流程'
date = 2024-03-06T19:00:40+08:00
draft = false
isCJKLanguage = true
+++

> Flannel版本：v0.24.2  

# 命令行参数  

先来看看在 flanneld 启动过程中跟绑定网卡有关联的几个命令行参数。  

| 参数               | 类型     | 默认值 | 含义                                                                       |
| ---------------- | ------ | --- | ------------------------------------------------------------------------ |
| -iface           | value  | 空   | 指定flannel用于主机间通信的接口，可以是IP或者名称。这个参数可以多次使用以指定多个接口，进程启动时会按顺序检查，并且会返回第一个匹配项。 |
| -iface-can-reach | string | 空   | 指定一个确定可用的接口用于主机间通信。可以在主机上执行 ip r get <其他主机IP> 来获取通信接口。                   |
| -iface-regex     | value  | 空   | 同第一个参数。通过正则表达式检查可用的接口，也可以多次使用。                                           |
| -public-ip       | string | 空   | 用于主机间通信的本机IPv4地址                                                         |
| -public-ipv6     | string | 空   | 用于主机间通信的本机IPv6地址                                                         |
# 启动流程  

flannel 的代码比较清晰，大体的启动流程代码基本就在 *main.go* 里面。  

## 初始化子网管理器  

在 *main.go#224*  

```go
sm, err := newSubnetManager(ctx)
```  

会初始化一个 SubnetManager，对应的方法在 *main.go#170*  

```go
func newSubnetManager(ctx context.Context) (subnet.Manager, error) {  
    if opts.kubeSubnetMgr {  
       return kube.NewSubnetManager(
       // ...
       )
    cfg := &etcd.EtcdConfig{
    // ...
    }
    // ...
}
```

### 使用Kubernetes API 

如果命令行启动参数同时指定了 `-kube-subnet-mgr` ，那么这里就会初始化一个连接 Kubernetes API 的 SubnetManager，对应的数据也会通过 Kubernetes API 进行存储。  

在 *pkg/subnet/kube/kube.go#81*，定义了 Kubernetes API SubnetManager 的初始化方法，其中跟接口绑定相关的代码如下    

```go
func NewSubnetManager(ctx context.Context, apiUrl, kubeconfig, prefix, netConfPath string, setNodeNetworkUnavailable bool) (subnet.Manager, error) {
// ...
	netConf, err := os.ReadFile(netConfPath)  
	if err != nil {  
	    return nil, fmt.Errorf("failed to read net conf: %v", err)  
	}  
  
	sc, err := subnet.ParseConfig(string(netConf))  
	if err != nil {  
	    return nil, fmt.Errorf("error parsing subnet config: %s", err)  
	}  
  
	sm, err := newKubeSubnetManager(ctx, c, sc, nodeName, prefix)  
	if err != nil {  
	    return nil, fmt.Errorf("error creating network manager: %s", err)  
	}
// ...
}
```  

netConfPath 是通过命令行参数传入的网络配置文件 `-net-config-path`，默认是 */etc/kube-flannel/net-conf.json* 。程序先读取 netConfPath ，然后再调用 `subnet.ParseConfig()` 解析配置。  

解析网络配置的方法在 *pkg/subnet/config.go#60*。  

```go
func ParseConfig(s string) (*Config, error) {
	cfg := new(Config)
	// Enable ipv4 by default
	cfg.EnableIPv4 = true
	err := json.Unmarshal([]byte(s), cfg)
	if err != nil {
		return nil, err
	}

	bt, err := parseBackendType(cfg.Backend)
	if err != nil {
		return nil, err
	}
	cfg.BackendType = bt

	cfg.Networks = make([]ip.IP4Net, 0)
	cfg.IPv6Networks = make([]ip.IP6Net, 0)

	return cfg, nil
}
```  

这里可以看到，默认会将 **EnableIPv4** 设置为 true。  

### 使用etcd存储  

如果命令行启动参数没有指定使用 Kubernetes API，那么默认就会返回一个使用 etcd 的 SubnetManager。

etcd 类型的 SubnetManager 仅初始化了 etcd 连接配置和节点的容器子网配置，初始化的代码中暂不涉及主机接口相关，这里就忽略了。  

## 获取IP栈配置  

SubnetManager 初始化好后，再回到 *main.go#248* ，会接着读取 IP 栈相关的信息。

```go
// Fetch the network config (i.e. what backend to use etc..).  
config, err := getConfig(ctx, sm)  
if err == errCanceled {  
    wg.Wait()  
    os.Exit(0)  
}
```  

程序首先会调用 `getConfig()` 从前面初始化好的 SubnetManager 里面读取配置，对应 *pkg/subnet/kube/kube.go#276* 和 *pkg/subnet/etcd/local_manager.go#86* 分别都实现了 `subnet.Manager` 的 `GetNetworkConfig()` 方法。  

对于使用 Kubernetes API 作为存储的方式，因为前面初始化 SubnetManager 的时候已经解析好了配置，所以这里就直接返回了对应的 `*subnet.Config`  

```go
func (ksm *kubeSubnetManager) GetNetworkConfig(ctx context.Context) (*subnet.Config, error) {  
    return ksm.subnetConf, nil  
}
```  

对于使用 etcd 作为存储的方式，在 `GetNetworkConfig()` 的同时会做配置解析的动作，其过程跟前面 Kubernetes API 方式一样。同样的，在调用 `subnet.ParseConfig()` 的过程中默认会将 **EnableIPv4** 设置为 true。

```go
func (m *LocalManager) GetNetworkConfig(ctx context.Context) (*subnet.Config, error) {  
    cfg, err := m.registry.getNetworkConfig(ctx)  
    if err != nil {  
       return nil, err  
    }  
  
    config, err := subnet.ParseConfig(cfg)  
    if err != nil {  
       return nil, err  
    }  
    err = subnet.CheckNetworkConfig(config)  
    if err != nil {  
       return nil, err  
    }  
    return config, nil  
}
```  

获取到 `*subnet.Config` 之后，回到 *main.go#255* 继续执行  

```go
// Get ip family stack  
ipStack, stackErr := ipmatch.GetIPFamily(config.EnableIPv4, config.EnableIPv6)  
if stackErr != nil {  
    log.Error(stackErr.Error())  
    os.Exit(1)  
}
```  

这里就会调用 *pkg/ipmatch/match.go#42* 的 `ipmatch.GetIPFamily()`  去构造一个 ipStack 对象  

```go
func GetIPFamily(autoDetectIPv4, autoDetectIPv6 bool) (int, error) {  
    if autoDetectIPv4 && !autoDetectIPv6 {  
       return ipv4Stack, nil  
    } else if !autoDetectIPv4 && autoDetectIPv6 {  
       return ipv6Stack, nil  
    } else if autoDetectIPv4 && autoDetectIPv6 {  
       return dualStack, nil  
    }  
    return noneStack, errors.New("none defined stack")  
}
```  

这里的逻辑比较简单，分别是几种不同配置的组合。  

如果 netConfigPath */etc/kube-flannel/net-conf.json* 中 IPv4 和 IPv6 都没有指定，前面已经分析过，在构造 SubnetManager 和获取 NetworkConfig 的时候，默认会将 **EnableIPv4** 设置为 true，所以这里就会返回 ipv4Stack。  

## 查询要绑定的接口

继续回到 *main.go#262*，接下来就进入主机接口的绑定流程了。这里首先会根据命令行参数构造一个 `ipmatch.PublicIPOpts{}` 对象，如果 flanneld 进程启动时没有指定 `-public-ip` 和 `-public-ipv6` 那么这里就是空的。  

```go
// Work out which interface to use  
var extIface *backend.ExternalInterface  
optsPublicIP := ipmatch.PublicIPOpts{  
    PublicIP:   opts.publicIP,  
    PublicIPv6: opts.publicIPv6,  
}
```  

然后在 *main.go#268*，正式进入查找接口的流程。

这里会根据前面 5 个命令行参数的不同组合，分别使用不同的传参，去调用 `ipmatch.LookupExtIface()` 去查询要绑定的接口。

```go
	// Check the default interface only if no interfaces are specified
	if len(opts.iface) == 0 && len(opts.ifaceRegex) == 0 && len(opts.ifaceCanReach) == 0 {
		if len(opts.publicIP) > 0 {
			extIface, err = ipmatch.LookupExtIface(opts.publicIP, "", "", ipStack, optsPublicIP)
		} else {
			extIface, err = ipmatch.LookupExtIface(opts.publicIPv6, "", "", ipStack, optsPublicIP)
		}
		if err != nil {
			log.Error("Failed to find any valid interface to use: ", err)
			os.Exit(1)
		}
	} else {
		// Check explicitly specified interfaces
		for _, iface := range opts.iface {
			extIface, err = ipmatch.LookupExtIface(iface, "", "", ipStack, optsPublicIP)
			if err != nil {
				log.Infof("Could not find valid interface matching %s: %s", iface, err)
			}

			if extIface != nil {
				break
			}
		}

		// Check interfaces that match any specified regexes
		if extIface == nil {
			for _, ifaceRegex := range opts.ifaceRegex {
				extIface, err = ipmatch.LookupExtIface("", ifaceRegex, "", ipStack, optsPublicIP)
				if err != nil {
					log.Infof("Could not find valid interface matching %s: %s", ifaceRegex, err)
				}

				if extIface != nil {
					break
				}
			}
		}

		if extIface == nil && len(opts.ifaceCanReach) > 0 {
			extIface, err = ipmatch.LookupExtIface("", "", opts.ifaceCanReach, ipStack, optsPublicIP)
			if err != nil {
				log.Infof("Could not find valid interface matching ifaceCanReach: %s: %s", opts.ifaceCanReach, err)
			}
		}

		if extIface == nil {
			// Exit if any of the specified interfaces do not match
			log.Error("Failed to find interface to use that matches the interfaces and/or regexes provided")
			os.Exit(1)
		}
	}
```

当 flanneld 进程启动时没有指定 `-iface` 、`-iface-can-reach` 和 `-iface-regex` 这三个参数，进一步会判断是否指定了 `-public-ip` 参数，如果有，就将 `-public-ip` 这个参数的值传递给 `ipmatch.LookupExtIface()` 继续查找；如果没有指定 `-public-ip` 参数，就将 `-public-ipv6` 的参数值传递进去继续查找。  

> #Q 如果 `-public-ip` 和 `-public-ipv6` 都没有指定，默认还是会走 ipv6 的查找流程，会不会有坑？  

反之，会先从 `-iface` 参数值开始查找，如果 `-iface` 为空，或者指定的 interfaces 都不可用，那就继续进入 `if extIface == nil {...}` 流程，再继续循环查找 `-iface-regex` ，如果还是没找到，再继续用 `-iface-ca-reach` 去查找，实在没找到就返回报错。  

从上面的流程可以看出，几个参数的优先级依次是：  

1. `-iface`  
2. `-iface-regex`
3. `-iface-can-reach`
4. `-public-ip`
5. `-public-ipv6`
## LookupExtIface()  

这是定义在 *pkg/ipmatch/match.go#53* 里面的函数  

```go
func LookupExtIface(ifname string, ifregexS string, ifcanreach string, ipStack int, opts PublicIPOpts) (*backend.ExternalInterface, error) {}
```

函数接收 5 个形参，分别是  

- ifname：接口名称，对应可以是命令行参数的一个 `-iface`，也可以是 `-public-ip` 或 `-public-ipv6`  
- ifregexS：用于查找接口的正则表达式字符串，对应命令行参数的一个 `-iface-regex`   
- ifcanreach：可以使用的接口，对应命令行参数 `-iface-can-reach`  
- ipStack：对应上面的 IP 栈配置
- opts：对应前面构造出的 optsPublicIP，里面也存储了 `-public-ip` 和 `-public-ipv6` 信息  

下面逐行分析这个函数  

```go
if ifregexS != "" {  
    ifregex, err = regexp.Compile(ifregexS)  
    if err != nil {  
       return nil, fmt.Errorf("could not compile the IP address regex '%s': %w", ifregexS, err)  
    }  
}
```  

如果传入的接口查找正则不为空，就先将正则编译出来。  

```go
// Check ip family stack  
if ipStack == noneStack {  
    return nil, fmt.Errorf("none matched ip stack")  
}
```  

如果传入的 IP 栈为 none，则返回错误。根据前面的代码分析，因为有默认设置 **EnableIPv4** ，所以这里一般不会出错。  

由于接下来的判断逻辑很长，下面就拆分成几种情况来分析

### 传入的 ifname 不为空，且是 IP 地址  

先用 `net.ParseIP(ifname)` 解析传入的 ifname，如果是 IP 地址的话，继续往下检查传入的 IP 栈配置 ipStack。  

```go
if ifaceAddr = net.ParseIP(ifname); ifaceAddr != nil {  
    log.Infof("Searching for interface using %s", ifaceAddr)  
    switch ipStack {  
    case ipv4Stack:  
       iface, err = ip.GetInterfaceByIP(ifaceAddr)  
       if err != nil {  
          return nil, fmt.Errorf("error looking up interface %s: %s", ifname, err)  
       }  
    case ipv6Stack:  
       iface, err = ip.GetInterfaceByIP6(ifaceAddr)  
       if err != nil {  
          return nil, fmt.Errorf("error looking up v6 interface %s: %s", ifname, err)  
       }  
    case dualStack:  
       if ifaceAddr.To4() != nil {  
          iface, err = ip.GetInterfaceByIP(ifaceAddr)  
          if err != nil {  
             return nil, fmt.Errorf("error looking up interface %s: %s", ifname, err)  
          }  
       }  
       if len(opts.PublicIPv6) > 0 {  
          if ifaceV6Addr = net.ParseIP(opts.PublicIPv6); ifaceV6Addr != nil {  
             v6Iface, err := ip.GetInterfaceByIP6(ifaceV6Addr)  
             if err != nil {  
                return nil, fmt.Errorf("error looking up v6 interface %s: %s", opts.PublicIPv6, err)  
             }  
             if ifaceAddr.To4() == nil {  
                iface = v6Iface  
                ifaceAddr = nil  
             } else {  
                if iface.Name != v6Iface.Name {  
                   return nil, fmt.Errorf("v6 interface %s must be the same with v4 interface %s", v6Iface.Name, iface.Name)  
                }  
             }  
          }  
       }  
    }  
}
```  

上面这一端逻辑，根据传入 ipStack 参数判断，

- 如果是 ipv4Stack，调用 `ip.GetInterfaceByIP(ifaceAddr)` 获取 IPv4 地址对应的接口对象；
- 如果是 ipv6Stack，调用 `ip.GetInterfaceByIP6(ifaceAddr)` 获取 IPv6 地址对应的接口对象；
- 如果启用了双栈 dualStack，先判断上面解析出来的 IP 地址如果是 IPv4 的话，还是先调用 `ip.GetInterfaceByIP(ifaceAddr)` 获取 IPv4 地址对应的接口对象。如果同时传入的 opts 里面还包含 PublicIPv6 信息的话，会解析出 ifaceV6Addr 之后，继续再调用 `ip.GetInterfaceByIP6(ifaceV6Addr)` 获取 IPv6 地址对应的接口对象，此时再判断如果前面的 ifaceAddr 不是 IPv4 的话，那么找到的接口就是 V6Iface。否则再判断前面找到的 iface 名和 V6Iface 名是否一致，不一致报错退出。

### 传入的 ifname 不为空，且是接口名称  

这一段就比较简单， 如果上面的 `net.ParseIP(ifname)` 解析出来 ifname 不是 IP 地址，就进入这一段 `else` 流程。    

```go
// ...
} else {  
    iface, err = net.InterfaceByName(ifname)  
    if err != nil {  
       return nil, fmt.Errorf("error looking up interface %s: %s", ifname, err)  
    }  
}
```

这里就会调用 `net.InterfaceByName(ifname)` 直接从主机上按名字查找可用的接口。

### ifname 为空，ifregexS 不为空  

根据传入的参数，如果上面的 `if len(ifname) > 0 {}` 不符合条件，就会进入下面的 `else if ifregex != nil {}`  逻辑。  

这里首先会从主机上将网卡清单全部取出来  

```go
// Use the regex if specified and the iface option for matching a specific ip or name is not used  
ifaces, err := net.Interfaces()  
if err != nil {  
    return nil, fmt.Errorf("error listing all interfaces: %s", err)  
}
```  

然后下面用了一个 **label loop**，去循环匹配主机上的网卡  

```go
    // Check IP  
ifaceLoop:  
    for _, ifaceToMatch := range ifaces {  
       switch ipStack {  
       case ipv4Stack:  
          ifaceIPs, err := ip.GetInterfaceIP4Addrs(&ifaceToMatch)  
          if err != nil {  
             // Skip if there is no IPv4 address  
             continue  
          }  
          if matched := matchIP(ifregex, ifaceIPs); matched != nil {  
             ifaceAddr = matched  
             iface = &ifaceToMatch  
             break ifaceLoop  
          }  
       case ipv6Stack:  
          ifaceIPs, err := ip.GetInterfaceIP6Addrs(&ifaceToMatch)  
          if err != nil {  
             // Skip if there is no IPv6 address  
             continue  
          }  
          if matched := matchIP(ifregex, ifaceIPs); matched != nil {  
             ifaceV6Addr = matched  
             iface = &ifaceToMatch  
             break ifaceLoop  
          }  
       case dualStack:  
          ifaceIPs, err := ip.GetInterfaceIP4Addrs(&ifaceToMatch)  
          if err != nil {  
             // Skip if there is no IPv4 address  
             continue  
          }  
  
          ifaceV6IPs, err := ip.GetInterfaceIP6Addrs(&ifaceToMatch)  
          if err != nil {  
             // Skip if there is no IPv6 address  
             continue  
          }  
  
          if matched := matchIP(ifregex, ifaceIPs); matched != nil {  
             ifaceAddr = matched  
          } else {  
             continue  
          }  
          if matched := matchIP(ifregex, ifaceV6IPs); matched != nil {  
             ifaceV6Addr = matched  
             iface = &ifaceToMatch  
             break ifaceLoop  
          }  
       }  
    }
```  

- 在 `for` 循环中，逐个处理主机取到的网卡接口；
- 在 `switch ... case` 语句中，还是根据传入的 ipStack 参数，先用 ifaceToMatch 取出 IP 地址，然后正则匹配；如果没有拿到 IP 地址，`continue` 继续处理下一个接口；如果正则匹配成功，`break ifaceLoop` 跳出循环。  

上面这个 ifaceLoop 中是用 IP 地址进行匹配，如果传入的 ifregexS 用的不是 IP 地址匹配规则，而是接口名称匹配规则，这个 ifaceLoop 还是匹配不到，这时会继续进入下面的判断  

```go
// Check Name  
if iface == nil && (ifaceAddr == nil || ifaceV6Addr == nil) {  
    for _, ifaceToMatch := range ifaces {  
       if ifregex.MatchString(ifaceToMatch.Name) {  
          iface = &ifaceToMatch  
          break  
       }  
    }  
}
```  

这里就会调用 `ifregex.MatchString(ifaceToMatch.Name)` 去用接口名称进行匹配。  

如果最后还是没有找到接口，会打印对应的报错日志

```go
// Check that nothing was matched  
if iface == nil {  
    var availableFaces []string  
    for _, f := range ifaces {  
       var ipaddr []net.IP  
       switch ipStack {  
       case ipv4Stack, dualStack:  
          ipaddr, _ = ip.GetInterfaceIP4Addrs(&f) // We can safely ignore errors. We just won't log any ip  
       case ipv6Stack:  
          ipaddr, _ = ip.GetInterfaceIP6Addrs(&f) // We can safely ignore errors. We just won't log any ip  
       }  
       availableFaces = append(availableFaces, fmt.Sprintf("%s:%v", f.Name, ipaddr))  
    }  
  
    return nil, fmt.Errorf("Could not match pattern %s to any of the available network interfaces (%s)", ifregexS, strings.Join(availableFaces, ", "))  
}
```  

### ifname/ifregexS 都为空，ifcanreach 不为空  

首先会判断主机 OS，Windows 不支持这个参数  

```go
if runtime.GOOS == "windows" {  
    return nil, fmt.Errorf("ifcanreach is not supported on windows")  
}
```  

然后就会调用 `ip.GetInterfaceBySpecificIPRouting()` 去通过主机路由信息查找对应的接口。

```go
log.Info("Determining interface to use based on given ifcanreach: ", ifcanreach)  
if iface, ifaceAddr, err = ip.GetInterfaceBySpecificIPRouting(net.ParseIP(ifcanreach)); err != nil {  
    return nil, fmt.Errorf("failed to get ifcanreach based interface: %s", err)  
}
```  

### ifname/ifregexS/ifcanreach 都为空  

如果三个参数都为空，那么就会根据主机上的路由表，去查找**默认路由**对应的接口  

```go
// ...
} else {  
    log.Info("Determining IP address of default interface")  
    switch ipStack {  
    case ipv4Stack:  
       if iface, err = ip.GetDefaultGatewayInterface(); err != nil {  
          return nil, fmt.Errorf("failed to get default interface: %w", err)  
       }  
    case ipv6Stack:  
       if iface, err = ip.GetDefaultV6GatewayInterface(); err != nil {  
          return nil, fmt.Errorf("failed to get default v6 interface: %w", err)  
       }  
    case dualStack:  
       if iface, err = ip.GetDefaultGatewayInterface(); err != nil {  
          return nil, fmt.Errorf("failed to get default interface: %w", err)  
       }  
       v6Iface, err := ip.GetDefaultV6GatewayInterface()  
       if err != nil {  
          return nil, fmt.Errorf("failed to get default v6 interface: %w", err)  
       }  
       if iface.Name != v6Iface.Name {  
          return nil, fmt.Errorf("v6 default route interface %s "+  
             "must be the same with v4 default route interface %s", v6Iface.Name, iface.Name)  
       }  
    }  
}
```
