# wwptt
```
# Update + install python3
sudo apt update
sudo apt install python3 python3-pip -y

# Bikin folder
mkdir wwptt && cd wwptt

# Buat file server.py, paste code yang tadi gue kasih
nano server.py
# ctrl+shift+v buat paste, ctrl+o enter ctrl+x buat save

# Buka port UDP di firewall
sudo ufw allow 500xx/udp
sudo ufw reload

# Jalankan server
python3 server.py

#Server bakal jalan dan nunggu HP connect. Biar auto start pas VPS reboot, pakai screen atau systemd. Gue kasih yang gampang pakai screen:
sudo apt install screen -y
screen -S wwptt
python3 server.py
# tekan ctrl+A lalu D buat lepas dari screen
Catat IP VPS lu: curl ifconfig.me
```
