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
Un bridge (couche 2 du modèle OSI), permet de relier des réseaux entre eux. C'est particulièrement pratique pour que des machines virtuelles (VMs) aient accès au LAN.
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
