# 2019 SDN大作业--数据中心类型网络拓扑的搭建与连接

## 实验概述
使用两个互为备份的中心交换机

连接两两互为备份的共计四个交换机

下接四组各连有两台主机的交换机

作为数据中心类型网络拓扑的一个小型实现

上、中、下层均可以扩展来实现对更多网络主机的支持

实现在网络中心区域防止单个设备故障所引发的网络中断

# 实验拓扑

实验拓扑图如下

![](https://img2018.cnblogs.com/blog/1329634/202001/1329634-20200105202749917-337166012.png)

# 建立实验网络

建立流程如下

1.先打开OpenDayLigtht作为remote控制器,否则先运行mininet则不能连接到控制器

2.运行mininet建立拓扑结构，运行代码如`sudo mn --custom datacenter.py --topo mytopo --controller=remote,ip=127.0.0.1,port=6633 --switch ovsk,protocols=OpenFlow13`

3.打开`http://127.0.0.1:8181/index.html#/topology`来查看拓扑

4.在mininet中输入net来获取网络接口信息，作为下发流表的依据

![](https://img2018.cnblogs.com/blog/1329634/202001/1329634-20200105211726293-686913768.png)

mininet的拓扑结构的Python代码如下：

```python
#!/usr/bin/python
#创建网络拓扑
"""Custom topology example
Adding the 'topos' dict with a key/value pair to generate our newly defined
topology enables one to pass in '--topo=mytopo' from the command line.
"""
 
from mininet.topo import Topo
from mininet.net import Mininet
from mininet.node import RemoteController,CPULimitedHost
from mininet.link import TCLink
from mininet.util import dumpNodeConnections
 
class MyTopo( Topo ):
    "Simple topology example."
 
    def __init__( self ):
        "Create custom topo."
 
        # Initialize topology
        Topo.__init__( self )
        L1 = 2
        L2 = L1 * 2 
        L3 = L2
        c = []
        a = []
        e = []
          
        # add core ovs  
        for i in range( L1 ):
                sw = self.addSwitch( 'c{}'.format( i + 1 ) )
                c.append( sw )
    
        # add aggregation ovs
        for i in range( L2 ):
                sw = self.addSwitch( 'a{}'.format( L1 + i + 1 ) )
                a.append( sw )
    
        # add edge ovs
        for i in range( L3 ):
                sw = self.addSwitch( 'e{}'.format( L1 + L2 + i + 1 ) )
                e.append( sw )
 
        # add links between core and aggregation ovs
        for i in range( L1 ):
                sw1 = c[i]
                for sw2 in a[i/2::L1/2]:
                # self.addLink(sw2, sw1, bw=10, delay='5ms', loss=10, max_queue_size=1000, use_htb=True)
			            self.addLink( sw2, sw1 )
 
        # add links between aggregation and edge ovs
        for i in range( 0, L2, 2 ):
                for sw1 in a[i:i+2]:
	                for sw2 in e[i:i+2]:
			            self.addLink( sw2, sw1 )
 
        #add hosts and its links with edge ovs
        count = 1
        for sw1 in e:
                for i in range(2):
                	host = self.addHost( 'h{}'.format( count ) )
                	self.addLink( sw1, host )
                	count += 1
topos = { 'mytopo': ( lambda: MyTopo() ) }
```

OpenDayLigtht的远程控制器代码如下：

先在控制台中运行`./karaf`打开容器

接着按照顺序安装feature如下

```shell
install odl-restconf
install odl-l2switch-switch-ui
install odl-openflowplugin-all
install odl-mdsal-apidocs
install odl-dlux-core
install odl-dlux-node
install odl-dlux-yangui
```
在退出前执行`logout`退出

运行完后在YangUI中可见拓扑结构如图

![](https://img2018.cnblogs.com/blog/1329634/202001/1329634-20200105210830872-1173192232.png)

![](https://img2018.cnblogs.com/blog/1329634/202001/1329634-20200105214007657-142591950.png)

节点编号与节点名称关系如下表（加载原因未能加载出全部）

![](https://img2018.cnblogs.com/blog/1329634/202001/1329634-20200105211053803-1146731411.png)

# 下发初始流表连接链路

1.清空所有流表项

```shell
sudo ovs-ofctl -O Openflow13 del-flows c1
sudo ovs-ofctl -O Openflow13 del-flows c2
sudo ovs-ofctl -O Openflow13 del-flows a3
sudo ovs-ofctl -O Openflow13 del-flows a4
sudo ovs-ofctl -O Openflow13 del-flows a5
sudo ovs-ofctl -O Openflow13 del-flows a6
sudo ovs-ofctl -O Openflow13 del-flows e7
sudo ovs-ofctl -O Openflow13 del-flows e8
sudo ovs-ofctl -O Openflow13 del-flows e9
sudo ovs-ofctl -O Openflow13 del-flows e10
```

2.下发下层流表

```shell
#e7
sudo ovs-ofctl -O OpenFlow13 add-flow e7 priority=2,in_port=1,actions=output:3,output:4
sudo ovs-ofctl -O OpenFlow13 add-flow e7 priority=2,in_port=2,actions=output:3,output:4
sudo ovs-ofctl -O OpenFlow13 add-flow e7 priority=2,in_port=3,actions=output:1,output:2,output:4
sudo ovs-ofctl -O OpenFlow13 add-flow e7 priority=2,in_port=4,actions=output:1,output:2,output:3
#e8
sudo ovs-ofctl -O OpenFlow13 add-flow e8 priority=2,in_port=1,actions=output:3,output:4
sudo ovs-ofctl -O OpenFlow13 add-flow e8 priority=2,in_port=2,actions=output:3,output:4
sudo ovs-ofctl -O OpenFlow13 add-flow e8 priority=2,in_port=3,actions=output:1,output:2,output:4
sudo ovs-ofctl -O OpenFlow13 add-flow e8 priority=2,in_port=4,actions=output:1,output:2,output:3
#e9
sudo ovs-ofctl -O OpenFlow13 add-flow e9 priority=2,in_port=1,actions=output:3,output:4
sudo ovs-ofctl -O OpenFlow13 add-flow e9 priority=2,in_port=2,actions=output:3,output:4
sudo ovs-ofctl -O OpenFlow13 add-flow e9 priority=2,in_port=3,actions=output:1,output:2,output:4
sudo ovs-ofctl -O OpenFlow13 add-flow e9 priority=2,in_port=4,actions=output:1,output:2,output:3
#e10
sudo ovs-ofctl -O OpenFlow13 add-flow e10 priority=2,in_port=1,actions=output:3,output:4
sudo ovs-ofctl -O OpenFlow13 add-flow e10 priority=2,in_port=2,actions=output:3,output:4
sudo ovs-ofctl -O OpenFlow13 add-flow e10 priority=2,in_port=3,actions=output:1,output:2,output:4
sudo ovs-ofctl -O OpenFlow13 add-flow e10 priority=2,in_port=4,actions=output:1,output:2,output:3
```

3.下发中层流表

```shell
#a3
sudo ovs-ofctl -O OpenFlow13 add-flow a3 priority=2,in_port=1,actions=output:3,output:4
sudo ovs-ofctl -O OpenFlow13 add-flow a3 priority=2,in_port=2,actions=output:3,output:4
sudo ovs-ofctl -O OpenFlow13 add-flow a3 priority=2,in_port=3,actions=output:1,output:2,output:4
sudo ovs-ofctl -O OpenFlow13 add-flow a3 priority=2,in_port=4,actions=output:1,output:2,output:3
#a4
sudo ovs-ofctl -O OpenFlow13 add-flow a4 priority=2,in_port=1,actions=output:3,output:4
sudo ovs-ofctl -O OpenFlow13 add-flow a4 priority=2,in_port=2,actions=output:3,output:4
sudo ovs-ofctl -O OpenFlow13 add-flow a4 priority=2,in_port=3,actions=output:1,output:2,output:4
sudo ovs-ofctl -O OpenFlow13 add-flow a4 priority=2,in_port=4,actions=output:1,output:2,output:3
#a5
sudo ovs-ofctl -O OpenFlow13 add-flow a5 priority=2,in_port=1,actions=output:3,output:4
sudo ovs-ofctl -O OpenFlow13 add-flow a5 priority=2,in_port=2,actions=output:3,output:4
sudo ovs-ofctl -O OpenFlow13 add-flow a5 priority=2,in_port=3,actions=output:1,output:2,output:4
sudo ovs-ofctl -O OpenFlow13 add-flow a5 priority=2,in_port=4,actions=output:1,output:2,output:3
#a6
sudo ovs-ofctl -O OpenFlow13 add-flow a6 priority=2,in_port=1,actions=output:3,output:4
sudo ovs-ofctl -O OpenFlow13 add-flow a6 priority=2,in_port=2,actions=output:3,output:4
sudo ovs-ofctl -O OpenFlow13 add-flow a6 priority=2,in_port=3,actions=output:1,output:2,output:4
sudo ovs-ofctl -O OpenFlow13 add-flow a6 priority=2,in_port=4,actions=output:1,output:2,output:3
```

4.下发上层流表

```shell
#c1
sudo ovs-ofctl -O OpenFlow13 add-flow c1 priority=2,in_port=1,actions=output:3,output:4
sudo ovs-ofctl -O OpenFlow13 add-flow c1 priority=2,in_port=2,actions=output:3,output:4
sudo ovs-ofctl -O OpenFlow13 add-flow c1 priority=2,in_port=3,actions=output:1,output:2
sudo ovs-ofctl -O OpenFlow13 add-flow c1 priority=2,in_port=4,actions=output:1,output:2
#c2
sudo ovs-ofctl -O OpenFlow13 add-flow c2 priority=2,in_port=1,actions=output:3,output:4
sudo ovs-ofctl -O OpenFlow13 add-flow c2 priority=2,in_port=2,actions=output:3,output:4
sudo ovs-ofctl -O OpenFlow13 add-flow c2 priority=2,in_port=3,actions=output:1,output:2
sudo ovs-ofctl -O OpenFlow13 add-flow c2 priority=2,in_port=4,actions=output:1,output:2
```

5.把上述代码封装入shell文件中可以实现一键下发和清空流表



# 测试链路可用性

1.在mininet中运行pingall命令

![](https://img2018.cnblogs.com/blog/1329634/202001/1329634-20200105212257537-947346364.png)

说明链路已经连接成功

2.在OpenDayLigtht拓扑图中可以清晰地看到每一个客户端![](https://img2018.cnblogs.com/blog/1329634/202001/1329634-20200105212506832-1322129181.png)
![](https://img2018.cnblogs.com/blog/1329634/202001/1329634-20200105223220695-1417509133.png)

# Iperf链路性能测试

h1连接h2测试:

![](https://img2018.cnblogs.com/blog/1329634/202001/1329634-20200105213028866-877986285.png)

可以看到带宽为9.92Gbits/sec

h1连接h3测试:

![](https://img2018.cnblogs.com/blog/1329634/202001/1329634-20200105213415454-62349588.png)

可以看到带宽为2.41Gbits/sec

h1连接h8测试:

![](https://img2018.cnblogs.com/blog/1329634/202001/1329634-20200105213645807-780360528.png)

可以看到带宽为122Mbits/sec

结论是随着跨交换机网络的转发，性能随着跨交换机网络而减弱，解决方法是用分时间片的方法来负载均衡。

# 负载均衡的实现

通过将某段时间流表设置成c1,a3,a5一组和c2,a4,a6一组来达到交换机的负载均衡，以提高性能。

负载均衡后再次执行链路性能测试

h1连接h2因为没有链路变化所以带宽基本不变

![](https://img2018.cnblogs.com/blog/1329634/202001/1329634-20200105220322254-1605352902.png)

h1连接h3可以看到带宽从2.41Gbit/sec提高到了3.87Gbit/sec

![](https://img2018.cnblogs.com/blog/1329634/202001/1329634-20200105221544811-2076309261.png)

h1连接h8可以看到带宽从122Mbit/sec提升到了292Mbit/sec

![](https://img2018.cnblogs.com/blog/1329634/202001/1329634-20200105221227057-1894868031.png)

成组打开并关闭代码：

```python
import os
import time
def runteam1():
	os.system("./addt1.sh")
	time.sleep(1)
	os.system("./delt2.sh")
	return 1;
def runteam2():
	os.system("./addt2.sh")
	time.sleep(1)
	os.system("./delt1.sh")
	return 1;
os.system("./delflows.sh")
os.system("./inite.sh")
while(True):
	runteam1()
	runteam2()
```

addt1.sh代码：

```shell
#c1
sudo ovs-ofctl -O OpenFlow13 add-flow c1 priority=2,in_port=1,actions=output:3,output:4
sudo ovs-ofctl -O OpenFlow13 add-flow c1 priority=2,in_port=2,actions=output:3,output:4
sudo ovs-ofctl -O OpenFlow13 add-flow c1 priority=2,in_port=3,actions=output:1,output:2
sudo ovs-ofctl -O OpenFlow13 add-flow c1 priority=2,in_port=4,actions=output:1,output:2
#a3
sudo ovs-ofctl -O OpenFlow13 add-flow a3 priority=2,in_port=1,actions=output:3,output:4
sudo ovs-ofctl -O OpenFlow13 add-flow a3 priority=2,in_port=2,actions=output:3,output:4
sudo ovs-ofctl -O OpenFlow13 add-flow a3 priority=2,in_port=3,actions=output:1,output:2,output:4
sudo ovs-ofctl -O OpenFlow13 add-flow a3 priority=2,in_port=4,actions=output:1,output:2,output:3
#a5
sudo ovs-ofctl -O OpenFlow13 add-flow a5 priority=2,in_port=1,actions=output:3,output:4
sudo ovs-ofctl -O OpenFlow13 add-flow a5 priority=2,in_port=2,actions=output:3,output:4
sudo ovs-ofctl -O OpenFlow13 add-flow a5 priority=2,in_port=3,actions=output:1,output:2,output:4
sudo ovs-ofctl -O OpenFlow13 add-flow a5 priority=2,in_port=4,actions=output:1,output:2,output:3
```

delt1.sh代码：

```shell
sudo ovs-ofctl -O Openflow13 del-flows c1
sudo ovs-ofctl -O Openflow13 del-flows a3
sudo ovs-ofctl -O Openflow13 del-flows a5
```



addt2.sh、delt2.sh、inite.sh、delflows.sh的代码和上述示例十分接近，故不在赘述

本次实验因为是在本机进行，所以使用的是ovs-ofctl命令，若要进行远程的流表的下发，则需要使用Restful接口远程下发流表，下发内容与以上近似。

# 实验总结

经过本次实验，对OpenFlow、Mininet、OpenDayLight、OVS控制器等有了更加深入的理解，对软件定义网络也更加熟悉和清晰。
