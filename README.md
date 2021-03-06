# 0: Introduction  
This guide will cover how to use an Intel Edisons as a gateway connecting to a
sensor sending temperature readings. The sensor will end data to the gateway 
Edison via BLE which will then send it to Appiot.

In order to get the gateway up and running you need to install some software 
on the edison, register the devices and build the code.

Here's an example on how to set up a sensor that transmits data over BLE:
http://appiot-docs.azurewebsites.net/?p=8021

# 1: Prerequisites
### 1.1: Edison setup
Follow the instructions on this page: (there are links to other OS'
s if you're not on Windows)
https://software.intel.com/en-us/get-started-edison-windows

### 1.2: Maven
```sh
$ wget http://apache.mirrors.spacedump.net/maven/maven-3/3.3.9/binaries/apache-maven-3.3.9-bin.tar.gz
$ tar xf apache-maven-3.3.9-bin.tar.gz
$ export PATH=${PWD}/apache-maven-3.3.9/bin/:$PATH
```

### 1.3: BlueZ
The version of BlueZ that comes with the Edison might be too old for the TinyB 
package that is used for the BLE communcation.

A newer version can be downloaded here:
http://www.bluez.org/release-of-bluez-5-42/

```sh
$ wget http://www.kernel.org/pub/linux/bluetooth/bluez-5.42.tar.xz
$ tar xf bluez-5.42.tar.xz
$ cd bluez-5.42/
$ ./configure --prefix=/usr --mandir=/usr/share/man --sysconfdir=/etc --localstatedir=/var
$ make && make install
```

### 1.4: Java Cacerts
if the file /usr/lib/jvm/java-8-openjdk/jre/lib/security/cacerts is empty (or 
non-existing) you can copy the cacerts from the computer you're working on.

In my case they were located under
/c/program\ Files/Java/jdk1.8.0_101/jre/lib/security/cacerts

# 2: Build TinyB
### 2.1: Clone project
```sh
$ git clone https://github.com/intel-iot-devkit/tinyb.git
```

### 2.2: Remove version check: 
(This is because the specificationVersion information is lost in Maven, would 
be nice to fix)
```sh
$ cd tinyb/
$ nano java/BluetoothManager.java
```
Add a block comment starting on line 219 and ending on line 239 (after the 
if case). The getBluetoothManager method should look like this when you're 
done:

```java
public static synchronized BluetoothManager getBluetoothManager() throws RuntimeException, BluetoothException
{
    if (inst == null)
    {
        inst = new BluetoothManager();
        inst.init();
        /*
        String nativeAPIVersion = getNativeAPIVersion();
        String APIVersion = BluetoothManager.class.getPackage().getSpecificationVersion();
        if (APIVersion.equals(nativeAPIVersion) == false) {
            String[] nativeAPIVersionCode = nativeAPIVersion.split("\\D");
            String[] APIVersionCode = APIVersion.split("\\D");
            if (APIVersionCode[0].equals(nativeAPIVersionCode[0]) == false) {
                if (Integer.valueOf(APIVersionCode[0]) < Integer.valueOf(nativeAPIVersionCode[0]))
                    throw new RuntimeException("Java library is out of date. Please update the Java library.");
                else throw new RuntimeException("Native library is out of date. Please update the native library.");
            }
            else if (APIVersionCode[0].equals("0") == true) {
                if (Integer.valueOf(APIVersionCode[1]) < Integer.valueOf(nativeAPIVersionCode[1]))
                    throw new RuntimeException("Java library is out of date. Please update the Java library.");
                else throw new RuntimeException("Native library is out of date. Please update the native library.");
            }
            else if (Integer.valueOf(APIVersionCode[1]) < Integer.valueOf(nativeAPIVersionCode[1]))
                System.err.println("Java library is out of date. Please update the Java library.");
            else System.err.println("Native library is out of date. Please update the native library.");
        }
        */
    }
    return inst;
}
```

### 2.3 Set Java variables:
```sh
$ export JAVA_HOME=/usr/lib/jvm/java-8-openjdk/
$ export PATH=$JAVA_HOME/bin:$PATH
```

### 2.4 Build:
Move to the top of the tinyb folder and execute the following shell commands:
```sh
$ mkdir build
$ cd build
$ cmake -DBUILDJAVA=ON -DCMAKE_INSTALL_PREFIX=/usr ..
$ make
$ sudo make install
```

There should now be a file called tinyb.jar under
/usr/lib/java/

# 3: Build the Gateway
### 3.1 Set up $SENSATION_HOME
Create sensation home folder and assign the variable $SENSATION_HOME to it's 
path.
```sh
$ mkdir ~/sensation_home
$ export SENSATION_HOME=~/sensation_home
```
Set up sensation-client.properties file
```sh
$ cd $SENSATION_HOME
$ echo "log_search_string=samplegateway" >> sensation-client.properties
$ echo "log_level=INFO" >> sensation-client.properties
$ echo "report_interval=1000" >> sensation-client.properties
$ echo "sendInterval=1000" >> sensation-client.properties
$ echo "queue_size=1000" >> sensation-client.properties
```

### 3.2 Build Gateway
###### 3.2.1 Configure gateway
```sh
$ git clone https://github.com/AxelUlmestig/Appiot-Edison-Gateway.git
```

```sh
$ cd Appiot-Edison-Gateway/
$ nano src/main/java/com/appiot/examples/gateway/samplegateway/EdisonGateway.java
```
 -  Edit lines 29 and set the variable 'macAddrees' to the mac address of your sensor.
 -  Set the variable 'deviceId' on line 30 to a unique id. You can use the
 mac address here as well if you're lacking in imagination.

Remember the deviceId value, it will be needed in step 6.2.

###### 3.2.2 Install dependencies and build
Install dependencies and package tinyb.jar for maven

```sh
$ mvn install:install-file \
        -Dfile=/usr/lib/java/tinyb.jar \
        -DgroupId=tinyb \
        -DartifactId=tinyb \
        -Dversion=1.0 \
        -Dpackaging=jar \
        -DgeneratePom=true
```

Compile a runnable jar file
```sh
$ mvn compile assembly:single
```

# 4 Register gateway in Appiot

### 4.1: Create gateway type
  - Go to the Appiot page.
  - Press 'Settings' -> 'Hardware Types'.
  - Press 'Create'.
  - Name it 'Edison Gateway' and give it a unique id (a large number preferably).
  - Press 'Save'

### 4.2: Register gateway
  - Go back to your Appiot front page.
  - Press 'Register Gateway'.
  - Give it a unique serial number and choose the Gateway Type that you just 
  created.
  - Name you gateway something creative, like "My Edison Gateway".
  - Press 'Register'.

### 4.3: Export device

  - Go to the your newly created gateway and press 'Actions' -> 'Download Tickets'.
  (if you came from the previous section you should be on the right page).
  - Write the content of the downloaded file to $SENSATION_HOME/registrationticket.json
  on the gateway Edison.

# 5: Run
### 5.1: Start Bluetooth

Execute the following commands in the terminal to enable bluetooth:
```sh
$ rfkill unblock bluetooth
$ killall bluetoothd
$ hciconfig hci0 up
```

### 5.2: Start the Gateway
There should now be a runnable jar file under
SampeGateway/target/EdisonGateway-0.0.1-jar-with-dependencies.jar
Run it with
```sh
$ java -jar Appiot-Edison-Gateway/target/EdisonGateway-0.0.1-jar-with-dependencies.jar
```

You should now be able to see the readings from the other Edison
written in the terminal.

# 6: Register device in Appiot
### 6.1 Create device type
  - Go to 'Settings' -> 'Hardware Types'
  - Then press Device Types and then Create.
  - Name the device type "Edison Temperature" and give it a unique id 
  in the range 20000-30000.
  - Press 'Add Sensor' and a choose the type 'Temperature'.
  - Press 'Save'.

### 6.2 Register Device
For this step to work the gateway Edison device needs to be running (it should 
be printing measurements in the console) and the gateway Edison must be 
connected to the internet.
  - Go back to the front page and press 'Register Device'.
  - Set the Serial Number to the deviceId you chose in step 3.2.1.
  - Choose your newly created Device Type "Edison Temperature".
  - Give your device a fitting name, e.g. "Edison Temperature Device".
  - Choose the gateway that you created in step 4.2.
  - Press 'Continue' and then 'Register'.

Now you should be able to go to your newly registered device and see a nice 
graph of temperature measurements.

# APPENDIX
### Mock data, no BLE
If you're not interested in BLE you can use the EdisonGatewayMock class instead of the
EdisonGateway. The mock gateway will send the hard coded temperature value 1337 to 
Appiot once when run.

In order to use this, edit the file EdisonGatewayMock.java and add the deviceId of a
device that you're intending to use. 

Then open the pom.xml file and edit the mainClass tag.
It should now end with EdisonGatewayMock instead of EdisonGateway.

After that, rebuild that application 
```sh
$ mvn compile assembly:single
```
and then you can run it as usual.

### opkg
If you're getting problems with opkg you might need to specify the repos for 
opkg:

```sh
$ echo "src all     http://iotdk.intel.com/repos/1.1/iotdk/all" >> /etc/opkg/base-feeds.conf
$ echo "src x86 http://iotdk.intel.com/repos/1.1/iotdk/x86" >> /etc/opkg/base-feeds.conf
$ echo "src i586    http://iotdk.intel.com/repos/1.1/iotdk/i586" >> /etc/opkg/base-feeds.conf
```
