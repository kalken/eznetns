# Why make this?
Most people who use a VPN today, connect to a vpn service online. On default settings, turning the VPN "on" will change the default route on the computer, making all traffic go thorough vpn. Sometimes you just want to run some programs via vpn, keeping everything else as it was. This is where eznetns comes in. It allows certain processes or programs to use a separate namespaces (with vpn), while keeping everything else as it is.

# eznetns
eznetns is a command line helper to configure separated network environments (namespaces), and control which processes that make use of them. the network environments do not affect the ordinary routing tables and default routes on the host. Its a simple solution built around ip netns, with support for firewall and wireguard interfaces using wg-quick config syntax. This method can be used to start chosen processes or services inside a netns (systemd scripts are included as an example). If wireguard is going to be configured, the wireguard package needs to be installed. You can create as many network namespaces as you need, and put each program in its desired space.

## Key features:

* Isolated network environments (supports as many as you need)
* Any process or program can easily be started inside the network environment
* Systemd startup script for configuring the isolated network environment
* Systemd startup scripts that explains how to start a process inside a network environment
* Systemd socket example, which explains how to bind a local port on the "host" to a service inside a isolated network environment.

## No external dependencies:
* No need for containers
* No need for bridges or virtual network adapters
* Default routing tables
* Default routing rules
* No firewall requirements (unless you want one)
* Each netns can have its own firewall
* Start services as any user through systemd


## Short story
1. Clone repository
2. Enable commands by linking them to /usr/local/sbin
3. create your /etc/eznetns/<name> (template files in repo)
4. create systemd services (template files in repo)
5. Enable systemd services to start at boot
6. done!

## Long story

### eznetns

eznetns is used to create and setup a network namespace. It is heavily based on the standard command "ip netns", and just makes it easier to configure the environment. It currently supports automagic configuration of nftables firewall, and wireguard interfaces. eznetns uses /etc/eznetns/[name]/ as base for its setup, with backward compability to /etc/netns (which ip netns uses). Put your files there. if /etc/eznetns/[name]/nftables.conf exist, nftables will be configured, if any wireguard config in /etc/eznetns/[netns]/wireguard is found, it will be configured.

### ezwgen

ezwgen is used to generate complete wireguard configs. It can also be used to change wireguard config. ezwgen also supports making system wide wg-quick config if no netns is specified. it expects settings in /root/.config/ezwgen. There is also support for netns with --netns which points to folder /root/.config/ezwgen/[netns]. this folder should contain <name>.conf (settings file) and <name> folder (with templates). Any setting in the settings file will always override any setting in a template file. A typical thing to specify in the settings file is **privatekey**. One file at random will be chosen from template folder and merged with the settings file any time this program is run. **ezwgen requires python 3**.

## Prevent dns leaks
You should use separate resolv.conf and nsswitch.conf in your netns to control how the netns resolves its hosts. example files are included in "templates" folder.

## Setup
This example assumes you want to create a netns named **vpn** and the source files are in **/opt/eznetns**

### Basic setup    
    # log in as root 
    sudo -Es

    # Enable commands 
    # Note: make sure /usr/local/sbin is in your shell PATH
    ln -s /opt/eznetns/bin/eznetns /usr/local/sbin/eznetns
    ln -s /opt/eznetns/bin/ezwgen /usr/local/sbin/ezwgen
    
    # help (as root)
    eznetns help
    ezwgen --help
    
    # add custom resolv.conf and nsswitch.conf
    cp -r /opt/eznetns/templates/eznetns /etc/eznetns/vpn

    # put your wireguard config in /etc/eznetns/vpn/wireguard/[interface].conf (with ezwgen or manually)

### Testing netns through cmd
    
    # manually setup netns
    eznetns vpn setup
    
    # list running netns
    eznetns list
    
    # exec through netns
    eznetns vpn execute mycommand
    # example: netns vpn exec curl ifconfig.co
    
    # enter netns
    eznetns vpn enter
    # Note: exit with "exit"
    
    # manually remove netns
    eznetns vpn remove
    
    
    
### systemd files
    # Note: microsocks needs to be installed on your system

    cp -r /opt/eznetns/templates/systemd /opt/systemd # or any other place you prefer
    # Edit files to match your needs

    # link to /etc/systemd/system
    systemctl link /opt/systemd/*.*
    systemctl link /opt/systemd/netns/*

    # start manually
    # Note: vpn-proxy.socket will trigger vpn-proxy.service when socket is activated.
    systemctl start microsocks vpn-proxy.socket vpn.target
    
    # enable services at boot
    systemctl enable microsocks vpn-proxy.socket vpn.target
    
## Notes: 
Services are available at localhost: 
       
* 1080 localhost -> microsocks host ip
* 1081 localhost -> microsocks netns ip through local socket
 
**If you add** more services to **/opt/systemd/eznetns**, also add them to **vpn.target** so they are started at boot.
    
# done!


## Tips & Trix

### More services:
If you want to add more services to vpn, just create a systemd file for it, and add the lines under [Service] section.

    myservice@.service:
    
    [Service]
    ...
    # namespace setup
    NetworkNamespacePath=/run/netns/%i
    BindReadOnlyPaths=/etc/eznetns/%i/resolv.conf:/etc/resolv.conf 
    BindReadOnlyPaths=/etc/eznetns/%i/nsswitch.conf:/etc/nsswitch.conf
    
Link it to systemd and enable myservice@vpn by itself, or add it to vpn.target

### Access from another computer

If you have this running on a server, and want to utilize the services you can easily use eternal terminal (or ssh) to tunnel ports. No need for vpn client!

1. Install eternal terminal on the server, and on the client
2. connect with something like: `et -t 6080:1080,6081:1081 mycomputer`
3. The proxys are now available on client localhost: 6080 (server ip) and 6081 (vpn ip)


Note: You can also put a **macvlan**-interface etc inside the netns, to open up services to the local network.


### Changing wireguard on the fly

something like this should would generate a new config and reload the interface wg0 inside the netns:
    
    ezwgen --netns vpn --dev wg0 && eznetns vpn wg.reload wg0

   
    


    
    
    
    
    
    
    
