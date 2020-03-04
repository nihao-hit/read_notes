# mptcp_fullmesh.c阅读笔记

​		文件中有四个重要数据结构，分别是

| 数据结构                                         | 注释                                                         |
| :----------------------------------------------- | ------------------------------------------------------------ |
| struct mptcp_pm_ops full_mesh                    | 路径管理器操作集接口                                         |
| struct pernet_operations full_mesh_net_ops       | 向mptcp命名空间注册路径管理器命名空间，用于接收网卡/地址通知事件。 |
| struct notifier_block mptcp_pm_inetaddr_notifier | 对地址通知事件，创建mptcp_addr_event，并向mptcp_wq入队延迟工作address_worker，推迟到下半部执行。 |
| struct notifier_block mptcp_pm_netdev_notifier   | 对网卡通知事件，创建mptcp_addr_event，并向mptcp_wq入队延迟工作address_worker，推迟到下半部执行。 |



​		路径管理器创建三种“工作”，分别为

| 工作                                   | 注释                                                         |
| -------------------------------------- | ------------------------------------------------------------ |
| struct work_struct subflow_work        | 关联工作线程create_subflow_worker：对所有本地地址和对端地址间可以建立额外子流的地址对，置位bitfield，立即尝试建立额外子流，失败则置位retry_bitfield，等待1s后入队subflow_retry_work重试。只有在函数full_mesh_create_subflows()中入队mptcp_wq。 |
| struct delayed_work subflow_retry_work | 关联工作线程retry_subflow_worker：对于retry_bitfield置位的地址，置位bitfield，清除retry_bitfield，重试一次建立连接。只有在工作线程create_subflow_worker()中被入队mptcp_wq。 |
| struct delayed_work address_worker     | 关联工作线程mptcp_address_worker：处理网卡/地址通知事件。    |



​		struct mptcp_pm_ops full_mesh操作集接口分析

```c
static struct mptcp_pm_ops full_mesh __read_mostly = {
	.new_session = full_mesh_new_session,
	.release_sock = full_mesh_release_sock,
	.fully_established = full_mesh_create_subflows,
	.new_remote_address = full_mesh_create_subflows,
	.get_local_id = full_mesh_get_local_id,
	.addr_signal = full_mesh_addr_signal,
	.add_raddr = full_mesh_add_raddr,
	.rem_raddr = full_mesh_rem_raddr,
	.delete_subflow = full_mesh_delete_subflow,
	.name = "fullmesh",
	.owner = THIS_MODULE,
};
```

1. full_mesh_new_session

   在主流建立后完成一些mptcp层pm相关初始化工作，记录已用本地与对端地址，通告额外可用地址，初始化subflow_work与subflow_retry_work工作。

   mptcp_update_metasocket()->full_mesh_new_session()函数调用栈共调用两次，一次是主动打开者三次握手后mptcp通告空闲地址，另一次是被动打开方三次握手后调用。

2. full_mesh_release_sock

   release_sock()->tcp_release_cb()->full_mesh_release_sock()

3. full_mesh_create_subflows

   向mptcp_wq入队工作subflow_work，推迟到下半部执行。

   追溯调用栈到mptcp_handle_options，established状态下的收包会调用此函数处理mptcp选项，若条件满足，调用full_mesh_create_subflows新建子流。