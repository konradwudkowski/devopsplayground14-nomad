# DevOps Playground - Nomad

![Nomad](https://hyzxph.media.zestyio.com/blog-nomad-list.svg)

What is Nomad? 

Nomad is a tool for managing highly available and distributed clusters and applications. You can declare your jobs to be run on the clusters, deploy applications and monitor their progress.
It's simple and built for scale. Supports docker by default.

Since it's a Hashicorp tool, it fully integrates with other Hashicorp tools such as Vault and Consul without many configurations.

For this use case, we will use Consul as well, to show the integration with Nomad, to monitor and auto-discover the nodes and services.

Other tools / technologies used in this use case:

* **Docker** - Container technology that allow to package applications with all it's dependencies in order to be run on any server.

* **Amazon Web Services** - On-demand cloud computing capabilities, where you have access to virtual clusters of computers and services through the internet.


# Use Case:

In order for you to get in touch with nomad, we prepared a use case.

In this use case you will perform the following steps:

1. **Set up our environment**  

   [1.1. Consul Server](#1.1) 

   [1.2. Nomad Server](#1.2) 

   [1.3. Nomad Client](#1.3) 

2. **Web APP Job**  

   [2.1. Generate job file](#2.1) 

   [2.2. Update the job file](#2.2) 

   [2.3. Run the job file](#2.3) 

   [2.4. Check status and allocation](#2.4) 

3. **Cluster Job**
  
   [3.1. Add more instances](#3.1)

   [3.2. Update the job file](#3.2)

   [3.3. Run the job file](#3.3)

   [3.4. Check status and allocation](#3.4)
    
   [3.5. Check the services on Consul](#3.5)


## 1. Set up our environment<a name="1"></a>

For this hands-on, i will run the consul server, Nomad server and Nomad client on the same machine.

A key will be needed to access the instance.

You can find the key in the link below.

```sh
https://s3-eu-west-1.amazonaws.com/devops-playground-configurations/NomadKey.pem
```

After downloading the key, you will need to change the key file permissions to be readable by SSH.

```sh
chmod 400 NomadKey.key
```

With OSX/Linux:

```sh
ssh ec2-user@IP-ADDRESS -i KEY
```

With windows, use the [Putty](https://the.earth.li/~sgtatham/putty/latest/w64/putty-64bit-0.70-installer.msi) tool.

---

### 1.1. Consul Server<a name="1.1"></a>

**Why we're going to use Consul and what is Consul?**

Consul is service discovery tool made by Hashicorp where you can monitor and check the health of your servers.
For this use case, we're going to use Consul to monitor our nomad server and clients to be easy to understand what's happening besides just looking at the command-line.

First, we need to ssh into the Consul server and then execute the following command:

```sh
nohup sudo docker run -p 8500:8500 consul > consul_logs&
```

To avoid exiting the instance or canceling the nomad service, we can use the command ```nohup``` that redirects all the logs to a nohup file insted of showing in the terminal. We add as well the ```&``` for the nomad to run on the background.

If you want to check the logs you can just:

```sh
cat consul_logs
```

To check if it's actually running, go for your browser and put the public ip of your server with the port 8500.
You should get a page like this:

![consul](/images/base_consul.png)

---

### 1.2. Nomad Server<a name="1.2"></a>

Since our consul server is all configured, we can now configure our Nomad server.

The configuration file is already provided. Just check for server.hcl

For this we need to use a text editor. In this use case,we used nano.
You can find the server.hcl file here in this repo on the Scripts folder.

```sh
sudo nano server.hcl
```

All the internal IP'S are already configured by default.

On the field ```bootstrap_expect``` the current value is ```3``` which specifies that exists 3 Nomad servers and one of the servers will need to be elect as leader. For this use case, we change the value to ```1``` since we need our server to be automatically be elect as leader.

```sh
bootstrap_expect =  3
```

For the field ```CONSUL_INTERNAL_IP``` place the internal IP's of your Consul server. In this case will be ```127.0.0.1:8500```

```sh
consul =  {
    address =  "CONSUL_INTERNAL_IP:8500"
}
```

Execute on the Nomad Server to start the server:

```sh
nohup sudo nomad agent -config=server.hcl > nomad_logs&
```

---

### 1.3. Nomad Client<a name="1.3"></a>

Doing the same thing for the client,

create the client.hcl file:

```sh
sudo nano client.hcl
```


For the field ```NOMAD_SERVER_INTERNAL_IP``` place the internal IP's of your Nomad server. In this case will be ```127.0.0.1:4647```

```sh
    servers = ["NOMAD_SERVER_INTERNAL_IP:4647"]
```

For the field ```CONSUL_INTERNAL_IP``` place the internal IP's of your Consul server. In this case will be ```127.0.0.1:8500```

```sh
consul =  {
    address =  "CONSUL_INTERNAL_IP:8500"
}
```

Execute on the Nomad Client to start the client:

```sh
nohup sudo nomad agent -config client.hcl > nomadclient_logs&
```

Now, we can go to the browser and check on consul for our nomad server and clients:

![consul](/images/consul_servers.png)

You can see the status of the server and the clients, check if they're responding correctly to the consul server and later on we will come back here to check the services running.

---

## 2. Web APP Job<a name="2"></a>

### 2.1. Generate job file<a name="2.1"></a>

Let's go back to our Nomad Server and do:

```sh
sudo nomad init
```

This will generate the a job example file.

For the use case, you can find your webapp.nomad already on the instance, so just keep in mind if you don't have job file, you can generate one.

---

### 2.2. Update the job file<a name="2.2"></a>

Edit the webapp.nomad with a text editor.

```sh
sudo nano webapp.nomad
```

On the field ```datacenters = ["DATACENTER"]``` substitute the variable DATACENTER with the datacenter specified in the server.hcl. (if it wasn't changed, it is ```ecsd```)

```sh
datacenters = ["DATACENTER"]
```

On the field ```type = "SERVICE"``` substitute the variable SERVICE with the type of service job that you're going to execute. 
Nomad defines three types: 

* Service: scheduling long lived services that should never go down.
* Batch: sheduling sensitive to short term performance services and are short lived. 
* System: sheduling jobs that should be run on all clients that meet the job's constraints.

In this case we're going to use ```service```.

```sh
type = "TYPE_SERVICE"
```

On the field ```driver = "DRIVER"``` substitute the variable DRIVER with the one we're going to use ```docker```, on```image = "DOCKER_IMAGE"``` substitute the variable DOCKER_IMAGE with the image we need to use to have the web application -> ```seqvence/static-site``` and substitute the variable HTTP with the http port we going to use ```80```

```sh
driver = "DRIVER"
config {
   image = "DOCKER_IMAGE"
  
   port_map {
      webapp = HTTP_PORT
   }
}
```

---

### 2.3. Run the job file<a name="2.3"></a>

After the file is edited, just execute:

```sh
sudo nomad run webapp.nomad
```

After the job is done, just go to the nomad client connection and check for the docker containers running with:

```sh
sudo docker ps
```

You can see the instance ip and port. Just copy and paste it in your browser and should be able to see a webpage.
Since the docker container uses the private IP of the instance, put the public ip of your instance with the same port as stated on 
docker ps.

The result will be a page similar to this:

![Result](/images/static.png)

---

### 2.4. Check status and allocation<a name="2.4"></a>

Check the status of the job:

```sh
sudo nomad status webapp
```

Check the resource allocation:
```sh
sudo nomad alloc-status ALLOC_ID
```

---

## 3. Cluster Job<a name="3"></a>

So far, we have a server and a client as infrastruture but if we want to apply a job this to a bigger scale, we will have most likely, clusters of servers and we will need to apply our jobs to the clusters.

So let's add one more instance for nomad use the two clients as a cluster.

### 3.1. Add more instances<a name="3.1"></a>

So for this example, we will run the cluster job on the same machine.
On a real environment, you will have a cluster of instances and the job would spread across those instances.

To add more machines, just do the same procedure as to create a nomad client. The nomad server will detect the new client and add it to the cluster.

---

### 3.2. Update the job file<a name="3.2"></a>

On the Nomad server, you have the webapp.nomad job file like this:

```sh
job "webapp" {
  region = "global"

  datacenters = ["dc1"]

  type = "service"
  
  group "webs" {
    count = 3

    task "webapp" {

      driver = "docker"

      config {
        image = "seqvence/static-site"
      
        port_map {
            webapp = 80
        }
      }

      logs {
        max_files     = 10
        max_file_size = 15
      }

      resources {
        cpu    = 500
        memory = 256 
        network {
          mbits = 10
          port "webapp" {}
        }
      }

      service {
        name = "webapp"
        tags = ["global", "webs"]
        port = "webapp"
        check {
          name     = "alive"
          type     = "tcp"
          interval = "10s"
          timeout  = "2s"
        }
      }
    }
  }
}
```

Add the ```count = 3``` below the group for the job tasks be scaled to 3 instances.

Change the name on ```job "webapp"``` for ```job "cluster"``` since is going to be a new job.

Now we are adding two more services to the job, a database and a elasticsearch instance.

Just add a database to the job file:

```sh
group "db" {
    count = VALUE

    task "redis" {
      driver = "docker"

      config {
        image = "IMAGE"
        port_map {
          db = PORT
        }
      }

      resources {
        cpu    = 500
        memory = 256
        network {
          mbits = 10
          port "db" {}
        }
      }

      service {
        name = "redis-database"
        tags = ["global", "db"]
        port = "db"
        check {
          name     = "alive"
          type     = "tcp"
          interval = "10s"
          timeout  = "2s"
        }
      }
    }
  }
```

For the field ```VALUE``` place the value of ```3``` to create 3 services on the cluster.

For the field ```IMAGE``` place the value of ```redis:3.2``` to use a docker image of Redis database.

For the field ```PORT``` place the value of ```6379``` to use port 6379 as access to the database.


and add as well a elasticsearch service:

```sh
  group "elasticsearch" {
    count = VALUE

    task "webapp" {

      driver = "docker"

      config {
        image = "IMAGE"
      
        port_map {
            elasticsearch = PORT
        }
      }

      logs {
        max_files     = 10
        max_file_size = 15
      }

      resources {
        cpu    = 500
        memory = 256
        network {
          mbits = 10
          port "elasticsearch" {}
        }
      }

      service {
        name = "elasticsearch"
        tags = ["global", "elasticsearch"]
        port = "elasticsearch"
        check {
          name     = "alive"
          type     = "tcp"
          interval = "10s"
          timeout  = "2s"
        }
      }
    }
  }
```

For the field ```VALUE``` place the value of ```3``` to create 3 services on the cluster.

For the field ```IMAGE``` place the value of ```elasticsearch``` to use a docker image of elasticsearch search engine.

For the field ```PORT``` place the value of ```9300``` to use port 9300 for elasticsearch access.


After adding this, your file will look like this (in this case a new file was created and saved with the same cluster.nomad):

```sh
job "cluster" {
  region = "global"

  datacenters = ["dc1"]

  type = "service"

  group "db" {
    count = 3

    task "redis" {
      driver = "docker"

      config {
        image = "redis:3.2"
        port_map {
          db = 6379
        }
      }

      resources {
        cpu    = 500
        memory = 256
        network {
          mbits = 10
          port "db" {}
        }
      }

      service {
        name = "redis-database"
        tags = ["global", "db"]
        port = "db"
        check {
          name     = "alive"
          type     = "tcp"
          interval = "10s"
          timeout  = "2s"
        }
      }
    }
  }
  group "webs" {
    count = 3

    task "webapp" {

      driver = "docker"

      config {
        image = "seqvence/static-site"
      
        port_map {
            webapp = 80
        }
      }

      logs {
        max_files     = 10
        max_file_size = 15
      }

      resources {
        cpu    = 500
        memory = 256
        network {
          mbits = 10
          port "webapp" {}
        }
      }

      service {
        name = "webapp"
        tags = ["global", "webs"]
        port = "webapp"
        check {
          name     = "alive"
          type     = "tcp"
          interval = "10s"
          timeout  = "2s"
        }
      }
    }
  }
  group "es" {
    count = 3

    task "elasticsearch" {

      driver = "docker"

      config {
        image = "elasticsearch"
      
        port_map {
            elasticsearch = 9300
        }
      }

      logs {
        max_files     = 10
        max_file_size = 15
      }

      resources {
        cpu    = 500
        memory = 256
        network {
          mbits = 10
          port "elasticsearch" {}
        }
      }

      service {
        name = "elasticsearch"
        tags = ["global", "elasticsearch"]
        port = "elasticsearch"
        check {
          name     = "alive"
          type     = "tcp"
          interval = "10s"
          timeout  = "2s"
        }
      }
    }
  }
}
```

You can find this full version of the file in the Jobs folder here on Git.

---

### 3.3. Run the job file<a name="3.3"></a>

Run the cluster job:
```sh
sudo nomad run cluster.nomad
```

---

### 3.4. Check status and allocation<a name="3.4"></a>

Check the status of the job:
```sh
sudo nomad status cluster
```

Check the resource allocation:
```sh
sudo nomad alloc-status ALLOC_ID
```


### 3.5. Check the services on Consul<a name="3.5"></a>

Go back your browser and go for the consul address.
Now we can see our Nomad server, clients and all the services deployed on the clients.


## All done!

I hope that all of you have enjoyed this hands-on with Nomad.
Any feedback or questions, you can contact me on the email Daniel@ecs-digital.co.uk

Thank you!

