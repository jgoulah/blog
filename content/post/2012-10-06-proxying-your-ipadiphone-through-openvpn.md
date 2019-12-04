---
title: Proxying Your iPad/iPhone Through OpenVPN
author: jgoulah
type: post
date: 2012-10-06T23:29:18+00:00
url: /2012/10/proxying-your-ipadiphone-through-openvpn/
categories:
  - iOS
  - VPN
tags:
  - ios
  - iPad
  - iphone
  - openvpn
  - PAC
  - proxy
  - SOCKS
  - virtual private network
  - VPN

---
## Intro

It comes up often how to connect to our office <a title="openvpn" href="http://openvpn.net/" target="_blank">openvpn</a> network using an iPad or iPhone. On OSX its pretty simple, use <a title="viscosity" href="http://www.sparklabs.com/viscosity/" target="_blank">Viscosity</a> or <a title="tunnelblick" href="http://code.google.com/p/tunnelblick/" target="_blank">Tunnelblick</a>. But to my knowledge there is nothing like that for iDevices. However its possible to connect these using a SOCKS proxy. The SOCKS server lives on your laptop connected to the VPN, and the iPhone/iPad will be setup to connect through that. Obviously you should only do this on a secured wireless network and/or secure the SOCKS server so that only you have access. I wrote these notes a couple years ago and figured its worth sharing since it comes up once in a while.

## Setting Up the SOCKS Server

Setting up the server is really easy, we can use ssh &#8211; just run this command on your laptop that is connected to your VPN

<pre>ssh -N -D 0.0.0.0:1080 localhost</pre>

If you want it to run in the background also use the -f option. You may also want to setup some access control with <a title="iptables" href="http://en.wikipedia.org/wiki/Iptables" target="_blank">iptables</a>, which is a bit out of scope of this article but more information can be found <a title="socks5" href="http://www.catonmat.net/blog/linux-socks5-proxy" target="_blank">here</a>.

## Setting Up the iPhone/iPad to use SOCKS

### Setup the PAC File

The only way to configure the iPhone/iPad to use SOCKS is to setup a <a title="PAC" href="http://en.wikipedia.org/wiki/Proxy_auto-config" target="_blank">PAC file</a>. Create a file with the _.pac_ extension, and put this into it:

<pre>function FindProxyForURL(url, host)
{
return "SOCKS 192.168.X.XXX";
}</pre>

Make sure to use the IP address of your laptop that we setup the SOCKS server on. Now put this file in any web accessible location. It doesn&#8217;t matter if its internal to your network or external, as long as you can access it from the web. How to actually serve a page is beyond the scope of this article, but if you&#8217;ve gotten this far you probably know how to do this.

## Configure the iPhone/iPad

Now you just have to tell the iPad to use the PAC file so that it will proxy web requests through the laptops VPN.

Click:  **Settings** -> **WiFi**

Then click the blue arrow to the right of your access point and under HTTP Proxy choose Auto. In the URL field, put the full URL to the PAC file that we setup. Make sure to put the _http://_ protocol in this URL line. For example this may look something like: http://yourserver.com/myproxy.pac

Sometimes getting this setting to stick is tricky. I recommend clicking out of the text field into another field and letting the iPhone spinner in the upper left finish.

## Conclusion

If you did everything right you should be able to hit websites behind your VPN connection. One way to debug that its working is to startup ssh with the -vvv option. When you request pages through the proxy you will see a bunch of output. If there is no output you&#8217;re not using the proxy.