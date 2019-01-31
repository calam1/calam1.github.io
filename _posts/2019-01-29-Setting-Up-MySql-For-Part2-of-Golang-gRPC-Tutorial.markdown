---
title: "Setting up MySql for Part 2 of the gRPC using Go"
date: 2019-01-29 13:41:00
description: "Setting up mySql on Docker for use in Part 2 of the gRPC and Go Tutorial"
---
This entry will go over setting up a docker instance of mySql.  We will need this database for Part 2 of our tutorial.  I am not going to go over the details of Docker or mySql; so I am assuming some comfort level with these technologies.  But this is pretty simple, so as long as you know how to install [Docker][docker-desktop-url], the rest should be relatively simple.

* Docker 18.09.1-ce
* MySql I am using the latest image from docker hub

&nbsp;

**Pull MySql Docker Image**

First you need to create a [Docker Hub Account][docker-hub-url]; once that is completed just run the following command
{% highlight ruby %}
> docker pull mysql/mysql-server
{% endhighlight %}

**Start the Docker Image of MySql**
&nbsp;
{% highlight ruby %}
> docker run --name mysql -d mysql/mysql-server:latest
{% endhighlight %}

**Retrieve the Generated Password**

&nbsp;
You will need the password for the next step
{% highlight ruby %}
> docker logs mysql1 2>&1 | grep GENERATED
{% endhighlight %}

**Go into container and run mysql**

&nbsp;
Enter the password from the previous step when prompted
{% highlight ruby %}
> docker exec -i -t -u root mysql mysql -p  
{% endhighlight %}

If you run into any errors like "No database selected" you will need to run the following

* show databases; // this will show you what databases/schemas exist
* for this particular example choose mysql by typing the following command 
{% highlight ruby %}
> use mysql;
{% endhighlight %}

* now you should be able to alter the root password and create the new user and grant permissions
* change password 

{% highlight ruby %}
> ALTER USER 'root'@'localhost' IDENTIFIED BY 'password';
{% endhighlight %}

mysql will require you to grant permissions to users to access the database from outside it's localhost.  So if you try to use the root and its password from an app, you will get refuse connection error.  I just created a new user and granted it permissions.

To create a new user and grant permissions, enter the container and mysql app as we did above or if you are still at the mysql prompt even better. Run this command using whatever user and password you like, I am using chris and password as the password 
{% highlight ruby %}
> CREATE USER 'chris'@'%' IDENTIFIED BY 'password';
{% endhighlight %}

Grant permissions to the user you just created above 
{% highlight ruby %}
> grant all privileges on *.* to 'chris'@'%' with grant option;      
{% endhighlight %}

**Create the database and table for Part 2 of the gRPC tutorial**

Firsty create the database.  So if you are not in mySql in the docker container log back in as explained above.

*Create the database*
{% highlight sql %}
> create DATABASE customers;
{% endhighlight %}

*Choose the database*
{% highlight sql %}
> use customers;
{% endhighlight %}

Now we will create a trivial "user" table
{% highlight sql %}
CREATE TABLE `User` (   `ID` bigint(20) NOT NULL AUTO_INCREMENT,   `FirstName` varchar(200) DEFAULT NULL,  `LastName` varchar(200) DEFAULT NULL, `Address` varchar(1024) DEFAULT NULL, PRIMARY KEY (`ID`),   UNIQUE KEY `ID_UNIQUE` (`ID`) );
{% endhighlight %}



[docker-desktop-url]: https://www.docker.com/products/docker-desktop
[docker-hub-url]: https://www.docker.com/products/docker-hub
