# docker-puppetserver
## Puppet Labs puppetserver 6.0.2
 
This is a general purpose puppetserver.   For more information on using puppet, see https://docs.puppet.com/puppet/6.0/ 

When running, it's a good idea to set the hostname inside the running container (_-h parameter to "docker run"_) to the DNS name your puppetserver will be listening for requests on, as that will be the default certname value used when creating the puppet CA cert.  Alternatively, you can edit the certname in the config later.

There are three volumes that you will want to map outside of the running container:

| Path | Purpose |
| ---- | ---- |
| /etc/puppetlabs | Configuration |
| /opt/puppetlabs/server/data/puppetserver | Runtime data, certs, reports, etc |
| /var/log/puppetlabs/puppetserver | Logs _(may not care about persisting this, but don't want to bloat the container)_ |

I prefer to map these to local volumes on my host, but they can be mapped to a data container as well.  

#### Local Volume Method
The first time you run the container, you'll want to initialize the volumes with their original content, which is not done automatically.   You can do so with the packaged alternative command "initialize", which will provision the local directories and exit:
```
# First run to initialize / provision config files on local volumes
docker run -it --rm \
        -v /home/docker/puppetserver/data:/opt/puppetlabs/server/data/puppetserver \
        -v /home/docker/puppetserver/etc:/etc/puppetlabs \
        -v /home/docker/puppetserver/logs:/var/log/puppetlabs/puppetserver \
        guyton/puppetserver initialize
```
*OPTIONAL* - From here, you can edit the config file (/home/docker/puppetserver/etc/puppet/puppet.conf) to set "certname" to the dns name for your container if you don't want to use the -h docker run option below, and possibly set autosign=true.
```
certname = my-puppetserver-hostname.example.net
autosign = true
```
Then, for standard operation:
```
# All subsequent runs after running the initial provisioning and configuration or similar
docker run -d --name puppetserver -h my-puppetserver-hostname.example.net -p 8140:8140 \
        -v /home/docker/puppetserver/data:/opt/puppetlabs/server/data/puppetserver \
        -v /home/docker/puppetserver/etc:/etc/puppetlabs \
        -v /home/docker/puppetserver/logs:/var/log/puppetlabs/puppetserver \
        guyton/puppetserver 
```

### Data Container Volume Method
First create the non-running data volume.  There's no need to initialize as we did above, since the data volume uses the same image and hence has the files already in place.  Simply run with bash, which will exit and leave you with a non-running data volume:
```
docker run -d --name puppetserver_data guyton/puppetserver bash
```
Then run the puppetserver with the "--volumes-from" switch and the -h switch to set the hostname the puppetserver will generate its certs for:
```
docker run -d --name puppetserver -h my-puppetserver-hostname.example.net -p 8140:8140 \
        --volumes-from puppetserver_data \
        guyton/puppetserver 
```
### Low Memory 
Puppetserver by default wants a little more than 2 GB of memory.  If you are operating in tight memory constraints you can set the env variable JAVA_ARGS to something different than the default "-Xmx2g -Xms2g", such as:
```
docker run -d --name puppetserver -h my-puppetserver-hostname.example.net -p 8140:8140 \
        -e JAVA_ARGS='-Xms500m -Xmx1g' \
        -v /home/docker/puppetserver/data:/opt/puppetlabs/server/data/puppetserver \
        -v /home/docker/puppetserver/etc:/etc/puppetlabs \
        -v /home/docker/puppetserver/logs:/var/log/puppetlabs/puppetserver \
        guyton/puppetserver 
```


