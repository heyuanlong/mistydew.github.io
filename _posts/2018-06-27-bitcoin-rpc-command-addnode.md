---
layout: post
title:  "比特币 RPC 命令剖析 \"addnode\""
date:   2018-06-27 14:55:52 +0800
author: mistydew
comments: true
categories: Blockchain Bitcoin
tags: CLI bitcoin-cli 区块链 比特币
excerpt: $ bitcoin-cli addnode "node" "add|remove|onetry"
---
## 提示说明

{% highlight shell %}
addnode "node" "add|remove|onetry" # 尝试从 addnode 列表中添加或移除一个节点，或尝试连接某节点一次
{% endhighlight %}

参数：<br>
1.node（字符串，必备）节点（见 [getpeerinfo](/blog/2018/07/bitcoin-rpc-command-getpeerinfo.html) 获取的节点）。<br>
2.add|remove|onetry（字符串，必备）add 添加一个节点到列表（不会主动连接），remove 从列表移除一个节点，onetry 尝试连接到节点一次。<br>

结果：无返回值。

## 用法示例

### 比特币核心客户端

用法一：添加节点，由于无法查看 addnode 节点列表，添加 2 次进行验证。

{% highlight shell %}
$ bitcoin-cli addnode "192.168.0.2:8333" "add"
$ bitcoin-cli addnode "192.168.0.2:8333" "add"
error code: -23
error message:
Error: Node already added
{% endhighlight %}

用法二：移除节点，由于无法查看 addnode 节点列表，先添加 1 次，再移除 2 次进行验证。

{% highlight shell %}
$ bitcoin-cli addnode "192.168.0.2:8333" "add"
$ bitcoin-cli addnode "192.168.0.2:8333" "remove"
$ bitcoin-cli addnode "192.168.0.2:8333" "remove"
error code: -24
error message:
Error: Node has not been added.
{% endhighlight %}

用法三：尝试连接节点一次，使用 getpeerinfo、getconnectioncount 查看是否成功建立连接。

{% highlight shell %}
$ bitcoin-cli addnode "192.168.0.2:8333" "onetry"
{% endhighlight %}

**注：端口号可以省略，使用默认端口。**

### cURL

{% highlight shell %}
$ curl --user myusername:mypassword --data-binary '{"jsonrpc": "1.0", "id":"curltest", "method": "addnode", "params": ["192.168.0.6:8333", "onetry"] }' -H 'content-type: text/plain;' http://127.0.0.1:8332/
{"result":null,"error":null,"id":"curltest"}
{% endhighlight %}

## 源码剖析
addnode 对应的函数在“rpcserver.h”文件中被引用。

{% highlight C++ %}
extern UniValue addnode(const UniValue& params, bool fHelp); // 添加节点
{% endhighlight %}

实现在“rpcnet.cpp”文件中。

