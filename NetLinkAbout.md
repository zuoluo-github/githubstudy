参考文献：

[Netlink基本使用]: http://blog.chinaunix.net/uid-28541347-id-5578403.html

​		Netlink socket 是一种Linux特有的socket，用于实现用户进程与内核进程之间通信的一种特殊的进程间通信方式(IPC) ，也是网络应用程序与内核通信的最常用的接口。Netlink 是一种在内核和用户应用间进行双向数据传输的非常好的方式，用户态应用使用标准的 socket API 就能使用 Netlink 提供的强大功能，内核态需要使用专门的内核 API 来使用 Netlink。

​		Linux中，uevent消息的发送等框架会用到Netlink的机制。

### 内核态Netlink

#### 内核态创建和销毁netlink socket

​		在内核空间中使用netlink_kernel_create（.\include\linux\netlink.h）来创建一个和用户空间通信的netlink socket。

```c
static inline struct sock *netlink_kernel_create(struct net *net, int unit, 
											 struct netlink_kernel_cfg *cfg);
```

​		参数net：是一个网络名字空间namespace，在不同的名字空间里面可以有自己的转发信息库，有自己的一套net_device等等。默认情况下都是使用 init_net （.\include\net\net_namespace.h）这个固定的全局变量。

​		参数uint：表示netlink协议类型，AF_NETLINK地址族支持最多32中协议，NETLINK_ROUTE、NETLINK_KOBJECT_UEVENT等(定义在 .\\include\uapi\linux\netlink\.h中)。需要注意的是，每种netlink协议在内核只能创建一个struct sock，如果试图创建的struct sock对应的协议已经被创建过了则会被报错，所以也可以理解为一个协议就代表内核中的一个netlink模块。

​		需要注意的是，当内核模块还没有创建uint对应协议的struct sock之前，用户态无法用socket接口创建基于uint协议的socket套接字。

​		参数cfg：早期的netlink_kernel_create接口参数比较多，后来为了方便，net和unit之外的参数都包含在了struct netlink_kernel_cfg结构体中（.\include\linux\netlink.h），该结构如下。

```c
struct netlink_kernel_cfg {
	unsigned int	groups;							  //表示多播地址
	unsigned int	flags;
	/*
		当用户空间有消息到达这个netlink socket时，内核就会调用这个input函数
		而且只有这个input函数返回之后，调用者的sendmsg才能返回
	*/
	void		(*input)(struct sk_buff *skb);		  //为内核模块定义的netlink消息处理函数
	struct mutex	*cb_mutex;
	int		(*bind)(struct net *net, int group);
	void		(*unbind)(struct net *net, int group);
	bool		(*compare)(struct net *net, struct sock *sk);
};

```

​		内核模块通过netlink_kernel_create获取到和用户空间通信的netlink socket之后便可以调用单播或者多播接口来向用户空间发送信息了。

​		同时内核通过使用netlink_kernel_release释放netlink_kernel_create创建的netlink socket。

```c
void netlink_kernel_release(struct sock *sk);
```

#### 内核态单播发送接口

```c
int netlink_unicast(struct sock *ssk, struct sk_buff *skb, __u32 portid, int nonblock);
```

​		参数ssk：ssk:为函数 netlink_kernel_create()返回的socket。

​		参数skb：存放消息，它的data字段指向要发送的netlink消息结构，而 skb的控制块保存了消息的地址信息，宏NETLINK_CB(skb)就用于方便设置该控制块。

​		参数portid：表示接收此消息用户进程的端口号，即目标地址，如果目标为内核，则把它设置为 0。

​		参数nonblock：表示该函数是否为非阻塞，如果为1，该函数将在没有接收缓存可利用时立即返回；而如果为0，该函数在没有接收缓存可利用定时睡眠。

​		这里需要再强调一下skb参数，

#### 内核态多播发送接口

```c
int netlink_broadcast(struct sock *ssk, struct sk_buff *skb, __u32 portid,
			     __u32 group, gfp_t allocation)
{
     return netlink_broadcast_filtered(ssk, skb, portid, group, allocation,NULL, NULL);
}
```

​		参数portid，因为是组播所以这个时候填0.

​		参数group：表示接收消息的多播组.该参数的每一个位代表一个多播组，因此如果发送给多个多播组，就把该参数设置为多个多播组组ID的位或。

​		参数allocation：表示内核分配内存的类型。GFP_ATOMIC用于原子的上下文（即不可以睡眠）；GFP_KERNEL用于非原子上下文。

### 用户态Netlink

​		用户态在使用Netlink时，使用的API是普通socket那一套API，但是在数据填充时略有不同。

#### 用户态创建netlink套接字

​		用户态使用socket接口创建套接字，例：

```c
int  socket（int portofamily，int  type，int  proto）
```

​		参数portofamily：在Netlink框架中，该参数始终为AF_NETLINK，表示地址族。

​		参数type：套接字类型，选择 SOCK_RAW 或者 SOCK_DGRAM，Netlink协议并不区分这两种类型，一般为SOCK_RAW。

​		参数proto：表示AF_NETLINK地址族中的协议，定义在.\\include\uapi\linux\netlink\.h中。

#### 用户态将netlink套接字和netlink地址族进行绑定

​		在绑定时，用户态需要使用struct sockaddr_nl 结构体（.\\include\uapi\linux\netlink\.h）：

