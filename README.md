﻿# TCP client/server API for C++ (with SSL/TLS support)
[![MIT license](https://img.shields.io/badge/license-MIT-blue.svg)](http://opensource.org/licenses/MIT)


## About
This is a simple TCP server/client for C++. Under Windows, it wraps WinSock and under Linux it wraps 
the related socket API (BSD compatible). It wraps also OpenSSL to create secure client/server sockets.

It is meant to be a portable and easy-to-use API to create a TCP server or client with or without SSL/TLS
support.

Upcoming features : using the sockets in an async way and proxy support.

Compilation has been tested with:
- GCC 5.4.0 (GNU/Linux Ubuntu 16.04 LTS)
- Microsoft Visual Studio 2015 (Windows 10)

## Usage
Create a server or client object and provide to its constructor a callable object (for log printing)
having this signature :

```cpp
void(const std::string&)
```

For now, you can disable log printing by choosing the flag ASocket::SettingsFlag::NO_FLAGS when creating
a socket or by providing a callable object that does nothing with the string message.

```cpp
#include "TCPClient.h"
#include "TCPServer.h"
#include "TCPSSLServer.h"
#include "TCPSSLClient.h"

auto LogPrinter = [](const std::string& strLogMsg) { std::cout << strLogMsg << std::endl;  }

CTCPClient TCPClient(LogPrinter); // creates a TCP client
CTCPServer TCPServer(LogPrinter, "12345"); // creates a TCP server to listen on port 12345

CTCPSSLClient SecureTCPClient(LogPrinter); // creates an SSL/TLS TCP client
CTCPSSLServer SecureTCPSSLServer(LogPrinter, "4242"); // creates an SSL/TLS TCP server to listen on port 4242
```

Please note that the constructor of CTCPServer or the SSL/TLS version throws only an exception in the Windows
version when the address resolution fails, so you should use the try catch block in this particular context.

To listen for an incoming TCP connection :

```cpp
ASocket::Socket ConnectedClient; // socket of the connected client, we can have a vector of them for example.
/* blocking call, should return true if the accept is OK, ConnectedClient should also be a valid socket
number */
m_pTCPServer->Listen(ConnectedClient);
```

A wait period (in milliseconds) can be set, to avoid waiting indefinitely for a client :

```cpp
m_pTCPServer->Listen(ConnectedClient, 1000); // waits for 1 second. Will return true, if a client connected to the server
```

To connect to a particular server (e.g 127.0.0.1:669)

```cpp
m_pTCPClient->Connect("127.0.0.1", "669"); // should return true if the connection succeeds
```

To send/receive data to/from a client :

```cpp
const std::string strSendData = "Hello World !";
m_pTCPServer->Send(ConnectedClient, strSendData);
/* or */
m_pTCPServer->Send(ConnectedClient, strSendData.c_str(), 13);
/* or even an std::vector<char> as a second parameter */

char szRcvBuffer[14] = {};
m_pTCPServer->Receive(ConnectedClient, szRcvBuffer, 13);
```

To send/receive data to/from a server :

```cpp
const std::string strSendData = "Hello World !";
m_pTCPClient->Send(strSendData);
/* or */
m_pTCPClient->Send(strSendData.c_str(), 13);
/* or even an std::vector<char> */

char szRcvBuffer[14] = {};
m_pTCPClient->Receive(szRcvBuffer, 13);
```

To disconnect from server or client side :

```cpp
m_pTCPClient->Disconnect();

m_pTCPServer->Disconnect(ConnectedClient);
```

Before using SSL/TLS secured classes, compile both library and the test program with the preprocessor macro OPENSSL.

Almost all the operations look similar to the operations above for unencrypted communications, the differences are :

The client socket to provide to the Listen method of an CTCPSSLServer is of type ASecureSocket::SSLSocket.
```cpp
ASecureSocket::SSLSocket ConnectedClient;
```

Before listenning for incoming SSL/TLS connections, you have to set the server's certificate and private key paths via
the proper methods :

```cpp
m_pSSLTCPServer->SetSSLCertFile(SSL_CERT_FILE);
m_pSSLTCPServer->SetSSLKeyFile(SSL_KEY_FILE);
```

You can also set CA file if you want. Otherwise, for now, passphrase must be included in the private key file.

To create SSL test files, you can use this command :

```Shell
openssl req -newkey rsa:2048 -new -nodes -x509 -days 3650 -keyout key.pem -out cert.pem
```

IMPORTANT: In the SSL/TLS server, ASecureSocket::SSLSocket objects must be disconnected with SSL/TLS server's
disconnect method to free used OpenSSL context and connection structures. Otherwise, you will have memory leaks.

## Thread Safety

Do not share ASocket or ASecureSocket objects across threads.

## Installation

You will need CMake to generate a makefile for the static library or to build the tests/code coverage program.

Also make sure you have Google Test installed and OpenSSL updated to the lastest version.

This tutorial will help you installing properly Google Test on Ubuntu: https://www.eriksmistad.no/getting-started-with-google-test-on-ubuntu/

If you work under Centos7 this tutorial may help you to install the latest version of OpenSSL : https://blacksaildivision.com/how-to-install-openssl-on-centos

The CMake script located in the tree will produce Makefiles for the creation of the static library and for the unit tests program.

To create a debug static library and a test binary, change directory to the one containing the first CMakeLists.txt and :

```Shell
mkdir build
cd build
cmake .. -DCMAKE_BUILD_TYPE:STRING=Debug
make
```

To create a release static library, just change "Debug" by "Release".

The library will be found under "build/[Debug|Release]/lib/libsocket.a" whereas the test program will be located in "build/[Debug|Release]/bin/test_socket"

To directly run the unit test binary, you must indicate the path of the INI conf file (see the section below)
```Shell
./[Debug|Release]/bin/test_socket /path_to_your_ini_file/conf.ini
```

## Run Unit Tests

[simpleini](https://github.com/brofield/simpleini) is used to gather unit tests parameters from
an INI configuration file. You need to fill that file with some parameters.
You can also disable some tests (SSL/TLS for instance) and indicate
parameters only for the enabled tests. A template of the INI file already exists under SocketTest/

e.g. to enable SSL/TLS tests :

```ini
[tests]
tcp-ssl=yes

[tcp-ssl]
server_port=4242
ca_file=CAfile.pem
ssl_cert_file=site.cert
ssl_key_file=privkey.pem
```

You can also generate an XML file of test results by adding --getst_output argument when calling the test program

```Shell
./[Debug|Release]/bin/test_socket /path_to_your_ini_file/conf.ini --gtest_output="xml:./TestSocket.xml"
```

An alternative way to compile and run unit tests :

```Shell
mkdir build
cd build
cmake .. -DCMAKE_BUILD_TYPE=Debug -DTEST_INI_FILE="full_or_relative_path_to_your_test_conf.ini"
make
make test
```

You may use a tool like https://github.com/adarmalik/gtest2html to convert your XML test result in an HTML file.

## Memory Leak Check

Visual Leak Detector has been used to check memory leaks with the Windows build (Visual Sutdio 2015)
You can download it here: https://vld.codeplex.com/

To perform a leak check with the Linux build, you can do so :

```Shell
valgrind --leak-check=full ./bin/Debug/test_socket /path_to_ini_file/conf.ini
```

## Code Coverage

The code coverage build doesn't use the static library but compiles and uses directly the 
socket API in the test program.

```Shell
mkdir build
cd build
cmake .. -DCMAKE_BUILD_TYPE=Coverage -DCOVERAGE_INI_FILE:STRING="full_path_to_your_test_conf.ini"
make
make coverage_socket
```

If everything is OK, the results will be found under ./SocketTest/coverage/index.html

Make sure you feed CMake with a full path to your test conf INI file, otherwise, the coverage test
will be useless.

Under Visual Studio, you can simply use OpenCppCoverage (https://opencppcoverage.codeplex.com/)

## CppCheck Compliancy

The C++ code of the Socket C++ API classes is Cppcheck compliant.

## Contribute
All contributions are highly appreciated. This includes updating documentation, cleaning and writing code and unit tests to increase code coverage and enhance tools.

Try to preserve the existing coding style (Hungarian notation, indentation etc...).
