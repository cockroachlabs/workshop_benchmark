Install JMeter locally

  Create Local Test Cluster

  Install JMeter on Laptop
        ## Base Jmeter
        wget https://apache.cs.utah.edu//jmeter/binaries/apache-jmeter-5.2.1.tgz

        ## bundle with plugins....
        wget https://github.com/glennfawcett/jmeter_cockroachdb/blob/master/apache-jmeter-5.2.1_CRDB_labs.tar.gz

  Download sample JMX file
        https://github.com/glennfawcett/jmeter_cockroachdb/blob/master/crdb_jmeter_example1.jmx

  Create DDL for LOCAL test
        https://github.com/glennfawcett/jmeter_cockroachdb/blob/master/crdb_jmeter_example1_ddl.sql


Run test locally
  Check to ensure accuracy of mix
  Experiment with changing the mix


Extra Credit ::
  Copy *.jmx file to driver VM
  Run CLI version of JMeter

