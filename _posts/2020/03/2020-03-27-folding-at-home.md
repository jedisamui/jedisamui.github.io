---
layout: post
title: "Folding@Home"
summary: "Help researchers around the world"
author: samui
date: "2020-03-27"
category: [project]
thumbnail: /images/2020/03/Screen-Shot-2020-03-26-at-3.33.49-PM.a0aa915afd634bd9976b88f4e620bfd1.png
permalink: /blog/folding-at-home/
published: true
---

The other day I mentioned that I would be starting a ‘Folding@Home’ project. You may be wondering what is “folding at home”? I had to explain it to my wife, as well. Let me see if I can make it more understandable for you, than I did for her.

Researchers around the world are working on various diseases that effect us, from Breast Cancer to the most recent coronavirus, Covid-19. While these researchers do have access to a lot of equipment and technology to aide in their efforts; many times, they need to run simulations and calculations that without help, could take an extremely long time. Thus delaying efforts to find cures or therapies.

This is where “Folding@Home” comes in. A client-side application was created that will allow researchers to send you what they call a “work unit” to be calculated. Using the horse power of your machine, the client will crunch the numbers for the work unit simulation and report the data back to the researchers. Many, like myself, in the home-lab community have some extra resources sitting idle that we can use towards this effort. So when I heard that several VMware employees were putting forth their resources for this effort, I decided that I would be one more among them.

For more info on [“Folding@Home”](https://foldingathome.org){:target="_blank"}, check out their website. Or watch this video.

\[embed\]https://youtu.be/RGGzMQ2oFrA\[/embed\]

To my surprise, the setup and configuration was super easy. I followed along with two readily available blogs.

- Paul Braren at TinkerTry: [https://tinkertry.com/vmware-fling-folding-at-home-vm](https://tinkertry.com/vmware-fling-folding-at-home-vm){:target="_blank"}
- William Lam at vGhetto: [https://www.virtuallyghetto.com/2020/03/how-to-manually-install-folding-home-on-vmware-photon-os.html](https://www.virtuallyghetto.com/2020/03/how-to-manually-install-folding-home-on-vmware-photon-os.html){:target="_blank"}  
    

I’m not going to detail out the configuration, as both Paul and William have done an outstanding job. But what I will say though is that I did not want to leave the Photon appliance with a DHCP configuration. With the various networks I have - in addition to the NSX-T overlay I have for my vCloud Director layout, I need to make sure my firewall can allow the needed traffic to the appliance - no matter where it ends up. Once the photon appliance booted and started running, I pressed “Alt-F2” to log in as ‘root’ and then configured a static IP address.

I will also add, that you should ensure to run the latest, [greatest version of the ova appliance](https://flings.vmware.com/vmware-appliance-for-folding-home){:target="_blank"} as the previous version would only work on vSphere versions that support vHW 13. Version 1.0.1 will run on vSphere 6.0. (This makes me happy as I had two Dell PE1950s sitting doing nothing in my garage.)

With the link I provided from William Lam, you could build a Photon appliance from scratch. However, what the appliance gives you that you don’t get building it from scratch, is the auto starting TOP and status display in the console. Of course, this could be added, but you would have to set that up.

![F@H Console](/images/2020/03/Screen-Shot-2020-03-26-at-3.35.22-PM.png)
F@H Console
{:style="color:gray;font-style:italic;font-size:90%;text-align:center;"}


Once up and running, open your web browser to http://<IP address>:7396 and check out the Folding@Home web client interface.

![F@H Web Interface](/images/2020/03/Screen-Shot-2020-03-26-at-3.33.49-PM.png)

F@H Web Interface
{:style="color:gray;font-style:italic;font-size:90%;text-align:center;"}


From the interface, you can ‘stop folding’, change how much dedicated CPU you give it, and view progress of the ‘folding’. From what I gather, work units may not process if you don’t provide 16vCPUs to the appliance. Also, the longer the appliance is up and running, you may notice that work units may not be assigned. A reboot of the appliance could jog yourself in the queue and restart getting more work units.

For now, I am no longer sitting idle and not doing anything to help with this pandemic. In addition to working from home and self-isolation, maybe by participating in this effort, I will help researchers find a cure, or possibly a treatment.