{% highlight C++ %}
UniValue addnode(const UniValue& params, bool fHelp)
{
    string strCommand;
    if (params.size() == 2) // 若有 2 个参数
        strCommand = params[1].get_str(); // 获取第二个参数作为命令
    if (fHelp || params.size() != 2 ||
        (strCommand != "onetry" && strCommand != "add" && strCommand != "remove")) // 参数必须为 2 个，命令必须为 "onetry" "add" "remove" 这 3 个中的一个
        throw runtime_error( // 命令帮助反馈
            "addnode \"node\" \"add|remove|onetry\"\n"
            "\nAttempts add or remove a node from the addnode list.\n"
            "Or try a connection to a node once.\n"
            "\nArguments:\n"
            "1. \"node\"     (string, required) The node (see getpeerinfo for nodes)\n"
            "2. \"command\"  (string, required) 'add' to add a node to the list, 'remove' to remove a node from the list, 'onetry' to try a connection to the node once\n"
            "\nExamples:\n"
            + HelpExampleCli("addnode", "\"192.168.0.6:8333\" \"onetry\"")
            + HelpExampleRpc("addnode", "\"192.168.0.6:8333\", \"onetry\"")
        );

    string strNode = params[0].get_str(); // 获取指定节点

    if (strCommand == "onetry") // 尝试连接操作
    {
        CAddress addr;
        OpenNetworkConnection(addr, NULL, strNode.c_str()); // 连接一次
        return NullUniValue;
    }

    LOCK(cs_vAddedNodes);
    vector<string>::iterator it = vAddedNodes.begin();
    for(; it != vAddedNodes.end(); it++) // 检查列表中是否存在该节点，若存在，则迭代器指向该节点，否则，指向列表末尾
        if (strNode == *it)
            break;

    if (strCommand == "add") // 添加节点操作
    {
        if (it != vAddedNodes.end()) // 若列表中已存在该节点
            throw JSONRPCError(RPC_CLIENT_NODE_ALREADY_ADDED, "Error: Node already added");
        vAddedNodes.push_back(strNode); // 添加该节点到列表
    }
    else if(strCommand == "remove") // 移除节点操作
    {
        if (it == vAddedNodes.end()) // 若列表中不存在该节点
            throw JSONRPCError(RPC_CLIENT_NODE_NOT_ADDED, "Error: Node has not been added.");
        vAddedNodes.erase(it); // 从列表中擦除该节点
    }

    return NullUniValue; // 无返回值
}
{% endhighlight %}

基本流程：<br>
1.处理命令帮助和参数个数。<br>
2.获取指定节点，并尝试连接一次。<br>
3.在添加节点的列表中查找指定节点。<br>
4.添加该节点到列表。<br>
5.或从列表中移除该节点。

添加节点的列表 vAddedNodes 对象在“net.h”文件中被引用。

{% highlight C++ %}
extern std::vector<std::string> vAddedNodes; // 添加的节点列表
{% endhighlight %}

定义在“net.cpp”文件中。

{% highlight C++ %}
vector<std::string> vAddedNodes; // 通过 "addnode" 命令添加的节点列表
{% endhighlight %}

第二步，尝试连接一次，调用 OpenNetworkConnection(addr, NULL, strNode.c_str()) 函数，声明在“net.h”文件中。

{% highlight C++ %}
bool OpenNetworkConnection(const CAddress& addrConnect, CSemaphoreGrant *grantOutbound = NULL, const char *strDest = NULL, bool fOneShot = false); // 打开网络连接
{% endhighlight %}

实现在“net.cpp”文件中。

{% highlight C++ %}
// if successful, this moves the passed grant to the constructed node // 如果连接成功，这将通过授权移动到构造的节点
bool OpenNetworkConnection(const CAddress& addrConnect, CSemaphoreGrant *grantOutbound, const char *pszDest, bool fOneShot)
{
    //
    // Initiate outbound network connection // 初始化向外的网络连接
    //
    boost::this_thread::interruption_point();
    if (!pszDest) {
        if (IsLocal(addrConnect) || // 检查是否为本地节点
            FindNode((CNetAddr)addrConnect) || CNode::IsBanned(addrConnect) || // 是否被禁止连接
            FindNode(addrConnect.ToStringIPPort()))
            return false;
    } else if (FindNode(std::string(pszDest)))
        return false;

    CNode* pnode = ConnectNode(addrConnect, pszDest); // 连接到指定节点
    boost::this_thread::interruption_point();

    if (!pnode)
        return false;
    if (grantOutbound) // NULL
        grantOutbound->MoveTo(pnode->grantOutbound);
    pnode->fNetworkNode = true;
    if (fOneShot)
        pnode->fOneShot = true;

    return true;
}
{% endhighlight %}

调用 ConnectNode(addrConnect, pszDest) 函数连接到指定节点，该函数实现在“net.cpp”文件中。

