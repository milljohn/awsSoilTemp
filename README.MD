# AWS IoT Soil Temperature Sensor

AWS IoT Thing
Greengrass Communication
DynamoDB data storage

# Installing ESP32 in the Arduino IDE

If you just want to use the Arduino IDE to test, you can [find the instructions here](https://randomnerdtutorials.com/installing-the-esp32-board-in-arduino-ide-windows-instructions/)

You need to add these links to the Board Manager URLs in Preferences:
`https://dl.espressif.com/dl/package_esp32_index.json, http://arduino.esp8266.com/stable/package_esp8266com_index.json`

Next, go into the Board Manager and search for ESP32. I used the **WEMOS LOLIN32** board with **115200 baud rate**.

Most boards come from the manufacturer with a test script that prints to the serial monitor. To do a quick test, just open the Serial Monitor with **115200 baud** and plugin the board. You should see some kind of readout.

# Setting up AWS

[The Amazon FreeRTOS Documentation](https://docs.aws.amazon.com/freertos/latest/userguide/freertos-account-and-permissions.html) Is a little opaque if this is your first time using AWS. The steps below are

1. [Create AWS account](https://aws.amazon.com/premiumsupport/knowledge-center/create-and-activate-aws-account/)
2. [Create IAM user and policy](#create-iam-user-and-policy)
3. [Create IoT Thing](#create-iot-thing)
4. Download Amazon FreeRTOS
5. Configuring the Demo

## Create IAM user and policy

When you create an AWS account, you accessed the console using your email as a login. You will need to create a couple of users to limit the scope of access to services. This is a best practice and Amazon makes it inconvenient to log into your root account.

### Setting Up IAM Group Policy

1. Go to [https://console.aws.amazon.com](https://console.aws.amazon.com)
2. Search for `IAM` and click on the service
3. Select **Groups** under **Access management** on the left

You need to create two groups; one for administrators, and one for users with IoT access.

1. Click **Create New Group** on the top of the screen
2. Give your group a name, e.g. `Administrators` or `Soil-Temp-Dev`
3. Give your admin group `AdministratorAccess`. Give your dev group `AmazonFreeRTOSFullAccess` & `AWSIoTFullAccess`
4. Continue to Create Group

### Creating IAM Users

1. Continue in the IAM console
2. Click on **Users** under **Access management** on the left side of the screen.
3. Click on **Add User**
4. Give your user a name

   - Here you can create multiple users in the same group by clicking **Add another user**
   - You may want to create a user that has read only access to the data, for instance.

5. Give your users programatic access and console access if needed

   - Most users need both
   -

6. Add user(s) to the group you just created
7. Add tags if you want. This is necessary if you have a lot of users or a lot of different services that you are being billed for.
8. Download the certificates and activate them. Be sure to download the rootCA.

NOTE: The ESP8266 cannot handle Amazon's encryption directly, so a user would be needed in order to update the Thing Shaddow. In this case, the MCU isn't using the full capability of Amazon's IoT services and is essentialy just using the cli. This is not a good method for enterprise level production. This is the main reason why the ESP8266 is not a good choice for AWS IoT applications. The ESP32 is capable of running Amazon FreeRTOS, which allows us to manage devices much easier. Imagine having to keep track of 1,000 users/devices.

## Create IoT Thing

## Find endpoint

Using the aws cli:

`aws iot describe-endpoint`

Output will looke like this:

`{ "endpointAddress": "xxxxxxxxxxxxxx.iot.us-west-2.amazonaws.com" }`

## find aws account id

Using the aws cli:

`aws sts get-caller-identity`

## AWS IoT console > Secure > Policies > Create

Choose 'Advanced mode'
`{ "Version": "2012-10-17", "Statement": [ { "Effect": "Allow", "Action": "iot:Connect", "Resource":"arn:aws:iot:<aws-region>:<aws-account-id>:*" }, { "Effect": "Allow", "Action": "iot:Publish", "Resource": "arn:aws:iot:<aws-region>:<aws-account-id>:*" }, { "Effect": "Allow", "Action": "iot:Subscribe", "Resource": "arn:aws:iot:<aws-region>:<aws-account-id>:*" }, { "Effect": "Allow", "Action": "iot:Receive", "Resource": "arn:aws:iot:<aws-region>:<aws-account-id>:*" } ] }`

# Setting up Development Environment

Amazon partially relies on the Espressif toolchain. We will install the entire Espressif toolchain before moving onto Amazon FreeRTOS.

## Setup Windows Subsystem Linux (WSL)

Windows 10 has a Linux terminal emulator. I use Ubuntu, but you have several options. [Go here for step-by-step instructions.](https://docs.microsoft.com/en-us/windows/wsl/install-win10)

1. Open the Powershell as administrator
2. Enter this command, `Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Windows-Subsystem-Linux`
3. Open the Windows App Store and download Ubuntu 18.04
   - At the time of this writing there were three options for Ubuntu; Ubuntu, Ubuntu 16.04, and Ubuntu 18.04
   - I had Ubuntu installed and updated, but installing the toolchain failed.
   - I had success when I installed Ubuntu 18.04

We will download dependancies for both Windows and Linux. You can probably just do one or the other, but setting up both is easy.

## Install Toolchain in WSL and Linux

[The AWS Documentation links to this guide.](https://docs.espressif.com/projects/esp-idf/en/v3.3/get-started-cmake/linux-setup.html)

0. Open Ubuntu 18.04 from the Start Menu
1. Be sure to update and upgrade the system:
   - `$ sudo apt update ; sudo apt upgrade -y`
   - You may need to do this a couple of times.
1. Create a project directory in the Windows side. `C:\Users\<USERNAME>\esp`
1. Open WSL and create a symbolic link in your home directory
   - `$ ln -s /mnt/c/Users/<USERNAME>/esp/ ~`
   - Files in the WSL are difficult to access in Windows, it's easier to make Windows files accessible in WSL.
1. Install aws cli now:
   - `$ sudo apt install awscli -y`
1. Install cmake
   - It is tricky to get the latest version since WSL doesn't support snap.
   - Install old version first: `$ sudo apt install cmake -y`
   - [Download the sourcecode from here](https://cmake.org/download/)
     - `$ wget https://github.com/Kitware/CMake/releases/download/v3.16.1/cmake-3.16.1.tar.gz`
     - `$ tar xzf cmake-3.16.1.tar.gz`
     - `$ sudo apt install libssl-dev -y`
   - Use older version of cmake to install the latest:
     - `$ cd cmake-3.16.1`
     - `$ cmake .`
       - I got an error about OpenSSl not found, I fixed it with this flag:
       - `$ cmake -DCMAKE_USE_OPENSSL=OFF .`
     - `$ make -j`
     - `$ sudo make install`
   - Uninstall old version of cmake:
     `$ sudo apt remove cmake -y`
   - Close WSL and reopen it.
   - Check the version:
     - `$ cmake --version`
1. Install dependancies:
   - `$ sudo apt-get install git wget libncurses-dev flex bison gperf python python-pip python-setuptools python-serial python-cryptography python-future python-pyparsing cmake ninja-build ccache libncursesw5-dev -y`
1. Install Espressif Toolchain:

   - `$ cd ~/esp`
   - `$ wget https://dl.espressif.com/dl/xtensa-esp32-elf-linux64-1.22.0-80-g6c4433a-5.2.0.tar.gz`
   - `$ tar -xzf ~/Downloads/xtensa-esp32-elf-linux64-1.22.0-80-g6c4433a-5.2.0.tar.gz`
   - `$ echo export PATH="$HOME/esp/xtensa-esp32-elf/bin:$PATH" >> ~/.profile`
   - I had to use vim to add this to path.
   - `$ source ~/.profile`
   - `$ sudo apt-get install gawk gperf grep gettext python python-dev automake bison flex texinfo help2man libtool libtool-bin make -y`
   - `$ cd ~/esp`
   - `$ git clone -b xtensa-1.22.x https://github.com/espressif/crosstool-NG.git`
   - `$ cd crosstool-NG`
   - `$ ./bootstrap && ./configure --enable-local && sudo make install`
   - `$ ./ct-ng xtensa-esp32-elf`
   - `$ ./ct-ng build`
   - `$ chmod -R u+w builds/xtensa-esp32-elf`
   - `$ echo export PATH="$HOME/esp/crosstool-NG/builds/xtensa-esp32-elf:$PATH" >> ~/.profile`
   - I had to use vim in order to add this to path.
   - `$ cd ~/esp`
   - `$ git clone -b v3.3 --recursive https://github.com/espressif/esp-idf.git`
   - `$ echo IDF_PATH="$HOME/esp/esp-idf`
   - `$ source ~/.profile`
   - `$ python -m pip install --user -r $IDF_PATH/requirements.txt`
   - `$ pip install --user pyserial tornado node boto3`

## Install Toolchain in Windows

I did not setup the toolchain in Windows becuase it involved using the command line, which is easier in the Windows Subsystem Linux, outlined above.

## Espressif Hello World

1. Open WSL
2. `$ cd ~/esp`
3. `$ cp -r $IDF_PATH/examples/get-started/hello_world .`
4. Open **Device Manager** and look under **Ports (COM & LPT)** to find the COM port for the USB Serial device. NOTE: COM3 in Windows is /dev/ttyS3 in WSL.
5. `$ cd ~/esp/hello_world`
6. `$ idf.py menuconfig`
7.
8. `$ idf.py build`
9. `$ idf.py -p PORT flash`
   - I had issues with the baud rate with these particular boards. I had to se the baud rate to 921600
   - `$ idf.py -p PORT -b 921600 flash`
10. `$ idf.py -p PORT monitor`

## Amazon FreeRTOS

1. If you haven't downloaded the Amazon FreeRTOS from the console, do so now. Create a project directory in your Windows home directory called **myThing** and make a symlink in WSL like you did when you did for the esp directory.
2. `$ cd myThings`
3.

`cmake -DVENDOR=espressif -DBOARD=esp32_wrover_kit -DCOMPILER=xtensa-esp32 -S . -B your-build-directory`

`cd your-build-directory`

`make all -j`

`cd ..`

The Amazon documentation says to use the idf.py located in the tools directory. I was abable to get it to work. I was, however, able to get the espressif toolchain we setup for the hello-world to work.

`idf.py -p PORT -b 921600 erase_flash -B your-build-directory`

`make flash`

`idf.py -p PORT -b 921600 flash -B your-build-directory`
