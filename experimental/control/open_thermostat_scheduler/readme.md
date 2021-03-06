# Open heating controller/thermostat scheduler software development v0.1

Energy use for space heating in buildings accounts for 19.5% of total primary energy or 28.6% 
of end use energy in the UK. The ZeroCarbonBritain report by the Center for Alternative Technology suggests that a space heating energy saving of 50-60% is achievable across all buildings. 40% from better insulation,a further 10% from better air-tightness and another 10% from better heating controls. See [OpenEnergyMonitor Sustainable Energy](http://openenergymonitor.org/emon/sustainable-energy)

The aim of the open thermostat scheduler is to explore how we can improve on heating controls to achieve space heating energy savings. If existing heating controls either over-heat rooms/zones or heat rooms/zones when they not in use energy savings are made by matching the heating of any room or zone as closely as possible to the use of that room/zone. The aim is to lower the average internal temperature as much as possible while still achieving comfortable temperatures for the period the rooms are occupied. There is a general rule of thumb that for every 1C the thermostat is turned down you can make a 10% heating energy saving, the average internal temperature can be reduced by both turning the occupied period temperature down (thermostat setpoint) and by reducing the length of the heating period. Savings are likely to be larger in buildings that are poorly insulated and have less thermal mass. 

In buildings which are under heated prior to installing heating controls with heating being turned on by the user manually for short periods, a scheduler may not result in energy savings and there is a possibility that the added convenience of being able to turn on heating remotely may result in larger energy use as the building may be heated for longer periods.

This software is work in progress and while the inital release has been tested to be basically functional there are still lots of parts to it that are missing: authentication, service scripts, etc see todo list below.

![topgraphic.png](docs/topgraphic.png)

# Install

This software is work in progress and while the inital release has been tested to be basically functional there are still lots of parts to it that are missing: authentication, service scripts, etc see todo list below.

This software is designed for running on a raspberrypi with an rfmpi adapter board connected. It can be run on any linux computer with a jeelink or equivalent serial to rfm12/69 interface board.

Start by following low write SD card emoncms installation guide here: [https://github.com/emoncms/emoncms/tree/bufferedwrite](https://github.com/emoncms/emoncms/tree/bufferedwrite) this will install most of the requirements needed it can be installed from scratch or using the ready to go image.

Download the **open_thermostat_scheduler** folder to the user directory (/home/pi on the raspberry pi)

Copy the files heating.html, jquery-1.9.0.min.js and folder open\_thermostat_scheduler/web/api to /var/www

Upload the RFM12Pi\_hardcoded_simple firmware to your RFMPi adapter board set your radio settings. You can do compile and upload the code using a tool called inotool. see [readme in inotool folder](https://github.com/emoncms/development/tree/master/experimental/control/open_thermostat_scheduler/inotool)

Run rfmpi2mqtt.py:

    python rfmpi2mqtt.py

Run runschedule.py:

    python runschedule.py

Open the heating controller interface:

    http://localhost/heating.html

Can be used with emoncms mqttdev branch:
https://github.com/emoncms/emoncms/tree/mqttdev

Run the phpmqtt\_input.php script in the scripts folder to subscribe to the mqtt node data. Set the userid in the phpmqtt\_input.php to your userid before running the script.

    cd /var/www/emoncms/scripts
    sudo phpmqtt_input.php

# Todo

- Service script for rfmpi2mqtt.py
- Service script for runschedule.py
- Add logging to rfmpi2mqtt.py
- Add logging to runschedule.py
- Authentication on integration with emoncms loging authentication for HTTP Api and heating.html page
- Watchdog for service scripts
- Extend heating.html to allow multi-zone control
- Develop android app version of heating.html
- Investigate replacing rfmpi2mqtt.py with emonhub once emonhub can carry out same functionality
- Investigate using nodejs with websockets as developed here [https://github.com/emoncms/development/tree/master/experimental/control/nodejs_websocket_thermostat](https://github.com/emoncms/development/tree/master/experimental/control/nodejs_websocket_thermostat) to allow pushing of updates to the client

## Development Notes:

![diagram.png](docs/diagram.png)

### rfmpi2mqtt.py

Bridge between serial IO of rfmpi and MQTT based on @pb66's emonhub decoder.

**RX:** Node data received is decoded according to config file and posted to rx MQTT topic & redis db

1) Received on serial:

	18,58,7,228,12
	
2) Decoded using config file:
	
	[nodes]

    [[18]]
        nodename = room
        names = temperature, battery
        codes = h, h
        units = C,V
        scale = 0.01,0.001
        
3) Output published to MQTT topic:
	
	*rxtopic/nodename/varname*
	rx/room/temperature		18.5
	rx/room/battery			3.30
	
**TX:** Data can be sent out on rfm network by publishing messages to the tx mqtt topic:

1) Pulish to tx mqtt topic:

	*txtopic/nodename*  csv variables
	tx/heating			1,1810
	
2) Encoded using config file:

    [[30]]
        nodename = heating
        names = state, setpoint
        codes = b, h
        
3) Serial write command to rfmpi:

    30,1,18,7,s


### heating.html 

The scheduler interface provides a UI that generates a schedule object detailing the heating schedule for every day of the week. The heating schedule can be overridden with a manual setpoint and heating state in manual mode. The variables required for this application are:
    
	js object:			value				redis/mqtt topic key
	heating.state			1/0				app/heating/state
	heating.manualsetpoint		<temperature>			app/heating/manualsetpoint
	heating.mode			manual/schedule			app/heating/mode
	heating.schedule		<schedule json>			app/heating/schedule

These variables are stored in redis and state changes are published to MQTT under topic names listed above

### runschedule.py 

