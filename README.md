# solr-kerberos-docker
Start Docker
------------------------
This works on Docker 1.12. Not sure if more recent versions work as is.

    $ sudo service docker start

Using a KDC docker image
------------------------
Download KDC docker image and keytab files:

    wget http://185.14.187.116/keytabs

    wget http://185.14.187.116/kdc-docker-image.tar.xz


Loading the image and creating a container:

    $ docker load < kdc-docker-image.tar.xz
    $ docker run -it --name kdc -h kdc kdc

Inside the container created:

    # service krb5-kdc start
    # service krb5-admin-server start
    # service ssh start

To check the ip address of this container, do this inside the container:

    # ifconfig

Assume the ip address is 172.17.0.2. Assume this terminal window is called window1.

Note:

    * Passwords are luc1d for principals: ishan, client
    * The keytabs file can be used for principals: HTTP/solr1, HTTP/solr2, zookeeper/zk1
    * Root password (for ssh) is: luc1d

Using a Solr development docker image along with the KDC image
--------------------------------------------------------------

Download docker image (do this in a different terminal window, lets call it window2):

    $ wget http://185.14.187.116/solr-dev-docker-image.tar.xz

Load the image and create a container:

    docker load < solr-dev-docker-image.tar.xz
    docker run -it --name solr1 -h solr1 solr-dev

Connect the containers with a common network. Do the following in a different terminal window (lets call it window3):

    docker network create solr-kerberos
    docker network connect solr-kerberos kdc
    docker network connect solr-kerberos solr1

Back inside the solr1 container (terminal window2), compile and run Solr:

    # cd lucene-solr
    # git pull
    # cd solr
    # ant server
    # bin/solr -c -force

Verify that kerberos is working:

    # curl http://solr1:8983/solr/

(should throw 403 error)

    # kinit client@EXAMPLE.COM
    (password: luc1d)
    # curl --negotiate -u : http://solr1:8983/solr/
    
(should show the HTML of the admin UI)

Setting up browser on host machine to access Solr Admin UI
----------------------------------------------------------
Install kerberos client:

    OS X: Pre-installed? (verify that /etc/krb5.conf file exists)
    Fedora: sudo yum install krb5-workstation, 
    Ubuntu: sudo apt-get install krb5-user)

Now add the KDC details:

    $ sudo vi /etc/krb5.conf

Add the following section (check the KDC's IP address from the first step, assuming 172.17.0.2):

    [realms]
     EXAMPLE.COM = {
      kdc = 172.17.0.2
      admin_server = 172.17.0.2
     }

Also, add the following line in the [libdefaults] section (create this section if not already there):

    default_realm = EXAMPLE.COM

Now, try to do kinit:

    $ kinit client@EXAMPLE.COM
    password: luc1d

Now, we have to add a hostname entry for solr1. First, find out the IP address of the solr1 container (inside window2):

    # ip addr
    
(assume the IP address is 172.17.0.3)

Now, in the host machine, open a new terminal window (say, window4) and add the host name entry:

    $ sudo vi /etc/hosts
    Add the following line:
    solr1  172.17.0.3
    
Open Firefox, and goto "about:config". Configure "network.negotiate-auth.delegation-uris" and "network.negotiate-auth.trusted-uris" as follows:

![Firefox Kerberos configuration](http://185.14.187.116/firefox-kerberos.png)


Now, access http://solr1:8983/solr and the Solr Admin UI should show up.


SOLR-15233 Reproduction
----------------------------------

JIRA link: https://issues.apache.org/jira/browse/SOLR-15233

This could be presented with a 2-node solrcloud but with this the distributed requests amongst the replicas can be checked as well so I added another solr service to have 3 nodes.

#### Steps:
1) Start the docker services
    ```
    docker-compose-up
    ```
2) Create a collection with client@EXAMPLE.COM. Collection name shall differ from 'test' and have exactly 2 replicas on 2 different nodes:<br />
    ```shell script
    kinit client@EXAMPLE.COM
    curl -i --negotiate -u : 'http://solr1:8983/solr/admin/collections?action=CREATE&name=test2&numShards=2'
    ```
3) Check which node doesn't have replica of the collection. (in the following sample solr3 is "empty")<br /> 
    ```json 
      {"responseHeader":{
          "status":0,
          "QTime":8727},
        "success":{
          "solr1:8983_solr":{
            "responseHeader":{
              "status":0,
              "QTime":6994},
            "core":"test2_shard1_replica_n1"},
          "solr2:8983_solr":{
            "responseHeader":{
              "status":0,
              "QTime":7108},
            "core":"test2_shard2_replica_n2"}}
    ```
4) Switch to client2@EXAMPLE.COM, pass: restrict (<- this user doesn't have any rights on newly created collection)
    ```shell script
    kinit client2@EXAMPLE.COM
    ```
5) Send a request to a node which has replica shall result 403
    ```shell script
    curl -i --negotiate -u : 'http://solr1:8983/solr/test2/select?q=*:*'
    ```
6) Send a request to the node which doesn't have a replica will result 200 and search response
    ```shell script
    curl -i --negotiate -u : 'http://solr3:8983/solr/test2/select?q=*:*'
    ```