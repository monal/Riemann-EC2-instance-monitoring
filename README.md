Riemann-EC2-instance-monitoring
===============================

The tool monitors the state of all Amazon EC2 instances in a given availability zone. This can be useful during software releases to check if all servers are back in service. The tool reports the data to Riemann, an open-source event stream processor.

(Documentation on Riemann can be found at http://riemann.io)

Pre-requisites:
========================
1) Riemann server must be running and must be configured to listen on all interfaces.

2) Riemann dashboard must point to the server running Riemann (The dashboard listens to localhost by default) 

3) Ruby1.9 must be installed on the machine running the Riemann-EC2-instance-monitoring script i.e the Riemann client

Using the Riemann-EC2-instance-monitoring tool
===============================================

1) Copy the contents of the /init_scripts folder into /etc/init.d on the machine running Riemann server

2) Riemann can now be started, stopped or restarted as:
	
	service riemann start	(Starting Riemann server)
	service riemann stop	(Stopping Riemann server)
	service riemann restart	(Restarting Riemann server)

3) Install the gem dependencies using the requirements.txt 
	
4) Run the riemann_instance_healtcheck script with the availability zone and aws credentials as parameters.
	
	Eg:	ruby riemann_instance_healthcheck -s "us-east-1a" --aws_access "access_key" --aws_secret "secret_key"
   
5) Monitor the results from the Riemann dashboard by setting the dashboard view to 'Grid' and querying for : state='InService' or state='OutOfService' to display the instances based on their current state.
