# Backport of HADOOP-15317 (NetworkTopology infinite loop)

This branch should only be used to build the custom `hadoop-common` version it contains.

That modified version of `hadoop-common` (based on the `HDP 2.6.5.0-292` version)
contains a backport of [HADOOP-15317](https://issues.apache.org/jira/browse/HADOOP-15317).

## Build instructions

Find below the instruction on how to build the JAR. It requires to build a
number of dependencies as they are not available in any public Maven repository
(or not in any repo we tried and have access to at least).

### Prerequisites

The steps below were taken on a Mac running macOS Catalina 10.15.4. Should work
with other environments with minimal changes.

Required tools:

* Homebrew (tested with version `2.2.14`)
* Maven (tested with `3.6.3`)
* Ant (tested with version `1.10.7`)
* Java 8 (tested with OpenJDK version `1.8.0_242`)
* Automake

Installing on MacOS:

```bash
brew cask install adoptopenjdk/openjdk/adoptopenjdk8
brew install maven ant automake
```

Make sure you have the following Maven repositories in your `~/.m2/settings.xml`:

```xml
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

```bash
# -- Temporary build dir
cd /tmp
mkdir hadoop265
cd hadoop265

# -- Zookeeper
git clone https://github.com/swisscom-bigdata/zookeeper-release.git
cd zookeeper-release
git checkout HDP-2.6.5.0-292_sbd
ant mvn-install
# Yes, it works on second attempt...!?
ant mvn-install
cd ..

# -- Protobuf 2.5 (we need the compiler installed)
curl -LJO https://github.com/protocolbuffers/protobuf/releases/download/v2.5.0/protobuf-2.5.0.tar.gz
tar zxf protobuf-2.5.0.tar.gz
cd protobuf-2.5.0
./configure
make
make check
sudo make install
cd ..

# -- Hadoop
git clone https://github.com/swisscom-bigdata/hadoop-release.git
cd hadoop-release
git checkout HDP-2.6.5.0-292_sbd
mvn -DskipTests=true clean compile package
```

The Hadoop common JAR will then be available under
`hadoop-common-project/hadoop-common/target/hadoop-common-2.7.3.2.6.5.0-292.jar`.

## Applying the patch

To limit the production changes to a minimum, only the affected class has been
patched in.

The built `hadoop-common-2.7.3.2.6.5.0-292.jar` can be used to extract the
needed class files:

```bash
cd /tmp/hadoop265
mkdir patching
cd patching

# Extract built classes
cp ../hadoop-release/hadoop-common-project/hadoop-common/target/hadoop-common-2.7.3.2.6.5.0-292.jar hadoop-common-build.jar
unzip hadoop-common-build.jar -d hadoop-common-build

# Replace in HDP JAR
scp <YOUR_HOST_HERE>:/usr/hdp/2.6.5.0-292/hadoop/hadoop-common-2.7.3.2.6.5.0-292.jar hadoop-common-orig.jar
cp hadoop-common-orig.jar hadoop-common-patched.jar
cd hadoop-common-build
for cf in org/apache/hadoop/net/NetworkTopology*; do zip -u ../hadoop-common-patched.jar $cf; done
cd ..
```

The resulting JAR `hadoop-common-patched.jar` can then be deployed to the
machines by replacing
`/usr/hdp/2.6.5.0-292/hadoop/hadoop-common-2.7.3.2.6.5.0-292.jar` with it and
restarting the services
