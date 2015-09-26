---
layout: post
title:  "Creating a Testing PostgreSQL Server"
date:   2015-09-26 12:35:35
categories: rails postgresql
---


## AWS RDS is really cool but... $$$ 

We are huge fans of AWS at [Kopo Kopo][4]. We therefore use RDS to run our databases that ship with our Rails application(s). RDS offers us a couple of advantages listed below.
However, when it comes to testing and staging environments, is RDS overkill ? I have come to think so.

{{ more }}

RDS offers numerous advactages that include but not limited to:

 - Automated backups and point in time recovery ability
 - Multi-AZ availability (redundancy)
 - Seamless on-demand scaling
 - Managed upgrades
 - Read-only replica management for increased read workloads

We isolate each client's setup in its own Virtual Private Cloud (VPC) on AWS. When creating staging environements for testing, our first pass involved trying to exactly re-create what the production environment looked like so as to test on a setup that was as close to production as possible. This meant having staging environments that were also running RDS databases. The only feature we subtracted from the production setup was the Multi-AZ availability since - well if an AWS AZ went down it would be non catastrophic since it was just a staging environment and we could get the whole setup back up and running in a matter of minutes or hours.

The problem with the above setup is cost. As you get many clients and get multiple staging environments, AWS resources cost go up since the "hands off" database management that RDS affords actually comes at a cost. Well, why should staging environments be accorded RDS resources especially after we learned how our apps behaved? Not to mention the fact, that for a Rails application, there is nothing really magical about where the db is. A rails application merely points to a db in database.yml and this could be a local or remote db. Why not just create a 'Database Server' to serve PostgreSQL databases for all the staging and testing environments ?


----------

## Creating a PostgreSQL database server ##

Start with a base Ubuntu image and create an EC2 instance. You could bring up your instance in EC2 Classic or like in our case, in a public subnet of a VPC of your chosing. Use a new AWS Security Group and allow inbound SSH traffic for now. We will modify this later to firewall the Database Server.

Install PostgreSQL by running the following commands

    sudo apt-get update
    sudo apt-get install -y postgresql postgresql-contrib

You then need to create a user that you will be using to access the database(s).

    sudo -u postgres createuser -P [Your user name]
    
The command above requires you to enter a password for the user in question. You can then proceed to create a single or multiple databases for applications that this database server will serve. You of course need to consider the resources this particular instance has. Staging and testing servers most likely (like in our case) will not be carrying production loads but the number of databases you create needs to put in consideration the resources available on the machine. To create the databases;

    sudo -u postgres createdb -O [Your user name] [Database 1 name] 
    sudo -u postgres createdb -O [Your user name] [Database 2 name]

Upon successful creation of the databases, you can test connection to said databases with the following command;

    psql -h localhost -U [Your user name] [Database 1 name]
    
    
After being prompted for the passord for [Your user name] you will then be at the postgres prompt.


If you have your database server on EC2, you will undoubtedly not want to expose the databse connection to the Internet for obvious security reasons. You could therefore avail connection to your database server via an SSH tunnel. To set up an SSH tunnel to your database server instance run the command;

    ssh -L 127.0.0.1:5433:127.0.0.1:5432 -i [path to pem file] [ssh user name]@[database server IP address] -N
    
We are not fans of passwords, so we use ssh keys to connect to the database server. The [path to pem file] is the path to the AWS key pair that you would use to connect to the EC2 instance (or the path to a private key whose public key is authorized to SSH into the database server). After running the above command, your local host's prot 5433 is mapped to the remote servers port 5432 (the port postgres server is running on). To test the tunnel, on another terminal on your local machine, connect to the db using;

    psql -h localhost -U [Your user name] -p 5433 [Database 1 name]
    
    
The connection to the remote database should succeed using SSH tunneling.


----------

## Setting up application deployment ##

So you now have a database server but use AWS Opsworks to deploy and manage your cloud infrastracture (did I mention we are AWS fans?). How do you set up your application deployment so that you can securely connect to the database server from your rails application instances? You can use two approaches

### 1. Using a combination of 'firewalling' and postgres access configuration

Locate the pg_hba.conf file for your postgres installation for example mine is located at:

    /etc/postgresql/9.3/main/pg_hba.conf

We would like to be able to connect to postgres from any location. Somewhere in the connection section, you need to add the following line

    host    all     all     0.0.0.0/0       md5
    
This tells postgres to accept connections from for all users, for all databases from any location. Yes this looks alarming, but don't worry as we will see below, we will firewall the instance to restrict connections. If you are still paranoid, you could still go ahead and add the specific IP's that your web application will be connecting from like so:

    host    all     all     X.X.X.X/32       md5
    
The problem of taking the above approach, is you can only use it if the IP's of your web applications will be fairly static. If your case is like ours where your web application IP's are somehow dynamic (due to frequently bringing up and down web application servers) then restricting IP's at the postgres level will be cumbersome.

We next need to enable remote connections to the database which is not enabled by default. This is done in the file postgresql.conf located in for example;

    /etc/postgresql/9.3/main/postgresql.conf
    
Locate the line (around line 59)

    #listen_addresses = 'localhost'          # what IP address(es) to listen on;

uncomment and change it to;

    listen_addresses = '*'          # what IP address(es) to listen on;
    
save and be sure to restart postgres:

    $ sudo service postgresql start
    
Next we need to firewall the Database Server. Locate the Security Group you created for the Database Server instance. Let us assume that all webservers are located in a public subnet of a VPC. Whenever we bring up a web application server, we assign it a special AWS Security Group called webserver-sg and in this case we will assume that the group id assigned by AWS is sg-442b9033. Our goal is to ensure that only machines located in our public subnet and using the SG webserver-sg are able to connect to the Database Server. To achieve this, modify the Database Server Security Group to look like this;

![Databse Server Security Group][1]


  
You can now test the connectivity to your database from your web application machine. SSH to the web application server and run:

    psql -h [DB Server IP address] -U [Your user name] [Database 1 name]
    
You should successfully be connected to [Database 1 name]


#### Setting up deployment with OpsWorks

We now need to modify our OpsWorks setup to get rid of the RDS functionality. In the staging OpsWorks stack, set the application data source to 'none'. 

![Set data source to none][2]


When deploying using AWS OpsWorks and RDS like our prevous staging setup was, OpsWorks automatically creates a database.yml file using the information from the app's [deploy attributes][3]

Since our app now does not have an attached database, we have to do it using a custom JSON to add database attributes to the app's deploy attributes that will contain the connection information.The attributes are all under["deploy"]["appshortname"]["database"], where appshortname is the app's short name, which AWS OpsWorks generates from the app name. Below is a sample custom JSON for our staging-app
   

     {
      "deploy" : {
        "staging_app" : {
          "database" : {
            "adapter" : "pstgresql",
            "database" : "database-1-name",
            "host" : "X.X.X.X",
            "password" : "password",
            "port" : portnumber
            "reconnect" : true,
            "username" : "username"
          }
        }
      }
    }

Replace the IP X.X.X.X with the IP address of the Database Server and the 'portnumber' with the port number you used above to run postgres. With this, upon deployment, the database.yml file will be created with the above settings and 


### 2. Using SSH Tunneling

Coming Soon in part II...


  [1]: ({{ site.url }}/img/posts/db-server-sg-image.png)
  [2]: #f805d223-a7d0-8526-73e9-bb0a44de020c
  [3]: http://docs.aws.amazon.com/opsworks/latest/userguide/workingcookbook-json.html#workingcookbook-json-deploy
  [4]: http://www.kopokopo.com