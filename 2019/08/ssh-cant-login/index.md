# 一次ssh免密无法登录的问题


自研数据库升级的过程中，需要配置一次ssh免密登录，以便在其中一台机器，很方便的升级集群所有服务器。但是在测试过程中，创建免密登录的脚本失效了。

问题定位
========

通过手动创建公钥，拷贝到其它机器的authrized\_keys，发现仍然需要输入密码登录；

通过ssh -v &lt;user@ip&gt;，查看详细信息，发现其提示： Authentication can
continue: publickey, gssapi-key…..password try publickey .ssh/….. try
publickey .ssh/….. using password:

而正常的服务器上，该行为： Servert accepted key…

1.  看日志，怀疑ssh客户端没有找到正确的公钥文件，但是该文件确实存在在正确的路径，且拥有正确的600权限。

2.  尝试使用其它端口，启动服务端的sshd，发现可以免密登录

3.  尝试在客户端新建其它用户，并使用22默认端口，一样可以免密登录

4.  执行以下命令，对比新建用户的目录，和问题用户的目录，发现问题

        ls -laZ

在问题用户的目录中，.ssh目录的label为unlabel，而正常用户的.ssh，为user\_t。通过查询及测试，发现该目录为user\_t或者ssh\_home\_t的标签，都可以测试通过，但是为ublabeled不行。
那么该目录为什么为unlabel呢？毕竟我们执行的只是ssh-keygen，目录并非我们生成。
其实这个目录的标签，会继承父目录的标签，而父目录的标签，由于未知原因，丢失了。因此，selinux的机制不允许ssh使用该目录作为公钥目录。该问题可以通过以下两种方法解决：

1.  restorecon -vv -r ~/.ssh

2.  setenforce 0等关闭selinux


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/2019/08/ssh-cant-login/  

