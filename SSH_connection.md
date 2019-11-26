# SSH connection tutorial

## MAC OSx

  - Before to connect the UDOO NEO through the USB cable open a terminal on your MAC
  - run the command "ifconfig" to check the network interfaces already present
  - Connect now the UDOO NEO through the USB cable
  - run again the "ifconfig" command in the terminal and note the name of the new interface that is added on the bottom of the output of the command (should be something similar to "en6")
  - assign 192.168.7.1 as IP address to the new network interface:<br>
    ``ifconfig <name_of_the_new_interface> 192.168.7.1 up `` <br> ( e.g ``ifconfig en6 192.168.7.1 up ``)
  - You can now check the connection with a ping to the UDOO NEO address: ``ping 192.172.7.2``

## Windows

## Linux