# Openssh存在漏洞，需将版本升级到8.3
* 下载openssh8.3升级包及依赖的zlib和openssl。

*openssh-8.3p1.tar.gz、zlib-1.2.11.tar.gz、openssl-1.1.1g.tar.gz（见附件：相关软件.zip）*

wget https://openbsd.hk/pub/OpenBSD/OpenSSH/portable/openssh-8.3p1.tar.gz

wget http://www.zlib.net/zlib-1.2.11.tar.gz

wget https://www.openssl.org/source/openssl-1.1.1g.tar.gz

* 解压升级包
tar  --no-same-owner -zxvf zlib-1.2.11. tar .gz
tar  --no-same-owner -zxvf openssh-8.3p1. tar .gz
tar  --no-same-owner -zxvf openssl-1.1.1g. tar .gz



* 编译安装zlib

cd zlib-1.2.11
./configure --prefix=/usr/local/zlib
make && make install


* 编译安装openssl

cd openssl-1.1.1g
./config --prefix=/usr/local/ssl -d shared
make && make install
echo '/usr/local/ssl/lib' >> /etc/ld.so.conf
ldconfig -v


* 安装openssh

cd openssh-8.3p1
./configure --prefix=/usr/local/openssh --with-zlib=/usr/local/zlib --with-ssl-dir=/usr/local/ssl
make && make install


* sshd_config文件修改

echo 'PermitRootLogin yes' >>/usr/local/openssh/etc/sshd_config
echo 'PubkeyAuthentication yes' >>/usr/local/openssh/etc/sshd_config
echo 'PasswordAuthentication yes' >>/usr/local/openssh/etc/sshd_config


* 备份原有文件，并将新的配置复制到指定目录
cp openssh编译的目录下/contrib/redhat/sshd.init /etc/init.d/sshd
mv /etc/ssh/sshd_config /etc/ssh/sshd_config.bak
cp /usr/local/openssh/etc/sshd_config /etc/ssh/sshd_config
mv /usr/sbin/sshd /usr/sbin/sshd.bak
cp /usr/local/openssh/sbin/sshd /usr/sbin/sshd
mv /usr/bin/ssh /usr/bin/ssh.bak
cp /usr/local/openssh/bin/ssh /usr/bin/ssh
mv /usr/bin/ssh-keygen /usr/bin/ssh-keygen.bak
cp /usr/local/openssh/bin/ssh-keygen /usr/bin/ssh-keygen
mv /etc/ssh/ssh_host_ecdsa_key.pub /etc/ssh/ssh_host_ecdsa_key.pub.bak
cp /usr/local/openssh/etc/ssh_host_ecdsa_key.pub /etc/ssh/ssh_host_ecdsa_key.pub
'''
* 启动sshd

systemctl restart sshd

* 查看信息版本

ssh -V

*  cd /etc/systemd/system/
vi sshd.service
在[Service]部分将
Type=notify 修改为
Type=simple 保存退出
没有这个Type就在[Service]下面一行加上
systemctl daemon-reload
systemctl enable sshd.service
systemctl restart sshd.service
