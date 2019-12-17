---
title: Ganglia Grill Graphs
author: jgoulah
type: post
date: 2013-03-26T03:55:53+00:00
url: /2013/03/ganglia-grill-graphs/
categories:
  - Graphs
tags:
  - cyberq
  - ganglia
  - gmetric
  - graphs
  - grill

---
One of my favorite things to do (other than stare at a computer all day) is to cook. I recently bought a new smoker grill, and quickly realized that I was running outside the entire day to tune the vents to keep the temperature in the range I wanted for a 12 hour slow cook. I did a little bit of research and turns out there are some decent devices that provide a fan to keep the temperature consistent. Even better, there was one equipped with a web server that can be accessed over wi-fi, called the <a href="http://store.thebbqguru.com/weborderentry/CyberQ%20WiFi" title="CyberQ" target="_blank">CyberQ</a>. 

Its not cheap, but life is short, and it will maintain the temperature you set up to 475 degrees Fahrenheit. I already have Ganglia setup in my house for basic server monitoring (you run linux servers in your house too right?) and since the device connects to the home network I figured it would be easy enough to query and graph. Luckily the people that made this device were nice enough to provide an XML endpoint that is easy to parse for all the information you want. There are a few different endpoints but I request the _config.xml_ version since it gives you the most information. 

So I <a href="https://github.com/jgoulah/gmetric-cyberq" title="gmetric-cyberq" target="_blank">wrote a gmetric</a> to query the box and throw the data into a few ganglia graphs. As of this writing I graph the fan speed, the pit temp, and the three food temps it can provide. There is a bunch of other data returned, but this was most important for me initially. 

While this device would normally be used for low and slow cooks, I did a faster high temp grill cook just to try it out. I set the temp to the max of 475 and you can see it took a bit of time to bring the grill up to that temp, and the fan output to get it there. Once it got there, you can see it drop when I opened up the lid to put the food on. The rest of the fluctuation is due to opening the lid and flipping the food:

<a href="http://blog.johngoulah.com/2013/03/ganglia-grill-graphs/pit-temp/" rel="attachment wp-att-955"><img src="http://blog.johngoulah.com/wp-content/uploads/2013/03/pit-temp.jpg" alt="pit-temp" width="482" height="184" class="alignleft size-full wp-image-955" srcset="http://blog.johngoulah.com/wp-content/uploads/2013/03/pit-temp.jpg 482w, http://blog.johngoulah.com/wp-content/uploads/2013/03/pit-temp-300x114.jpg 300w" sizes="(max-width: 482px) 100vw, 482px" /></a>

<a href="http://blog.johngoulah.com/2013/03/ganglia-grill-graphs/fan/" rel="attachment wp-att-960"><img src="http://blog.johngoulah.com/wp-content/uploads/2013/03/fan.jpg" alt="fan" width="480" height="183" class="alignleft size-full wp-image-960" srcset="http://blog.johngoulah.com/wp-content/uploads/2013/03/fan.jpg 480w, http://blog.johngoulah.com/wp-content/uploads/2013/03/fan-300x114.jpg 300w" sizes="(max-width: 480px) 100vw, 480px" /></a>

<br style="clear:both;" />

and then here is the food, two steaks cook rather quickly at that high of a temperature:

<br style="clear:both;" />

<a href="http://blog.johngoulah.com/2013/03/ganglia-grill-graphs/food1/" rel="attachment wp-att-962"><img src="http://blog.johngoulah.com/wp-content/uploads/2013/03/food1.jpg" alt="food1" width="480" height="185" class="alignleft size-full wp-image-962" srcset="http://blog.johngoulah.com/wp-content/uploads/2013/03/food1.jpg 480w, http://blog.johngoulah.com/wp-content/uploads/2013/03/food1-300x115.jpg 300w" sizes="(max-width: 480px) 100vw, 480px" /></a>

<a href="http://blog.johngoulah.com/2013/03/ganglia-grill-graphs/food-temp/" rel="attachment wp-att-961"><img src="http://blog.johngoulah.com/wp-content/uploads/2013/03/food-temp.jpg" alt="food-temp" width="480" height="183" class="alignleft size-full wp-image-961" srcset="http://blog.johngoulah.com/wp-content/uploads/2013/03/food-temp.jpg 480w, http://blog.johngoulah.com/wp-content/uploads/2013/03/food-temp-300x114.jpg 300w" sizes="(max-width: 480px) 100vw, 480px" /></a>

<br style="clear:both;" />
