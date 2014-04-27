---
layout: post
title: "Set up your own IRC bouncer and custom hostname"
date: 2013-09-02 04:22:10 -0700
comments: true
categories: [IRC, Linux, Software]
---

A bunch of my friends have been raving about [IRC Cloud](https://www.irccloud.com/), which gives users a persistent connection to an IRC server, allowing them to stay in chatrooms even when they close their clients. It’s a cool product, and while it has a native mobile app, the only way to access the service on a desktop computer is through a browser. As I like to use all my chat services in a single client inasmuch as it’s possible (/me shakes fist at Skype), I resigned myself to the idea of signing out whenever I close the lid of my laptop. Sure, it’s nice having continuous logs and not missing mentions in group chat, but switching from my Adium setup wasn’t enough to sway me.

But it turns out I can eat my cake and have it too! My friend [Phil](http://philkates.com/) told me about IRC bouncers, and specifically recommended [ZNC](http://wiki.znc.in/ZNC), so I looked into it. It works with Adium, so I can have the benefits of a persistent connection while continuing to consolidate my chat experience in my favorite client! Also, if I happen to be away from my computer, [ZNC Push](https://github.com/jreese/znc-push) notifies me whenever I’m mentioned or privately messaged. As a bonus, I was also able to configure a custom hostname so that my personal domain appears when someone does a /whois on my nick. I’ll be walking you through the steps I took to set all of this up.

 
Server Setup
------------

The first thing you’ll need is a server. If you don’t already have one, I recommend setting up an [Amazon EC2 micro-instance](http://aws.amazon.com/ec2/), which is free for a year. Mine is running Ubuntu 12.04 LTS. (An EC2 setup guide is outside the scope of this tutorial, but [here is Amazon’s documentation](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EC2_GetStarted.html).)

Before you do anything else, pick a port for ZNC to listen on (I chose 6667) and make sure it’s open. If you’re using Amazon EC2, go to the [management console](http://aws.amazon.com/console/), click on Security Groups, and make a new inbound TCP rule with port 6667, leaving the default setting for source. If you’re using some other VPS host, the following commands should work:

    $ iptables -I INPUT -p tcp --dport 6667 --syn -j ACCEPT
    $ netstat -vatn

The netstat output should show something like this:

    Proto Recv-Q Send-Q Local Address           Foreign Address         State      
    tcp        0      0 0.0.0.0:6667            0.0.0.0:*               LISTEN

You may also want to set your server’s timezone, so that the timestamps shown on your chat playback are accurate. Following the instructions in [this Stack Overflow post](http://stackoverflow.com/questions/11931566/how-to-set-the-time-zone-in-amazon-ec2), I used `tzselect` to find out that my timezone was “America/Los_Angeles”, got annoyed that it wasn’t “America/San_Francisco”, and then did the following:

    $ echo "America/Los_Angeles" | sudo tee /etc/timezone
    $ sudo dpkg-reconfigure --frontend noninteractive tzdata
 

Installing ZNC
--------------

Now it’s time to install ZNC! At the time of writing, [using IPV6 caused issues with ZNC Push notifications](https://github.com/jreese/znc-push/issues/48), so I chose to compile from source, using the following commands. [*Update 9/4/13: Apparently there's a branch that fixes this bug [here](https://github.com/jreese/znc-push/tree/ipv6); if you install it, don't pass the* `--disable-ipv6` *flag to configure, obviously.*]

    $ wget http://znc.in/releases/znc-1.0.tar.gz
    $ tar -xzvf znc-1.0.tar.gz
    $ cd znc-1.0
    $ ./configure --disable-ipv6
    $ make
    $ sudo make install

After installation completes, you’ll want to start ZNC for the first time with the `--makeconf` flag to set up the initial configuration:

    $ znc --makeconf

Here are the relevant settings that I used. You likely won’t need the partyline or webadmin global modules, but I enabled all the user modules it prompted me to load. If you’re enabling SSL, you may need to look up the appropriate port on your desired IRC server’s FAQ page. For [Freenode](http://freenode.net/irc_servers.shtml#ssl), it’s 6697.

    [ ?? ] What port would you like ZNC to listen on? (1025 to 65535): 6667
    [ ?? ] Would you like ZNC to listen using SSL? (yes/no) [no]: y
    [ ?? ] Would you like ZNC to listen using ipv6? (yes/no) [yes]: n
    [ ?? ] Listen Host (Blank for all ips):
    ...
    [ ?? ] Would you like this user to be an admin? (yes/no) [yes]: y
    ...
    [ ?? ] Would you like to keep buffers after replay? (yes/no) [no]: n
    ...
    [ ?? ] IRC server (host only): chat.freenode.net
    [ ?? ] [chat.freenode.net] Port (1 to 65535) [6667]: 6697
    [ ?? ] [chat.freenode.net] Password (probably empty): 
    [ ?? ] Does this server use SSL? (yes/no) [no]: y


Connecting to ZNC
-----------------

If everything has worked up until this point, ZNC should be running, and you should be able to use your chat client (e.g. Adium) to connect to the server hosting your IRC bouncer. In your chat client, make sure to enter the username and password you created during ZNC configuration under the nickname and password fields. The hostname field should contain the IP address of your personal server, not the IRC server you ultimately want to connect to. If you enabled SSL, don’t forget to check that option.

Once you’ve connected, try joining a channel. Stay in the channel, but quit your client and reopen it. You should see a convenient playback of the n (default 50) most recent lines, and you can ask a friend to help you confirm that your nick stayed connected even though your client disconnected from ZNC.


Loading ZNC Modules
-------------------

If you need to edit your configuration file, you COULD do the following from the terminal:

    $ pkill -SIGUSR1 znc
    $ pkill znc
    $ vim ~/.znc/configs/znc.conf
    $ znc

…BUT the safest way is to use your chat client to send either of these commands and follow the given instructions:

    /msg *status help
    /msg *controlpanel help

If you plan to configure a custom hostname, remember this – we’ll come back to it later. However, one variable that I recommend you set now is AutoClearChanBuffer, so that you don’t come back to 50 lines of chat playback that you’ve already seen if you’ve only exited your client for a second or two:

    /msg *controlpanel Set AutoClearChanBuffer $me true
 

ZNC Push
--------

I installed ZNC Push so that I could receive push notifications on my phone whenever I’m pinged and I don’t have a chat client actively connected to ZNC. [The documentation here](https://github.com/jreese/znc-push) is very helpful, but I’ll list the steps I went through to set this up:

    $ sudo apt-get install znc-dev
    $ git clone https://github.com/jreese/znc-push.git
    $ cd znc-push
    $ make
    $ cp push.so ~/.znc/modules/
    
I decided to use Boxcar to receive push notifications on my phone. After downloading the app and creating an account, I sent the following commands in my chat client:

    /msg *status LoadMod push
    /msg *push set service boxcar
    /msg *push set username myemail@domain.tld
    /msg *push subscribe

I also set the following options so that any mentions or private messages would be pushed to my phone as soon as my chat client has disconnected from ZNC. (The first command shows the current settings.)

    /msg *push get
    /msg *push set idle 0
    /msg *push set last_active 0
    /msg *push set last_notification 0
    /msg *push client_count_less_than 1
 

Custom Hostname
---------------

Finally, I wanted a vanity host to be displayed whenever I join a channel or someone performs a /whois lookup on me. I first purchased a domain (from Namecheap, you can use whichever domain registrar you prefer), and then I modified the domain’s host records,  [setting an A record](https://www.namecheap.com/support/knowledgebase/article/settingup_hostrecords) that points to the IP address of the server running ZNC.

After completing this step, I configured the reverse DNS of the ZNC server so that its IP address would resolve back to my custom domain. Disclaimer – I didn’t end up using my Amazon EC2 micro-instance for ZNC, but if I did, these are the likely steps I would take to set up reverse DNS:

1. In the [management console](management console), allocate an elastic IP address and associate it with your instance.
2. Fill out the [form mentioned here](http://aws.amazon.com/ec2/faqs/#Can_I_configure_the_reverse_DNS_record_for_my_Elastic_IP_address), specifying your custom domain as the reverse DNS record corresponding to your instance’s elastic IP.
If you’re using a VPS that utilizes a management system like SolusVM, there may be a field where you can provide reverse DNS records under the network settings. If this step doesn’t complete, and you only recently set your domain’s A record, you might have to wait a little while longer for the changes to go through.

Once this was done, I went back to my chat client and sent the following commands (replace “domain.tld” with your own domain):

    /msg *status AddBindHost domain.tld
    /msg *status SetBindHost domain.tld
    /msg *status SetUserBindHost domain.tld

That’s it! I had fun spending a few hours Googling SO MANY THINGS, and I hope that documenting all of it in this post helps some of you who are inspired to set up ZNC for yourselves.
