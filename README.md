# network_howto
Mémo : tout ce qui est relatif au réseau en vrac (avant d'avoir une bonne compréhension)

# Accés réseau par le domaine local (différent de DNS)
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

# Accès réseau par le nom seul
ping xxx.local fonctionne ou pas MAIS ping xxx (seul) fonctionne

Je pense que ça vient d'un paquet lié à Samba (depuis emmabuntus avec samba installé par défaut, 
après désinstallation de Samba => ping xxx ne marche plus
