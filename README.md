# PXaaS integration with Middleware API
## Local test-bed
Here are the steps followed to integrate the PXaaS VNF with the mAPI. Rundeck and Middleware API were deployed on Ubuntu 14.04 
### Deploy Rundeck
In order to deploy Rundeck follow:
http://stash.i2cat.net/projects/TNOV/repos/wp5/browse/WP5/mAPI. Proceed with the following steps:
#### Install Rundeck
#### Install MySQL
#### Install Python dependencies
#### Get mAPI code
Clone the wp5 repository from stash
#### Configuration
```
vim wp5/WP5/mAPI/Config/mAPI.cfg
```
```
[authentication]
#authentication_method can be basic or gatekeeper
authentication_method = basic
#basic authentication configuration
username = admin
password = changeme
#gatekeeper authentication configuration
gatekeeper_host = 0.0.0.0
gatekeeper_port = 1234
service_key = secret
[server]
ip = 0.0.0.0
port = 1234
[general]
# the path where the mAPI is located
folder = /home/george/Desktop/wp5/WP5/mAPI/
[rundeck]
host = localhost
port = 4440
# get the token from Rundeck
token = OdPBTMKkI8mjk63UZXwssVg4ryhS43AB
project_folder = /var/rundeck/projects/
TNOVA_user = george
[db]
user = root
password = <password>
ip = localhost
```
### Create a pair of public/private key
This step is needed in order for the middleware API to send remote commands to the VNF
```
ssh-keygen -t rsa
add the public key under ./ssh/authorized_keys on the VNF
add the private key on the VNFD
```
### Create a folder on the VNF
In this folder the middleware API will copy the configuration files
```
# on the VNF
mkdir /home/vagrant/container
```
### Create VNFD.json
```
{
  "id":"PXaaSDimokritos",
  "vnfd": {
    "lifecycle_event":{
      "Driver":"SSH",
      "Authentication":"-----BEGIN RSA PRIVATE KEY-----\n
MIIEogIBAAKCAQEAylira/1FMscFzGTLlv7XgjP/NNvbYlLecwrAotbvqizPmtPS\n
...
dd/rn1fs8YZ6+EPGATRwn1aJPhD7Hc5o0FhE8ONanhGXAUavGdU=\n
-----END RSA PRIVATE KEY-----",
      "Authentication_Type":"private key",
      "Authentication_Username":"vagrant",
      "VNF_Container":"/home/vagrant/container/",
      "events":[
        {
          "Event":"start",
          "Command":"/home/vagrant/scripts/start",
          "Template File Format":"json",
          "Template File":"{ \"controller\":\"get_attr[vdu1,PublicIp]\", \"vdu1\":\"get_attr[vdu1,PublicIp]\"}"
        },
        {
          "Event":"stop",
          "Command":"/home/vagrant/scripts/stop"
        }
      ]
    }
  }
}
```
### Running Middleware API
```
cd wp5/WP5/mAPI/
sudo python northboundinterface.py
```
### Load VNFD and create project in Rundeck
From a new console run:
```
curl -X POST 0.0.0.0:1234/vnf_api/ -u admin:changeme -d @<path to VNFD>/VNFD.json -v
```
### Run commands
#### Create start and stop files
Normally those files will be created automatically (I am not sure from which component) based on the template files described in the VNFD. However, in order to test the mAPI we create them manually.

start.json
```
{
  "event":"start",
  "vnf_controller":["10.10.1.90"],
  "parameters":{
    "vdu1_PublicIp":["10.10.1.90"]
  }
}
```
stop.json
```
{
  "event":"stop",
  "vnf_controller":["10.10.1.90"]
}
```

#### Run the start command
```
curl -X POST 0.0.0.0:1234/vnf_api/PXaaS/config/ -u admin:changeme -d @start.json
```
#### Run the stop command
```
curl -X PUT 0.0.0.0:1234/vnf_api/PXaaS/config/ -u admin:changeme -d @stop.json
```