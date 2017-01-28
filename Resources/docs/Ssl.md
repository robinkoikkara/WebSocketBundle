#SSL Configuration

For wss:// connections we recommend using stunnel. It is used to open a secured port and then forward it to a not secured port on the same other different machine. You can also use Nginx or HaProxy

##Using stunnel

Install [Stunnel](https://www.stunnel.org/index.html) : 

```cmd
 sudo apt-get install stunnel
 ```

Create a config file in /etc/stunnel/. Preferably named stunnel.conf : 

```cmd
nano /etc/stunnel/stunnel.conf
```

```ini
# Certificate
cert = /my/way/to/ssl.crt
key = /my/way/to/not_crypted.key

# Remove TCP delay for local and remote.
socket = l:TCP_NODELAY=1
socket = r:TCP_NODELAY=1

chroot = /var/run/stunnel4/
pid = /stunnel.pid

# Only use this options if for making it more secure after you get it to work.
# User id
#setuid = nobody
# Group id
#setgid = nobody

# IMPORTANT: If the websocketserver is on the same server as the webserver use this:
#local = my.domainname.com # Insert here your domain that is secured with https.

[websockets]
accept = 8443
connect = 8888
# IMPORTANT: If you use the local variable above, you have to add the domainname here aswell.
# connect = my.domainname.com:8888 
# ALSO *: When starting your websocket server, you have to use the -a parameter to specify the domainname
```


(*) Starting the websocketserver when on same server :

```cmd
php app/console gos:websocket:server -a my.domainname.com -e=prod -n
```


Save the file and start stunnel : 

 ```cmd
/etc/init.d/stunnel4 start
```


For running stunnel automated edit properties in /etc/default/stunnel4:

```ini
ENABLED=1
```

## Using Nginx

Create a folder named `stream.d` at the root of your nginx install (for Debian: `/etc/nginx`) 
Create the file `websocket.conf` inside the folder you've just created, and copy / adjust the following content.
```
stream {
    upstream websocket_backend {
        server YOUR-LOCAL-IP:RUNNING-PORT;
    }
    server {
        listen ACCEPT-PORT ssl;
        proxy_pass websocket_backend;
        proxy_timeout 4h; # adjust it to your needs

        ssl_certificate /my/way/to/ssl.crt;
        ssl_certificate_key /my/way/to/not_crypted.key;

        ssl_handshake_timeout 5s; # adjust it to your needs
        proxy_buffer_size 16k;
        ssl_session_timeout 4h; # adjust it to your needs
    }
}
```

Then, edit `nginx.conf` located in `/etc/nginx`

After the `http` directive, place the following statement `include /etc/nginx/stream.d/*.conf;`
It has to be at the same hierarchy level of `http`.
This configuration was tested on nginx 1.10 but it should works on nginx 1.9 as well.

Execute `/etc/init.d/nginx reload` and run your websocket server using the following command line `bin/console gos:websocket:server --port RUNNING-PORT -a YOUR-LOCAL-IP`


## Finally

Launch your websocket server in the *connectport*.

Set-up Gos Web Socket Bundle in AWS (Production) with SSL

Introduction

Gos Web Socket is a Symfony2 Bundle designed to bring together WS functionality in a easy to use application architecture.It provides both server and client side code ensuring you have to write as little as possible to get your app up and running. Powered By Ratchet, AutobahnJS, with Symfony2. This can be used to implement WS clients using AngularJS.

However, getting this to work in production that too in a AWS EC2 instance behind a load balancer and autoscaling group with SSl is tricky.

Examples

Step By Step Configuration
1. Deploy the Code in AWS EC2 Instance

In my project AngularJS websocket client talks to the socket server implemented with AutobahnJS and gos-websocket.js. This is packaged in an separate application for deployment in Apache. This application talks to my symfony application (seperately deployed) which acts as the web socket server and it uses Ratchet. Basic documentation is available here - https://github.com/GeniusesOfSymfony/WebSocketBundle

I was able to execute the web socket on this EC2 instance.

2. Configure AWS for Load balancer (out of scope of this documentation)

Kindly go through the AWS documentation http://docs.aws.amazon.com/autoscaling/latest/userguide/attach-load-balancer-asg.html

Make sure the AWS firewall configurations are altered to allow websocket port to receive connections.

I was able to execute the web socket with this configuration too.

3. Configure the server for SSL (out of scope of this documentation) Kindly go through the AWS documentation http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/SSL-on-an-instance.html

However, after configuring SSL my web socket stopped working. Reason: The web socket server implementation does not support secured ports. So the TCP request on secured connection are not handled. In next step we will configure stunnel to overcome this challenge.

4. Configuring stunnel (I am using AWS Linux AMI)

sudo yum install stunnel

This will install the stunnel application in the server.

repoquery -l stunnel

This command will list all the files installed with stunnel application. Now create the configuration file with below command

sudo vim /etc/stunnel/stunnel.conf

Copy paste the below configuartion

 debug = debug
cert = /etc/ssl/certs/domainname.com.cert
key = /etc/ssl/certs/domainname.com.cert.key
#Remove TCP delay for local and remote.
socket = l:TCP_NODELAY=1
socket = r:TCP_NODELAY=1
chroot = /var/run/stunnel/ #the changed root directory in which the stunnel process runs, for greater security
pid = /stunnel.pid
output = stunnel.log
# IMPORTANT: If the websocketserver is on the same server as the webserver use this:
local = 10.0.0.22 # Insert here your Private IP of EC2 instance that is secured with https.

[websockets]
accept = 9090 # This is the port from which my client will connect ans stunnel will intercept
#connect = 8888
# IMPORTANT: If you use the local variable above, you have to add the domainname here aswell.
connect = 10.0.0.22:8443 # This is ip (EC2 private IP) and port on which ratchet is listening and stunnel will forward the request
5. AWS firewall setting changes

For both load balancer and the EC2 instance change the inbond setting. Since my stunnel is listening on 9090, Ihave opened that port for communication For both load balancer and the EC2 instance change the inbond setting. Since my stunnel is listening on 9090, Ihave opened that port for communication

6. Verify the wss/ip/port setting in ws client code (Angular code)
a. configuration in app-run.js

 >var config_port = [{host: "domainname.com", port: "9090", secured: true}];
b. configuration in bower_components/gos-websocket/src/gos-websocket.js

this.websocket = WS.connect('wss' + '://' + 'domainname.com' + ':' + '9090');

7. Verify the setting in ws server side (PHP)

app/config/config_prod.yml

  gos_web_socket:
     server:
         port: 8443        #The port to which ratchet WS is listening 
         host: ec2-51-11-111-11.ap-south-1.compute.amazonaws.com   #The EC2 instance bind to
         router:
             resources:
                 - '@AppBundle/Resources/config/pubsub/routing.yml'
8. Lets start the stunnel server

sudo stunnel /etc/stunnel/stunnel.conf

Watch for any errors in log

sudo vim /var/run/stunnel/stunnel.log

9. lets start the gos web socket server (ratchet)

php app/console gos:websocket:server -e=prod -n

Clear the cache of Symfony application before the websocket server is started. Also you can kill the stunnel server if any change is needed in stunnel configuration.

sudo kill cat /var/run/stunnel/stunnel.pid

Now try to hit the secured wss:// URL and it should work.

Connect the client on wss://my.domainname.com:*acceptport*

