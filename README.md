RO en attente de la fin de migration





































# network_howto
Mémo : tout ce qui est relatif au réseau en vrac (avant d'avoir une bonne compréhension)

# DHCP
## DHCP Linux
### DHCP Linux Client
Afficher serveurs DHCP et leurs baux
```sh
grep -IR "DHCPOFFER" /var/log/*
```
Sources et autres méthodes [ici](https://www.putorius.net/find-dhcp-server-ip-linux.html)
## DHCP Windows
### DHCP Windows client
Afficher serveur DHCP du bail actuel
```batch
ipconfig /all
```

# Sniff
Voir [rubrique dédiée](https://github.com/lenainjaune/sniff)

# Accés réseau par nom
## Accés réseau par le domaine local (différent de DNS)
Debian
```sh
# domaine local (tout ne doit pas être utile)
apt install avahi-daemon avahi-discover winbind libnss-winbind libnss-mdns
# semblent inutiles : dnsutils
```
Nota : nsswitch est modifié comme ceci
```sh
grep hosts: /etc/nsswitch.conf 
hosts:          files mdns4_minimal [NOTFOUND=return] dns
```
## Accès réseau par le nom seul
ping xxx.local fonctionne ou pas MAIS ping xxx (seul) fonctionne

Je pense que ça vient d'un paquet lié à Samba (depuis emmabuntus avec samba installé par défaut, 
après désinstallation de Samba => ping xxx ne marche plus
## Bonjour (Apple) depuis Windows (7, etc.)
https://www.manageengine.com/products/desktop-central/software-installation/silent_install_Bonjour-Print-Services-2.0.2.html

Silent Installation Switch 	cscript.exe install_bonjour.vbs
Ou bien directement : msiexec /i BonjourPS64.msi /qn (voir aussi [post-install_win7.bat](https://github.com/lenainjaune/post-install/blob/main/README.md))
```vbs
install_bonjour.vbs ouvert
'ManageEngine Desktop Central 

'Script to install Bonjour
'======================================================
'Get the Agent Installed directory from the Registry location details
'====================================================================
Err.Clear

Set WshShell = WScript.CreateObject("WScript.Shell")
checkOSArch = WshShell.RegRead("HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Session Manager\Environment\PROCESSOR_ARCHITECTURE")
returnValue = 0
'Wscript.Echo checkOSArch 

if Err Then
	Err.Clear
	'WScript.Echo "The OS Architecture is unable to find ,so it was assumed to be 32 bit"
	regkey = "HKEY_LOCAL_MACHINE\SOFTWARE\AdventNet\DesktopCentral\DCAgent"
else
	if checkOSArch = "x86" Then
		'Wscript.Echo "The OS Architecture is 32 bit"
		regkey = "HKEY_LOCAL_MACHINE\SOFTWARE\AdventNet\DesktopCentral\DCAgent"
	else
		'Wscript.Echo "The OS Architecture is 64 bit"
		regkey = "HKEY_LOCAL_MACHINE\SOFTWARE\Wow6432Node\AdventNet\DesktopCentral\DCAgent"
	End IF
End If

agentDir = WshShell.RegRead(regkey & "\DCAgentInstallDir")

'WScript.Echo agentDir

location = agentDir&"bin\7za.exe"  

'WScript.Echo location
'Extraction of the BonjourPSSetup
returnValue =WshShell.Run(Chr(34)& location &Chr(34)& " x -y BonjourPSSetup.exe",0,TRUE)

'Once the extraction is sucess , install the pre-requisite and msi in the specific order 
'==============================================================================

if returnValue <> 0 Then
	Wscript.Quit returnValue
else
	if checkOSArch = "x86" Then
		returnValue=WshShell.Run("msiexec /i Bonjour.msi /qn", 0, TRUE)
		returnValue=WshShell.Run("msiexec /i BonjourPS.msi /qn", 0, TRUE)
	else
		WScript.Echo "x64"
		returnValue=WshShell.Run("msiexec /i Bonjour64.msi /qn", 0, TRUE)
		returnValue=WshShell.Run("msiexec /i BonjourPS64.msi /qn", 0, TRUE)
	End IF
	
end if
Wscript.Quit returnValue
```
## Bugs
### host.local -> host-2.local
Parfois avahi ne fonctionne pas comme attendu et depuis système distant ```ping host.local``` indique que l'hôte n'est pas accessible MAIS si on utilise **host-2.local** à la place, ça fonctionne.

TODO : centraliser autres trouvailles ici

Voir aussi [cette piste pour forcer le nom](https://raspberrypi.stackexchange.com/questions/22485/how-to-debug-avahi-daemon-hostname-not-coming-through/120864#120864)
# Pont niv2 (bridge)
Un bridge (couche 2 du modèle OSI), permet de relier des réseaux entre eux. C'est particulièrement pratique pour que des machines virtuelles (VMs) aient accès au LAN et aient accès entre elles.
## Network Manager
Normalement présent ou installable sur toutes les distributions Linux modernes.
```sh
# https://www.cyberciti.biz/faq/how-to-add-network-bridge-with-nmcli-networkmanager-on-linux/

# Ajouter bridge
nmcli con add ifname br0 type bridge con-name br0
nmcli con add type bridge-slave ifname eno1 master br0
# nmcli -t connection show
nmcli con modify br0 bridge.stp no
# nmcli -t -f bridge con show br0
nmcli con down "Connexion filaire 1" && nmcli con up br0
# patienter un peu des fois ça met du temps (?)

# Supprimer bridge
nmcli con down br0 && nmcli con up "Connexion filaire 1"
# ping google.fr
nmcli connection delete br0
# TODO : trouver méthode pour supprimer automatiquement tous les ports du bridge
nmcli connection delete bridge-slave-eno1
# nmcli -t connection show

# Problème bridge
# PB #01 : remarqué quand je n'ai pas de réseau au boot (ex : BOX éteinte)
# => bascule sur Wired et br0 ne peut se connecter au réseau
nmcli -t connection show --active
br0:843bb6a4-064a-4151-a71d-da8b5c76d344:bridge:br0
# => br0 actif
ip addr show dev br0
41: br0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default qlen 1000
    link/ether 2a:e9:6b:38:0a:62 brd ff:ff:ff:ff:ff:ff
# => pas d'IP
# Solution : basculer
nmcli con down "Wired" && nmcli con up br0
# => Connexion « Wired » désactivée (chemin D-Bus actif : /org/freedesktop/NetworkManager/ActiveConnection/52)
#    Connexion activée (master waiting for slaves) (Chemin D-Bus actif : /org/freedesktop/NetworkManager/ActiveConnection/54)
# ... après quelques instants (PAS immédiat)
ip ad li br0
41: br0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 2a:e9:6b:38:0a:62 brd ff:ff:ff:ff:ff:ff
    inet 192.168.0.11/24 brd 192.168.0.255 scope global dynamic noprefixroute br0
       valid_lft 863611sec preferred_lft 863611sec
...
# => résolu !
```
Voir aussi [ici](https://www.golinuxcloud.com/configure-network-bridge-nmcli-static-dhcp/) (utile)
# Wifi
## Linux
Suivre méthodiquement [ce lien](https://forum.ubuntu-fr.org/viewtopic.php?id=1997812) qui donne toutes les commandes utiles et [cette procédure](https://forum.ubuntu-fr.org/viewtopic.php?pid=21716388#p21716388) qui explique comment installer le pilote pour mon dongle TL-WN823N (test en live session) ou [celle là](https://askubuntu.com/questions/1211157/how-do-i-get-a-tp-link-tl-wn823n-v3-wireless-adapter-working/1212939#1212939) plus détaillée apparemment.

Mon essai pour installer le dongle depuis session Live (sans reboot donc) :
```sh
# https://askubuntu.com/questions/1211157/how-do-i-get-a-tp-link-tl-wn823n-v3-wireless-adapter-working/1212939#1212939
# https://forum.ubuntu-fr.org/viewtopic.php?pid=21716388#p21716388

root@CZ-LIVE:~# uname -a
Linux CZ-LIVE 5.18.0-2-amd64 #1 SMP PREEMPT_DYNAMIC Debian 5.18.5-1 (2022-06-16) x86_64 GNU/Linux

   56  apt update && apt install dkms linux-headers-generic
  # => headers 5.18.0-4-amd64
   57  wget https://github.com/Mange/rtl8192eu-linux-driver/archive/refs/heads/realtek-4.4.x.zip
   58  unzip realtek-4.4.x.zip
  # headers pour 5.18.0-2 introuvables => on force avec le generic
   92  dkms install rtl8192eu-linux-driver-realtek-4.4.x -k 5.18.0-4-amd64
  # long à installer
   
  100  lshw -c network
  # => You should see the line driver=8192eu

   93  lspci -k -nn | grep -A 3 -i net 
   94  lsmod
   99  iwconfig

  # 
  101  echo "blacklist rtl8xxxu" > /etc/modprobe.d/rtl8xxxu.conf


  # supprimer
   67  dkms status
   68  dkms remove rtl8192eu/1.0 --all

  # désinstaller    
  107  dkms uninstall rtl8192eu/1.0 --all
  # désinstallation immédiate, et reinstall pas possible, il faut supprimer avant ...
```
# Access Point Wifi (brouillon)
## Linux
Objectif : j'ai 2 dongles USB wifi et je veux m'en servir en tant que point d'accès. Ainsi, je rend disponible la wifi en local.
### Ralink RT5372 (dongle sans nom)
```sh
# Pour la clé Wifi Ralink RT5372 (https://wiki.debian.org/fr/rt2800usb), achat Leclerc < 10€, il n'y a pas besoin de pilote
# Peut être que si, voir : https://www.mediatek.com/products/broadband-wifi/rt5370 (DPO_RT5572_LinuxSTA_2.6.1.3_20121022)

user@host:~# sudo cat /etc/apt/sources.list
# deb http://ftp.fr.debian.org/debian/ stretch main
deb http://ftp.fr.debian.org/debian/ stretch main non-free

user@host:~# sudo apt-get update

# indispensables et NON indispensables (haveged est recommandé en raison de la Low entropy et rfkill - voir dessous)
user@host:~$ sudo apt install -y make git iw hostapd haveged firmware-misc-nonfree rfkill linux-headers-$(uname -r)
user@host:~$ #sudo apt install -y util-linux procps iproute2 dnsmasq iptables wireless-tools
user@host:~$ 

user@host:~$ git clone https://github.com/oblique/create_ap
user@host:~$ cd create_ap
user@host:~$ make install
user@host:~$ cd .. && rm -rf create_ap
user@host:~$ ip ad
...
3: br0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether d8:cb:8a:36:82:ed brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.127/24 brd 192.168.1.255 scope global xenbr0
       valid_lft forever preferred_lft forever
...
19: wlx70f11c07152e: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc mq state DOWN group default qlen 1000
    link/ether 70:f1:1c:07:15:2e brd ff:ff:ff:ff:ff:ff

# On active l'AP Wifi (wlx... IF USB et xenbr0 IF avec IP - ici le bridge Xen)
user@host:~$ sudo bash -c "rfkill unblock wlan ; create_ap --no-virt -m bridge wlx70f11c07152e br0 test-ap azerty123456"
# nota 1 : environnement virtuel ? WARN: Your adapter does not fully support AP virtual interface, enabling --no-virt
# nota 2 : WARN: Low entropy detected. We recommend you to install `haveged'

=> OK j'ai la wifi sur mon smartphone

user@host:~$ ping 8.8.8.8
64 bytes from 8.8.8.8: icmp_seq=1 ttl=116 time=12.4 ms
...
^C
--- 8.8.8.8 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 4ms
rtt min/avg/max/mdev = 11.650/12.554/13.661/0.833 ms

=> OK quand on interrompt la commande on a toujours accès au réseau


# nota : dépendance rfkill car au 1er lancement de create_ap j'ai eu les messages
rfkill: WLAN soft blocked
...
Error: Failed to run hostapd, maybe a program is interfering.
If an error like 'n80211: Could not configure driver mode' was thrown
try running the following before starting create_ap:
    nmcli r wifi off
    rfkill unblock wlan
# nmcli r ... n'a pas fonctionné mais rfkill ... a fonctionné
```
### TP-Link TL-WN823N V3
Pilotes Linux : https://www.tp-link.com/us/support/download/tl-wn823n/ (TL-WN823N(US)_V3_160315_Linux)
## Windows
### Ralink RT5372 (dongle sans nom)
https://www.mediatek.com/products/broadband-wifi/rt5370 (IS_Setup_ICS_011916_1.5.39.173)
### TP-Link TL-WN823N V3
https://www.tp-link.com/us/support/download/tl-wn823n/ (TL-WN823N(US)_V3_181024_Win)
# Ping
## Statistiques sans interruption
Linux : [Ctrl + \\](https://unix.stackexchange.com/questions/143845/check-ping-statistics-without-stopping)

# Windows - Accès entre PCs en zero-conf
Je voudrais copier les données de PC1 vers PC2 le plus simplement. Je branche un câble ethernet entre les 2 PCs sur une interface réseau RJ-45. Les cartes réseau des PCs actuels possèdent la fonctionnalité auto-MDX ainsi on peut prendre indistinctement un câble droit ou croisé. Vu que les PCs sont connectés entre eux, chaque PC aura une adresse automatique par autonégociation APIPA dans le réseau **169.254.0.0/16**. La connectivité APIPA pourra se vérifier en testant avec pc1 <- **ping** *hostname* -> pc2 (on peut utiliser directement les hostname mais aussi les IPs. Si rien ne se passe, c'est qu'il y a un élément qui bloque la communication tel un parefeu ; dans ce cas il faut le désactiver. Si après avoir éliminé toute cause de blocage, la connectivité n'est toujours pas bonne il faut creuser (voir [1]). La connectivité est à ce stade opérationnelle.

## Windows édition Pro, Entreprise etc. (autre que familiale)
On peut directement ouvrir un partage administratif depuis PC1 vers PC2, en ouvrant un navigateur et en tapant dans la barre d'adresse **\\pc2\c$**. Si tout se passe bien, une fenêtre d'authentification devrait s'afficher. Cela indique qu'il faut un compte pour se connecter et celui qui passera dans tous les cas c'est le compte **administrateur** MAIS par défaut ce compte est désactivé et sans mot de passe. Il faut donc l'activer, fournir un mot de passe et recommencer l'opération. Une fois la tâche achevée, il conviendra de désactiver à nouveau **administrateur** et d'éliminer son mot de passe.
TODO : éliminer le mot de passe administrateur (et non le rendre vide à priori).

Nota : pour le cas de Windows édition familiale, on ne pourra pas gérer le compte administrateur. Il faudra donc partager un dossier (%userprofile% par exemple) et donner les droits "Controle Total" pour *utilisateur* dont on connait le mot de passe (sinon on peut le redéfinir) et se connecter au partage avec lui et son mot de passe.

# Windows gestion IP depuis CLI (admin)
[Source](https://tweaks.com/windows/40339/configure-ip-address-and-dns-from-command-line/)
```bash
netsh interface ipv4 show interfaces
:: affiche "Connexion au réseau local" ; 'é' et non ''' ou '�' ... ah les encodages sous Windows :|
netsh interface ip set address name="Connexion au r�seau local" static 192.168.1.70 255.255.255.0 192.168.1.1 1"
netsh interface ip set dns name="Connexion au r�seau local" static 192.168.1.10
```
Attention : ces modifications sont persistantes, il existe surement une manière de faire comme avec ```ip ad```
TODO : permettre la configuration temporaire

# Gestionnaire de réseau sous Linux (à compléter, pour mémoire)
Sous Linux, il existe plusieurs gestionnaires de réseau. [Cette procédure](https://askubuntu.com/questions/1031439/am-i-running-networkmanager-or-networkd/1246465#1246465) permet de savoir lequel est en place.

A noter aussi qu'on peut mixer les technologies. Par exemple on peut confier la gestion de lo à /etc/network/interfaces et les autres interfaces à NetworkManager.

TODO : retrouver la source pour mixer avec NetworkManager

# Parefeu
## Windows : autoriser ICMP depuis LAN (évite de désactiver Parefeu)
Invite de commande en admin
```batch
C:\Users\USER\Desktop>netsh advfirewall firewall add rule name="ICMP Allow incoming LAN echo request" protocol=icmpv4:8,any dir=in action=allow remoteip=localsubnet
```
Sources : généralistes sur [malekal](https://www.malekal.com/netsh-advfirewall-configurer-pare-feu-windows-defender-invite-de-commandes/) et [ici](https://www.howtogeek.com/howto/windows-vista/allow-pings-icmp-echo-request-through-your-windows-vista-firewall/) pour ce qui concerne ICMP (voir aussi [post-install_win7.bat](https://github.com/lenainjaune/post-install/blob/main/README.md))
# Bascule filaire/wifi
## Linux - Network Manager
Basé sur [Automatically enable and disable WiFi based on Ethernet connection with NetworkManager](https://blog.christophersmart.com/2021/11/02/automatically-enable-and-disable-wifi-based-on-ethernet-connection-with-networkmanager/comment-page-1/).

Nota : j'ai modifié les droits en passant de 744 à 745 (droit d'exécution pour les autres, pour exécuter à l'ouverture de session par exemple)

Testé avec succès sous Debian Bullseye
```sh
root@host:~$ cat << \EOF |sudo tee /etc/NetworkManager/dispatcher.d/70-wifi-wired-exclusive.sh
#!/bin/bash
export LC_ALL=C
 
enable_disable_wifi ()
{
    result=$(nmcli dev | grep "ethernet" | grep -w "connected")
    if [ -n "$result" ]; then
        nmcli radio wifi off
    else
        nmcli radio wifi on
    fi
}
 
if [ "$2" = "up" ]; then
    enable_disable_wifi
fi
 
if [ "$2" = "down" ]; then
    enable_disable_wifi
fi
EOF
user@host:~$ chown root:root /etc/NetworkManager/dispatcher.d/70-wifi-wired-exclusive.sh
user@host:~$ chmod 745 /etc/NetworkManager/dispatcher.d/70-wifi-wired-exclusive.sh
user@host:~$ systemctl restart NetworkManager
```
Sous XFCE4 on peut améliorer en déclenchant le script à chaque ouverture de session
```sh
user@host:~# mkdir -p /data/script $HOME/.config/autostart
user@host:~# cat << EOF > /data/script/wired_wifi_toggler.desktop 
[Desktop Entry]
Name=Bascule filaire/Wifi
Comment=Bascule automatiquement de la connexion filaire prioritaire à la Wifi
Exec=/etc/NetworkManager/dispatcher.d/70-wifi-wired-exclusive.sh "" up
Terminal=false
Type=Application
Icon=network-wired-disconnected
Categories=Network;
StartupNotify=false
EOF
user@host:~# chmod +x /data/script/wired_wifi_toggler.desktop
user@host:~# ln -s /data/script/wired_wifi_toggler.desktop $HOME/.config/autostart/wired_wifi_toggler.desktop 
```
# Désactiver totalement IPv6
## Linux
Attention : dans un cas, n'a pas fonctionné alors que ça aurait du, donc à revoir ! (en attendant j'ai désactivé depuis NetworkManager

[Source](https://askubuntu.com/questions/309461/how-to-disable-ipv6-permanently/309463#309463)

```sh
root@host:~# cat /etc/sysctl.conf | grep ^net.ipv6.conf
net.ipv6.conf.all.disable_ipv6 = 1
root@host:~# sysctl -p
```
# Routage
## Linux
### Passerelle par défaut
```sh
@# route add default gw 10.10.1.20
```
Ou
```sh
@# ip route add default via 10.10.1.20
```




[1]
Les causes possibles sont : un mauvais branchement, un câble défectueux, un défaut sur la carte réseau ou la carte mère, un pilote non installé ou non opérationnel, le système d'exploitation défectueux.
