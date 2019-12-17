---
title: Configuring a SonicWall to Pass OpenVPN Traffic
author: jgoulah
type: post
date: 2017-05-21T18:18:33+00:00
draft: true
url: /?p=1341
categories:
  - VPN
tags:
  - firewall
  - networking
  - openvpn
  - security
  - sonicwall
  - VPN

---
There came a time last year when it started making sense for me to look into office firewall and VPN solutions &#8211; we wanted to be able to get to the internal network securely from home, primarily as a means for our employees to reach our internal fileserver. In addition I needed some seamless and easy wireless access points, and so upon some recommendations went with the <a href="https://www.sonicwall.com/products/tz400/" title="sonicwall tz400" target="_blank">SonicWall TZ400</a>. I&#8217;m sure people have opinions about this and I know its not necessarily _the best_ thing out there, but it worked for our time and resource constraints (ie. solely me at the time). In any case, the thing is pretty easy to configure and fit my needs rather well, but I would say the VPN solutions that work with the device are close to terrible. It mostly comes down to the lack of compatible clients that are offered, not so much the sonicwall server side configuration, but you get locked into what I&#8217;d consider bulky clients with licensing costs. The best client that wasn&#8217;t absolutely unusable is <a title="VPN Tracker" href="https://www.vpntracker.com" target="_blank">VPN Tracker</a>. It is a little clunky (requires an account that users must login to their client with) but it does work, and they did have really great support when I was trying to initially get some settings to match the server. However, it costs per user, per year and there are a lot of <a href="https://www.sparklabs.com/viscosity/" title="viscosity" target="_blank">excellent</a> <a href="https://tunnelblick.net/" title="tunnelblick" target="_blank">clients</a> out there that are free to cheap if only <a title="openvpn" href="https://openvpn.net/" target="_blank">OpenVPN</a> were supported by SonicWall. But it turns out Sonicwall wants to also charge a yearly licensing cost and so it doesn&#8217;t make sense for them to support the open solution.

To get around this limitation, you can install OpenVPN on any server and then use that as a proxy to your internal network. And by configuring the Sonicwall (or any firewall you have really) to pass the traffic correctly, you won&#8217;t be bound to their protocols or service agreements. For OpenVPN setup I&#8217;d recommend the <a title="chef openvpn cookbook" href="https://supermarket.chef.io/cookbooks/openvpn" target="_blank">chef cookbooks</a> which will get you up and running in no time. There is similar support for <a href="https://github.com/luxflux/puppet-openvpn" title="puppet openvpn" target="_blank">puppet</a> or <a href="https://github.com/Stouts/Stouts.openvpn" title="ansible openvpn" target="_blank">ansible</a>. Do yourself a favor here and use some kind of configuration management.

The chef cookbook sets up the vpn <a title="subnet 10.8.0.0/16" href="https://github.com/sous-chefs/openvpn/blob/master/attributes/default.rb#L56" target="_blank">subnet on 10.8.0.0/16</a> so thats what this example is using. Additionally we run the vpn server as <a title="port 1194" href="https://github.com/sous-chefs/openvpn/blob/master/attributes/default.rb#L83" target="_blank">UDP on port 1194</a>.

There&#8217;s only a few rules needed once your server is setup and <a title="test openvpn port" href="https://serverfault.com/questions/262474/how-to-check-that-an-openvpn-server-is-listening-on-a-remote-port-without-using" target="_blank">communicating</a>. I&#8217;m going into detail about how this was done on the Sonicwall but the concept can translate pretty easily to any firewall based device/software that you&#8217;re running.

### Preliminary Setup

On the Sonicwall, you first have to setup a &#8220;service&#8221; for OpenVPN, under **Network->Services**. This should be using UDP protocol on port 1194.

You also need some address objects, under **Network -> Address Objects**. We need to define the OpenVPN Subnet as _10.8.0.0/255.255.0.0_. We&#8217;ll also define our VPN Server which may be hypothetically something like _192.168.1.172/255.255.255.255_.

### Firewall Rule

Go to **Firewall -> Access Rules**. We need a firewall rule from the external network to our local network. It will accept any source port, destined to our OpenVPN service on port 1194. The source of the packets could be anything and destination would be our external IP address.

<table style="width:50%" align="center">
  <tr>
    <td>
      From
    </td>
    
    <td>
      WAN
    </td>
  </tr>
  
  <tr>
    <td>
      To
    </td>
    
    <td>
      LAN
    </td>
  </tr>
  
  <tr>
    <td>
      Source Port
    </td>
    
    <td>
      Any
    </td>
  </tr>
  
  <tr>
    <td>
      Service
    </td>
    
    <td>
      OpenVPN
    </td>
  </tr>
  
  <tr>
    <td>
      Source
    </td>
    
    <td>
      Any
    </td>
  </tr>
  
  <tr>
    <td>
      Destination
    </td>
    
    <td>
      WAN IP
    </td>
  </tr>
</table>



### NAT Policy

Go to **Network -> NAT Policies**. We need to setup <a href="https://en.wikipedia.org/wiki/Network_address_translation" title="NAT" target="_blank">Network Address Translation</a> for the packets communicating through our OpenVPN. 

<table style="width:50%" align="center">
  <tr>
    <td>
      Original Source
    </td>
    
    <td>
      Any
    </td>
  </tr>
  
  <tr>
    <td>
      Translated Source
    </td>
    
    <td>
      Original
    </td>
  </tr>
  
  <tr>
    <td>
      Original Destination
    </td>
    
    <td>
      WAN IP
    </td>
  </tr>
  
  <tr>
    <td>
      Translated Destination
    </td>
    
    <td>
      VPN Server IP
    </td>
  </tr>
  
  <tr>
    <td>
      Translated Service
    </td>
    
    <td>
      Original
    </td>
  </tr>
  
  <tr>
    <td>
      Inbound/Outbound Interface
    </td>
    
    <td>
      Any
    </td>
  </tr>
</table>



### Route

Go to **Network -> Routing**. The SonicWall needs to know how to route to your VPN clients because they are going to be connected to this other subnet that we defined on the OpenVPN server. So we need to tell it anything coming from our LAN destined back out to the VPN clients will use the VPN Server as the gateway.

<table style="width:70%" align="center">
  <tr>
    <td>
      Source
    </td>
    
    <td>
      LAN Subnets
    </td>
  </tr>
  
  <tr>
    <td>
      Destination
    </td>
    
    <td>
      OpenVPN Subnet (10.8.0.0/255.255.0.0)
    </td>
  </tr>
  
  <tr>
    <td>
      Service
    </td>
    
    <td>
      Any
    </td>
  </tr>
  
  <tr>
    <td>
      Gateway
    </td>
    
    <td>
      VPN Server IP
    </td>
  </tr>
  
  <tr>
    <td>
      Translated Service
    </td>
    
    <td>
      Original
    </td>
  </tr>
  
  <tr>
    <td>
      Interface
    </td>
    
    <td>
      LAN (X0 in my case)
    </td>
  </tr>
</table>



## Conclusion

That should be all there is to configuring the rules needed to pass the packets correctly and you should be able to connect with easier to use clients. Two clients that I&#8217;d recommend would be <a href="https://www.sparklabs.com/viscosity/" title="viscosity" target="_blank">viscosity</a> and <a href="https://tunnelblick.net/" title="tunnelblick" target="_blank">tunnelblick</a> but there are others including openvpn itself. This frees us from both client and server side licensing burdens of the solutions built directly into the SonicWall.
