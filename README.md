# ProtoTwin Cloud Gateway for Linux

This directory contains the files necessary to build the Cloud Gateway for linux. It uses a docker container to build the application, which is then packaged into both an AppImage and a portable tarball. It is also possible to run the built gateway inside of the container. The docker compose file exposes port 443. If connecting through the Cloud Gateway to a server running on your local machine, you must use your machine's IP address (run `IPConfig`), rather than `127.0.0.1` or `localhost`.

The docker container runs Ubuntu 22.04, ensuring an old version of the standard libraries (e.g. glibc), so that users don't need a very up-to-date linux installation.

## Build Steps

1. Use the docker-compose.yml file to "Compose Up".
2. Attach a shell to the running container.
3. Run `sh package.sh`
4. The `ProtoTwinCloudGateway.tar.gz` and `ProtoTwinCloudGateway.AppImage` files are generated in the `/app` directory.
5. Download these files from the container and place in the `/static/installers` directory of the ProtoTwin website.

## Running on AWS LightSail

Create AWS LightSail instance running Ubuntu
Configure the instance's settings:

1. Switch to Networking tab and add a IPv4 Firewall rule: [Custom, TCP, 443]
2. Attach a public Static IP
3. Point (sub-)domain at IP by creating an DNS A record

Connect to the instance via SSH (e.g. using the terminal button under the Connect tab).

Update: `sudo apt-get update`
Switch to home directory: `cd ~`
Download the tarball: `wget https://prototwin.com/installers/ProtoTwinCloudGateway.tar.gz`
Extract the tarball: `tar -xzvf ProtoTwinCloudGateway.tar.gz`
Install certbot: `sudo apt-get install certbot -y`
Generate LetsEncrypt certificate and key (**replace domain and email**): `sudo certbot certonly --standalone -d "example.domain.com" --non-interactive --agree-tos --email "user@domain.com" --preferred-challenges http`
Note that if the above command fails to generate the certificate, you may need to wait for the DNS changes to propagate before trying again.

Create startup shell script to start the gateway with the desired arguments:

`nano ~/start.sh`

```
#!/bin/bash
exec /home/ubuntu/gateway -key="/etc/letsencrypt/live/gateway.prototwin.com/privkey.pem" -cert="/etc/letsencrypt/live/gateway.prototwin.com/fullchain.pem"
```

Note that you can provide additional command line arguments in the startup script.

Assign permission to execute: `sudo chmod +x ~/start.sh`

Run startup script: `sudo sh start.sh`
Ensure that the gateway starts correctly and that no errors are reported. If it fails then it is likely that something is already running on port 443 or your key and cert files are not at the locations specified in the startup script.

Leave the gateway running whilst we test the connection.

### Test the WebSocket Connection

Use [Websocketking](websocketking.com) or similar to test the connection.
The address should be: wss://your.domain.com
Ensure that you can connect to the server

You can now kill the gateway by pressing CTRL+C

### Creating the Service

We'll create a service that runs the startup script whenever the VPS reboots.

Create the service file: `sudo nano /etc/systemd/system/gateway.service`

```
[Unit]
Description=ProtoTwin Cloud Gateway
After=network.target

[Service]
ExecStart=sudo /bin/bash /home/ubuntu/start.sh
Restart=always
User=ubuntu

[Install]
WantedBy=multi-user.target
```

Enable and start the gateway service:

```
sudo systemctl daemon-reload
sudo systemctl enable gateway.service
sudo systemctl start gateway.service
```

Check that the service is running: `sudo systemctl status gateway.service`

We'll now create a certbot depoly hook, so that the gateway service is restarted after the certificate renews:

`sudo nano /etc/letsencrypt/renewal-hooks/deploy/restart-gateway.sh`

```
#!/bin/bash
pkill -f /home/ubuntu/gateway
pkill -f /home/ubuntu/start.sh
sudo nohup /bin/bash /home/ubuntu/start.sh &
```

Assign permission to execute: `sudo chmod +x /etc/letsencrypt/renewal-hooks/deploy/restart-gateway.sh`

It's important that the service is restarted after the certificate renews, otherwise the gateway will continue to use the old certificate which will expire after 90 days. If the gateway uses an expired certificate then you won't be able to connect to the gateway using most web browsers. This is because most web browsers block connections to servers using expired SSL certificates.

Restart the server: `sudo reboot`

Wait a few seconds and retest the connection.
If the service and startup script were all setup correctly, then the gateway should be started automatically after rebooting.

## Installing Hamachi

1. Download the LogMeIn Hamachi Debian package: `wget https://www.vpn.net/installers/logmein-hamachi_2.1.0.203-1_amd64.deb`
2. Install the package: `sudo dpkg -i logmein-hamachi_*.deb`
3. Login to Hamachi: `sudo hamachi login`
4. Attach the Hamachi client to your LogMeIn Hamachi user account (**replace email**): `sudo hamachi attach user@domain.com`
5. Approve the attachment request in [LogMeIn Central](https://secure.logmein.com/central/Central.aspx)
6. Ensure that the attachment was successful (the lmi account should not say "pending"): `sudo hamachi`
7. Request to join the Hamachi network (**replace Network ID**): `sudo hamachi do-join 123-456-789`
8. Approve the join request in [LogMeIn Central](https://secure.logmein.com/central/Central.aspx)

## Command Line Options

ProtoTwin Cloud Gateway supports a number of command line arguments. These arguments can be specified in the startup script. The default settings are very conservative and designed to reduce the load on connected devices.

### Port

Specifies the port for the websocket server. Defaults to 443.

```
./gateway -port=443
```

### Broadcast Interval

Specifies the interval (in milliseconds) between broadcasting outputs to connected clients. Note that this is not the same as the scan rate. The scan rate defines the interval at which the cloud gateway reads outputs from the device. The broadcast interval defines the interval at which the cloud gateway broadcasts the most recently read outputs to connected ProtoTwin clients. Defaults to 100ms.

```
./gateway -broadcastInterval=100
```

### Minimum Scan Interval

Specifies the minimum interval (in milliseconds) between reads from connected devices. Note that ProtoTwin clients can request a particular scan rate. This setting is used to clamp the minimum scan rate so as to avoid high load on the device. Defaults to 25ms.

```
./gateway -minScanInterval=25
```

### Write Inputs

Specifies whether to allow inputs to be written to connected devices. Caution! For security reasons, it is highly recommended to protect access to the gateway when enabling this setting. Defaults to disabled.

Input writing can be enabled by providing the `-writeInputs` argument:

```
./gateway -writeInputs
```

### Key and Certificate

Specifies the path to the key and certificate files, used for WebSockets over TLS.

```
./gateway -key="/etc/letsencrypt/live/example.domain.com/privkey.pem" -cert="/etc/letsencrypt/live/example.domain.com/fullchain.pem"
```