```c
////////////////////////////////////////////////////////////////////////////////////////////////////
//在使用netlink机制时，用户态需要使用该结构体
//struct sockaddr_nl(12B)结构体和 struct sockaddr(16B)、struct sockaddr_in(16B) 结构体属于并列关系
////////////////////////////////////////////////////////////////////////////////////////////////////
struct sockaddr_nl {
	__kernel_sa_family_t	nl_family;	/* AF_NETLINK	*/  //该字段恒为AF_NETLINK
	unsigned short	nl_pad;		/* zero		*/			  //一般为0
	__u32		nl_pid;		/* port ID	*/ 				 //端口id，可以用进程号或者其它表示，能够唯一表示一个套接字即可.最好不要用0
	/*
		该成员变量表示多播组的组号掩码，注意，并不是表示多播组的组号
		例如： 
			用户态设置：			 nl_groups = 3
			内核态组播发送： netlink_broadcast(hsk, skb, 0, 3, GFP_ATOMIC)
		此时内核通过netlink_broadcast发送组播消息时会报错-3，表示没有socket在监听（内核直接使用数字表述组号，不需要用掩码）
		因为用户态在给nl_groups赋值时直接使用了组号3，但是该字段表示的是组号的掩码，所以需要将组号转换为组号的掩码
		nl_groups = 0 表示不加入任何组（单播）
		static int netlink_group_mask(int group){
			return group ? 1 << (group - 1) : 0;
		}
		上面这个函数可以实现组号到组号掩码的转换
		nl_groups = netlink_group_mask(3);
		这样给nl_groups字段赋值之后，内核通过netlink_broadcast接口发送给组播号为3的组时，内核能够成功接收到数据
		AF_NETLINK地址族对应的每种协议号最多可以支持32个组（nl_groups为32位的数据，每一位表示一个组）
	*/
    __u32		nl_groups;	/* multicast groups mask */	  	//多播组掩码，单播时设置为0
};
```

​		这里需要注意的是nl_groups这个字段，如果想让被绑定的套接字不进行组播监听，则需要将该字段设置为0，即不加入任何组，反之则按照注释中的方法来设置被绑定的套接字监听在某一个组播上。

​		nl_pid这个字段表示被绑定的套接字的端口号是多少。

​		将struct sockaddr_nl结构体设置好之后，就可以使用bind接口将之前创建的套接字和sockaddr_nl进行绑定。

​		绑定结束之后就可以调用recvfrom、sendto、recvmsg、sendmsg等UDP数据接收相关的接口来收发数据了。所以Netlink套接字也可以和UDP套接字进行类比。

#### 用户态通过netlink套接字发送和接收数据

​		用户态在发送数据的时候需要用到struct nlmsghdr结构体（.\\include\uapi\linux\netlink\.h）：

```c
////////////////////////////////////////////////////////////////////////////////////////////////////
//在使用netlink机制发送信息给内核时，需要用到该结构体
//这个结构体只是一个消息头，数据部分在内存能上紧跟在这个结构体后面
//数据部可以是一个结构体也可以就是一段buff，只需跟在struct nlmsghdr结构体后面即可
//内核定义了一系列的来帮助消息的处理。例如： NLMSG_ALIGN、NLMSG_HDRLEN等
////////////////////////////////////////////////////////////////////////////////////////////////////
struct nlmsghdr {
	__u32		nlmsg_len;	 //表示消息头和数据的总长度
	__u16		nlmsg_type;	 //表示消息状态
	__u16		nlmsg_flags; /* Additional flags */
	__u32		nlmsg_seq;	/* Sequence number */
	__u32		nlmsg_pid;	//用户态的netlink socket在bind struct sockaddr_nl时设置的端口号（源端口号），内核获取到消息头之后可以通过这个字段回复消息
};
```

​		nlmsg_type表示消息状态，内核在include/uapi/linux/netlink.h中定义了4种通用的消息状态：

```c
#define NLMSG_NOOP		0x1	/*不执行任何动作，必须将该消息丢弃*/
#define NLMSG_ERROR		0x2	/*消息发生错误*/
#define NLMSG_DONE		0x3	/*标识分组消息的末尾*/
#define NLMSG_OVERRUN	0x4	/*缓冲区溢出，表示某些消息已经丢失*/
```

​		Netlink的报文由消息头和消息体构成，struct nlmsghdr即为消息头，数据就紧跟在消息头后面，可以使用NLMSG_DATA宏来将数据和消息头进行组装。这里以sendto为例来发送消息。

```c
struct sockaddr_nl dest_addr;
struct nlmsghdr *nlh = NULL;
nlh = (struct nlmsghdr *)malloc(NLMSG_SPACE(MAX_PAYLOAD));
memset(&dest_addr,0,sizeof(dest_addr));

dest_addr.nl_family = AF_NETLINK;//恒定为AF_NETLINK
dest_addr.nl_pid = 0;			//0表示发送给内核
dest_addr.nl_groups = 0;

nlh->nlmsg_len = NLMSG_SPACE(MAX_PAYLOAD);
nlh->nlmsg_pid = 100;		    //内核中对应的模块可以通过这个字段回复消息
nlh->nlmsg_flags = 0;
strcpy(NLMSG_DATA(nlh),"Hello you!");

sendto(sock_fd,nlh,NLMSG_LENGTH(MAX_PAYLOAD),0,(struct sockaddr*)(&dest_addr),sizeof(dest_addr));
```

​			当然用户态直接通过sendto对应的接口recvfrom来接收数据就好了。

```c
recvfrom(sock_fd,nlh,NLMSG_LENGTH(MAX_PAYLOAD),0,NULL,NULL);
```

