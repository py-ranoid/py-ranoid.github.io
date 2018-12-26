---
title: " DIY Smart Home : Google Home + Raspberry Pi"
layout: post
date: 2018-12-26 17:36
tag:
- python
- iot
- raspberry pi
image: /assets/images/smart-light.png
headerImage: true
projects: false
hidden: false
description: "Voice-activated Switchboard"
category: blog
author: johndoe
externalLink: false
---
<iframe width="100%" height = "350" src="https://www.youtube.com/embed/jAB0Iscjj68" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

Imagine. You're snug and warm in your bed, and are about to happily doze off when you realize- you've forgotten to turn your lights off. Don't worry though- it's possible for you to do so without climbing out of those soft sheets. Here's how you can set up your room to be infinitely cooler.

# What you'll need
1. [Google Home](https://store.google.com/product/google_home
) (Mini/Regular/Max) ([Buy in India](https://www.flipkart.com/automation-robotics/google~brand/pr?sid=igc))
2. [Raspberry Pi](https://www.raspberrypi.org/products/raspberry-pi-3-model-b-plus/) (Any Model) ([Buy in India](https://www.amazon.in/Raspberry-Model-1-2GHz-Starter-Kit/dp/B017TZFZ3U))
3. Female-to-Female [Jumper Wires](https://en.wikipedia.org/wiki/Jump_wire) ([Buy in India](https://www.amazon.in/REES52-Jumper-Wire-Male-Female/dp/B072FLQBQ4))
4. [5v Relay Module](https://www.youtube.com/watch?v=51f3ZazNW-w) ([Buy in India](https://www.amazon.in/REES52-Optocoupler-Channel-Control-Arduino/dp/B01HXM1G9Q))
5. (Wireless) LAN with Internet Access


*Optional Items*
- Raspberry Pi Case <br> Note : Make sure your case does not cover the GPIO ports.
- RPi accessories (only needed the first time)
  - HDMI Cable (to display RPi screen)
  - Micro USB Charger (to power RPi)
  - USB Keyboard and Mouse (to control RPi)
- If you're tweaking an existing switchboard :
  - Wire of [appropriate thickness](https://www.thespruce.com/matching-wire-size-to-circuit-amperage-1152865), to connect switch ends to relay
  - Screwdriver, to open your switchboard and secure wires in the relay)
  - Insulation tape
  - Safety Gloves


# How it works
![](/assets/images/SmartSwimlane.png)

---


## Step 1 : Controlling switches with Python
The first thing that you need to do is set up the wiring so that you can programmatically flip the switch using our Raspberry Pi. While it's easy to trigger GPIO ports, they can only emit 5V. In order to close/break a 220V circuit, you need a relay. A relay is an [electromagnetic switch](https://www.explainthatstuff.com/howrelayswork.html) operated by a relatively small electric current that can turn on or off a much larger electric current. This is why you'll hear a clicking sound when the relay is triggered.
### 1.0 Setting up your RPi
- Follow [this](https://projects.raspberrypi.org/en/projects/raspberry-pi-setting-up) guide to get your Pi running and then [enable SSH](https://www.raspberrypi.org/documentation/remote-access/ssh/).
- Alternatively, if you don't have a keyboard, mouse and a display, check out this [headless setup](https://howtoraspberrypi.com/how-to-raspberry-pi-headless-setup/).
- Make sure you enable SSH or VNC to access your RPi using another device on the same LAN.


### 1.1 Triggering GPIO ports with Python
Run the following in a Python Shell on your Pi.

```python
import RPi.GPIO as GPIO

pin = 33

# Setup
GPIO.setmode(GPIO.BCM)
GPIO.setwarnings(False)
GPIO.setup(pin,GPIO.OUT)

# To turn on pin
GPIO.output(pin,GPIO.HIGH)

# To turn off pin
GPIO.output(pin,GPIO.LOW)
```
Refer [this](https://thepihut.com/blogs/raspberry-pi-tutorials/27968772-turning-on-an-led-with-your-raspberry-pis-gpio-pins) detailed guide about **Turning on an LED with your Raspberry Pi's GPIO Pins**

### 1.2 Triggering appliances with Python
If you'd like to control a switch in an existing wall socket, grab a screwdriver and a pair of safety gloves. Open the socket and disconnect the switch's live wire and connect it to `NC` port of the 1st Relay. Take another wire and connect the `NO` port of the 1st Relay to the switch's live input (that you just opened). This lets you break the circuit with the switch when the relay circuit is closed. <br>Recreate the following circuit using your Pi, Relay and Wires.

![/assets/images/circuit.png](/assets/images/circuit.png)

## Step 2 : Controlling switches with HTTP requests
- You need to run a server on your Pi so that we can interact with it remotely. (I'll be using Flask since it's fairly simple.)

```python
from flask import Flask, request, jsonify

app = Flask(__name__)

@app.route('/prime', methods=['POST'])
def switch():
    query = request.json.get('obj',"NA")
    # Process query and turn appliance on/off with RPi.GPIO


if __name__ == '__main__':
    app.run(host= '0.0.0.0',debug=True,port=8000)
```

- You could also develop a **Mobile app** to send requests to this server with an intuitive UI on your phone or tablet.
- However the Flask server can only be accessed locally. Though you can control your switch locally, you can't do it outside your LAN.
- The easiest way to expose your local servers to the public internet is using secure tunnels.
- Some options and their free tier features are :
  - [ngrok](https://ngrok.com/download) : Provides a new subdomain every time client is run. (It is infeasible to keep updating IFTTT applet's webhook time and again.)
  - [pagekite](https://pagekite.net/) : Provides custom subdomain for 30 days on verifying email  ID.


### Exposing your Pi to the Internet
1. Run Flask server in one terminal (`python server.py`)
2. Initiate port forwarding in the another terminal<br>
  ```python pagekite.py 80 homeboy.pagekite.me```<br>
  or<br>
  ```./ngrok http 80```

Any request sent to your ngrok or pagekite URL should hopefully get tunnelled/forwarded to your flask server.

## Step 3 : Controlling switches with the Google Home/Assistant
Now that you have a server to control your smart home remotely, you need to call the endpoint using your Google Home. To do this, you will be using [IFTTT](https://ifttt.com/) ie, *If This Then That*, which is a free web-based service to create chains of simple conditional statements, called applets. An applet has 2 components : a trigger (which initiates the action) and the action itself. To bridge the Google Home with your Raspberry Pi, you need to create an IFTTT applet. Download the IFTTT app and connect it to the Google account associated wth your Google Home.
### Setting the Trigger.
1. Click on New Applet and then "**this**"
2. Choose *Google Assistant* as the trigger service.
3. Select "Say a phrase with a text ingredient". This text field will tell your server which appliance to trigger and what to do (On/Off)
4. Enter the sample unterance with **$** as a placeholder for the query.
5. Click on **Create Trigger** when you're done.

![/assets/images/trigger.png](/assets/images/trigger.png)

### Setting the Action
1. Click on "**that**" to set the action
2. Choose *Webhooks* as the action service. This lets you make a HTTP request to an endpoint when the trigger is activated.
3. Paste your ngrok/pagekite URL in the URL field
4. Set method as POST and content type as `application/json`
5. Enter the sample JSON with **TextField** as a placeholder for the query replace **$** in the Google Assistant trigger.

![/assets/images/action.png](/assets/images/action.png)
---
### Troubleshooting
There are quite a few ways above steps could go wrong. Here's a checklist to help you debug.
- Step 1
  - **GPIO ports work** : Test with an LED before moving on to a Relay.
  - **Port numbers are correct** : It's possible that you're triggering the wrong GPIO port.
  - **Relay inputs are correct** : It's possible that you're triggering the wrong switch.
  - **Loose connections** : 🤷‍♂
- Step 2
  - **Flask server is running and on the right port** : Send requests locally with cURL, Postman or HTTPie. Use [`nohup`](https://stackoverflow.com/questions/23029443/run-python-flask-on-ec2-in-the-backgroud) to run the server in the background. Add logs to monitor parameters of incoming requests.
  - **Tunnel is running and forwarding requests to the right port** : Send requests to the right endpoint on the right tunnel URL (`*.ngrok.com/method_path` or `*.pagekite.com/method_path`). It is also possible that your subdomain may expire or your tunnel may stop responding after some time.
  - **Both processes are running simaltaneously**
- Step 3
  - **IFTTT applet is triggered.** : Click on the ⚙️ icon to the top-right of your applet to configure it and enable "Receive notifications when the Applet is run". It's helps to add a response message to your Google Assistant trigger service.
  - **Request is received at server** : Monitor tunnel and Flask logs to make sure request is received.


---

🚀 Server Code  : [github.com/py-ranoid/PyCasa](https://github.com/py-ranoid/PyCasa)
<br> ☹️ Can't get it to work ? [Drop a mail](mailto:vishalg8897@gmail.com)
<br> 🙋‍♂ Follow me on [Github](http://github.com/py-ranoid) or Connect on [LinkedIn](https://linkedin.com/in/vishalg8897)
<br> ➕ Check out my other [posts](http://vishalgupta.me/blog)

---

<div>Header Icon made by <a href="https://www.freepik.com/" title="Freepik">Freepik</a> from <a href="https://www.flaticon.com/" 			    title="Flaticon">www.flaticon.com</a> is licensed by <a href="http://creativecommons.org/licenses/by/3.0/" 			    title="Creative Commons BY 3.0" target="_blank">CC 3.0 BY</a></div>