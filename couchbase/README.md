# Couchbase Docker container

This is a collection of images and scripts to help you run Couchbase 2.2.0 community edition (build-837) in Docker containers.
Docker containers are great to provision ephemeral topologies for testing and development purpose.
This project was partly inspired by [dustin/couchbase](https://gist.github.com/dustin/6605182).

## Prerequisites

You need to override memlock and nofile limits to make Couchbase run correctly. 
Add the following lines at the end of the `/etc/init/docker.conf` file:

	limit memlock unlimited unlimited
	limit nofile 262144

Finally, restart the Docker daemon: `/etc/init.d/docker restart`.

## Single Couchbase node

1. Create the Couchbase container

		docker run -d -name cb ncolomer/couchbase

	This command starts a Couchbase node as a new cluster and setup a `default` bucket of 128m (can be changed later in the admin interface).
	You can also add the options generated by the `$(./scripts/ports raz)` script to bind Couchbase client ports (8091, 8092, 11210, 11211) to host.

		docker run -d $(./scripts/ports raz) -name cb ncolomer/couchbase

2. Check the Couchbase instance is up&running

	Using Telnet (Memcached text protocol)

		$ telnet $(./scripts/ipof cb) 11211
		# Check health
		stats
		# Set and get a key
		set test_key 0 0 1
		get test_key
		# Disconnect
		quit

	Using `couchbase-cli`

		docker run -rm ncolomer/couchbase couchbase-cli server-info -c $(./scripts/ipof cb) -u Administrator -p couchbase

	Using HTTP

		curl -u Administrator:couchbase http://$(./scripts/ipof cb):8091/nodes/ns_1@127.0.0.1

	Using admin interface: open the output of `echo http://$(./scripts/ipof cb):8091` in your favorite browser. 
	The admin login is `Administrator` and the password is `couchbase`.

## 3-node cluster

Create the containers as following:

	docker run -d -name cb1 ncolomer/couchbase
	docker run -d -name cb2 ncolomer/couchbase couchbase-start $(./scripts/ipof cb1)
	docker run -d -name cb3 ncolomer/couchbase couchbase-start $(./scripts/ipof cb1)

The nodes `cb2` and `cb3` will automatically join the cluster via `cb1`. The cluster needs a rebalance to be fully operationnal. To do so, run the following command:

	docker run -rm ncolomer/couchbase couchbase-cli rebalance -c $(./scripts/ipof cb1) -u Administrator -p couchbase

Note that you can also trigger a rebalance and show progress from the admin interface.

## Two 2-node cluster (XDCR base)

Create the containers as following:

	docker run -d $(./scripts/ports raz) -name cb11 ncolomer/couchbase
	docker run -d -name cb12 ncolomer/couchbase couchbase-start $(./scripts/ipof cb11)
	docker run -d $(./scripts/ports) -name cb21 ncolomer/couchbase
	docker run -d -name cb22 ncolomer/couchbase couchbase-start $(./scripts/ipof cb21)

The node `cb12` automatically joins the cluster #1 via `cb11` and the node `cb22` automatically joins the cluster #2 via `cb11`. The clusters #1 and #2 both need a rebalance to be fully operationnal. To do so, run the following commands:

	docker run -rm ncolomer/couchbase couchbase-cli rebalance -c $(./scripts/ipof cb11) -u Administrator -p couchbase
	docker run -rm ncolomer/couchbase couchbase-cli rebalance -c $(./scripts/ipof cb21) -u Administrator -p couchbase

* Open the URL `http://$CB11_URL:8091` (replace `$CB11_URL` by the output of `./scripts/ipof cb11`) in your favorite browser
* Setup the XDCR as described in the [Set Source and Destination Clusters](http://docs.couchbase.com/couchbase-manual-2.2/#set-source-and-destination-clusters) documentation page. You will have to provide at lease one IP of a node in the cluster #2, the IP returned by `./scripts/ipof cb21` is a good choice.
* Note you can also do all this stuff using command line. See the documentation for the details.

## Some routing magic

If you host Docker on a Virtual Machine (a Vagrant managed VM for instance), you can make docker containers reachable from your VM's host by routing request to the docker bridge:

	sudo route add -net $DOCKER_BRIDGE $VM_IP

Example:

	sudo route add -net 172.17.0.0 192.168.10.10