The scheduler interface needs to be used in conjunction with an always running script that runs the schedule when the web page interface is not loaded by the user. The scheduler UI needs to pass the above configuration variables to the runschedule.py script.

### api/index.php

provides a HTTP interface too:

- rfmpi rx: node data received from wireless nodes, ie: room/temperature
- rfmpi tx: used to send data out to the nodes (heating state variables/comands)
- application state data (user schedule)

**Testing the API**

The API used both the GET and POST HTTP methods, run these commands in terminal to test:

GET:

    GET http://localhost/api/rx/room/temperature
    GET http://localhost/api/app/heating/mode
    GET http://localhost/api/app/heating/state
    GET http://localhost/api/app/heating/schedule
    
POST:
    
    curl --data 16.5 http://localhost/api/rx/room/temperature
    curl --data 1 http://localhost/api/app/heating/mode
    
To send a command to the rfmpi adapter board, ie to set heating on and set point to 21C:

    curl --data 1,2100 http://localhost/api/tx/heating
    

###  Using MQTT + Redis for responsive control

One of the main design ideas used here is that any property which might be an: integer, float, json, csv is stored in a server side key:value database (i.e redis in this case) and is also passed to MQTT. The HTTP api url mirror's the database and MQTT key for that variable. When a property is updated it is **both** saved to the redis database and published to a MQTT topic of the same key name. Publishing to MQTT rather than having other scripts poll redis on a ususally slower basis makes it possible for the control application to be very responseive to user input, turning on a light via a relay as soon as a web html button in the browser is pressed.

# Development questions

## Persistance

At the moment all state is recorded in the redis database including:

- program state (schedule object, manual heating settings)
- node rx state (node data, room temperature etc)
- node tx state (last sent command state to rfmpi)

The redis database can be configured to persist to disk every x key update or to just run in ram. It would be beneficial to be able to select which keys must get persisted and which keys do not need to be persisted. Perhaps another storage engine is needed in addition to redis? another key value store and a setting in the program config to say which key:value's are stored in the disk based db and which in the ram db. The user heating schedule is an example of data that would be critical to persist. While last value node data does not need to be persisted as persisting would result in regular disk writes and on system reboot the nodes will updated new values within a short timeframe.

### Command routing

In the implementation above the scheduler interface can send manual heating commands directly to rfmpi2mqtt.py via api.php in addition to sending heating schedule settings changes to runschedule.py. In the event that runschedule.py crashes/fails this provides a failure mode that still allows manual control of the heating. The alternative is that all commands go via runschedule.py which then in turn sends commands to rfmpi2mqtt.py, is there an event where commands could be simultaneously sent by runsheduler.py and directly causing the program to get into a 'mixed' state?

### Server API

While developing the server side component that provides a HTTP interface for heating.html to access data in MQTT and Redis I tried a couple of different ideas. My ideal API I think would provide several options so that in addition to being able to fetch a single variable by its key:

    GET http://localhost/api/rx/room/temperature

It would also be possible to request the entire node or even all the nodes by going along the hierarchy. 

    GET http://localhost/api/rx/room -> {"temperature":18.5, "battery":3.3}
    GET http://localhost/api/rx      -> {"room":{"temperature":18.5, "battery":3.3}}

The code to do this gets complex fast resulting in much harder to read and understand code, so in the end I decided to revert to the simple full key approach. see attempt03\_extended_api.php for my attempt:

- [archive/attempt01_verymanual.php](https://github.com/emoncms/development/blob/master/experimental/control/open_thermostat_scheduler/archive/attempt01_verymanual.php)
- [archive/attempt03_extended_api.php](https://github.com/emoncms/development/blob/master/experimental/control/open_thermostat_scheduler/archive/attempt03_extended_api.php)

It would be great however to achieve this fuller API if a more elgant solution could be found. (attempt02 was my chosen solution in the end see: [api/index.php](https://github.com/emoncms/development/blob/master/experimental/control/open_thermostat_scheduler/web/api/index.php))


### Remote emoncms dispatchers

In the implementation above node data is published to a MQTT topic key for each node/variable. This works ok for transfering data locally between scripts but if we wanted to dispatch node data to a remote emoncms account translating a mqtt variable update to a http request to emoncms.org for every single input would not be the most efficient way of sending data and would multiply the connection load on emoncms.org considerably. At the moment emonhub has a dispatcher that buffers node data which can in turn contain many node variables. It might be nice to hang dispatchers off the MQTT rx/node/variable bus but to then dispatch an efficient message would require recombination of the individual variables into nodes and would have associated challenges with defining which variable to trigger dispatching with. Perhaps another MQTT topic is required which holds this node data without it having being split into sperate variables.

The current format looks like this:

    data=[
        [0,18,18.5,3.3],
        [5,18,18.5,3.3],
        [10,18,18.5,3.3]
    ]
    &sentat=12
    
where [0,18,18.5,3.3] = [time,node,var1,var2]

perhaps a less space efficient more verbose format could be sent, which would allow naming to be sent to the remote emoncms.org account:

    bulkdata = [
    	{"time":0, "node":"room", "var":{"temperature":18.5, "battery":3.3}}
        {"time":5, "node":"room", "var":{"temperature":18.5, "battery":3.3}}
        {"time":10, "node":"room", "var":{"temperature":18.5, "battery":3.3}}
    ]

or perhaps a dispatcher could create bulk uploads of full keys:

    bulkdata = {
        "room/temperature":{"time":0,"val":18.5},
        "room/battery":{"time":0,"val":3.3},
        "room/temperature":{"time":5,"val":18.5},
        "room/battery":{"time":5,"val":3.3},
        "room/temperature":{"time":10,"val":18.5},
        "room/battery":{"time":10,"val":3.3}
    }

Im not sure it is as nice a solution as the current approach.