{% highlight C++ %}
CNode* ConnectNode(CAddress addrConnect, const char *pszDest)
{
    if (pszDest == NULL) {
        if (IsLocal(addrConnect)) // 本地连接
            return NULL;

        // Look for an existing connection
        CNode* pnode = FindNode((CService)addrConnect); // 发现一个已存在的连接
        if (pnode)
        {
            pnode->AddRef(); // 引用计数加 1
            return pnode;
        }
    }

    /// debug print
    LogPrint("net", "trying connection %s lastseen=%.1fhrs\n", // 调试打印
        pszDest ? pszDest : addrConnect.ToString(),
        pszDest ? 0.0 : (double)(GetAdjustedTime() - addrConnect.nTime)/3600.0);

    // Connect // 开始连接
    SOCKET hSocket;
    bool proxyConnectionFailed = false;
    if (pszDest ? ConnectSocketByName(addrConnect, hSocket, pszDest, Params().GetDefaultPort(), nConnectTimeout, &proxyConnectionFailed) :
                  ConnectSocket(addrConnect, hSocket, nConnectTimeout, &proxyConnectionFailed)) // 进行连接
    {
        if (!IsSelectableSocket(hSocket)) { // 若连接创建不成功
            LogPrintf("Cannot create connection: non-selectable socket created (fd >= FD_SETSIZE ?)\n");
            CloseSocket(hSocket); // 关闭套接字
            return NULL;
        }

        addrman.Attempt(addrConnect); // 接收该连接的连入请求

        // Add node
        CNode* pnode = new CNode(hSocket, addrConnect, pszDest ? pszDest : "", false); // 创建一个对应的节点对象
        pnode->AddRef(); // 引用计数加 1

        {
            LOCK(cs_vNodes);
            vNodes.push_back(pnode); // 加入成功建立连接的节点列表
        }

        pnode->nTimeConnected = GetTime(); // 记录建立连接的时间

        return pnode;
    } else if (!proxyConnectionFailed) { // 若连接节点失败，且失败是由连接到代理的问题导致，标记该连接为尝试
        // If connecting to the node failed, and failure is not caused by a problem connecting to
        // the proxy, mark this as an attempt.
        addrman.Attempt(addrConnect);
    }

    return NULL;
}
{% endhighlight %}

调用 ConnectSocket(addrConnect, hSocket, nConnectTimeout, &proxyConnectionFailed) 函数连接到 socket，声明在“netbase.h”文件中。

{% highlight C++ %}
bool ConnectSocket(const CService &addr, SOCKET& hSocketRet, int nTimeout, bool *outProxyConnectionFailed = 0); // 连接套接字
{% endhighlight %}

实现在“netbase.cpp”文件中。

{% highlight C++ %}
bool ConnectSocket(const CService &addrDest, SOCKET& hSocketRet, int nTimeout, bool *outProxyConnectionFailed)
{
    proxyType proxy;
    if (outProxyConnectionFailed)
        *outProxyConnectionFailed = false;

    if (GetProxy(addrDest.GetNetwork(), proxy)) // 通过代理连接
        return ConnectThroughProxy(proxy, addrDest.ToStringIP(), addrDest.GetPort(), hSocketRet, nTimeout, outProxyConnectionFailed);
    else // no proxy needed (none set for target network)
        return ConnectSocketDirectly(addrDest, hSocketRet, nTimeout); // 直接连接到套接字
}
{% endhighlight %}

又调用 ConnectSocketDirectly(addrDest, hSocketRet, nTimeout) 函数直接连接到指定节点，该函数实现在“netbase.cpp”文件中。

