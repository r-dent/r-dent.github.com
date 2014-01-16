---
layout: post
category : pebble
tagline: "Quick writedown"
tags : [pebble, smartwatch, install, tutorial]
---
{% include JB/setup %}

I had to face some problems while installing Pebble SDK with [this guide](https://developer.getpebble.com/2/getting-started/macosx/). So i wrote down what was working in my case.
....
    
    echo 'export PATH=~/pebble-dev/PebbleSDK-2.0-BETA5/bin:$PATH' >> ~/.bash_profile 
    source .bash_profile 
    sudo easy_install pip
    pip install --no-use-wheel --upgrade setuptools // This was the line that helped.
    pip install --user -r ~/pebble-dev/PebbleSDK-2.0-BETA5/requirements.txt
    
After that i was ready to create and run my first project

    pebble new-project my-test-project
    pebble build --debug // The debug flag helps finding problems.
    pebble install --phone 192.168.*.*
    pebble screenshot --phone 192.168.*.*
    
Maybe this helps someone.
