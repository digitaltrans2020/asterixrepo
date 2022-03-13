# asterixrepo
asterixrepo
# -----------------------------------
# Asterisk with FreePBX - First Setup 
# -----------------------------------
# 
# Dr. Nauman 
# http://recluze.net/learn  
# 
# -----------------------------------



## Environment Setup 
# -----------------------------------

sudo apt update && sudo apt upgrade -y 
sudo reboot 

# -----------------------------------
## Optional ZSH Setup 
# -----------------------------------
apt install zsh 

# Repeat for user and for root 
wget https://github.com/robbyrussell/oh-my-zsh/raw/master/tools/install.sh -O - | zsh
chsh -s `which zsh`
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git
# Logout and log back in  
echo "source ${(q-)PWD}/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh" >> ${ZDOTDIR:-$HOME}/.zshrc

# END Optioanl ZSH Setup 


# -----------------------------------
# Asterisk Setup 
# -----------------------------------

# Let's go root: 
sudo -i 
cd /usr/local/src 

# Get Asterisk: 
wget https://downloads.asterisk.org/pub/telephony/asterisk/asterisk-14-current.tar.gz
tar zxvf asterisk-14-current.tar.gz
cd asterisk-14.7.7

# Setup the prereqs 
cd contrib/scripts 
./install_prereq install 
apt -f install   # to resolve conflicts (if any)


cd /usr/local/asterisk-14.7.7 
./configure 
make menuconfig   
# Select format_mp3 and EXTRA SOUND PACKAGES
# Then exit saving changes  

contrib/scripts/get_mp3_source.sh  # needed for mp3 support 

make 
make install 
make config    

# check status 
/etc/init.d/asterisk status     # shouldn't be running yet. Don't start it. 

# Details from: https://wiki.asterisk.org/wiki/display/AST/Hello+World
make samples 

cd /etc/asterisk

# vi extensions.conf 
[from-internal]
exten = 100,1,Answer()
same = n,Wait(1)
same = n,Playback(hello-world)
same = n,Hangup()


# vi sip.conf 
[general]
context=default

[6001]
type=friend
context=from-internal
host=dynamic
secret=temp123
disallow=all
allow=ulaw


# start asterisk on the console 
asterisk -cvvvvvv       # start with lots of verbosity (many messages) 

# -----------------------------------
# Try Asterisk Hello World 
# -----------------------------------
Install Linphone 
Setup with the following information: 
Username: 6001 
Password: temp123
SIP Proxy: [IP Address of Server]

# -----------------------------------
# FREEPBX Setup                      
# -----------------------------------

sudo -i  # Go root 

# We need php 5.6 as php7 is not supported 

# Setup repo for php5.6 
# apt-get install python-software-properties
add-apt-repository ppa:ondrej/php
apt-get update
apt-get install -y php5.6


# Setup other prereqs
apt install -y build-essential linux-headers-`uname -r` openssh-server apache2 mysql-server\
  mysql-client bison flex php5.6 php5.6-curl php5.6-cli php5.6-mysql php-pear php5.6-db php5.6-simplexml php5.6-gd curl sox\
  libncurses5-dev libssl-dev libmysqlclient-dev mpg123 libxml2-dev libnewt-dev sqlite3\
  libsqlite3-dev pkg-config automake libtool autoconf git subversion unixodbc-dev uuid uuid-dev\
  libasound2-dev libogg-dev libvorbis-dev libcurl4-openssl-dev libical-dev libneon27-dev libsrtp0-dev\
  libspandsp-dev libapache2-mod-php5.6


  (Also might need php-xml)


php -v   # Ensure 5.6 

# Get some extra sounds 

cd /var/lib/asterisk/sounds
wget http://downloads.asterisk.org/pub/telephony/sounds/asterisk-extra-sounds-en-wav-current.tar.gz
tar xfz asterisk-extra-sounds-en-wav-current.tar.gz
rm -f asterisk-extra-sounds-en-wav-current.tar.gz


# Finally, for FreePBX 

cd /usr/local/src 
wget http://mirror.freepbx.org/modules/packages/freepbx/freepbx-14.0-latest.tgz
tar zxf freepbx-14.0-latest.tgz

/etc/init.d/asterisk stop 

# Change ownership to dedicated asterisk user 
useradd -m asterisk
chown asterisk: /var/run/asterisk
chown -R asterisk: /etc/asterisk
chown -R asterisk: /var/{lib,log,spool}/asterisk
chown -R asterisk: /usr/lib/asterisk
rm -rf /var/www/html    # Make sure you don't have anything here that you would like to keep  


vi /etc/php/5.6/apache2/php.ini
# Change: upload_max_filesize = 120M

vi /etc/apache2/apache2.conf
Change: ------------------------
User asterisk
#  ${APACHE_RUN_USER}
Group asterisk
# ${APACHE_RUN_GROUP}
End Change: --------------------


# Setup database for FreePBX
export ASTERISK_DB_PW=tempo123 
mysqladmin -u root create asterisk
mysqladmin -u root create asteriskcdrdb


mysql -u root -e "GRANT ALL PRIVILEGES ON asterisk.* TO asterisk@localhost IDENTIFIED BY '${ASTERISK_DB_PW}';"
mysql -u root -e "GRANT ALL PRIVILEGES ON asteriskcdrdb.* TO asterisk@localhost IDENTIFIED BY '${ASTERISK_DB_PW}';"
mysql -u root -e "flush privileges;"

/etc/init.d/asterisk stop 

# Remove the files we created earlier 
cd /etc/asterisk 
mv extensions.conf extensions.conf.sample
mv sip.conf sip.conf.sample


cd /usr/local/src/freepbx 
./start_asterisk start
./install 

# Everything else is default. Just select appropriate username and password 
Database username [root]: asterisk
Database password: tempo123 


vi /etc/apache2/apache2.conf  
# Change: ------------------------------
<Directory /var/www/>
        Options Indexes FollowSymLinks
        AllowOverride All
        Order allow,deny
        Allow from all
</Directory>
# End Change: --------------------------


# Only do this if you get a permission error
vi /etc/apache2/sites-enabled/000-default.conf 

# Change: ------------------------------
<Directory "/var/www">
    Require all granted
</Directory>
# End Change: --------------------------


# Enable modules needed by FreePBX
a2enmod php5.6   # Enable PHP Module 
a2enmod rewrite  # Allow .htaccess 
/etc/init.d/apache2 restart

Start at: http://ipaddress/admin


# Fix some settings for client usability 
Settings (Top Menu) -> Asterisk SIP Settings -> Chan SIP -> IP Configuration -- Public IP 
Settings (Top Menu) -> Asterisk SIP Settings -> Chan SIP -> Bind Port -- 5060 
Settings (Top Menu) -> Asterisk SIP Settings -> Chan SIP -> TLS Bind Port -- 5061 

# Change listen port to 5065 in PJSIP Settings 

/etc/init.d/asterisk stop 
/etc/init.d/asterisk start 

# Create extensions and then check if listening on correct port 
netstat -ntulp | grep 5060 

# Connect to running asterisk instance 
asterisk -rvvvvv 

# Check peers (from the asterisk CLI, not bash CLI)
sip show peers 



# -----------------------------------
# Asterisk with FreePBX - First Setup 
# -----------------------------------
# 
# Dr. Nauman 
# http://recluze.net/learn  
# 
# -----------------------------------
