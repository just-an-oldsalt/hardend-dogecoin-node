
# make etc folder
sudo mkdir /etc/dogecoin/
# copy contents of the conf file in to file in etc
sudo nano /etc/dogecoin/dogecoin.conf

# copy contents of service file in to system folder 
nano /etc/systemd/system/dogecoind.service

# create doge user
sudo adduser --disabled-password --gecos "" dogeuser
# create doge group 
sudo addgroup dogegroup
# add dogeuser to dogegroup
sudo usermod -aG dogegroup dogeuser
# reload daemon
sudo systemctl daemon-reload
# start doge service
sudo systemctl start dogecoind
# enable doge service
sudo systemctl enable dogecoind
