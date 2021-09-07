1. 配置好/etc/ganesha/ganesha.conf
2. icfs-fuse + 挂载目录（比如/mnt/icfs/nfs）
3. 重启ganesha，systemctl restart ganesha
4. showmount -e
5. 即可执行挂载nfs：mount -t nfs 188.188.0.122:/nfs mnt/test -o vers=3