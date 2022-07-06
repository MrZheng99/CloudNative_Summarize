# Docker网络浅析

- veth pair
  
  - 连通不同network namspace的设备，数据从一端发出，在另一端就可以收到数据。在docker中，它一端插在容器内名叫eth0，另一端插在docker0网桥中，名叫vethxxx。
    

- docker0
  
  - 由docker创建的一个虚拟网桥设备，正因为它是虚拟的，也就意味着不是单纯的二层转发设备（docker0有ip地址可以充当网关），当它遇到不能处理的数据包时，它就会将请求丢给上层网络协议栈去处理（宿主机路由表）。
    
  - 交换机角色
    
    - 当同一宿主机上的docker容器进行通信时，直接经由docker0进行二层转发。这里涉及到docker0的mac地址自学习功能。
      
  - 网关角色
    
    - 当容器需要访问外网或者host网络时，容器就会将数据发给docker0，然后由docker0再转发给上层网络协议栈。
      
    - 当在外网或者host访问容器时，将会根据路由表转发给docker0，docker0查询arp表项得到vethxxx的mac地址，然后通过mac表转发给对应的端口，进而到达容器的eth0。
      
- ip_forward
  
  - 如果需要同**非Host**通信，则需要设置此内核参数【默认不开启】，因为需要由docker0将ip数据包转发给宿主机的物理网卡，经由物理网卡发出，***注意*** 需要利用到iptables做NAT操作
    

# Fannel

> 特洛伊木马

### udp

- 三次用户态和内核态的切换
  
  - todo 插入图片
    
- route表项 根据路由转发给fannel.0
  
- 通过fannel.0【这是一个tun0设备】将数据传输到用户态完成udp封包，然后再转发给宿主机网卡。在这里fanneld会维护一张子网和宿主机对应关系的表，在udp封包时，会根据不同的子网封装不同的ip头，而目的宿主机的mac则是自我学习得来。
  
- fanneld在节点加入集群时会配置好这些表项（以前是直接查etcd，现在是连api-server）
  

## vxlan

- 一次用户态和内核态的切换
  
- route表项 根据vtep的ip 转发到fannel.1
  
- arp表项 根据vtep ip查询其mac
  
- fdb表项，根据vtep获取其宿主机的ip
  
- fanneld在节点加入集群时会配置好这些表项（以前是直接查etcd，现在是连api-server）
  

## gateway

- 利用宿主机作为网关，只有一个网桥设备连接docker容器，无其他设备。
  
- 配置宿主机的路由表，表项大概意思就是：发往子网xxx的包，需要经过eth0设备，并且目的ip为子网xxx所在的宿主机。
