# rpi-services

The purpose of this repository is to provide a `docker-compose.yml` to easily create, run and delete common docker containers on a raspberry (for example influxdb, grafana, rpi-monitor, ...).

## Hardware

The hardware you need depends on your field of application. But I recommend a `Raspberry 3 B` (4 cores, 1 GB RAM) and at least a 16GB sd-card inserted (we are going to download a lot of docker images).

## Preparations

First you have to install docker on your raspberry. There is a real good documentation from official docker sources, so I won't get much into detail.
Follow the steps from [here](https://docs.docker.com/engine/installation/linux/docker-ce/debian/#upgrade-docker-ce). 
Please note that you need at least raspbian jessie or stretch.

After installation you can try your docker installation by running:

    sudo docker run --rm arm32v7/hello-world

The next step is to enable the docker daemon on boot:

    sudo systemctl enable docker
    sudo systemctl start docker

Enable the user pi to issue docker commands without sudo

    sudo usermod -aG docker pi
    # You should now be able to run (without sudo)
    docker run --rm arm32v7/hello-world

Unfortunately there are no pre-build `docker-compose` binaries for the ARM-architecture, so we have to install `docker-compose` via python-pip:

    sudo apt-get update
    sudo apt-get -yy python python-pip
    sudo pip install docker-compose 

Gratz! You are now running a docker featured multi-purpose raspberry server.

## RPI-Monitor 

I use [RPI-Monitor](http://rpi-experiences.blogspot.de/p/rpi-monitor.html) to monitor service and the raspberry overall health.
Unfortunately you have to install the `rpimonitor` on the host machine itself, cause when running in a docker environment it would collect stats about the guest system and not the host system itself.
To install `rpimonitor` you can follow the [installation guide](http://rpi-experiences.blogspot.de/p/rpi-monitor-installation.html).

I've included a `services.conf` inside the `template` directory. This will enable `rpi-monitor` to monitor the availability of your services like `influxdb` and `grafana`

    sudo cp ./templates/rpimonitor.services.conf /etc/rpimonitor/services.conf

Next you have to activate it inside `/etc/rpimonitor/data.conf`. Just search for services.conf, uncomment it and adjust the path. Then restart the service:

    sudo service rpimonitor restart

## Services 

To run services via `docker-compose` you only have to run

    # Run influxdb; -d stands for daemon and will cause influx to run in background
    docker-compose up -d influxdb

It is easy as that.

To view the log output of any running service

    # You can Ctrl+C out of it when finished
    docker-compose logs --tail=50 -f influxdb

### Portainer

`Portainer` is a lightweight management ui for managing your docker containers, volumes and images. Further documentation is available [here](https://portainer.io/).

    docker-compose up -d portainer

Next navigate to the url `http://<your-raspi-name-or-ip>:9000` to get into the `portainer` management ui. 

### InfluxDB

`Influx DB` is a database to store, monitor and analyze time series data. Further documentation is available [here](https://docs.influxdata.com/influxdb/v1.4/).

    docker-compose up -d influxdb
    # If you need the influx client and to test your running service
    docker-compose exec influxdb influx

If you need further customization you can modify the configuration file `./conf/influxdb.conf` before firing up the service.
To change default credentials of `root:root` and the pre-created databases (so far only `smarthome`) you have to modify the appropiate settings in `docker-compose.yml`.

### Grafana

`Grafana` is a data visualization and monitoring tool for (mostly) time series data. Further documentation is available [here](https://grafana.com/).

    docker-compose up -d grafana

You will notice that `influxdb` is firing up as well. That's because `influxdb` should be available as a data source to `grafana`. 
In my setup I basically write sensor data to `influxdb` to monitor and analyze it via `grafana`.
If you need further customization you can modify the configuration file `./conf/grafana.ini` before firing up the service.

`Grafana` is accessable by browser: `http://<your-raspi-name-or-ip>:3000`. Default credentails are `admin:admin`.

TODO: Howto create Influx Data Source

### MQTT

`MQTT` is the short-form of `Message Queue Telemtry Transport` and is used to transfer sensor data (telemetry) from a source to potential multiple sinks. 
The message queue (aka broker) is the `man in the middle` and receives messages that represent sensor data or complex objects and pusblishes them to a topic (known as publishing). 
Consumer can subscribe to topics and receive data when it is available (known as subscribing).

Example: A temperature sensor publishes its data to the message queue and a subscriber takes this data and forwards it to an influx database to visualize those sensor data.

I use `mosquitto` as my `MQTT` service. `mosquitto` is best-suited for the Internet of Things (IoT) and the requirements of a `raspberry pi`.

    # Have to change permissions for certain folders cause container runs code with another user
    sudo mkdir /var/mqtt && sudo chown 777 /var/mqtt
    sudo mkdir /var/log/mqtt && sudo chown 777 /var/log/mqtt
    docker-compose up -d mqtt

You can install the client tools to test your running container

    sudo apt-get mosquitto-clients
    mosquitto_sub -h localhost -v -t 'home/#'  # Subscribe for new messages on any home topic -> home/*
    # Open up another terminal window
    mosquitto_pub -h localhost -t 'home/sensor' -m "MQTT is running"  # Publish message to Topic: home/sensor
    # After publishing the message you should see 'MQTT is running' on the first terminal window 

### RPI-Monitor

`RPI-Monitor` keeps an eye on your raspberry itself and monitors the global health status (cpu, mem, temperature) as well as service availability.

This service is only integrated for educational purpose and it doesn't make sense to run it inside a container. It will monitor your container environment rather than your actual raspberry.
So I've decided to install the monitor the old fashioned way by running `apt-get install` ;-) (see instructions above)
