# juniper-input-jti
Unofficial fork of the [native JTI input plugin for Juniper Open-NTI.](https://github.com/Juniper/open-nti/tree/master/plugins/input-jti)

__Key changes:__
- Port 50000 is exposed by default.
    - This is the default destination port for Junos telemetry streaming.
    - Allows the container to be deployed standalone without requiring a Dockerfile edit and rebuild (of the original Juniper implementation).
- Kafka is enabled by default.
    - Default destination is localhost:9092.
    - Default topic is "jnpr.jti".
    - This can be disabled/changed via specifying ENV variables at runtime.

## Supported Outputs
The same outputs as the main Juniper input plugin are supported:
- Kafka (default)
- InfluxDB
- Stdout

These can be toggled on/off when running the container via ENV variables.

## Environment variables
Here is the list of variables available with their default value.

```yaml
OUTPUT_KAFKA: true
OUTPUT_INFLUXDB: false
OUTPUT_STDOUT: false

## Ports Numbers for Juniper Telemetry Input Plugins
PORT_JTI: 50000

## Information for Influxdb Output plugin
INFLUXDB_ADDR: localhost
INFLUXDB_PORT: 8086
INFLUXDB_DB: juniper
INFLUXDB_USER: telemetry
INFLUXDB_PWD: telemetry1
INFLUXDB_FLUSH_INTERVAL: 2

## Information for Kafka
KAFKA_ADDR: localhost
KAFKA_PORT: 9092
KAFKA_DATA_TYPE: json
KAFKA_COMPRESSION_CODEC: none
KAFKA_TOPIC: jnpr.jti
```

## Running the container
You can use the `-e` parameter of `docker run` to modify the above environment variables at runtime, and `-p` to alter the destination port to listen on for telemetry data.

##### Changing ENV variables
For example, the below disables Kafka output and enables InfluxDB output to a database located at 1.1.1.1 (other variables are kept as the default).
```
docker run -d -e OUTPUT_INFLUXDB:true -e OUTPUT_KAFKA=false -e INFLUXDB_ADDR='1.1.1.1' -p 50000:50000/udp -i nathancatania/juniper-input-jti
```

To alter Kafka settings to connect to a broker located at '1.1.1.1:9094' and push data to a topic called "telemetry":
```
docker run -d -e KAFKA_ADDR='1.1.1.1' -e KAFKA_PORT:9094 -e KAFKA_TOPIC='telemetry' -p 50000:50000/udp -i nathancatania/juniper-input-jti
```

##### Changing the destination port
The container will listen on port 50000 by default for incoming telemetry data.

For example, to change this to port 44444 instead:
```
docker run -d -p 44444:50000/udp -i nathancatania/juniper-input-jti
```
This maps port 44444 of the host to the exposed port 50000 (UDP) of the container.


## Supported Input Sensors
[This container functions as a collector for the __native__ UDP Junos Telemetry Interface (JTI) sensors.][jtinative] It will not function as a (gpb) gRPC/OpenConfig Telemetry collector.

The following sensors are currently supported:

| Sensor                                 | Compatible Device/OS                                                                                      | Resource String                                 |
| -------------------------------------- |:---------------------------------------------------------------------------------------------------------:|:-----------------------------------------------:|
| Physical Interface Telemetry           | MX - 15.1F5<br>MX150 - 17.3R1<br>PTX - 15.1F3<br>QFX10k - 17.2R1<br>PTK1k - 17.2R1<br>EX9200 - 17.3R1     | /junos/system/linecard/interface/               |
| Logical Interface Telemetry            | MX - 15.1F5<br>QFX10k - 17.2R1<br>EX9200 - 17.3R1                                                         | /junos/system/linecard/interface/logical/usage/ |
| Firewall Filter Telemetry              | MX - 15.1F5<br>QFX10k - 17.2R1<br>PTK1k - 17.3R1<br>EX9200 - 17.3R1                                       | /junos/system/linecard/firewall/                |
| PFE CPU Memory Utilization Telemetry   | MX - 16.1R3<br>QFX10k - 17.2R1<br>PTK1k - 17.2R1<br>EX9200 - 17.3R1                                       | /junos/system/linecard/cpu/memory/              |


## Example: Configuring Telemetry
Assume the collector is running on an IP address of 1.1.1.1, listening on port 50000, and you wish to turn on Physical Interface telemetry.

All native telemetry configuration occurs under the `edit services analytics` context.

##### Define a destination
The destination server for the JTI telemetry is defined as a `streaming-server`.

You must give the destination server a name. This is just used for identification within Junos.
```
set services analytics streaming-server <SERVER-NAME> remote-address 1.1.1.1 remote-port 50000
```

##### Define an source
You must define an `export-profile` before you can stream telemetry from Junos.

- You must give the profile a name. This is just used for identification within Junos.
- You must also specify a:
    - Reporting rate (the frequency - in seconds - to push telemetry data out to the remote server)
    - Transport (which for native sensors, will be UDP)
- You can also specify a source IP address and port which will be the source address for exported telemetry. If you define this, set this to the management interface IP or you may encounter issues with telemetry not working.
```
set services analytics export-profile <PROFILE-NAME> reporting-rate 30 transport udp
```

##### Define a sensor profile
The telemetry sensor you wish to turn on will be tagged/set with the profiles you created above.

You must give the sensor profile a unique name. This is only used for identification within Junos.

The reason for defining streaming-server and export-profiles separately, is it allows you to have the same sensor sent to different destinations or exported under different parameters.

```
set services analytics <SENSOR-NAME> server-name <SERVER-NAME> export-profile <PROFILE-NAME> resource /junos/system/linecard/interface/
```

##### Commit and check
```
commit check
commit and-quit
```

You can verify the configured profiles and sensor:
```
show agent sensors
```

[jtinative]:https://www.juniper.net/documentation/en_US/junos/topics/reference/configuration-statement/sensor-edit-services-analytics.html
