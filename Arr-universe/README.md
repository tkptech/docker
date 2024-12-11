# Self-Hosted Media Server and Aggregation

Đây hiện là một công việc đang được tiến hành. Vui lòng tham khảo [Thiết lập Docker Servarr](https://wiki.servarr.com/docker-guide) để biết thêm chi tiết về cách cài đặt stack.

## Data Directory
### Folder Mapping
Cách tốt nhất là cấp cho tất cả các vùng chứa quyền truy cập như nhau vào cùng một thư mục gốc hoặc chia sẻ. Đây là lý do tại sao tất cả các vùng chứa trong tệp soạn thảo đều có gắn ổ đĩa liên kết ```/data:/data```. Nó làm cho mọi thứ trở nên dễ dàng hơn, cộng với việc chuyển vào hai tập như /tv, /movies và /downloads thường được đề xuất làm cho chúng trông giống như hai hệ thống tệp khác nhau, ngay cả khi chúng là một hệ thống tệp duy nhất bên ngoài vùng chứa. Xem thiết lập hiện tại của tôi dưới đây.
```
media
├── books
├── downloads
│   ├── qbittorrent
│   │   ├── completed
│   │   ├── incomplete
│   │   └── torrents
│   └── nzbget
│       ├── completed
│       ├── intermediate
│       ├── nzb
│       ├── queue
│       └── tmp
├── movies
├── music
├── shows
└── youtube
```
Lệnh dễ dàng để tạo sơ đồ thư mục tải xuống..
```
mkdir -p downloads/qbittorrent/{completed,incomplete,torrents} && mkdir -p downloads/nzbget/{completed,intermediate,nzb,queue,tmp}
```
### Network Share
Điều này được tạo bằng cách thêm chia sẻ vào tệp fstab trong máy chủ Docker của tôi. Do lưu ý, ứng dụng ```cifs-utils``` là bắt buộc đối với phương pháp này.
```
//192.168.1.103/data /media/data cifs uid=1000,gid=1000,username=user,password=password,iocharset=utf8 0 0
```
Storing the user creditentials within this file it's the best idea. Check out [this question](https://unix.stackexchange.com/questions/178187/how-to-edit-etc-fstab-properly-for-network-drive) on Stack Exchange to learn more.
## User Permissions
Using bind mounts (path/to/config:/config) may lead to permissions conflicts between the host operating system and the container. To avoid this problem, you we specify the user ID (PUID) and group ID (PGID) to use within some of the containers. This will give your user permission to read and write configuration files, etc.

In the compose files I use PUID=1000 and PGID=1000, as those are generally the default ID's in most Linux systems, but depending on your setup you may need to chage this.

```
id your_user
uid=1000(tkp),gid=1000(tkp),groups=1000(data-share),988(docker)
```
In the example output above, If using a network share I would need to edit the compose.yaml with gid=1003. If you are using a network share mounted though ```/etc/fstab``` match the permissions there. I use Cockpit with a custom group fpr shares so my permissions are ```uid=1000(brandon),gid=1000(data-share)```.
If you run into errors, after creating all the folders you can assign the permissions using chmod. For example,
```
sudo chown -R 1000:1000 /data
```

## VPN Information
### Testing Gluetun Connectivity 
Once your containers are up and running, you can test your connection is correct and secured. This assumes you keeo the gluetun container name. Learn more at the [gluetun wiki](https://github.com/qdm12/gluetun-wiki/blob/main/setup/test-your-setup.md).
```
docker run --rm --network=container:gluetun alpine:3.18 sh -c "apk add wget && wget -qO- https://ipinfo.io"
```
### Passing Through Containers 
When containers are in the same docker compose they all you need to add is a ```network_mode: service:container_name``` and open the ports through the the gluetun container. See example from the compose.yaml below.
```
services:
  gluetun: # This config is for wireguard only tested with AirVPN
    image: qmcgaw/gluetun
    container_name: gluetun
    ...
    ports:
      - 8888:8112 # deluge web interface
      - 58846:58846 # deluge RPC
  deluge:
    image: linuxserver/deluge:latest
    container_name: deluge
    ...
    network_mode: service:gluetun
```
### External Container to Gluetun
Add ```--network=container:gluetun``` when launching the container, provided Gluetun is already running on the same machine.

### Container in another docker-compose.yml
Add network_mode: "container:gluetun" to your docker-compose.yml, provided Gluetun is already running. Ensure you open the ports through the the gluetun container.

### Gluetun Proxmox Fix

"cannot Unix Open TUN device file: operation not permitted and cannot create TUN device file node: operation not permitted" May happen if you're running this on LXC containers.

Find your container number, for example mine is 101

Edit `/etc/pve/lxc/101.conf` and add:

```
lxc.cgroup2.devices.allow: c 10:200 rwm
lxc.mount.entry: /dev/net dev/net none bind,create=dir
lxc.mount.entry: /dev/net/tun dev/net/tun none bind,create=file
```
Make sure you pass through the tun device (/dev/net/tun:/dev/net/tun) as shown in my compose file.

### Testing Other Containers
Jump into the Exec console and run the wget command below. Tested with nzbget, deluge, and prowlarr. Ensure you open the ports through the the gluetun container.
```
docker exec -it conatiner_name bash
wget -qO- https://ipinfo.io
```

