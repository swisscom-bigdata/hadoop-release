# Hadoop auth with server name overrides support

This branch contains a modified version of `hadoop-auth` contained in `HDP 2.6.4.91-3` which allows to specify
server name overrides (through an environment variable) to retrieve the right Kerberos ticket when authenticating clients.
This allows to shortcut the normal mechanism (reverse DNS lookup), which can cause problem in complex setups where the client 
and server do not resolve to the same value, hence making authentication simply not possible.

Note that it is possible to specify multiple overrides for a given server name. In that case, overrides will be tried one by 
one in the order they are defined until one allows successful authentication. In case none allows successful authentication,
the exception thrown by the last one is raised.

## Usage
Simply set the environment variable `KERBEROS_SERVERNAME_OVERRIDES` with list of overrides, following the following syntax:

```
server_name=[a-zA-Z0-9.-_]+
override_def=<server_name>=<server_name>(,<server_name>)*
KERBEROS_SERVERNAME_OVERRIDES_ENV_NAME=<override_def>(;<override_def>)*
```

Example:
```
KERBEROS_SERVERNAME_OVERRIDES_ENV_NAME=server1=override_server1,override2_server2;server2=override_server2
```

## Build instructions
Find below the instruction on how to build the JAR. It requires to build a number of dependencies as they are not available in any public Maven repository (or not in any repo I tried and have access to at least).

### Prerequisites
The steps below were taken on a Mac running macOS Sierra 10.13.6. Should work with other environments with minimal changes.

Required tools:
* Homebrew (tested with version `2.0.1`)
* mvnvm (Maven Version Manager)
* Ant (tested with version `1.10.5`)
* Java 8 (tested with version `1.8.0_181`)

Make sure you have the following Maven repositories in your `~/.m2/settings.xml`:

```
<repository>
  <snapshots />
  <id>spring-plugins</id>
  <name>spring-plugins</name>
  <url>http://repo.spring.io/plugins-release/</url>
</repository>
<repository>
  <snapshots />
  <id>hortonworks</id>
  <name>hortonworks</name>
  <url>http://repo.hortonworks.com/content/repositories/releases/</url>
</repository>
```

### Step by step instructions
```
cd /tmp
mkdir ranger_plugin
cd ranger_plugin

git clone https://github.com/hortonworks/zookeeper-release.git
cd zookeeper-release
git checkout tags/HDP-2.6.4.91-3-tag
sed -i -- 's/3.4.6/3.4.6.2.6.3.0-SNAPSHOT/g' build.xml
brew install automake
ant mvn-install
# Yes, it works on second attempt...!?
ant mvn-install
cd ..

git clone https://github.com/swisscom-bigdata/hadoop-release.git
cd hadoop-release
git checkout HDP-2.6.4.91-3_sbd
brew install protobuf@2.5
mvn -DskipTests=true clean compile package install
cd ..
```

The Hadoop Auth JAR will then be available under `hadoop-common-project/hadoop-auth/target/hadoop-auth-2.7.3.2.6.3.0-SNAPSHOT-sources.jar`.