{% highlight C++ %}
bool static ConnectSocketDirectly(const CService &addrConnect, SOCKET& hSocketRet, int nTimeout)
{
    hSocketRet = INVALID_SOCKET;

    struct sockaddr_storage sockaddr;
    socklen_t len = sizeof(sockaddr);
    if (!addrConnect.GetSockAddr((struct sockaddr*)&sockaddr, &len)) { // 获取 sock 地址
        LogPrintf("Cannot connect to %s: unsupported network\n", addrConnect.ToString());
        return false;
    }

    SOCKET hSocket = socket(((struct sockaddr*)&sockaddr)->sa_family, SOCK_STREAM, IPPROTO_TCP); // socket
    if (hSocket == INVALID_SOCKET)
        return false;

    int set = 1;
#ifdef SO_NOSIGPIPE
    // Different way of disabling SIGPIPE on BSD
    setsockopt(hSocket, SOL_SOCKET, SO_NOSIGPIPE, (void*)&set, sizeof(int));
#endif

    //Disable Nagle's algorithm
#ifdef WIN32
    setsockopt(hSocket, IPPROTO_TCP, TCP_NODELAY, (const char*)&set, sizeof(int));
#else
    setsockopt(hSocket, IPPROTO_TCP, TCP_NODELAY, (void*)&set, sizeof(int));
#endif

    // Set to non-blocking
    if (!SetSocketNonBlocking(hSocket, true))
        return error("ConnectSocketDirectly: Setting socket to non-blocking failed, error %s\n", NetworkErrorString(WSAGetLastError()));

    if (connect(hSocket, (struct sockaddr*)&sockaddr, len) == SOCKET_ERROR) // connect
    {
        int nErr = WSAGetLastError();
        // WSAEINVAL is here because some legacy version of winsock uses it
        if (nErr == WSAEINPROGRESS || nErr == WSAEWOULDBLOCK || nErr == WSAEINVAL)
        {
            struct timeval timeout = MillisToTimeval(nTimeout);
            fd_set fdset;
            FD_ZERO(&fdset);
            FD_SET(hSocket, &fdset);
            int nRet = select(hSocket + 1, NULL, &fdset, NULL, &timeout);
            if (nRet == 0)
            {
                LogPrint("net", "connection to %s timeout\n", addrConnect.ToString());
                CloseSocket(hSocket);
                return false;
            }
            if (nRet == SOCKET_ERROR)
            {
                LogPrintf("select() for %s failed: %s\n", addrConnect.ToString(), NetworkErrorString(WSAGetLastError()));
                CloseSocket(hSocket);
                return false;
            }
            socklen_t nRetSize = sizeof(nRet);
#ifdef WIN32
            if (getsockopt(hSocket, SOL_SOCKET, SO_ERROR, (char*)(&nRet), &nRetSize) == SOCKET_ERROR)
#else
            if (getsockopt(hSocket, SOL_SOCKET, SO_ERROR, &nRet, &nRetSize) == SOCKET_ERROR)
#endif
            {
                LogPrintf("getsockopt() for %s failed: %s\n", addrConnect.ToString(), NetworkErrorString(WSAGetLastError()));
                CloseSocket(hSocket);
                return false;
            }
            if (nRet != 0)
            {
                LogPrintf("connect() to %s failed after select(): %s\n", addrConnect.ToString(), NetworkErrorString(nRet));
                CloseSocket(hSocket);
                return false;
            }
        }
#ifdef WIN32
        else if (WSAGetLastError() != WSAEISCONN)
#else
        else
#endif
        {
            LogPrintf("connect() to %s failed: %s\n", addrConnect.ToString(), NetworkErrorString(WSAGetLastError()));
            CloseSocket(hSocket);
            return false;
        }
    }

    hSocketRet = hSocket;
    return true;
}
{% endhighlight %}

这里可以看到，经过层层封装最终调用 socket 和 connect 系统调用连接到指定 IP 和端口。

Thanks for your time.

## 参照
* [Developer Documentation - Bitcoin](https://bitcoin.org/en/developer-documentation)
* [Bitcoin Developer Reference - Bitcoin](https://bitcoin.org/en/developer-reference#addnode)
* [精通比特币（第二版） \| 巴比特图书](http://book.8btc.com/masterbitcoin2cn)
* [...](https://github.com/mistydew/blockchain)
