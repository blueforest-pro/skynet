
# demo 解析
https://www.jianshu.com/p/d9ecdf852dcb

按照skynet的惯例，网络服务器一般是用gate/watchdog/agent三剑客。这个例子也不例外，因为skynet底层的socket不能算是非常的易用。

服务端与node.js的客户端之间使用json传输数据。

目录结构仍然像第2篇中的一样，项目根目录为skynet_howto。

搭建基本的网络功能:

main.lua 管理服务，启动服务。
agent.lua agent服务，负责请求路由。
watchdog.lua 看门狗，负责启动agent。
可能有的人疑问，为什么没有gate.lua?gate服务已经被标准化了，放在skynet框架里了。所以不需要自己写gate.lua，直接拿来用就好了。

那么gate/agent/watchdog三者的关系是什么？还是从网络请求的处理过程来讲比较直观，首先一个连接进来，先到gate，gate会给watchdog发一个请求。watchdog就会启动一个agent。agent启动以后会给gate发个请求forward，gate就会给连接加上agent属性。当这个连接再有数据进来的时候，还是经过gate，但是gate检查到这个连接已经有agent属性以后，数据就直接发给agent了，不会再发给watchdog。

用一个简单的图形表示就如下。

``` bash
client -> gate -> watchdog -> agent

```

client->gate->agent
搞清楚了这个过程以后，去看一下skynet/example下的watchdog.lua和agent.lua就很容易了。可以简单地理解为watchdog实际上只有建立连接的时候有用，连接建立时它启动了一个agent。在agent没启动好的时候，连接上过来的所有数据都发到watchdog，watchdog把数据直接扔掉。当agent启动好以后，gate的数据就不再经过watchdog，直发agent。

讲原理讲了很大的篇幅了，下面来写代码。watchdog.lua不需要更改，直接把skynet/example/watchdog.lua拷过来就好了。要改的就要是agent.lua。

``` lua
agent.lua v1

local skynet = require "skynet"
local socket = require "skynet.socket"

local WATCHDOG
local host
local send_request

local CMD = {}
local REQUEST = {}
local client_fd

function CMD.start(conf)
    local fd = conf.client
    local gate = conf.gate
    WATCHDOG = conf.watchdog
    -- slot 1,2 set at main.lua
    skynet.fork(function()
        while true do
            socket.write(client_fd, "heartbeat")
            skynet.sleep(500)
        end
    end)

    client_fd = fd
    skynet.call(gate, "lua", "forward", fd)
end

function CMD.disconnect()
    -- todo: do something before exit
    skynet.exit()
end

skynet.start(function()
    skynet.dispatch("lua", function(_,_, command, ...)
        local f = CMD[command]
        skynet.ret(skynet.pack(f(...)))
    end)
end)

```

这个版本的agent.lua只实现了最基本的网络功能，还有发心跳包的功能。

然后还需要一个main.lua

main.lua
``` lua
local skynet = require "skynet"

local max_client = 64

skynet.start(function()
        skynet.error("Server start")
        if not skynet.getenv "daemon" then
                local console = skynet.newservice("console")
        end
        skynet.newservice("debug_console",8000)
        skynet.newservice("echo")
        local watchdog = skynet.newservice("watchdog")
        skynet.call(watchdog, "lua", "start", {
                port = 8888,
                maxclient = max_client,
                nodelay = true,
        })
        skynet.error("Watchdog listen on", 8888)
        skynet.exit()
end)

```
最后把config/config文件里的start改为main，然后运行。一切正常的话，就可以用telnet试一下能不能连上了。
telnet 127.0.0.1 8888

