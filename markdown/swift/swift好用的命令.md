# swift 好用的命令

1. swift-container-info /..../containers/.../*.db

   查看container（bucket）的相关信息

2. swift-ring-builder /../../*.buidler list_parts --ip --device 

   查看某些设备上的分区，用于检查分区和ring*.gz是否一致

3. select local_user.name, role.name from local_user, assignment,  role where assignment.actor_id = local_user.user_id and assignment.role_id = role.id; 

   查看user和role的对应关系。(使用assignment表中获取对应关系，通过id连接) 

4. swift-get-nodes -a /etc/swift/object.ring.gz account container object


   显示和此object相关的设备，可以通过显示出来的路径去找到对应的文件，也包括了handoff的partition。

5. swift-get-nodes -a /etc/swift/container.ring.gz accont container

   显示container相关的路径

6. curl -i http://object-server-ip:6200/recon/async

   https://docs.openstack.org/swift/latest/admin_guide.html

