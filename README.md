# network_howto
Mémo : tout ce qui est relatif au réseau en vrac (avant d'avoir une bonne compréhension)

# Accés réseau par nom
## Accés réseau par le domaine local (différent de DNS)
Debian
```sh
# domaine local (tout ne doit pas être utile)
apt install avahi-daemon avahi-discover winbind libnss-winbind 
# semblent inutiles : dnsutils libnss-mdns
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

# Supprimer bridge
nmcli con down br0 && nmcli con up "Connexion filaire 1"
# ping google.fr
nmcli connection delete br0
# TODO : trouver méthode pour supprimer automatiquement tous les ports du bridge
nmcli connection delete bridge-slave-eno1
# nmcli -t connection show
```
# Access Point Wifi (brouillon)
```sh
# Pour la clé Wifi Ralink RT5372 (https://wiki.debian.org/fr/rt2800usb), achat Leclerc < 10€

user@host:~# sudo cat /etc/apt/sources.list
# deb http://ftp.fr.debian.org/debian/ stretch main
deb http://ftp.fr.debian.org/debian/ stretch main non-free

user@host:~# sudo apt-get update

# indispensables et NON indispensables (haveged est recommandé en raison de la Low entropy et rfill - voir dessous)
user@host:~$ sudo apt-get install -y make git iw hostapd haveged firmware-misc-nonfree rfkill
user@host:~$ #sudo apt-get install -y util-linux procps iproute2 dnsmasq iptables wireless-tools

user@host:~$ git clone https://github.com/oblique/create_ap
user@host:~$ cd create_ap
user@host:~$ make install
user@host:~$ cd .. && rm -rf create_ap
user@host:~$ ip ad
...
3: xenbr0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
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
# Ping
## Statistiques sans interruption
Linux : [Ctrl + \\](https://unix.stackexchange.com/questions/143845/check-ping-statistics-without-stopping)

# Windows - Accès entre PCs en zero-conf
Je voudrais copier les données de PC1 vers PC2 le plus simplement. Je branche un câble ethernet entre les 2 PCs sur une interface réseau RJ-45. Les cartes réseau des PCs actuels possèdent la fonctionnalité auto-MDX ainsi on peut prendre indistinctement un câble droit ou croisé. Vu que les PCs sont connectés entre eux, chaque PC aura une adresse automatique par autonégociation APIPA dans le réseau **169.254.0.0/16**. La connectivité APIPA pourra se vérifier en testant avec pc1 <- **ping** *hostname* -> pc2 (on peut utiliser directement les hostname mais aussi les IPs. Si rien ne se passe, c'est qu'il y a un élément qui bloque la communication tel un parefeu ; dans ce cas il faut le désactiver. Si après avoir éliminé toute cause de blocage, la connectivité n'est toujours pas bonne il faut creuser (voir [1]). La connectivité est à ce stade opérationnelle.

## Windows édition Pro, Entreprise etc. (autre que familiale)
On peut directement ouvrir un partage administratif depuis PC1 vers PC2, en ouvrant un navigateur et en tapant dans la barre d'adresse **\\pc2\c$**. Si tout se passe bien, une fenêtre d'authentification devrait s'afficher. Cela indique qu'il faut un compte pour se connecter et celui qui passera dans tous les cas c'est le compte **administrateur** MAIS par défaut ce compte est désactivé et sans mot de passe. Il faut donc l'activer, fournir un mot de passe et recommencer l'opération.

Nota : pour le cas de Windows édition familiale, on ne pourra pas gérer le compte administrateur. Il faudra donc partager un dossier (%userprofile% par exemple) et donner les droits "Controle Total" pour *utilisateur* dont on connait le mot de passe (sinon on peut le redéfinir) et se connecter au partage avec lui et son mot de passe


[1]
Les causes possibles sont : un mauvais branchement, un câble défectueux, une carte réseau défectueuse ou la carte mère, un pilote non installé ou opérationnel, le système d'exploitation défectueux.
