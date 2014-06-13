accumulo-jmxtrans-ganglia
=========================

Accumulo JMX metrics integration with jmxtrans, ganglia and nagios


Test setup

In this test setup, there are 2 servers (c6401.ambari.apache.org) and (c6402.ambari.apache.org) referred in short as c6401 and c6402 in the document. Ambari was used to install HDP on these nodes and accumulo was installed on top based on rpms. 

Services on c6401 (only related to ambari and ganglia/nagios)

1. Accumulo Master
2. Accumulo TabletServer
3. Accumulo GC
4. Accumulo Tracer
5. Ganglia (individual gmond services running to collect metrics for multiple services)


Service on c6402

1. Accumulo TabletServer
2. Nagios Server
3. Ganglia (gmond and gmetad) (indiviual gmond services running to collection metrics for multiple services, we add a new gmond here to collect accumulo metrics from all hosts)

WARNING:: If ganglia and nagios are restarted from ambari UI, the configuration changes are overwritten. Careful not to do that and take a backup. Ultimately, all these changes will be integrated into ambari to let ambari configure for you. 

Steps:


1. Edit accumulo-env.sh on both hosts to enable JMX monitoring restart accumulo. 

export ACCUMULO_TSERVER_OPTS="-Dcom.sun.management.jmxremote.port=12341 -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false $ACCUMULO_TSERVER_OPTS"
export ACCUMULO_MASTER_OPTS="-Dcom.sun.management.jmxremote.port=12342 -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false $ACCUMULO_MASTER_OPTS"
export ACCUMULO_MONITOR_OPTS="-Dcom.sun.management.jmxremote.port=12343 -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false $ACCUMULO_MONITOR_OPTS"
export ACCUMULO_GC_OPTS="-Dcom.sun.management.jmxremote.port=12344 -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false $ACCUMULO_GC_OPTS"

This enables JMX with no SSL or authentication (TODO: Need to change them to enable SSL and authentication). 

JMX Metrics for TabletServer can now be collected on port 12341. 


2. gmond (on c6402): I created a new gmond service to collect all accumulo related metrics. This gmond service is only run on c6402 (unlike other gmonds that run on all 2 hosts and collect metrics locally - can be changed to collect metrics locally)

For convenience, I am copying gmond config from /etc/ganglia/hdp/HDPHBaseRegionServers. (copy whole directory to /etc/ganglia/hdp/HDPAcccumuloServers). Note that ganglia metrics for HBase runs muliple collectors, one each for HBaseMaster and HBaseRegionServers while we will run one to collect all metrics from Accumulo)  

Edit /etc/ganglia/hdp/HDPAccumuloServers/gmond.core.conf and change cluster {name} to HDPAccumuloServers and  tcp_accept_channel {port} to 9666 (or any other port you would like)
Edit /etc/ganglia/hdp/HDPAccumuloServers/conf.d/gmond.master.conf to change udp_rcv_channel {port} to 9666
Edit /etc/ganglia/hpd/HDPAccumuloServers/conf.d/gmond.slave.conf to change udp_send_channel {port} to 9666

Create a directory /var/run/ganglia/hdp/HDPAccumuloServers and start gmond for collecting accumulo metrics
/usr/sbin/gmond --conf=/etc/ganglia/hdp/HDPAccumuloServers/gmond.core.conf --pid-file=/var/run/ganglia/hdp/HDPAccumuloServers/gmond.pid

3. gmetad (on C6402). Edit /etc/ganglia/hdp/gmetad.conf and add  data_source "HDPAccumuloServers" c6402.ambari.apache.org:9666
where datasources are listed

Restart gmetad. service hdp-metad restart


