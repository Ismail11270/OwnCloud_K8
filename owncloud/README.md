Setting up nfs server

sudo mkdir /private

sudo chmod 777 /private

sudo vim /etc/exports 

/private *(rw,sync,no_subtree_check,no_root_squash)

sudo exportfs -arvf
sudo systemctl start nfs-kernel-server
