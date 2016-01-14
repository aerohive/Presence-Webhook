This document describes how to write an application that will receive streaming presence API calls over a secure SSL session. Avoid using plain HTTP endpoints because unencrypted data (client MAC addresses and positions) should not be transmitted over the internet.
 
**Some prerequisites:**
 
1. Obtain a valid publicly signed SSL certificate for the web server. This is important. HMMG will not post to untrusted SSL endpoints.
2. Make sure the certificate contains the whole CA certificate chain. This usually means concatenating all the public certificates into a single bundle. A good guide is available here: Certificate Installation: NGINX - Powered by Kayako Help Desk Software
3. Make sure your web server is reachable on the internet (public IP, DNS...)
4. Configure HMNG API Data Management settings to point to the URL of your web server
 
The message flow of the app is as follows:
 
HMNG ------ HTTPS -------> nginx  ----Thin----> Ruby App #1
                                                 *|
                                                 * ----Thin----> Ruby App #2
                                                 *|
                                                 * ----Thin----> Ruby App #3
 
The API Data Management configuration in HiveManager NG points to a https endpoint on the web server.

**1. Install Ruby**
 
Ruby is a programming language that is very handy and practical to use. Combined with the Rails framework it is well suited for writing web applications.
To install Ruby, follow your OS specific instructions here: Installing Ruby
 
**2. Install gems**
 
Gems are Ruby packages that contain libraries and extensions. We need three libraries:
Sinatra: Sinatra is domain specific language for HTTP. It simplifies dealing with HTTP requests.
Thin: a very lightweight web server library.
Openssl: Openssl library provides SSL, TLS and cryptography features.
 
To install, execute:
$ gem install sinatra  
$ gem install thin  
$ gem install openssl  
 
**3. Install nginx proxy**
 
The three gems above are usually enough to receive API calls over SSL. However, it is a good practice to avoid exposing the applications directly to the internet. Therefore, the web hook callbacks should be implemented behind a proxy/loadbalancer. This provides additional security, handles SSL handshakes and provides load balancing capabilities which is especially handy when we are dealing with several sources of presence data (i.e. several VHMs streaming location data).
 
There are different ways of installing, depending on the operating system. In Ubuntu Linux, nginx can be installed by running
 
$ sudo apt-get install nginx  
 
**4. Configure nginx**
 
Edit the nginx configuration file. Add SSL support and configure the Ruby app backend, where the HTTP requests get proxied to. The required configuration is found below.
 
sudo vi /etc/nginx/sites-available/default  
 
```
# Ruby app backend
upstream backend {
  server 127.0.0.1:4000;
}
 
# HTTPS server
 
server {
        listen 3000;
        server_name ec2-52-33-75-85.us-west-2.compute.internal;
 
        root html;
        index index.html index.htm;
 
        ssl on;
        ssl_certificate /home/ubuntu/code/ssl-bundle.crt;
        ssl_certificate_key /home/ubuntu/code/ahtraining_in.key;
        ssl_verify_client optional;
        ssl_session_timeout 5m;
 
        ssl_protocols SSLv2 TLSv1 TLSv1.1 TLSv1.2;
        ssl_ciphers "HIGH:!aNULL:!MD5 or HIGH:!aNULL:!MD5:!3DES";
        ssl_prefer_server_ciphers on;
        location @backend {
                proxy_pass http://backend;
        }
        location / {
                try_files $uri @backend;
        }
}
``` 
 
To test nginx configuration, run
nginx -t  
 
To restart nginx with the new config
sudo service nginx restart  
 
**5. Write the Ruby application**
 
The source code for the Ruby application that will process the streaming API HTTP POST requests and write to a file is given below. Make sure the port settings between the application and the nginx upstream configuration are correct. Nginx should proxy the request to the Ruby application.

```Ruby 
require 'sinatra'  
require 'thin'  
  
  
#initialize thin web server  
class MyThinBackend < ::Thin::Backends::TcpServer  
  def initialize(host, port, options)  
    super(host, port)  
#set SSL - set to true if there is no nginx proxy  
    @ssl = false   
    @ssl_options = options  
  end  
end  
  
  
#initialize thin web server  
configure do  
  set :environment, :production  
  #bind to 127.0.0.1 when using with nginx, otherwise bind with 0.0.0.0  
  set :bind, '127.0.0.1'  
  set :port, 4000  
  set :server, "thin"  
  class << settings  
    def server_settings  
      {  
        :backend          => MyThinBackend,  
        #SSL certificate settings. Aerohive streaming presence API requires  
        #requires certificates signed by a publicly trusted CA  
        :private_key_file => File.dirname(__FILE__) + "/ahtraining_in.key",  
        #cert chain needs to include the complete cert chain   
        :cert_chain_file  => File.dirname(__FILE__) + "/ssl-bundle.crt",  
        #verify HiveManager NG certificate  
        :verify_peer      => true  
      }  
    end  
  end  
end  
  
  
#Handle HTTP GET request. Used for SSL testing.  
get '/' do  
  "Hello. You are using SSL!"  
end  
  
  
#Handle HTTP POST requests  
post '/' do  
  status 204 #successful request with no body content  
  request.body.rewind  
  request_payload = JSON.parse(request.body.read)  
#append the payload to a file  
File.open("events.txt", "a") do |f|  
    f.puts(request.body.read)  
end  
end  
```
**6. Finally, run the app**
 
Run the ruby app and enjoy the show. You should start seeing requests from the VHM coming in over an encrypted SSL session.
 
== Sinatra (v1.4.6) has taken the stage on 4000 for production with backup from Thin
Thin web server (v1.6.4 codename Gob Bluth)
Maximum connections set to 1024
Listening on 127.0.0.1:4000, CTRL+C to stop
127.0.0.1 - - [30/Dec/2015:12:38:56 +0000] "POST / HTTP/1.0" 204 - 0.0017
127.0.0.1 - - [30/Dec/2015:12:38:56 +0000] "POST / HTTP/1.0" 204 - 0.0025
