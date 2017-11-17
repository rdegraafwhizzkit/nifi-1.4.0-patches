# nifi-1.4.0-patches

## Patch and recompile
export NIFI_SRC=~/Code/nifi
export NIFI_HOME=~/Development/nifi-1.4.0
export PATH=~/Development/maven/bin:$PATH
export JAVA_HOME=`/usr/libexec/java_home -v1.8`

curl -L -O http://www-eu.apache.org/dist/nifi/1.4.0/nifi-1.4.0-bin.tar.gz
cd $NIFI_HOME/../
tar zxf nifi-1.4.0-bin.tar.gz
rm -f nifi-1.4.0-bin.tar.gz

cd $NIFI_SRC/../
git clone -b "rel/nifi-1.4.0" https://github.com/apache/nifi.git

cd $NIFI_SRC

curl -L -O https://github.com/rdegraafwhizzkit/nifi-1.4.0-patches/archive/master.zip
unzip master.zip "nifi-1.4.0-patches-master/nifi-nar-bundles/*"
cp -rf nifi-1.4.0-patches-master/nifi-nar-bundles/* nifi-nar-bundles/
rm -rf nifi-1.4.0-patches-master/ master.zip 

cd $NIFI_SRC/nifi-nar-bundles/nifi-standard-bundle/nifi-standard-processors
mvn -Dmaven.test.skip=true clean install

cd $NIFI_SRC/nifi-nar-bundles/nifi-standard-bundle/nifi-standard-nar
mvn clean nifi-nar:nar

cp target/nifi-standard-nar-1.4.0.nar $NIFI_HOME/lib

cd $NIFI_SRC/nifi-nar-bundles/nifi-hadoop-libraries-bundle/nifi-hadoop-libraries-nar
curl -O https://raw.githubusercontent.com/rdegraafwhizzkit/nifi-hadoop-libraries-bundle/master/nifi-hadoop-libraries-nar/pom.xml
mvn clean install

cp target/nifi-hadoop-libraries-nar-1.4.0.nar $NIFI_HOME/lib

cd $NIFI_HOME
bin/nifi.sh start


## Install on a MapR Cluster

### Download distribution
wget http://www-eu.apache.org/dist/nifi/1.4.0/nifi-1.4.0-bin.tar.gz
cp nifi-1.4.0-bin.tar.gz /mapr/mapr.whizzkit.nl/tmp

### Set up and extract
clush -a mkdir -p /opt/whizzkit/nifi
clush -a tar -C /opt/whizzkit/nifi/ -zxf /mapr/mapr.whizzkit.nl/tmp/nifi-1.4.0-bin.tar.gz
clush -a 'rm -f `find /opt/whizzkit/nifi/ -name "*bat"`'

clush -a cp /mapr/mapr.whizzkit.nl/tmp/nifi-hadoop-libraries-nar-1.4.0.nar /opt/whizzkit/nifi/nifi-1.4.0/lib/
clush -a cp /mapr/mapr.whizzkit.nl/tmp/nifi-standard-nar-1.4.0.nar /opt/whizzkit/nifi/nifi-1.4.0/lib/

clush -a chown -R mapr.mapr /opt/whizzkit/nifi/nifi-1.4.0/

### Configure 
clush -a 'sed -i -e "s/^nifi\.web\.http\.host=.*$/nifi.web.http.host=$HOSTNAME/g" /opt/whizzkit/nifi/nifi-1.4.0/conf/nifi.properties'
clush -a 'sed -i -e "s/^nifi\.web\.http\.port=.*$/nifi.web.http.port=7070/g" /opt/whizzkit/nifi/nifi-1.4.0/conf/nifi.properties'
clush -a 'sed -i -e "s/^nifi\.zookeeper\.connect\.string=.*$/nifi.zookeeper.connect.string=server1.whizzkit.nl:5181,server2.whizzkit.nl:5181,server3.whizzkit.nl:5181/g" /opt/whizzkit/nifi/nifi-1.4.0/conf/nifi.properties'
clush -a 'sed -i -e "s/^nifi\.cluster\.is\.node=.*$/nifi.cluster.is.node=true/g" /opt/whizzkit/nifi/nifi-1.4.0/conf/nifi.properties'
clush -a 'sed -i -e "s/^nifi\.cluster\.node\.address=.*$/nifi.cluster.node.address=$HOSTNAME/g" /opt/whizzkit/nifi/nifi-1.4.0/conf/nifi.properties'
clush -a 'sed -i -e "s/^nifi\.cluster\.node\.protocol\.port=.*$/nifi.cluster.node.protocol.port=12000/g" /opt/whizzkit/nifi/nifi-1.4.0/conf/nifi.properties'
clush -a 'sed -i -e "s/^nifi\.cluster\.node\.protocol\.threads=.*$/nifi.cluster.node.protocol.threads=15/g" /opt/whizzkit/nifi/nifi-1.4.0/conf/nifi.properties'
clush -a 'sed -i -e "s/^nifi\.cluster\.node\.connection\.timeout=.*$/nifi.cluster.node.connection.timeout=30 sec/g" /opt/whizzkit/nifi/nifi-1.4.0/conf/nifi.properties'
clush -a 'sed -i -e "s/^nifi\.cluster\.node\.read\.timeout=.*$/nifi.cluster.node.read.timeout=30 sec/g" /opt/whizzkit/nifi/nifi-1.4.0/conf/nifi.properties'

### Fix zookeeper connection for state management
clush -a 'sed -i -e "s/^.*Connect String.*$/<property name=\"Connect String\">server1.whizzkit.nl:5181,server2.whizzkit.nl:5181,server3.whizzkit.nl:5181<\/property>/g" /opt/whizzkit/nifi/nifi-1.4.0/conf/state-management.xml'

### Set login information
expr `clush -a 'grep -E "java\.arg\.[0-9]+" /opt/whizzkit/nifi/nifi-1.4.0/conf/bootstrap.conf|cut -d "." -f 3|cut -d "=" -f 1'|cut -d " " -f 2|sort -nr|head -1` + 1 > /tmp/newarg
export newarg=`awk '{print "java.arg."$0"=-Djava.security.auth.login.config=/opt/mapr/conf/mapr.login.conf"}' < /tmp/newarg` 
clush -a "echo $newarg >> /opt/whizzkit/nifi/nifi-1.4.0/conf/bootstrap.conf"
rm /tmp/newarg

### Follow instructions above if you need the pached nar stuff

# (Re)start
clush -a /opt/whizzkit/nifi/nifi-1.4.0/bin/nifi.sh stop
clush -a /opt/whizzkit/nifi/nifi-1.4.0/bin/nifi.sh status
clush -a /opt/whizzkit/nifi/nifi-1.4.0/bin/nifi.sh restart
