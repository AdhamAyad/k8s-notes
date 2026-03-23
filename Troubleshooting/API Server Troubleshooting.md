cat /var/log/container/
cat /var/log/pod/
 if there is any thisng 
 cat /var/log/syslog | grep -C 5 "apiserver" 
 journalctl -u kubelet -e
 grep -C 3 -ir "" /var/log/

