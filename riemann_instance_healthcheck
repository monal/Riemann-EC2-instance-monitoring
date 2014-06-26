#!/usr/bin/env ruby

require 'fog'
require 'riemann/client'
require 'trollop'
require 'parallel'
require 'aws-sdk'

class ElbManager

  attr_accessor :connection, :ec2_connection

  def initialize(options,region)
    @connection ||= ElbManager.connection(options,region)
    @ec2_connection ||= ElbManager.ec2_connection(options,region) 
  end

  def self.connection(options,region)
    connection = Fog::AWS::ELB.new(
	:aws_access_key_id => options[:aws_access],
	:aws_secret_access_key =>options[:aws_secret],
	:region => region)
    connection
  end

  def self.ec2_connection(options,region)
    ec2_connection = Fog::Compute.new(
	:aws_access_key_id => options[:aws_access],
	:aws_secret_access_key =>options[:aws_secret],
    :provider => :aws,
	:region => region)
    ec2_connection
  end

  def load_balancers()
    connection.describe_load_balancers().instance_variable_get(:@data)[:body]["DescribeLoadBalancersResult"]["LoadBalancerDescriptions"]
  end

  def instances(elb)
	connection.describe_instance_health(elb).instance_variable_get(:@data)[:body]["DescribeInstanceHealthResult"]["InstanceStates"]
  end

  def elbs_in_az(az, patterns)
    elbs_in_az = []
    load_balancers.each do |loadbalancer|
	  zone = loadbalancer["AvailabilityZones"]
      if zone.include?(az)
        if not patterns.select { |pattern| loadbalancer["LoadBalancerName"].start_with?(pattern)}.empty? 
          elbs_in_az.push(loadbalancer["LoadBalancerName"])
        end
      end
    end
    elbs_in_az
  end

  def instances_in_az(az)
   instances_map=Hash.new
   instances_info=ec2_connection.describe_instances({'availability-zone' => az}).data[:body]['reservationSet'].select { |k| k['instancesSet'].first['placement']['availabilityZone']}
   puts instances_info
   instances_info.map{|k| instances_map["#{k['instancesSet'].first['instanceId']}"] = k['instancesSet'].first['tagSet']['Name']}
   instances_map
   end  
end

def emit_riemann_event(elb,name,state,options)
  event=Riemann::Client.new
  event << {
      host: "#{name}",
      service: "instance_status",
      ttl: options[:ttl],
      description: "Instance #{name} attached to elb #{elb} is #{state}",
      tags: [ "production", "instance_status" ],
      state: state
    }
end

options=Trollop::options do
  opt :aws_access, "AWS Access Key", :type => String
  opt :aws_secret, "AWS Secret Key", :type => String
  opt :aws_az, "List of AZs to aggregate against", :type => String, :default => "us-east-1a"
  opt :ttl, "TTL for the event", :default => 30
end


region=options[:aws_az].slice(0..-2)
elb_conn = ElbManager.new(options,region)
elbs_in_az = elb_conn.elbs_in_az(options[:aws_az], ["clb","plb"])
instances_in_az=elb_conn.instances_in_az(options[:aws_az])

puts Time.now()
SLICE=2

while true
 Parallel.map(elbs_in_az, :in_threads => SLICE) do |elb|
   name=elb
   instances= elb_conn.instances(name)
   instances.each do |instance|
     id, state = instance["InstanceId"],instance["State"]
     if instances_in_az.key?(id)
       puts "#{instances_in_az[id]} #{id} #{state}"
       emit_riemann_event(elb,instances_in_az[id],state,options)
	 end
   end
   sleep(0.25)
 end
 puts "--------------------------------------------------------#{Time.now()}------------------------------------------------"
end