4. JMXTRANS (on c6402):: Install jmxtrans using rpm (https://github.com/downloads/jmxtrans/jmxtrans/jmxtrans-20121016.145842.6a28c97fbb-0.noarch.rpm). Its installed in /usr/share/jmxtrans (executable) and picks configuration from /var/lib/jmxtrans/. 
Created accumulo.json (https://raw.githubusercontent.com/whiz/accumulo-jmxtrans-ganglia/master/accumulo.json) in /var/lib/jmxtrans. 
Start jmxtrans (/usr/share/jmxtrans start /var/lib/jmxtrans/accumulo.json). In this configuration, we are using jmxtrans on c6402 to read JMX metrics of TabletServers from both c6401 and c6402 and write into accumulo gmond collector running on c6402.  



5. Nagios (on c6402):: Once we have metrics in place, we can use nagios to send alerts in any metrics are out of range or if any service goes down. 

Edit /etc/nagios/object/hadoop-hostgroups.cfg to add all accumulo service hosts 

define hostgroup {
        hostgroup_name  accumulomasters
        alias           accumulomasters
        members         c6401.ambari.apache.org
}
define hostgroup {
        hostgroup_name  accumulotservers
        alias           accumulotservers
        members         c6401.ambari.apache.org,c6402.ambari.apache.org
}
define hostgroup {
        hostgroup_name  accumulotracers
        alias           accumulotracers
        members         c6401.ambari.apache.org
}

Edit /etc/nagios/object/hadoop-commands.cfg to all check_ganglia that can check a gmond service

define command {
  command_name check_ganglia
  command_line $USER1$/check_ganglia.py -h $HOSTNAME$ -m $ARG1$ -w $ARG2$ -c $ARG3$ -p $ARG4$
}

Copy check_ganglia.py(https://raw.githubusercontent.com/ganglia/monitor-core/master/contrib/check_ganglia.py)  to /usr/lib64/nagios/plugins/

Edit /etc/nagios/object/hadoop-servicegroups.cfg and add

define servicegroup {
  servicegroup_name ACCUMULO
  alias ACCUMULO Checks
}

Edit /etc/nagios/object/hadoop-services.cfg and add 

# ACCUMULO::TSERVER Checks
define service {
        hostgroup_name          accumulotservers
        use                     hadoop-service
        service_description     TSERVER::TabletServer process
        servicegroups           ACCUMULO
        check_command           check_tcp_wrapper!9997!-w 1 -c 1
        normal_check_interval   1
        retry_check_interval    0.5
        max_check_attempts      3
}

define service {
        hostgroup_name          accumulotservers
        use                     hadoop-service
        service_description     TSERVER:: TableServer load
        servicegroups           ACCUMULO
        check_command           check_ganglia!Accumulo.Ingest!50!60!9666
        normal_check_interval   1
        retry_check_interval    0.5
        max_check_attempts      3
}

define service {
        hostgroup_name          accumulotservers
        use                     hadoop-service
        service_description     TSERVER:: TableServer load
        servicegroups           ACCUMULO
        check_command           check_ganglia!Accumulo.ScanAvgTime!4!6!9666
        normal_check_interval   1
        retry_check_interval    0.5
        max_check_attempts      3
}

define service {
        hostgroup_name          accumulomasters
        use                     hadoop-service
        service_description     MASTER::Master process
        servicegroups           ACCUMULO
        check_command           check_tcp_wrapper!9999!-w 1 -c 1
        normal_check_interval   1
        retry_check_interval    0.5
        max_check_attempts      3
}

define service {
        hostgroup_name          accumulotracers
        use                     hadoop-service
        service_description     TRACER::Tracer process
        servicegroups           ACCUMULO
        check_command           check_tcp_wrapper!12234!-w 1 -c 1
        normal_check_interval   1
        retry_check_interval    0.5
        max_check_attempts      3
}

define service {
        host_name               c6401.ambari.apache.org
        use                     hadoop-service
        service_description     GC::gc process
        servicegroups           ACCUMULO
        check_command           check_tcp_wrapper!50091!-w 1 -c 1
        normal_check_interval   1
        retry_check_interval    0.5
        max_check_attempts      3
}

define service {
        host_name               c6401.ambari.apache.org
        use                     hadoop-service
        service_description     MONITOR::monitor process
        servicegroups           ACCUMULO
        check_command           check_tcp_wrapper!50095!-w 1 -c 1
        normal_check_interval   1
        retry_check_interval    0.5
        max_check_attempts      3
}

Restart Nagios (service nagios restart)




