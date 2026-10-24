# 装完 Server 后安装 GUI 桌面环境

```bash
sudo apt update
```

# 安装 **GNOME 完整版**

```bash
# ubuntu
# 会存在网络问题可以重试多次
sudo apt install -y ubuntu-desktop
sudo systemctl set-default graphical.target
sudo reboot
sudo ls /dev/dri/card0 
sudo timedatectl set-timezone Asia/Shanghai
sudo  timedatectl
sudo  systemctl stop ufw
sudo systemctl disable ufw


# centos
sudo dnf install epel-release -y
sudo dnf groupinstall "Server with GUI"
sudo systemctl set-default graphical.target
reboot
```

![](D:/marktext/images/2026-10-15-18-21-23-image.png)

# 安装RDP远程连接

```bash
# ubuntu
sudo apt update
sudo apt install -y xrdp xorgxrdp
sudo adduser xrdp ssl-cert
sudo systemctl enable --now xrdp
sudo vim /etc/gdm3/custom.conf
WaylandEnable=false
sudo systemctl restart gdm3
sudo systemctl restart xrdp
# 配置网络
echo "network: {config: disabled}" | sudo tee /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg
sudo netplan apply


# centos

sudo dnf install -y epel-release
sudo dnf install -y xrdp xorgxrdp
sudo dnf remove -y gnome-terminal xterm
sudo dnf install -y terminator


sudo sed -i 's/#WaylandEnable=false/WaylandEnable=false/g' /etc/gdm/custom.conf


sudo sed -i 's/^EnableUserWindowManager=.*/EnableUserWindowManager=false/g' /etc/xrdp/sesman.ini
sudo sed -i 's/^DefaultWindowManager=.*/DefaultWindowManager=startwm.sh/g' /etc/xrdp/sesman.ini


cat <<'EOF' > /tmp/startwm.sh
#!/bin/sh
unset DBUS_SESSION_BUS_ADDRESS
unset XDG_RUNTIME_DIR
export XDG_SESSION_TYPE=x11
export XDG_CURRENT_DESKTOP=GNOME
export DESKTOP_SESSION=gnome
exec dbus-run-session gnome-session
EOF


sudo cp /tmp/startwm.sh /etc/xrdp/startwm.sh
sudo cp /tmp/startwm.sh /usr/libexec/xrdp/startwm.sh
sudo chmod +x /etc/xrdp/startwm.sh
sudo chmod +x /usr/libexec/xrdp/startwm.sh
rm -f /tmp/startwm.sh


sudo systemctl restart gdm

sudo systemctl enable --now xrdp
sudo systemctl restart xrdp-sesman
sudo systemctl restart xrdp


```
