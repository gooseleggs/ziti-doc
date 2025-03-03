# YubiKey by Yubico

[Yubico](https://www.yubico.com/) is a manufacturer of HSM deviceis. A popular line of HSM produced by Yubico is the
YubiKey. This quickstart guide will use specific device from Yubico - the [YubiKey 5
nfc](https://www.yubico.com/product/YubiKey-5-nfc).

## Overview

The [YubiKey 5 nfc](https://www.yubico.com/product/YubiKey-5-nfc) is a multi-purpose device with a few different
security-minded uses. One of the applications on the device is an application called PIV or "Personal Identity
Verification". PIV is a [standard published by
NIST](https://csrc.nist.gov/projects/piv/piv-standards-and-supporting-documentation) and describes the kinds of
credentials which make up the standard. PIV credentials have certificates and key pairs, pin numbers, biometrics
like fingerprints and pictures, and other unique identifiers. When put together into a PIV credential, it provides
the capability to implement multi-factor authentication for networks, applications and buildings.

In this quickstart you will see the commands and tools needed to use a [YubiKey 5
nfc](https://www.yubico.com/product/YubiKey-5-nfc) with a Ziti Network. This document is intended to serve as a
quickstart. That means limited context will be provided for each step. When appropriate there will be a small amount of
context or a comment included to aid in understanding of what is happening and why. Most if not all of these commands
are easily searched for using your search engine of choice.

> [!WARNING]
This quickstart intended audience is for more technically savvy indiviuals. You will need to be familar with the command
line interface of your operating system.

## Prerequistites

* [YubiKey 5 nfc](https://www.yubico.com/product/YubiKey-5-nfc) - clearly you'll need one in order to use this
  quickstart!
* [OpenSC](https://github.com/OpenSC/OpenSC/wiki) is installed and `pkcs11-tool` is either on the PATH or at a known
  location. Not required however this quickstart uses the `pkcs11-tool` to illustrate that the device is PKCS#11
  compliant. HSM manufacturers will generally provide a similar tool and often expand it's usage. See more below.
  [yubico-piv-tool](https://developers.yubico.com/yubico-piv-tool/) - YubiKey privides a similar tool to the
  `pkcs11-tool`. This tool is needed to be installed because it contains the pkcs#11 module (driver) for the HSM. As
  this is a tool specific to Yubico we've chosen to not use this in the following commands.
* Ensure the YubiKey is factory reset. To avoid any compliations with existing information in the YubiKey ensure the
  device is factory reset using the [YubiKey
  Manager](https://www.yubico.com/products/services-software/download/YubiKey-manager/).
* In order to successfully use the YubiKey the libraries provided by the `yubico-piv-tool` *MUST* either be on the path
  or in a common location that is known to the OS. On linux this is likely done by the YubiKey software installation but
  on Windows you'll need to take any additional actions highlighted in the Windows-specific sections.
* Linux Only: If you're using linux - you'll need to follow the [build
  instructions](https://developers.yubico.com/yubico-piv-tool/) provided by Yubico. Before you can do anything with the
  Yubikey you'll need to make sure the `libykcs11.so` exists on your system. When creating this quickstart the library
  was built to `./yubico-piv-tool-2.0.0/ykcs11/.libs/libykcs11.so`.
* [Ubuntu](https://ubuntu.com/) was used to test this guide. An attempt was also made with [linux
  mint](https://linuxmint.com/) however the attempt failed when trying to compile the Yubikey software and was aborted.
  It would likely have worked if enough effort was put into discovering why that linux variant had issues pulling and
  compiling the necessary software. If you see strange errors when following this guide and are not using Ubuntu it may
  be related.
* ziti and ziti-tunnel are both downloaded and on the path.

## Let's Use the YubiKey!

Here's the list of steps we'll accomplish in this quickstart:

* Establish a bunch of environment variables to make it easy to copy/paste the other commands. You'll want to *look at
  these environment variables*. They need to be setup properly. If you have problems with this guide it is almost
  certainly because you have an environment variable setup incorrectly. Double check them.
* Make a directory and generate a configuration files for Ziti
* Use the [Ziti CLI](https://netfoundry.jfrog.io/netfoundry/ziti-release/ziti/) to:
    * create two identities - one demonstrating an RSA key, one EC
    * enroll the identities
    * create a test service
    * create test router/service policies
* Use the pkcs11-tool provided by OpenSC to interact with the YubiKey to:
    * initialize the PIV app
    * create a key
* Use `ziti-tunnel` to enroll the identities using the YubiKey
* Use `ziti-tunnel` in proxy mode to verify things are working and traffic is flowing over the Ziti Network

> [!WARNING]
Do *NOT* use id `2` or `02` for any keys you add. Id `02` corresponds to the YubiKey's "Management Key". You will not be
able to write a key into this location. The error you will see will indicate: `CKR_USER_NOT_LOGGED_IN`.

### Establish Environment Variables

Open a command line and establish the following environment varibles. Note that for the YubiKey we do _not_ use id `02`
as it appears to be reserved by the YubiKey.  The default SOPIN and PIN are used as well. When using the YubiKey Manager
software the "SO PIN" corresponds to the "Management Key" while "pin" is the same both here and in the YubiKey Manager.

# [Linux/MacOS](#tab/env-var-linux)

    # the name of the ziti controller you're logging into
    export ZITI_CTRL=local-edge-controller
    # the location of the certificate(s) to use to validate the controller
    export ZITI_CTRL_CERT=/path/to/controller.cert

    export ZITI_USER=myUserName
    export ZITI_PWD=myPassword

    # a name for the configuration
    export HSM_NAME=yubikey_demo

    # path to the yubikey pkcs11 libraries
    export HSM_ROOT=/path/to/yubico-piv-tool-2.0.0
    export PKCS11_MODULE=${HSM_ROOT}/ykcs11/.libs/libykcs11.so

    # the id of the key - you probably want to leave these alone unless you know better
    export HSM_ID1=01
    export HSM_ID2=03

    # the pins used when accessing the pkcs11 api
    export HSM_SOPIN=010203040506070801020304050607080102030405060708
    export HSM_PIN=123456
    export RSA_ID=${HSM_NAME}${HSM_ID1}_rsa
    export EC_ID=${HSM_NAME}${HSM_ID2}_ec

    # location for the config files to be placed
    export HSM_DEST=${HSM_ROOT}/${HSM_NAME}
    export HSM_LABEL=${HSM_NAME}-label

    # make an alias for ease
    alias p='pkcs11-tool --module $PKCS11_MODULE'


# [Windows](#tab/env-var-windows)

> [!WARNING]
Ensure you use the correct dll. If you use an x86 dll with x64 binaries you'll get an error.

> [!WARNING]
With Windows - make sure you update the path to include the folder of libykcs11-1.dll as additional libraries are needed
by the pkcs11 driver!

    REM the name of the ziti controller you're logging into
    SET ZITI_CTRL=local-edge-controller
    REM the location of the certificate(s) to use to validate the controller
    SET ZITI_CTRL_CERT=c:\path\to\controller.cert
    
    SET ZITI_USER=myUserName
    SET ZITI_PWD=myPassword

    REM a name for the configuration
    SET HSM_NAME=yubikey_windemo

    REM the path to the root of the yubikey piv tool
    SET HSM_ROOT=c:\path\to\yubico-piv-tool-2.0.0

    REM path to the pkcs11 library
    SET PATH=%PATH%;%HSM_ROOT%\bin
    SET PKCS11_MODULE=%HSM_ROOT%\bin\libykcs11-1.dll

    REM the id of the key - you probably want to leave these alone unless you know better
    SET HSM_ID1=01
    SET HSM_ID2=03
    SET RSA_ID=%HSM_NAME%%HSM_ID1%_rsa
    SET EC_ID=%HSM_NAME%%HSM_ID2%_ec
    
    REM the pins used when accessing the pkcs11 api
    SET HSM_SOPIN=010203040506070801020304050607080102030405060708
    SET HSM_PIN=123456

    SET HSM_DEST=%HSM_ROOT%\%HSM_NAME%
    SET HSM_LABEL=%HSM_NAME%-label
    SET HSM_TOKENS_DIR=%HSM_DEST%\tokens\

    REM make an alias for ease
    doskey p="c:\Program Files\OpenSC Project\OpenSC\tools\pkcs11-tool.exe" --module %PKCS11_MODULE% $*

***

### Make Directories for Config Files

# [Linux/MacOS](#tab/dirs-and-config-linux)

    cd ${HSM_ROOT}

    rm -rf ${HSM_NAME}
    mkdir -p ${HSM_NAME}

    cd ${HSM_NAME}

# [Windows](#tab/dirs-and-config-windows)

    cd /d %HSM_ROOT%

    rmdir /s /q %HSM_NAME%
    mkdir %HSM_NAME%

    cd /d %HSM_NAME%

***

### Use the Ziti CLI

# [Linux/MacOS](#tab/ziti-cli-linux)

    ziti edge login $ZITI_CTRL:1280 -u $ZITI_USER -p $ZITI_PWD -c $ZITI_CTRL_CERT

    # create a new identity and output the jwt to a known location
    ziti edge create identity device "${RSA_ID}" -o "${HSM_DEST}/${RSA_ID}.jwt"

    # create a second new identity and output the jwt to a known location
    ziti edge create identity device "${EC_ID}" -o "${HSM_DEST}/${EC_ID}.jwt"

# [Windows](#tab/ziti-cli-windows)

    ziti edge login %ZITI_CTRL%:1280 -u %ZITI_USER% -p %ZITI_PWD% -c %ZITI_CTRL_CERT%

    REM create a new identity and output the jwt to a known location
    ziti edge create identity device "%RSA_ID%" -o "%HSM_DEST%\%RSA_ID%.jwt"

    REM create a second new identity and output the jwt to a known location
    ziti edge create identity device "%EC_ID%" -o "%HSM_DEST%\%EC_ID%.jwt"

***

### Use pkcs11-tool to Setup the YubiKey

# [Linux/MacOS](#tab/pkcs11-tool-linux)

    p --init-token --label "ziti-test-token" --so-pin $HSM_SOPIN

    # create a couple of keys - one rsa and one ec
    p -k --key-type rsa:2048 --usage-sign --usage-decrypt --login --id $HSM_ID1 --login-type so --so-pin $HSM_SOPIN --label defaultkey
    p -k --key-type EC:prime256v1 --usage-sign --usage-decrypt --login --id $HSM_ID2 --login-type so --so-pin $HSM_SOPIN --label defaultkey

# [Windows](#tab/pkcs11-tool-windows)

    p --init-token --label "ziti-test-token" --so-pin %HSM_SOPIN%

    REM create a couple of keys - one rsa and one ec
    p -k --key-type rsa:2048 --usage-sign --usage-decrypt --login --id %HSM_ID1% --login-type so --so-pin %HSM_SOPIN% --label defaultkey
    p -k --key-type EC:prime256v1 --usage-sign --usage-decrypt --login --id %HSM_ID2% --login-type so --so-pin %HSM_SOPIN% --label defaultkey

***

### Use ziti-tunnel to Enroll the Identities

# [Linux/MacOS](#tab/enroll-linux)

    ziti-tunnel enroll -j "${HSM_DEST}/${RSA_ID}.jwt" -k "pkcs11://${PKCS11_MODULE}?id=${HSM_ID1}&pin=${HSM_PIN}" -v
    ziti-tunnel enroll -j "${HSM_DEST}/${EC_ID}.jwt" -k "pkcs11://${PKCS11_MODULE}?id=${HSM_ID2}&pin=${HSM_PIN}" -v

# [Windows](#tab/enroll-windows)

    ziti-tunnel enroll -j "%HSM_DEST%\%RSA_ID%.jwt" -k "pkcs11://%PKCS11_MODULE%?id=%HSM_ID1%&pin=%HSM_PIN%" -v
    ziti-tunnel enroll -j "%HSM_DEST%\%EC_ID%.jwt" -k "pkcs11://%PKCS11_MODULE%?id=%HSM_ID2%&pin=%HSM_PIN%" -v

***

### Use ziti-tunnel to Verify Things Work

# [Linux/MacOS](#tab/start-tunnel-linux)

    # if you only have a single edge router this command will work without the need for copy/paste
    EDGE_ROUTER_ID=$(ziti edge list edge-routers | cut -d " " -f2)
    
    # IF the above command doesn't work - run this command and get the id from the first edge-router.
    # ziti edge list edge-routers

    # then use the id returned from the above command and put it into a variable for use in a momment
    # EDGE_ROUTER_ID={insert the 'id' from above - example: 64d4967b-5474-4f06-8548-5700ed7bfa80}

    # remove/recreate the config - here we'll be instructing the tunneler to listen on localhost and port 9000
    ziti edge delete config wttrconfig
    ziti edge create config wttrconfig ziti-tunneler-client.v1 "{ \"hostname\" : \"localhost\", \"port\" : 9000 }"
    
    # recreate the service with the EDGE_ROUTER_ID from above. Here we are adding a ziti service that will
    # send a request to wttr.in to retreive a weather forecast
    ziti edge delete service wttr.ziti
    ziti edge create service wttr.ziti "${EDGE_ROUTER_ID}" tcp://wttr.in:80 --configs wttrconfig

    # start one or both proxies
    ziti-tunnel proxy -i "${HSM_DEST}/${RSA_ID}.json" wttr.ziti:8000 -v &
    ziti-tunnel proxy -i "${HSM_DEST}/${EC_ID}.json" wttr.ziti:9000 -v &

    # use a browser - or curl to verify the ziti tunneler is listening locally and the traffic has flowed over the ziti network
    curl -H "Host: wttr.in" http://localhost:8000
    curl -H "Host: wttr.in" http://localhost:9000

# [Windows](#tab/start-tunnel-windows)

    REM these two commands can't be copied and pasted - you need to get the result of the first command and use it in the next
    REM run this command and get the id from the first edge-router.
    ziti edge list edge-routers
    
    REM use the id returned from the above command and put it into a variable for use in a momment
    SET EDGE_ROUTER_ID={insert the 'id' from above - example: 64d4967b-5474-4f06-8548-5700ed7bfa80}

    REM remove/recreate the config - here we'll be instructing the tunneler to listen on localhost and port 9000
    ziti edge delete config wttrconfig
    ziti edge create config wttrconfig ziti-tunneler-client.v1 "{ \"hostname\" : \"localhost\", \"port\" : 9000 }"
    
    REM recreate the service with the EDGE_ROUTER_ID from above. Here we are adding a ziti service that will
    REM send a request to wttr.in to retreive a weather forecast
    ziti edge delete service wttr.ziti
    ziti edge create service wttr.ziti "%EDGE_ROUTER_ID%" tcp://wttr.in:80 --configs wttrconfig

    REM start one or both proxies - use ctrl-break or ctrl-pause to terminate these processes 
    start /b ziti-tunnel proxy -i "%HSM_DEST%/%RSA_ID%.json" wttr.ziti:8000 -v
    start /b ziti-tunnel proxy -i "%HSM_DEST%/%EC_ID%.json" wttr.ziti:9000 -v

    REM use a browser - or curl to verify the ziti tunneler is listening locally and the traffic has flowed over the ziti network
    curl -H "Host: wttr.in" http://localhost:8000 > "%HSM_DEST%\example_%RSA_ID%.txt"
    curl -H "Host: wttr.in" http://localhost:9000 > "%HSM_DEST%\example_%EC_ID%.txt"

    REM set the codepage for the cmd prompt so that the output looks nice
    chcp 65001

    REM show the results in the console
    type "%HSM_DEST%\example_%RSA_ID%.txt"
    type "%HSM_DEST%\example_%EC_ID%.txt"

    REM ctrl-break or ctrl-pause to kill the tunnelers
    
***

### Putting It All Together

Above we've only shown the commands that need to run and not what the output of those commands would look like. Here
we'll see all the commands put together along with all the output from the commands. This section is long - you are
warned! Also note that this content is subject to change. If the output you see is not identical it's because the
software has changed since this information was captured. File an [issue](https://github.com/openziti/ziti-doc/issues)
if you'd like to see it updated.

# [Sample Output](#tab/hidden-linux)

The tabs to the right contain example output from running all the commands in sequence. If you want to see what the
output would likely look like click one of the tabs to the right. Reminder - it's a lot of commands and a lot of output!
:)

# [Linux/MacOS](#tab/verify-linux)

    -- coming soon--    
    cd@cd-ubuntuvm: # the name of the ziti controller you're logging into
    cd@cd-ubuntuvm: export ZITI_CTRL=local-edge-controller
    cd@cd-ubuntuvm: # the location of the certificate(s) to use to validate the controller
    cd@cd-ubuntuvm: export ZITI_CTRL_CERT=/path/to/controller.cert
    cd@cd-ubuntuvm:
    cd@cd-ubuntuvm: export ZITI_USER=myUserName
    cd@cd-ubuntuvm: export ZITI_PWD=myPassword
    cd@cd-ubuntuvm:
    cd@cd-ubuntuvm: # a name for the configuration
    cd@cd-ubuntuvm: export HSM_NAME=yubikey_demo
    cd@cd-ubuntuvm:
    cd@cd-ubuntuvm: # path to the yubikey pkcs11 libraries
    cd@cd-ubuntuvm: export HSM_ROOT=/path/to/yubico-piv-tool-2.0.0
    cd@cd-ubuntuvm: export PKCS11_MODULE=${HSM_ROOT}/ykcs11/.libs/libykcs11.so
    cd@cd-ubuntuvm:
    cd@cd-ubuntuvm: # the id of the key - you probably want to leave these alone unless you know better
    cd@cd-ubuntuvm: export HSM_ID1=01
    cd@cd-ubuntuvm: export HSM_ID2=03
    cd@cd-ubuntuvm:
    cd@cd-ubuntuvm: # the pins used when accessing the pkcs11 api
    cd@cd-ubuntuvm: export HSM_SOPIN=010203040506070801020304050607080102030405060708
    cd@cd-ubuntuvm: export HSM_PIN=123456
    cd@cd-ubuntuvm: export RSA_ID=${HSM_NAME}${HSM_ID1}_rsa
    cd@cd-ubuntuvm: export EC_ID=${HSM_NAME}${HSM_ID2}_ec
    cd@cd-ubuntuvm:
    cd@cd-ubuntuvm: # location for the config files to be placed
    cd@cd-ubuntuvm: export HSM_DEST=${HSM_ROOT}/${HSM_NAME}
    cd@cd-ubuntuvm: export HSM_LABEL=${HSM_NAME}-label
    cd@cd-ubuntuvm:
    cd@cd-ubuntuvm: # make an alias for ease
    cd@cd-ubuntuvm: alias p='pkcs11-tool --module $PKCS11_MODULE'
    cd@cd-ubuntuvm: cd ${HSM_ROOT}
    rm -rf ${HSM_NAME}
    cd@cd-ubuntuvm:
    cd@cd-ubuntuvm: rm -rf ${HSM_NAME}
    mkdir -p ${HSM_NAME}
    cd@cd-ubuntuvm: mkdir -p ${HSM_NAME}
    cd@cd-ubuntuvm:
    cd@cd-ubuntuvm: cd ${HSM_NAME}
    cd@cd-ubuntuvm: ziti edge login $ZITI_CTRL:1280 -u $ZITI_USER -p $ZITI_PWD -c $ZITI_CTRL_CERT
    # create a new identity and output the jwt to a known location
    ziti edge create identity device "${RSA_ID}" -o "${HSM_DEST}/${RSA_ID}.jwt"

    # create a second new identity and output the jwt to a known location
    ziti edge create identity device "${EC_ID}" -o "${HSM_DEST}/${EC_ID}.jwt"Token: 7083e601-cc2f-4636-94c0-959174e76264
    cd@cd-ubuntuvm:
    cd@cd-ubuntuvm: # create a new identity and output the jwt to a known location
    cd@cd-ubuntuvm: ziti edge create identity device "${RSA_ID}" -o "${HSM_DEST}/${RSA_ID}.jwt"
    76877717-9bce-4f34-ae7a-2ce313b8267a
    Enrollment expires at 2020-02-24T14:56:19.961322754Z
    cd@cd-ubuntuvm:
    cd@cd-ubuntuvm: # create a second new identity and output the jwt to a known location
    cd@cd-ubuntuvm: ziti edge create identity device "${EC_ID}" -o "${HSM_DEST}/${EC_ID}.jwt"
    bdd18e58-e474-4ed3-98eb-c68ed96377be
    Enrollment expires at 2020-02-24T14:56:21.673599433Z
    cd@cd-ubuntuvm: p --init-token --label "ziti-test-token" --so-pin $HSM_SOPIN
    p -k --key-type rsa:2048 --usage-sign --usage-decrypt --login --id $HSM_ID1 --login-type so --so-pin $HSM_SOPIN --label defaultkey
    p -k --key-type EC:prime256v1 --usage-sign --usage-decrypt --login --id $HSM_ID2 --login-type so --so-pin $HSM_SOPIN --label defaultkeyUsing slot 0 with a present token (0x0)
    Token successfully initialized
    cd@cd-ubuntuvm:
    cd@cd-ubuntuvm: # create a couple of keys - one rsa and one ec
    cd@cd-ubuntuvm: p -k --key-type rsa:2048 --usage-sign --usage-decrypt --login --id $HSM_ID1 --login-type so --so-pin $HSM_SOPIN --label defaultkey
    Using slot 0 with a present token (0x0)
    Key pair generated:
    Private Key Object; RSA
      label:      Private key for PIV Authentication
      ID:         01
      Usage:      decrypt, sign
    Public Key Object; RSA 2048 bits
      label:      Public key for PIV Authentication
      ID:         01
      Usage:      encrypt, verify
    cd@cd-ubuntuvm: p -k --key-type EC:prime256v1 --usage-sign --usage-decrypt --login --id $HSM_ID2 --login-type so --so-pin $HSM_SOPIN --label defaultkey
    Using slot 0 with a present token (0x0)
    Key pair generated:
    Private Key Object; EC
      label:      Private key for Key Management
      ID:         03
      Usage:      decrypt, sign
    Public Key Object; EC  EC_POINT 256 bits
      EC_POINT:   044104ee01d2c979d4050a2f6e357b2cbe518c3adf816cb08c5b3116318b9cc92328213bb923ea1a3ce9bc46093ffec03912b1e7eda679287fd65419e47d337ca0bb8c
      EC_PARAMS:  06082a8648ce3d030107
      label:      Public key for Key Management
      ID:         03
      Usage:      encrypt, verify
    cd@cd-ubuntuvm: ziti-tunnel enroll -j "${HSM_DEST}/${RSA_ID}.jwt" -k "pkcs11://${PKCS11_MODULE}?id=${HSM_ID1}&pin=${HSM_PIN}" -v
    DEBU[0000] jwt to parse: eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJlbSI6Im90dCIsImV4cCI6MTU4MjU1NjE3OSwiaXNzIjoiaHR0cHM6Ly9sb2NhbC1lZGdlLWNvbnRyb2xsZXI6MTI4MCIsImp0aSI6Ijk5NjVlZmQzLTliMjItNDk3ZS04NDdiLTM1NWI3MjE1NTZjMyIsInN1YiI6Ijc2ODc3NzE3LTliY2UtNGYzNC1hZTdhLTJjZTMxM2I4MjY3YSJ9.ohrYUZQK5xum-70AYARu8hYYxhvA9Cm2rEln6jVp8VDvMLRuIQi9bXNO1e2NQtkZLGzo3akg4Brx_QxM-9SIRKdBDYzgaN_otpV6y9XAQjVlarVvtA1bBA9OfQU1SyVGNkkaA2l660cY5gfsGb5TqBQt6wQ7KvMfzZvPhXFdQAMoXW-O2OaRh3jJGPne29-Hoz_thObv5PMTVlCGpzO6tKKfPnoV9p8HchjLHMMybEZO_xITyY_6UmkOPpeSJHdjWxJdkXu0xT8wC9PzAXK2xYYwgmW4yA8ygz2RJP6gXYd5maD-f1jgh87ELeN5dG2ksuy5Fh3_kyFAGg1X7dSnfmXSA0fWH-YPK54aSJe2fcCIt41O9pnqwepPT_7nCB4wSxVhuIugUBh_wYeJzXKI8tVnyv8nLyWhYqMwKRqTtiJM7EtxpBUPD6E35x3d4bw_fkVJ5z36NOIaGRmqhltPIsYMaFDwIE-nUU714Ra-cf8xN9lHuNoi_4Ehf8sR2YwLTsk7mJ9TKZkhLIj2KgwjL2KNCoM9n6MQtJJMq3Ae6H_xVxi2-DA_9zkbusdNi2j11OoIAtBGJQ4ONC5ekRsp_bTbb9GAxF-fO8oAQeFUXWyCpywRV8gT3G4huAlcofZTDnMtcyVvTJbvs8zYRtBM9jCKpUOmHuzb4FefgU-6kP0
    INFO[0000] using engine : pkcs11
    DEBU[0000] loading key                                   context=pkcs11 url="pkcs11:///path/to/yubico-piv-tool-2.0.0/ykcs11/.libs/libykcs11.so?id=01&pin=123456"
    INFO[0000] using driver: /path/to/yubico-piv-tool-2.0.0/ykcs11/.libs/libykcs11.so  context=pkcs11
    WARN[0000] slot not specified, using first slot reported by the driver (0)  context=pkcs11
    DEBU[0001] found signing mechanism                       context=pkcs11 sign mechanism=0
    DEBU[0001] no cas provided in caPool. using system provided cas
    DEBU[0001] fetching certificates from server
    DEBU[0001] loading key                                   context=pkcs11 url="pkcs11:///path/to/yubico-piv-tool-2.0.0/ykcs11/.libs/libykcs11.so?id=01&pin=123456"
    INFO[0001] using driver: /path/to/yubico-piv-tool-2.0.0/ykcs11/.libs/libykcs11.so  context=pkcs11
    WARN[0002] slot not specified, using first slot reported by the driver (0)  context=pkcs11
    DEBU[0002] found signing mechanism                       context=pkcs11 sign mechanism=0
    enrolled successfully. identity file written to: /path/to/yubico-piv-tool-2.0.0/yubikey_demo/yubikey_demo01_rsa.jsoncd@cd-ubuntuvm: ziti-tunnel enroll -j "${HSM_DEST}/${EC_ID}.jwt" -k "pkcs11://${PKCS11_MODULE}?id=${HSM_ID2}&pin=${HSM_PIN}" -v
    DEBU[0000] jwt to parse: eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJlbSI6Im90dCIsImV4cCI6MTU4MjU1NjE4MSwiaXNzIjoiaHR0cHM6Ly9sb2NhbC1lZGdlLWNvbnRyb2xsZXI6MTI4MCIsImp0aSI6IjQwYTg1NmMxLWFiZmItNDQ4ZS1hN2E3LWE3YjZkMzVmYWU4OSIsInN1YiI6ImJkZDE4ZTU4LWU0NzQtNGVkMy05OGViLWM2OGVkOTYzNzdiZSJ9.o5hGXqZXWUuj-zkg-Xd335AbZUe9zNtvQmhE4cNFZNw40eRXNTBeFntLWjWv7-41AtsHbPwmQgWQfDdHS8JuLbhLS-8ZNSL3ZThBwnTiKtgYQvg_aArcu1FbPiw3jdiiHqXu0JoIToCEFX4kMjAJIZlzRi05J0d5w6Wvvjy3V7SqT_8viLla6l3PXsa1_xnHCc89HebuKB_qYds-XO2JvXTOFP0A3dSe6UgierjrTPvRxLMbf8TIfSOFtAc6bbSy17H8S0XrNiL-1bA9Ja9KKpB_ynGTygpM3h1vXda0niJN-_TpPdTX9kFEXOn-DpvAGKW8X6ggPlnpF7Rmfx9AEarA9ft-nRjx4Z4vyoAlukSc7LfnjAqTg9CCsN0-hKB1d1PnAOuWfHP5IZ2w1zibcUvdRQMoW2Twk_P75MP8rjr19VtZu5bDweZjP10-eHt3_QYeujaylBfb7VkJuRtSyh6IeWAb-IlIaJx7hzjGqssqNPn0vqVdf43Sc0h-PJcVTOkNPpcBIbxHPL5IVyujCWUSj4NUSgVh72uQIKjxC-WrkT8i-hEnGbJRYUL2NDcS190yt21AJAAkDPBTiFA_LR0aR7lEPzQV-lsjJzjmta7V460OKATsU8oo82_S8dVv8oYndk2kRboVsCWvR7_6H95CEnmRjAfI0Emxwkg1raA
    INFO[0000] using engine : pkcs11
    DEBU[0000] loading key                                   context=pkcs11 url="pkcs11:///path/to/yubico-piv-tool-2.0.0/ykcs11/.libs/libykcs11.so?id=03&pin=123456"
    INFO[0000] using driver: /path/to/yubico-piv-tool-2.0.0/ykcs11/.libs/libykcs11.so  context=pkcs11
    WARN[0000] slot not specified, using first slot reported by the driver (0)  context=pkcs11
    DEBU[0001] found signing mechanism                       context=pkcs11 sign mechanism=0
    DEBU[0001] EC oid[1.2.840.10045.3.1.7], rest: [], err: <nil>  context=pkcs11
    DEBU[0001] no cas provided in caPool. using system provided cas
    DEBU[0001] fetching certificates from server
    DEBU[0001] loading key                                   context=pkcs11 url="pkcs11:///path/to/yubico-piv-tool-2.0.0/ykcs11/.libs/libykcs11.so?id=03&pin=123456"
    INFO[0001] using driver: /path/to/yubico-piv-tool-2.0.0/ykcs11/.libs/libykcs11.so  context=pkcs11
    WARN[0001] slot not specified, using first slot reported by the driver (0)  context=pkcs11
    DEBU[0001] found signing mechanism                       context=pkcs11 sign mechanism=0
    DEBU[0001] EC oid[1.2.840.10045.3.1.7], rest: [], err: <nil>  context=pkcs11
    enrolled successfully. identity file written to: /path/to/yubico-piv-tool-2.0.0/yubikey_demo/yubikey_demo03_ec.jsoncd@cd-ubuntuvm:
    cd@cd-ubuntuvm: # if you only have a single edge router this command will work without the need for copy/paste
    cd@cd-ubuntuvm: EDGE_ROUTER_ID=$(ziti edge list edge-routers | cut -d " " -f2)
    # IF the above command doesn't work - run this command and get the id from the first edge-router.
    # ziti edge list edge-routers

    # then use the id returned from the above command and put it into a variable for use in a momment
    # EDGE_ROUTER_ID={insert the 'id' from above - example: 64d4967b-5474-4f06-8548-5700ed7bfa80}

    # remove/recreate the config - here we'll be instructing the tunneler to listen on localhost and port 9000
    ziti edge delete config wttrconfig
    ziti edge create config wttrconfig ziti-tunneler-client.v1 "{ \"hostname\" : \"localhost\", \"port\" : 9000 }"

    # recreate the service with the EDGE_ROUTER_ID from above. Here we are adding a ziti service that will
    cd@cd-ubuntuvm:
    cd@cd-ubuntuvm: # IF the above command doesn't work - run this command and get the id from the first edge-router.
    cd@cd-ubuntuvm: # ziti edge list edge-routers
    cd@cd-ubuntuvm:
    cd@cd-ubuntuvm: # then use the id returned from the above command and put it into a variable for use in a momment
    cd@cd-ubuntuvm: # EDGE_ROUTER_ID={insert the 'id' from above - example: 64d4967b-5474-4f06-8548-5700ed7bfa80}
    cd@cd-ubuntuvm:
    cd@cd-ubuntuvm: # remove/recreate the config - here we'll be instructing the tunneler to listen on localhost and port 9000
    cd@cd-ubuntuvm: ziti edge delete config wttrconfig
    ziti edge delete service wttr.ziti
    ziti edge create service wttr.ziti "${EDGE_ROUTER_ID}" tcp://wttr.in:80 --configs wttrconfig

    # start one or both proxies
    ziti-tunnel proxy -i "${HSM_DEST}/${RSA_ID}.json" wttr.ziti:8000 -v &
    ziti-tunnel proxy -i "${HSM_DEST}/${EC_ID}.json" wttr.ziti:9000 -v &
    cd@cd-ubuntuvm: ziti edge create config wttrconfig ziti-tunneler-client.v1 "{ \"hostname\" : \"localhost\", \"port\" : 9000 }"
    4ff768fa-0e7e-4b4d-ab4e-5c5aec5bd28d
    cd@cd-ubuntuvm:
    cd@cd-ubuntuvm: # recreate the service with the EDGE_ROUTER_ID from above. Here we are adding a ziti service that will
    cd@cd-ubuntuvm: # send a request to wttr.in to retreive a weather forecast
    cd@cd-ubuntuvm: ziti edge delete service wttr.ziti
    cd@cd-ubuntuvm: ziti edge create service wttr.ziti "${EDGE_ROUTER_ID}" tcp://wttr.in:80 --configs wttrconfig
    1827c649-9ba6-4f5a-b122-a8cb9d4e4893
    cd@cd-ubuntuvm:
    cd@cd-ubuntuvm: # start one or both proxies
    cd@cd-ubuntuvm: ziti-tunnel proxy -i "${HSM_DEST}/${RSA_ID}.json" wttr.ziti:8000 -v &
    [1] 5969
    cd@cd-ubuntuvm: ziti-tunnel proxy -i "${HSM_DEST}/${EC_ID}.json" wttr.ziti:9000 -v &
    [2] 5970
    cd@cd-ubuntuvm: [   0.001]    INFO github.com/netfoundry/ziti-edge/tunnel/intercept/proxy.(*interceptor).Start: starting proxy interceptor
    [   0.001]    INFO github.com/netfoundry/ziti-edge/tunnel/intercept/proxy.(*interceptor).Start: starting proxy interceptor
    [   0.002]   DEBUG github.com/netfoundry/ziti-foundation/identity/engines/pkcs11.(*engine).LoadKey [pkcs11]: {url=[pkcs11:///path/to/yubico-piv-tool-2.0.0/ykcs11/.libs/libykcs11.so?id=03&pin=123456]} loading key
    [   0.002]    INFO github.com/netfoundry/ziti-foundation/identity/engines/pkcs11.(*engine).LoadKey [pkcs11]: using driver: /path/to/yubico-piv-tool-2.0.0/ykcs11/.libs/libykcs11.so
    [   0.004]   DEBUG github.com/netfoundry/ziti-foundation/identity/engines/pkcs11.(*engine).LoadKey [pkcs11]: {url=[pkcs11:///path/to/yubico-piv-tool-2.0.0/ykcs11/.libs/libykcs11.so?id=01&pin=123456]} loading key
    [   0.004]    INFO github.com/netfoundry/ziti-foundation/identity/engines/pkcs11.(*engine).LoadKey [pkcs11]: using driver: /path/to/yubico-piv-tool-2.0.0/ykcs11/.libs/libykcs11.so
    [   0.085] WARNING github.com/netfoundry/ziti-foundation/identity/engines/pkcs11.(*engine).LoadKey [pkcs11]: slot not specified, using first slot reported by the driver (0)
    [   1.793]   DEBUG github.com/netfoundry/ziti-foundation/identity/engines/pkcs11.(*engine).LoadKey [pkcs11]: {sign mechanism=[0]} found signing mechanism
    [   1.793]   DEBUG github.com/netfoundry/ziti-foundation/identity/engines/pkcs11.loadECDSApub [pkcs11]: EC oid[1.2.840.10045.3.1.7], rest: [], err: <nil>
    [   1.793]    INFO github.com/netfoundry/ziti-sdk-golang/ziti.(*contextImpl).Authenticate: attempting to authenticate
    [   1.912]   DEBUG github.com/netfoundry/ziti-sdk-golang/ziti.(*contextImpl).Authenticate: {token=[426a9b25-5f9c-4d3d-9319-30096cb584fc] id=[30b4cc4a-1e71-4e34-971c-bd5bead0477b]} Got api session: {30b4cc4a-1e71-4e34-971c-bd5bead0477b 426a9b25-5f9c-4d3d-9319-30096cb584fc}
    [   1.912]   DEBUG github.com/netfoundry/ziti-sdk-golang/ziti.(*contextImpl).getServices: using api session token 426a9b25-5f9c-4d3d-9319-30096cb584fc
    [   1.956]   DEBUG github.com/netfoundry/ziti-sdk-golang/ziti.(*contextImpl).getServices: using api session token 426a9b25-5f9c-4d3d-9319-30096cb584fc
    [   2.000]    INFO github.com/netfoundry/ziti-edge/tunnel/intercept.updateServices: starting tunnel for newly available service wttr.ziti
    [   2.000]   DEBUG github.com/netfoundry/ziti-sdk-golang/ziti.(*contextImpl).getSession: requesting session from https://local-edge-controller:1280/sessions
    [   2.000]   DEBUG github.com/netfoundry/ziti-sdk-golang/ziti.(*contextImpl).getSession: {service_id=[1827c649-9ba6-4f5a-b122-a8cb9d4e4893]} requesting session
    [   2.033]   DEBUG github.com/netfoundry/ziti-edge/tunnel/intercept/proxy.interceptor.Intercept: {id=[ee783e84-c888-4912-ba1b-b79334590d3e] service=[wttr.ziti]} acquired network session
    [   2.034]    INFO github.com/netfoundry/ziti-edge/tunnel/intercept.updateServices: starting tunnel for newly available service netcat7256
    [   2.034]   DEBUG github.com/netfoundry/ziti-edge/tunnel/intercept/proxy.interceptor.Intercept: {service=[netcat7256]} service netcat7256 was not specified at initialization. not intercepting
    [   2.034]    INFO github.com/netfoundry/ziti-edge/tunnel/intercept/proxy.(*interceptor).handleTCP: {service=[wttr.ziti] addr=[0.0.0.0:9000]} service is listening
    [   2.038] WARNING github.com/netfoundry/ziti-foundation/identity/engines/pkcs11.(*engine).LoadKey [pkcs11]: slot not specified, using first slot reported by the driver (0)
    [   3.863]   DEBUG github.com/netfoundry/ziti-foundation/identity/engines/pkcs11.(*engine).LoadKey [pkcs11]: {sign mechanism=[0]} found signing mechanism
    [   3.864]    INFO github.com/netfoundry/ziti-sdk-golang/ziti.(*contextImpl).Authenticate: attempting to authenticate
    [   4.099]   DEBUG github.com/netfoundry/ziti-sdk-golang/ziti.(*contextImpl).Authenticate: {token=[a2d721d4-4b4a-466f-b310-b7eabd539e75] id=[deaf5b97-e467-431a-9f16-4546e4ef42fc]} Got api session: {deaf5b97-e467-431a-9f16-4546e4ef42fc a2d721d4-4b4a-466f-b310-b7eabd539e75}
    [   4.099]   DEBUG github.com/netfoundry/ziti-sdk-golang/ziti.(*contextImpl).getServices: using api session token a2d721d4-4b4a-466f-b310-b7eabd539e75
    [   4.146]   DEBUG github.com/netfoundry/ziti-sdk-golang/ziti.(*contextImpl).getServices: using api session token a2d721d4-4b4a-466f-b310-b7eabd539e75
    [   4.189]    INFO github.com/netfoundry/ziti-edge/tunnel/intercept.updateServices: starting tunnel for newly available service netcat7256
    [   4.189]   DEBUG github.com/netfoundry/ziti-edge/tunnel/intercept/proxy.interceptor.Intercept: {service=[netcat7256]} service netcat7256 was not specified at initialization. not intercepting
    [   4.189]    INFO github.com/netfoundry/ziti-edge/tunnel/intercept.updateServices: starting tunnel for newly available service wttr.ziti
    [   4.189]   DEBUG github.com/netfoundry/ziti-sdk-golang/ziti.(*contextImpl).getSession: requesting session from https://local-edge-controller:1280/sessions
    [   4.189]   DEBUG github.com/netfoundry/ziti-sdk-golang/ziti.(*contextImpl).getSession: {service_id=[1827c649-9ba6-4f5a-b122-a8cb9d4e4893]} requesting session
    [   4.222]   DEBUG github.com/netfoundry/ziti-edge/tunnel/intercept/proxy.interceptor.Intercept: {service=[wttr.ziti] id=[a53212ef-404e-4528-9d36-02852db9c595]} acquired network session
    [   4.222]    INFO github.com/netfoundry/ziti-edge/tunnel/intercept/proxy.(*interceptor).handleTCP: {service=[wttr.ziti] addr=[0.0.0.0:8000]} service is listening

    cd@cd-ubuntuvm:
    cd@cd-ubuntuvm: # use a browser - or curl to verify the ziti tunneler is listening locally and the traffic has flowed over the ziti network
    cd@cd-ubuntuvm: curl -H "Host: wttr.in" http://localhost:8000
    [  12.111]   DEBUG github.com/netfoundry/ziti-foundation/channel2.(*classicDialer).Create [tls:local-edge-router:3022]: started
    [  12.315]   DEBUG github.com/netfoundry/ziti-foundation/transport/tls.Dial: server provided [2] certificates
    [  12.315]   DEBUG github.com/netfoundry/ziti-foundation/channel2.(*classicDialer).sendHello [u{classic}->i{}]: started
    [  12.316]   DEBUG github.com/netfoundry/ziti-foundation/channel2.(*classicDialer).sendHello [u{classic}->i{}]: exited
    [  12.316]   DEBUG github.com/netfoundry/ziti-foundation/channel2.(*classicDialer).Create [tls:local-edge-router:3022]: exited
    [  12.316]   DEBUG github.com/netfoundry/ziti-sdk-golang/ziti/edge.(*muxAddSinkEvent).Handle: {connId=[1]} Added sink to mux. Current sink count: 1
    [  12.316]   DEBUG github.com/netfoundry/ziti-sdk-golang/ziti/edge.(*MsgMux).AddMsgSink: {connId=[1]} added to msg mux
    [  12.316]   DEBUG github.com/netfoundry/ziti-foundation/channel2.(*channelImpl).txer [ch{ziti-sdk}->u{classic}->i{oEzq}]: started
    [  12.316]   DEBUG github.com/netfoundry/ziti-foundation/channel2.(*channelImpl).rxer [ch{ziti-sdk}->u{classic}->i{oEzq}]: started
    [  12.557]   DEBUG github.com/netfoundry/ziti-foundation/channel2.(*channelImpl).rxer [ch{ziti-sdk}->u{classic}->i{oEzq}]: waiter found for message. type [60784], sequence [1], replyFor [1]
    [  12.557]   DEBUG github.com/netfoundry/ziti-sdk-golang/ziti/internal/edge_impl.(*edgeConn).Connect: {connId=[1]} connected
    [  12.557]    INFO github.com/netfoundry/ziti-edge/tunnel.Run: {src-remote=[127.0.0.1:41838] src-local=[127.0.0.1:8000] dst-local=[:1] dst-remote=[wttr.ziti]} tunnel started
    [  12.558]   DEBUG github.com/netfoundry/ziti-sdk-golang/ziti/edge.(*MsgChannel).WriteTraced: {connId=[1] type=[EdgeDataType] chSeq=[-1] edgeSeq=[1]} writing 71 bytes
    [  12.558]   DEBUG github.com/netfoundry/ziti-sdk-golang/ziti/internal/edge_impl.(*edgeConn).Read: {connId=[1]} read buffer = 32768 bytes
    [  12.735]   DEBUG github.com/netfoundry/ziti-sdk-golang/ziti/edge.(*MsgEvent).Handle: {seq=[1] connId=[1]} handling received EdgeDataType
    [  12.735]   DEBUG github.com/netfoundry/ziti-sdk-golang/ziti/internal/edge_impl.(*edgeConn).Read: {connId=[1]} got buffer from queue 8852 bytes
    [  12.735]   DEBUG github.com/netfoundry/ziti-sdk-golang/ziti/internal/edge_impl.(*edgeConn).Read: {connId=[1]} read buffer = 32768 bytes
    Weather report: Laurelton, United States

                  Overcast
          .--.     14..24 °F
      .-(    ).   ↘ 13 mph
      (___.__)__)  9 mi
                  0.0 in
                                                          ┌─────────────┐
    ┌──────────────────────────────┬───────────────────────┤  Fri 14 Feb ├───────────────────────┬──────────────────────────────┐
    │            Morning           │             Noon      └──────┬──────┘     Evening           │             Night            │
    ├──────────────────────────────┼──────────────────────────────┼──────────────────────────────┼──────────────────────────────┤
    │    \  /       Partly cloudy  │     \   /     Sunny          │     \   /     Clear          │     \   /     Clear          │
    │  _ /"".-.     6..19 °F       │      .-.      8..21 °F       │      .-.      8..19 °F       │      .-.      3..14 °F       │
    │    \_(   ).   ↘ 11-13 mph    │   ― (   ) ―   ↘ 11-13 mph    │   ― (   ) ―   ↗ 11-13 mph    │   ― (   ) ―   ↓ 8-13 mph     │
    │    /(___(__)  5 mi           │      `-’      6 mi           │      `-’      6 mi           │      `-’      6 mi           │
    │               0.0 in | 0%    │     /   \     0.0 in | 0%    │     /   \     0.0 in | 0%    │     /   \     0.0 in | 0%    │
    └──────────────────────────────┴──────────────────────────────┴──────────────────────────────┴──────────────────────────────┘
                                                          ┌─────────────┐
    ┌──────────────────────────────┬───────────────────────┤  Sat 15 Feb ├───────────────────────┬──────────────────────────────┐
    │            Morning           │             Noon      └──────┬──────┘     Evening           │             Night            │
    ├──────────────────────────────┼──────────────────────────────┼──────────────────────────────┼──────────────────────────────┤
    │     \   /     Sunny          │    \  /       Partly cloudy  │    \  /       Partly cloudy  │    \  /       Partly cloudy  │
    │      .-.      14..17 °F      │  _ /"".-.     17..24 °F      │  _ /"".-.     21..30 °F      │  _ /"".-.     17..24 °F      │
    │   ― (   ) ―   ↖ 3 mph        │    \_(   ).   ↑ 6-8 mph      │    \_(   ).   ↑ 9-14 mph     │    \_(   ).   ↑ 8-14 mph     │
    │      `-’      6 mi           │    /(___(__)  6 mi           │    /(___(__)  6 mi           │    /(___(__)  6 mi           │
    │     /   \     0.0 in | 0%    │               0.0 in | 0%    │               0.0 in | 0%    │               0.0 in | 0%    │
    └──────────────────────────────┴──────────────────────────────┴──────────────────────────────┴──────────────────────────────┘
                                                          ┌─────────────┐
    ┌──────────────────────────────┬───────────────────────┤  Sun 16 Feb ├───────────────────────┬──────────────────────────────┐
    │            Morning           │             Noon      └──────┬──────┘     Evening           │             Night            │
    ├──────────────────────────────┼──────────────────────────────┼──────────────────────────────┼──────────────────────────────┤
    │    \  /       Partly cloudy  │    \  /       Partly cloudy  │               Overcast       │               Overcast       │
    │  _ /"".-.     24..30 °F      │  _ /"".-.     35..39 °F      │      .--.     37..39 °F      │      .--.     32..33 °F      │
    │    \_(   ).   ↗ 4-7 mph      │    \_(   ).   ↗ 7-8 mph      │   .-(    ).   → 6-9 mph      │   .-(    ).   ↗ 3-8 mph      │
    │    /(___(__)  6 mi           │    /(___(__)  6 mi           │  (___.__)__)  6 mi           │  (___.__)__)  6 mi           │
    │               0.0 in | 0%    │               0.0 in | 0%    │               0.0 in | 0%    │               0.0 in | 0%    │
    └──────────────────────────────┴──────────────────────────────┴──────────────────────────────┴──────────────────────────────┘

    Follow @igor_chubin for wttr.in updates
    [  12.735]    INFO github.com/netfoundry/ziti-edge/tunnel.myCopy: {dst-remote=[wttr.ziti] src-remote=[127.0.0.1:41838] src-local=[127.0.0.1:8000] dst-local=[:1]} stopping pipe
    [  12.735]   DEBUG github.com/netfoundry/ziti-sdk-golang/ziti/internal/edge_impl.(*edgeConn).Close: {connId=[1]} close: begin
    [  12.735]   DEBUG github.com/netfoundry/ziti-sdk-golang/ziti/edge.(*muxRemoveSinkEvent).Handle: {connId=[1]} Removed sink from mux. Current sink count: 0
    [  12.736]   DEBUG github.com/netfoundry/ziti-sdk-golang/ziti/edge.(*MsgMux).RemoveMsgSinkById: {connId=[1]} removed from msg mux
    [  12.736]   DEBUG github.com/netfoundry/ziti-sdk-golang/ziti/internal/edge_impl.(*edgeConn).Close: {connId=[1]} close: end
    cd@cd-ubuntuvm: [  13.736]   DEBUG github.com/netfoundry/ziti-sdk-golang/ziti/internal/edge_impl.(*edgeConn).Read: {connId=[1]} sequencer closed, closing connection
    [  13.736]   DEBUG github.com/netfoundry/ziti-sdk-golang/ziti/internal/edge_impl.(*edgeConn).Read: {connId=[1]} return EOF from closing/closed connection
    [  13.736]    INFO github.com/netfoundry/ziti-edge/tunnel.myCopy: {dst-remote=[127.0.0.1:41838] src-remote=[wttr.ziti] src-local=[:1] dst-local=[127.0.0.1:8000]} stopping pipe
    [  13.736]    INFO github.com/netfoundry/ziti-edge/tunnel.Run: {src-remote=[127.0.0.1:41838] src-local=[127.0.0.1:8000] dst-local=[:1] dst-remote=[wttr.ziti]} tunnel closed: 71 bytes sent; 8852 bytes received

    cd@cd-ubuntuvm:
    cd@cd-ubuntuvm: curl -H "Host: wttr.in" http://localhost:9000
    [  17.991]   DEBUG github.com/netfoundry/ziti-foundation/channel2.(*classicDialer).Create [tls:local-edge-router:3022]: started
    [  18.099]   DEBUG github.com/netfoundry/ziti-foundation/transport/tls.Dial: server provided [2] certificates
    [  18.099]   DEBUG github.com/netfoundry/ziti-foundation/channel2.(*classicDialer).sendHello [u{classic}->i{}]: started
    [  18.100]   DEBUG github.com/netfoundry/ziti-foundation/channel2.(*classicDialer).sendHello [u{classic}->i{}]: exited
    [  18.100]   DEBUG github.com/netfoundry/ziti-foundation/channel2.(*classicDialer).Create [tls:local-edge-router:3022]: exited
    [  18.100]   DEBUG github.com/netfoundry/ziti-sdk-golang/ziti/edge.(*muxAddSinkEvent).Handle: {connId=[1]} Added sink to mux. Current sink count: 1
    [  18.100]   DEBUG github.com/netfoundry/ziti-sdk-golang/ziti/edge.(*MsgMux).AddMsgSink: {connId=[1]} added to msg mux
    [  18.100]   DEBUG github.com/netfoundry/ziti-foundation/channel2.(*channelImpl).rxer [ch{ziti-sdk}->u{classic}->i{Jn2J}]: started
    [  18.100]   DEBUG github.com/netfoundry/ziti-foundation/channel2.(*channelImpl).txer [ch{ziti-sdk}->u{classic}->i{Jn2J}]: started
    [  18.211]   DEBUG github.com/netfoundry/ziti-foundation/channel2.(*channelImpl).rxer [ch{ziti-sdk}->u{classic}->i{Jn2J}]: waiter found for message. type [60784], sequence [1], replyFor [1]
    [  18.211]   DEBUG github.com/netfoundry/ziti-sdk-golang/ziti/internal/edge_impl.(*edgeConn).Connect: {connId=[1]} connected
    [  18.211]    INFO github.com/netfoundry/ziti-edge/tunnel.Run: {dst-remote=[wttr.ziti] src-remote=[127.0.0.1:57980] src-local=[127.0.0.1:9000] dst-local=[:1]} tunnel started
    [  18.211]   DEBUG github.com/netfoundry/ziti-sdk-golang/ziti/edge.(*MsgChannel).WriteTraced: {connId=[1] type=[EdgeDataType] chSeq=[-1] edgeSeq=[1]} writing 71 bytes
    [  18.211]   DEBUG github.com/netfoundry/ziti-sdk-golang/ziti/internal/edge_impl.(*edgeConn).Read: {connId=[1]} read buffer = 32768 bytes
    [  18.379]   DEBUG github.com/netfoundry/ziti-sdk-golang/ziti/edge.(*MsgEvent).Handle: {seq=[1] connId=[1]} handling received EdgeDataType
    [  18.379]   DEBUG github.com/netfoundry/ziti-sdk-golang/ziti/internal/edge_impl.(*edgeConn).Read: {connId=[1]} got buffer from queue 8852 bytes
    [  18.379]   DEBUG github.com/netfoundry/ziti-sdk-golang/ziti/internal/edge_impl.(*edgeConn).Read: {connId=[1]} read buffer = 32768 bytes
    Weather report: Laurelton, United States

                  Overcast
          .--.     14..24 °F
      .-(    ).   ↘ 13 mph
      (___.__)__)  9 mi
                  0.0 in
                                                          ┌─────────────┐
    ┌──────────────────────────────┬───────────────────────┤  Fri 14 Feb ├───────────────────────┬──────────────────────────────┐
    │            Morning           │             Noon      └──────┬──────┘     Evening           │             Night            │
    ├──────────────────────────────┼──────────────────────────────┼──────────────────────────────┼──────────────────────────────┤
    │    \  /       Partly cloudy  │     \   /     Sunny          │     \   /     Clear          │     \   /     Clear          │
    │  _ /"".-.     6..19 °F       │      .-.      8..21 °F       │      .-.      8..19 °F       │      .-.      3..14 °F       │
    │    \_(   ).   ↘ 11-13 mph    │   ― (   ) ―   ↘ 11-13 mph    │   ― (   ) ―   ↗ 11-13 mph    │   ― (   ) ―   ↓ 8-13 mph     │
    │    /(___(__)  5 mi           │      `-’      6 mi           │      `-’      6 mi           │      `-’      6 mi           │
    │               0.0 in | 0%    │     /   \     0.0 in | 0%    │     /   \     0.0 in | 0%    │     /   \     0.0 in | 0%    │
    └──────────────────────────────┴──────────────────────────────┴──────────────────────────────┴──────────────────────────────┘
                                                          ┌─────────────┐
    ┌──────────────────────────────┬───────────────────────┤  Sat 15 Feb ├───────────────────────┬──────────────────────────────┐
    │            Morning           │             Noon      └──────┬──────┘     Evening           │             Night            │
    ├──────────────────────────────┼──────────────────────────────┼──────────────────────────────┼──────────────────────────────┤
    │     \   /     Sunny          │    \  /       Partly cloudy  │    \  /       Partly cloudy  │    \  /       Partly cloudy  │
    │      .-.      14..17 °F      │  _ /"".-.     17..24 °F      │  _ /"".-.     21..30 °F      │  _ /"".-.     17..24 °F      │
    │   ― (   ) ―   ↖ 3 mph        │    \_(   ).   ↑ 6-8 mph      │    \_(   ).   ↑ 9-14 mph     │    \_(   ).   ↑ 8-14 mph     │
    │      `-’      6 mi           │    /(___(__)  6 mi           │    /(___(__)  6 mi           │    /(___(__)  6 mi           │
    │     /   \     0.0 in | 0%    │               0.0 in | 0%    │               0.0 in | 0%    │               0.0 in | 0%    │
    └──────────────────────────────┴──────────────────────────────┴──────────────────────────────┴──────────────────────────────┘
                                                          ┌─────────────┐
    ┌──────────────────────────────┬───────────────────────┤  Sun 16 Feb ├───────────────────────┬──────────────────────────────┐
    │            Morning           │             Noon      └──────┬──────┘     Evening           │             Night            │
    ├──────────────────────────────┼──────────────────────────────┼──────────────────────────────┼──────────────────────────────┤
    │    \  /       Partly cloudy  │    \  /       Partly cloudy  │               Overcast       │               Overcast       │
    │  _ /"".-.     24..30 °F      │  _ /"".-.     35..39 °F      │      .--.     37..39 °F      │      .--.     32..33 °F      │
    │    \_(   ).   ↗ 4-7 mph      │    \_(   ).   ↗ 7-8 mph      │   .-(    ).   → 6-9 mph      │   .-(    ).   ↗ 3-8 mph      │
    │    /(___(__)  6 mi           │    /(___(__)  6 mi           │  (___.__)__)  6 mi           │  (___.__)__)  6 mi           │
    │               0.0 in | 0%    │               0.0 in | 0%    │               0.0 in | 0%    │               0.0 in | 0%    │
    └──────────────────────────────┴──────────────────────────────┴──────────────────────────────┴──────────────────────────────┘

    Follow @igor_chubin for wttr.in updates
    [  18.379]    INFO github.com/netfoundry/ziti-edge/tunnel.myCopy: {dst-local=[:1] dst-remote=[wttr.ziti] src-remote=[127.0.0.1:57980] src-local=[127.0.0.1:9000]} stopping pipe
    [  18.379]   DEBUG github.com/netfoundry/ziti-sdk-golang/ziti/internal/edge_impl.(*edgeConn).Close: {connId=[1]} close: begin
    [  18.379]   DEBUG github.com/netfoundry/ziti-sdk-golang/ziti/edge.(*muxRemoveSinkEvent).Handle: {connId=[1]} Removed sink from mux. Current sink count: 0
    [  18.379]   DEBUG github.com/netfoundry/ziti-sdk-golang/ziti/edge.(*MsgMux).RemoveMsgSinkById: {connId=[1]} removed from msg mux
    [  18.379]   DEBUG github.com/netfoundry/ziti-sdk-golang/ziti/internal/edge_impl.(*edgeConn).Close: {connId=[1]} close: end
    cd@cd-ubuntuvm: [  19.382]   DEBUG github.com/netfoundry/ziti-sdk-golang/ziti/internal/edge_impl.(*edgeConn).Read: {connId=[1]} sequencer closed, closing connection
    [  19.382]   DEBUG github.com/netfoundry/ziti-sdk-golang/ziti/internal/edge_impl.(*edgeConn).Read: {connId=[1]} return EOF from closing/closed connection
    [  19.382]    INFO github.com/netfoundry/ziti-edge/tunnel.myCopy: {dst-remote=[127.0.0.1:57980] src-remote=[wttr.ziti] src-local=[:1] dst-local=[127.0.0.1:9000]} stopping pipe
    [  19.383]    INFO github.com/netfoundry/ziti-edge/tunnel.Run: {src-remote=[127.0.0.1:57980] src-local=[127.0.0.1:9000] dst-local=[:1] dst-remote=[wttr.ziti]} tunnel closed: 71 bytes sent; 8852 bytes received

    cd@cd-ubuntuvm: killall ziti-tunnel
    [  25.856]   DEBUG github.com/netfoundry/ziti-edge/tunnel/intercept.ServicePoller: caught signal terminated
    [  25.856]   DEBUG github.com/netfoundry/ziti-edge/tunnel/intercept.ServicePoller: caught signal terminated
    [  25.856]   DEBUG github.com/netfoundry/ziti-edge/tunnel/intercept.ServicePoller: caught signal terminated
    [  25.856]   DEBUG github.com/netfoundry/ziti-edge/tunnel/intercept.ServicePoller: caught signal terminated
    [  25.856]    INFO github.com/netfoundry/ziti-edge/tunnel/intercept.updateServices: stopping tunnel for unavailable service: wttr.ziti
    [  25.856]    INFO github.com/netfoundry/ziti-edge/tunnel/intercept.updateServices: stopping tunnel for unavailable service: netcat7256
    cd@cd-ubuntuvm: [  25.856]   ERROR github.com/netfoundry/ziti-edge/tunnel/intercept.updateServices: failed to stop intercepting: StopIntercepting not implemented by proxy interceptor
    [  25.856]   ERROR github.com/netfoundry/ziti-edge/tunnel/intercept.updateServices: failed to stop intercepting: StopIntercepting not implemented by proxy interceptor
    [  25.856]    INFO github.com/netfoundry/ziti-edge/tunnel/intercept.updateServices: stopping tunnel for unavailable service: netcat7256
    [  25.856]    INFO github.com/netfoundry/ziti-edge/tunnel/intercept.updateServices: stopping tunnel for unavailable service: wttr.ziti
    [  25.856]   ERROR github.com/netfoundry/ziti-edge/tunnel/intercept.updateServices: failed to stop intercepting: StopIntercepting not implemented by proxy interceptor
    [  25.856]   ERROR github.com/netfoundry/ziti-edge/tunnel/intercept.updateServices: failed to stop intercepting: StopIntercepting not implemented by proxy interceptor
    [  25.856]    INFO github.com/netfoundry/ziti-edge/tunnel/intercept/proxy.(*interceptor).Stop: stopping proxy interceptor
    [  25.856]    INFO github.com/netfoundry/ziti-edge/tunnel/intercept/proxy.(*interceptor).Stop: stopping proxy interceptor

# [Windows](#tab/verify-windows)

    c:\path\to\yubico-piv-tool-2.0.0\yubikey_windemo>REM the name of the ziti controller you're logging into

    c:\path\to\yubico-piv-tool-2.0.0\yubikey_windemo>SET ZITI_CTRL=local-edge-controller

    c:\path\to\yubico-piv-tool-2.0.0\yubikey_windemo>REM the location of the certificate(s) to use to validate the controller

    c:\path\to\yubico-piv-tool-2.0.0\yubikey_windemo>SET ZITI_CTRL_CERT=c:\path\to\controller.cert

    c:\path\to\yubico-piv-tool-2.0.0\yubikey_windemo>
    c:\path\to\yubico-piv-tool-2.0.0\yubikey_windemo>SET ZITI_USER=myUserName

    c:\path\to\yubico-piv-tool-2.0.0\yubikey_windemo>SET ZITI_PWD=myPassword

    c:\path\to\yubico-piv-tool-2.0.0\yubikey_windemo>
    c:\path\to\yubico-piv-tool-2.0.0\yubikey_windemo>REM a name for the configuration

    c:\path\to\yubico-piv-tool-2.0.0\yubikey_windemo>SET HSM_NAME=yubikey_windemo

    c:\path\to\yubico-piv-tool-2.0.0\yubikey_windemo>SET HSM_ROOT=c:\path\to\yubico-piv-tool-2.0.0

    c:\path\to\yubico-piv-tool-2.0.0\yubikey_windemo>
    c:\path\to\yubico-piv-tool-2.0.0\yubikey_windemo>REM path to the pkcs11 library

    c:\path\to\yubico-piv-tool-2.0.0\yubikey_windemo>SET PATH=%PATH%;%HSM_ROOT%\bin

    c:\path\to\yubico-piv-tool-2.0.0\yubikey_windemo>SET PKCS11_MODULE=%HSM_ROOT%\bin\libykcs11-1.dll

    c:\path\to\yubico-piv-tool-2.0.0\yubikey_windemo>
    c:\path\to\yubico-piv-tool-2.0.0\yubikey_windemo>REM the id of the key - you probably want to leave these alone unless you know better

    c:\path\to\yubico-piv-tool-2.0.0\yubikey_windemo>SET HSM_ID1=01

    c:\path\to\yubico-piv-tool-2.0.0\yubikey_windemo>SET HSM_ID2=03

    c:\path\to\yubico-piv-tool-2.0.0\yubikey_windemo>SET RSA_ID=%HSM_NAME%%HSM_ID1%_rsa

    c:\path\to\yubico-piv-tool-2.0.0\yubikey_windemo>SET EC_ID=%HSM_NAME%%HSM_ID2%_ec

    c:\path\to\yubico-piv-tool-2.0.0\yubikey_windemo>
    c:\path\to\yubico-piv-tool-2.0.0\yubikey_windemo>REM the pins used when accessing the pkcs11 api

    c:\path\to\yubico-piv-tool-2.0.0\yubikey_windemo>SET HSM_SOPIN=010203040506070801020304050607080102030405060708

    c:\path\to\yubico-piv-tool-2.0.0\yubikey_windemo>SET HSM_PIN=123456

    c:\path\to\yubico-piv-tool-2.0.0\yubikey_windemo>
    c:\path\to\yubico-piv-tool-2.0.0\yubikey_windemo>SET HSM_DEST=%HSM_ROOT%\%HSM_NAME%

    c:\path\to\yubico-piv-tool-2.0.0\yubikey_windemo>SET HSM_LABEL=%HSM_NAME%-label

    c:\path\to\yubico-piv-tool-2.0.0\yubikey_windemo>SET HSM_TOKENS_DIR=%HSM_DEST%\tokens\

    c:\path\to\yubico-piv-tool-2.0.0\yubikey_windemo>
    c:\path\to\yubico-piv-tool-2.0.0\yubikey_windemo>REM make an alias for ease

    c:\path\to\yubico-piv-tool-2.0.0\yubikey_windemo>doskey p="c:\Program Files\OpenSC Project\OpenSC\tools\pkcs11-tool.exe" --module %PKCS11_MODULE% $*

    c:\path\to\yubico-piv-tool-2.0.0\yubikey_windemo>cd /d %HSM_ROOT%

    c:\path\to\yubico-piv-tool-2.0.0>
    c:\path\to\yubico-piv-tool-2.0.0>rmdir /s /q %HSM_NAME%

    c:\path\to\yubico-piv-tool-2.0.0>mkdir %HSM_NAME%

    c:\path\to\yubico-piv-tool-2.0.0>
    c:\path\to\yubico-piv-tool-2.0.0>cd /d %HSM_NAME%

    c:\path\to\yubico-piv-tool-2.0.0\yubikey_windemo>ziti edge login %ZITI_CTRL%:1280 -u %ZITI_USER% -p %ZITI_PWD% -c %ZITI_CTRL_CERT%
    Token: 2e4b9d55-fe89-4e35-8101-7d8a5f492b8b

    c:\path\to\yubico-piv-tool-2.0.0\yubikey_windemo>
    c:\path\to\yubico-piv-tool-2.0.0\yubikey_windemo>REM create a new identity and output the jwt to a known location

    c:\path\to\yubico-piv-tool-2.0.0\yubikey_windemo>ziti edge create identity device "%RSA_ID%" -o "%HSM_DEST%\%RSA_ID%.jwt"
    43b76397-c65e-43a9-a326-b58f83cacfea
    Enrollment expires at 2020-02-24T14:47:02.783676578Z

    c:\path\to\yubico-piv-tool-2.0.0\yubikey_windemo>
      
    c:\path\to\yubico-piv-tool-2.0.0\yubikey_windemo>REM create a second new identity and output the jwt to a known location

    c:\path\to\yubico-piv-tool-2.0.0\yubikey_windemo>ziti edge create identity device "%EC_ID%" -o "%HSM_DEST%\%EC_ID%.jwt"
    fa116b87-d322-4058-a8c1-d41f21a9e051
    Enrollment expires at 2020-02-24T14:47:03.521726623Z
    
    c:\path\to\yubico-piv-tool-2.0.0\yubikey_windemo>p --init-token --label "ziti-test-token" --so-pin %HSM_SOPIN%
    Using slot 0 with a present token (0x0)
    Token successfully initialized

    c:\path\to\yubico-piv-tool-2.0.0\yubikey_windemo>
    c:\path\to\yubico-piv-tool-2.0.0\yubikey_windemo>REM create a couple of keys - one rsa and one ec

    c:\path\to\yubico-piv-tool-2.0.0\yubikey_windemo>p -k --key-type rsa:2048 --usage-sign --usage-decrypt --login --id %HSM_ID1% --login-type so --so-pin %HSM_SOPIN% --label defaultkey
    Using slot 0 with a present token (0x0)
    Key pair generated:
    Private Key Object; RSA
      label:      Private key for PIV Authentication
      ID:         01
      Usage:      decrypt, sign
      Access:     sensitive, always sensitive, never extractable, local
    Public Key Object; RSA 2048 bits
      label:      Public key for PIV Authentication
      ID:         01
      Usage:      encrypt, verify
      Access:     local

    c:\path\to\yubico-piv-tool-2.0.0\yubikey_windemo>p -k --key-type EC:prime256v1 --usage-sign --usage-decrypt --login --id %HSM_ID2% --login-type so --so-pin %HSM_SOPIN% --label defaultkey
    Using slot 0 with a present token (0x0)
    Key pair generated:
    Private Key Object; EC
      label:      Private key for Key Management
      ID:         03
      Usage:      decrypt, sign
      Access:     sensitive, always sensitive, never extractable, local
    Public Key Object; EC  EC_POINT 256 bits
      EC_POINT:   044104c5f6b528121fbe67e44dc2f966a2df9df97d5e712e40fdf6df74ba8c166d069d0eee65886c115034821b3e3c3c890e910ffe05efc70d281e531ae8073915f579
      EC_PARAMS:  06082a8648ce3d030107
      label:      Public key for Key Management
      ID:         03
      Usage:      encrypt, verify
      Access:     local

    c:\path\to\yubico-piv-tool-2.0.0\yubikey_windemo>ziti-tunnel enroll -j "%HSM_DEST%\%RSA_ID%.jwt" -k "pkcs11://%PKCS11_MODULE%?id=%HSM_ID1%&pin=%HSM_PIN%" -v
    time="2020-02-14T09:47:46-05:00" level=debug msg="jwt to parse: eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJlbSI6Im90dCIsImV4cCI6MTU4MjU1NTYyMiwiaXNzIjoiaHR0cHM6Ly9sb2NhbC1lZGdlLWNvbnRyb2xsZXI6MTI4MCIsImp0aSI6ImYwYjIxZjQ4LTk3N2QtNDVlZi04ZDZlLWY5YzllMzlkYTg1YSIsInN1YiI6IjQzYjc2Mzk3LWM2NWUtNDNhOS1hMzI2LWI1OGY4M2NhY2ZlYSJ9.lAfmkZb_3-qPtW4hWaaHsL_7rZiHlN3LKGIrudX4K91idw5C-AdcjT-smbfSlDBqmZElU0iomrmT4TJ3SbGM9yRhrnRZBgsbC76dKO6sbNuCEGLeMJ_pJFGo7ZZC32wLm2HsvCXszwjMXOrr1ndvWUKs3KkUhvmbBEIA4x566McsngeZ5B4ILso4BmL4KJcef-n-PuI14V42n7uzOu_KZomEGSNZ34LCg0cbWcsUmoeV8d3Vgb3xSR16wyyl7tC_pGvH1qPvaWmNUQ9BWk1j730C6RXLHZ3zECOUXqIjYTOaXkoDEpuXceNF5gNm9Cx76UwpKJlSEyD_LFbkULJL4_zmKy9w9JreSUh6cnkPW7ltXluEt-Wy1VckkwOPUCRXCQ_AuxvClhOPUW_Y-PW8Qbooou7eo8v-ZPAuFqx04_DkPdhMPn2h6CyZiqEM5ZqN_ln-Q6BAUgpBHmAFUuPDnCXk0R1l74Yc23z55hrfFreR9-t4BGl6ZBiV9oAa6kz4-8MTkg_osGJOAn3S0T_mFORCES68dppVAF9Lv8kaFkZsJE3pyQeApyYqtIZWZUUiQH4S3ssFfwuoy9KmP9lvSo4Ampl7whygx6B6nf5EiSvkQyvNPSW1L5VQeI3eShQbrUHVnrBMY2BQHa86Wqz6ICdSLQHW1P_Z6yX71HKa06A"
    time="2020-02-14T09:47:46-05:00" level=info msg="using engine : pkcs11\n"
    time="2020-02-14T09:47:46-05:00" level=debug msg="loading key" context=pkcs11 url="pkcs11://c:%5Cpath%5Cto%5Cyubico-piv-tool-2.0.0%5Cbin%5Clibykcs11-1.dll?id=01&pin=123456"
    time="2020-02-14T09:47:46-05:00" level=info msg="using driver: c:\\path\\to\\yubico-piv-tool-2.0.0\\bin\\libykcs11-1.dll" context=pkcs11
    time="2020-02-14T09:47:46-05:00" level=warning msg="slot not specified, using first slot reported by the driver (0)" context=pkcs11
    time="2020-02-14T09:47:46-05:00" level=debug msg="found signing mechanism" context=pkcs11 sign mechanism=0
    time="2020-02-14T09:47:46-05:00" level=debug msg="no cas provided in caPool. using system provided cas"
    time="2020-02-14T09:47:46-05:00" level=debug msg="fetching certificates from server"
    time="2020-02-14T09:47:46-05:00" level=debug msg="loading key" context=pkcs11 url="pkcs11://c:%5Cpath%5Cto%5Cyubico-piv-tool-2.0.0%5Cbin%5Clibykcs11-1.dll?id=01&pin=123456"
    time="2020-02-14T09:47:46-05:00" level=info msg="using driver: c:\\path\\to\\yubico-piv-tool-2.0.0\\bin\\libykcs11-1.dll" context=pkcs11
    time="2020-02-14T09:47:46-05:00" level=warning msg="slot not specified, using first slot reported by the driver (0)" context=pkcs11
    time="2020-02-14T09:47:46-05:00" level=debug msg="found signing mechanism" context=pkcs11 sign mechanism=0
    enrolled successfully. identity file written to: c:\path\to\yubico-piv-tool-2.0.0\yubikey_windemo\yubikey_windemo01_rsa.json
    c:\path\to\yubico-piv-tool-2.0.0\yubikey_windemo>ziti-tunnel enroll -j "%HSM_DEST%\%EC_ID%.jwt" -k "pkcs11://%PKCS11_MODULE%?id=%HSM_ID2%&pin=%HSM_PIN%" -v
    time="2020-02-14T09:47:47-05:00" level=debug msg="jwt to parse: eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJlbSI6Im90dCIsImV4cCI6MTU4MjU1NTYyMywiaXNzIjoiaHR0cHM6Ly9sb2NhbC1lZGdlLWNvbnRyb2xsZXI6MTI4MCIsImp0aSI6ImY1NzVkZTI0LTcyYzctNDcxZS1hMzI0LTUyMjE3ZWFlYmExNCIsInN1YiI6ImZhMTE2Yjg3LWQzMjItNDA1OC1hOGMxLWQ0MWYyMWE5ZTA1MSJ9.cMyzS7o4ou2XUN6d3C-fx6i7sleIZO5b_LeqOxBNkAZO6l7JFIAp54ak8TUYD-chsVnSZ-bYPiuO-lbPIR2UrZvN6SshOowiQpAKQPggL2y0UZJAqzRoxfzlkjBZXi5BjRSTGTW3nRd9y40AwR_aAkOr4IztYbREGBYvfTK4QX-NSVvSoGVCiBeKFC6PO_8nuAUxMgL_ODVpPVA1ThO5l6Jzut_N5LgysaxTjp8WWsAIvUIbnIxJ7y_hTvmKFQ9hbLuTLaWZ091AA-469dxzbMzQVe_xDSwpQDGfYgRm02_uRlKUlm_w_sPdytaNb54u0I0-la83r31S8ugyGz3xMD-xLDHVKwxZ7dUArQf3oKzGZNhCc6MMJQBI81jbHaeSCNlpDJugYDDG0bx1E87cU7Nt88h-jaLOwEOqozsG_4myhoFaMRGx-unNP8Lr-zlPtU3AGMTu21AvCq4CCg0HYLdKVjCkQ3m6N29wOrMgF_3G_hdvTLDTeHY2wKnJIBHg3lg106a7J6WtiM9CyS_N_01kCtwFWK49662GQPv6_IVb6eKokhtcA4-2E1F6H8v5BCUljzfjpfQ4EQds7uDtQSL3i3muBLuXkLUW86H98q8XVnQZms_41T2n6MWmo5smOYrs4--rM1R-in3lmKymd8nmmN-L2GuJF5zsRaiq0r4"
    time="2020-02-14T09:47:47-05:00" level=info msg="using engine : pkcs11\n"
    time="2020-02-14T09:47:47-05:00" level=debug msg="loading key" context=pkcs11 url="pkcs11://c:%5Cpath%5Cto%5Cyubico-piv-tool-2.0.0%5Cbin%5Clibykcs11-1.dll?id=03&pin=123456"
    time="2020-02-14T09:47:47-05:00" level=info msg="using driver: c:\\path\\to\\yubico-piv-tool-2.0.0\\bin\\libykcs11-1.dll" context=pkcs11
    time="2020-02-14T09:47:47-05:00" level=warning msg="slot not specified, using first slot reported by the driver (0)" context=pkcs11
    time="2020-02-14T09:47:48-05:00" level=debug msg="found signing mechanism" context=pkcs11 sign mechanism=0
    time="2020-02-14T09:47:48-05:00" level=debug msg="EC oid[1.2.840.10045.3.1.7], rest: [], err: <nil>" context=pkcs11
    time="2020-02-14T09:47:48-05:00" level=debug msg="no cas provided in caPool. using system provided cas"
    time="2020-02-14T09:47:48-05:00" level=debug msg="fetching certificates from server"
    time="2020-02-14T09:47:48-05:00" level=debug msg="loading key" context=pkcs11 url="pkcs11://c:%5Cpath%5Cto%5Cyubico-piv-tool-2.0.0%5Cbin%5Clibykcs11-1.dll?id=03&pin=123456"
    time="2020-02-14T09:47:48-05:00" level=info msg="using driver: c:\\path\\to\\yubico-piv-tool-2.0.0\\bin\\libykcs11-1.dll" context=pkcs11
    time="2020-02-14T09:47:48-05:00" level=warning msg="slot not specified, using first slot reported by the driver (0)" context=pkcs11
    time="2020-02-14T09:47:48-05:00" level=debug msg="found signing mechanism" context=pkcs11 sign mechanism=0
    time="2020-02-14T09:47:48-05:00" level=debug msg="EC oid[1.2.840.10045.3.1.7], rest: [], err: <nil>" context=pkcs11
    enrolled successfully. identity file written to: c:\path\to\yubico-piv-tool-2.0.0\yubikey_windemo\yubikey_windemo03_ec.json
    c:\path\to\yubico-piv-tool-2.0.0\yubikey_windemo>REM these two commands can't be copied and pasted - you need to get the result of the first command and use it in the next

    c:\path\to\yubico-piv-tool-2.0.0\yubikey_windemo>REM run this command and get the id from the first edge-router.

    c:\path\to\yubico-piv-tool-2.0.0\yubikey_windemo>ziti edge list edge-routers
    id: 0c2c2049-3577-479c-aa09-625fa0e15d31    name: local-edge-router    role attributes: {}

    c:\path\to\yubico-piv-tool-2.0.0\yubikey_windemo>
    c:\path\to\yubico-piv-tool-2.0.0\yubikey_windemo>REM use the id returned from the above command and put it into a variable for use in a momment

    c:\path\to\yubico-piv-tool-2.0.0\yubikey_windemo>SET EDGE_ROUTER_ID=0c2c2049-3577-479c-aa09-625fa0e15d31

    c:\path\to\yubico-piv-tool-2.0.0\yubikey_windemo>
    c:\path\to\yubico-piv-tool-2.0.0\yubikey_windemo>REM remove/recreate the config - here we'll be instructing the tunneler to listen on localhost and port 9000

    c:\path\to\yubico-piv-tool-2.0.0\yubikey_windemo>ziti edge delete config wttrconfig

    c:\path\to\yubico-piv-tool-2.0.0\yubikey_windemo>ziti edge create config wttrconfig ziti-tunneler-client.v1 "{ \"hostname\" : \"localhost\", \"port\" : 9000 }"
    b2c7af06-072d-4ff2-9f73-8d1456a16d5a

    c:\path\to\yubico-piv-tool-2.0.0\yubikey_windemo>
    c:\path\to\yubico-piv-tool-2.0.0\yubikey_windemo>REM recreate the service with the EDGE_ROUTER_ID from above. Here we are adding a ziti service that will

    c:\path\to\yubico-piv-tool-2.0.0\yubikey_windemo>REM send a request to wttr.in to retreive a weather forecast

    c:\path\to\yubico-piv-tool-2.0.0\yubikey_windemo>ziti edge delete service wttr.ziti

    c:\path\to\yubico-piv-tool-2.0.0\yubikey_windemo>ziti edge create service wttr.ziti "%EDGE_ROUTER_ID%" tcp://wttr.in:80 --configs wttrconfig
    5a11bfce-b716-4d7a-944b-ea72e6225549

    c:\path\to\yubico-piv-tool-2.0.0\yubikey_windemo>
    c:\path\to\yubico-piv-tool-2.0.0\yubikey_windemo>REM start one or both proxies - use ctrl-break or ctrl-pause to terminate these processes

    c:\path\to\yubico-piv-tool-2.0.0\yubikey_windemo>start /b ziti-tunnel proxy -i "%HSM_DEST%/%RSA_ID%.json" wttr.ziti:8000 -v

    c:\path\to\yubico-piv-tool-2.0.0\yubikey_windemo>start /b ziti-tunnel proxy -i "%HSM_DEST%/%EC_ID%.json" wttr.ziti:9000 -v

    c:\path\to\yubico-piv-tool-2.0.0\yubikey_windemo>
    c:\path\to\yubico-piv-tool-2.0.0\yubikey_windemo>REM[   0.013]    INFO github.com/netfoundry/ziti-edge/tunnel/intercept/proxy.(*interceptor).Start: starting proxy interceptor
    [   0.013]   DEBUG github.com/netfoundry/ziti-foundation/identity/engines/pkcs11.(*engine).LoadKey [pkcs11]: {url=[pkcs11://c:%5Cpath%5Cto%5Cyubico-piv-tool-2.0.0%5Cbin%5Clibykcs11-1.dll?id=01&pin=123456]} loading key
    [   0.014]    INFO github.com/netfoundry/ziti-foundation/identity/engines/pkcs11.(*engine).LoadKey [pkcs11]: using driver: c:\path\to\yubico-piv-tool-2.0.0\bin\libykcs11-1.dll
    [   0.012]    INFO github.com/netfoundry/ziti-edge/tunnel/intercept/proxy.(*interceptor).Start: starting proxy interceptor
    [   0.012]   DEBUG github.com/netfoundry/ziti-foundation/identity/engines/pkcs11.(*engine).LoadKey [pkcs11]: {url=[pkcs11://c:%5Cpath%5Cto%5Cyubico-piv-tool-2.0.0%5Cbin%5Clibykcs11-1.dll?id=03&pin=123456]} loading key
    [   0.013]    INFO github.com/netfoundry/ziti-foundation/identity/engines/pkcs11.(*engine).LoadKey [pkcs11]: using driver: c:\path\to\yubico-piv-tool-2.0.0\bin\libykcs11-1.dll
    [   0.025] WARNING github.com/netfoundry/ziti-foundation/identity/engines/pkcs11.(*engine).LoadKey [pkcs11]: slot not specified, using first slot reported by the driver (0)
    [   0.195] WARNING github.com/netfoundry/ziti-foundation/identity/engines/pkcs11.(*engine).LoadKey [pkcs11]: slot not specified, using first slot reported by the driver (0)
    [   0.688]   DEBUG github.com/netfoundry/ziti-foundation/identity/engines/pkcs11.(*engine).LoadKey [pkcs11]: {sign mechanism=[0]} found signing mechanism
    [   0.689]    INFO github.com/netfoundry/ziti-sdk-golang/ziti.(*contextImpl).Authenticate: attempting to authenticate
    [   0.699]   DEBUG github.com/netfoundry/ziti-foundation/identity/engines/pkcs11.(*engine).LoadKey [pkcs11]: {sign mechanism=[0]} found signing mechanism
    [   0.699]   DEBUG github.com/netfoundry/ziti-foundation/identity/engines/pkcs11.loadECDSApub [pkcs11]: EC oid[1.2.840.10045.3.1.7], rest: [], err: <nil>
    [   0.700]    INFO github.com/netfoundry/ziti-sdk-golang/ziti.(*contextImpl).Authenticate: attempting to authenticate
    [   0.812]   DEBUG github.com/netfoundry/ziti-sdk-golang/ziti.(*contextImpl).Authenticate: {token=[3b8bbb76-5984-495e-94cd-9752fe6a4d61] id=[55a4e9aa-eb78-4a19-a685-92aa19fd7177]} Got api session: {55a4e9aa-eb78-4a19-a685-92aa19fd7177 3b8bbb76-5984-495e-94cd-9752fe6a4d61}
    [   0.813]   DEBUG github.com/netfoundry/ziti-sdk-golang/ziti.(*contextImpl).getServices: using api session token 3b8bbb76-5984-495e-94cd-9752fe6a4d61
    [   0.866]   DEBUG github.com/netfoundry/ziti-sdk-golang/ziti.(*contextImpl).getServices: using api session token 3b8bbb76-5984-495e-94cd-9752fe6a4d61
    [   0.920]    INFO github.com/netfoundry/ziti-edge/tunnel/intercept.updateServices: starting tunnel for newly available service wttr.ziti
    [   0.936]   DEBUG github.com/netfoundry/ziti-sdk-golang/ziti.(*contextImpl).getSession: requesting session from https://local-edge-controller:1280/sessions
    [   0.939]   DEBUG github.com/netfoundry/ziti-sdk-golang/ziti.(*contextImpl).getSession: {service_id=[5a11bfce-b716-4d7a-944b-ea72e6225549]} requesting session
    [   0.960]   DEBUG github.com/netfoundry/ziti-sdk-golang/ziti.(*contextImpl).Authenticate: {token=[79e3ef58-d712-4821-9f8b-3e2a947bd7b5] id=[4af36742-61ac-49cc-8935-08a8ab14d6c6]} Got api session: {4af36742-61ac-49cc-8935-08a8ab14d6c6 79e3ef58-d712-4821-9f8b-3e2a947bd7b5}
    [   0.975]   DEBUG github.com/netfoundry/ziti-sdk-golang/ziti.(*contextImpl).getServices: using api session token 79e3ef58-d712-4821-9f8b-3e2a947bd7b5
    [   0.983]   DEBUG github.com/netfoundry/ziti-edge/tunnel/intercept/proxy.interceptor.Intercept: {service=[wttr.ziti] id=[9eb75dd2-88df-4b6c-8a1e-f9d1a6e15adc]} acquired network session
    [   0.984]    INFO github.com/netfoundry/ziti-edge/tunnel/intercept.updateServices: starting tunnel for newly available service netcat7256
    [   1.002]    INFO github.com/netfoundry/ziti-edge/tunnel/intercept/proxy.(*interceptor).handleTCP: {service=[wttr.ziti] addr=[0.0.0.0:9000]} service is listening
    [   1.003]   DEBUG github.com/netfoundry/ziti-edge/tunnel/intercept/proxy.interceptor.Intercept: {service=[netcat7256]} service netcat7256 was not specified at initialization. not intercepting
    [   1.023]   DEBUG github.com/netfoundry/ziti-sdk-golang/ziti.(*contextImpl).getServices: using api session token 79e3ef58-d712-4821-9f8b-3e2a947bd7b5
    [   1.071]    INFO github.com/netfoundry/ziti-edge/tunnel/intercept.updateServices: starting tunnel for newly available service wttr.ziti
    [   1.071]   DEBUG github.com/netfoundry/ziti-sdk-golang/ziti.(*contextImpl).getSession: requesting session from https://local-edge-controller:1280/sessions
    [   1.072]   DEBUG github.com/netfoundry/ziti-sdk-golang/ziti.(*contextImpl).getSession: {service_id=[5a11bfce-b716-4d7a-944b-ea72e6225549]} requesting session
    [   1.102]   DEBUG github.com/netfoundry/ziti-edge/tunnel/intercept/proxy.interceptor.Intercept: {service=[wttr.ziti] id=[5589a3dc-0301-4563-bf65-cf1143f9db3b]} acquired network session
    [   1.102]    INFO github.com/netfoundry/ziti-edge/tunnel/intercept.updateServices: starting tunnel for newly available service netcat7256
    [   1.103]   DEBUG github.com/netfoundry/ziti-edge/tunnel/intercept/proxy.interceptor.Intercept: {service=[netcat7256]} service netcat7256 was not specified at initialization. not intercepting
    [   1.104]    INFO github.com/netfoundry/ziti-edge/tunnel/intercept/proxy.(*interceptor).handleTCP: {service=[wttr.ziti] addr=[0.0.0.0:8000]} service is listening
    use a browser - or curl to verify the ziti tunneler is listening locally and the traffic has flowed over the ziti network

    c:\path\to\yubico-piv-tool-2.0.0\yubikey_windemo>curl -H "Host: wttr.in" http://localhost:8000 > "%HSM_DEST%\example_%RSA_ID%.txt"
    [   7.803]   DEBUG github.com/netfoundry/ziti-foundation/channel2.(*classicDialer).Create [tls:local-edge-router:3022]: started
    % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                    Dload  Upload   Total   Spent    Left  Speed
      0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0[   7.948]   DEBUG github.com/netfoundry/ziti-foundation/transport/tls.Dial: server provided [2] certificates
    [   7.948]   DEBUG github.com/netfoundry/ziti-foundation/channel2.(*classicDialer).sendHello [u{classic}->i{}]: started
    [   7.949]   DEBUG github.com/netfoundry/ziti-foundation/channel2.(*classicDialer).sendHello [u{classic}->i{}]: exited
    [   7.950]   DEBUG github.com/netfoundry/ziti-foundation/channel2.(*classicDialer).Create [tls:local-edge-router:3022]: exited
    [   7.950]   DEBUG github.com/netfoundry/ziti-sdk-golang/ziti/edge.(*muxAddSinkEvent).Handle: {connId=[1]} Added sink to mux. Current sink count: 1
    [   7.950]   DEBUG github.com/netfoundry/ziti-foundation/channel2.(*channelImpl).rxer [ch{ziti-sdk}->u{classic}->i{nJXo}]: started
    [   7.951]   DEBUG github.com/netfoundry/ziti-foundation/channel2.(*channelImpl).txer [ch{ziti-sdk}->u{classic}->i{nJXo}]: started
    [   7.951]   DEBUG github.com/netfoundry/ziti-sdk-golang/ziti/edge.(*MsgMux).AddMsgSink: {connId=[1]} added to msg mux
    [   8.082]   DEBUG github.com/netfoundry/ziti-foundation/channel2.(*channelImpl).rxer [ch{ziti-sdk}->u{classic}->i{nJXo}]: waiter found for message. type [60784], sequence [1], replyFor [1]
    [   8.082]   DEBUG github.com/netfoundry/ziti-sdk-golang/ziti/internal/edge_impl.(*edgeConn).Connect: {connId=[1]} connected
    [   8.083]    INFO github.com/netfoundry/ziti-edge/tunnel.Run: {src-local=[127.0.0.1:8000] dst-local=[:1] dst-remote=[wttr.ziti] src-remote=[127.0.0.1:52952]} tunnel started
    [   8.084]   DEBUG github.com/netfoundry/ziti-sdk-golang/ziti/internal/edge_impl.(*edgeConn).Read: {connId=[1]} read buffer = 32768 bytes
    [   8.085]   DEBUG github.com/netfoundry/ziti-sdk-golang/ziti/edge.(*MsgChannel).WriteTraced: {edgeSeq=[1] connId=[1] type=[EdgeDataType] chSeq=[-1]} writing 71 bytes
    [   8.252]   DEBUG github.com/netfoundry/ziti-sdk-golang/ziti/edge.(*MsgEvent).Handle: {seq=[1] connId=[1]} handling received EdgeDataType
    [   8.252]   DEBUG github.com/netfoundry/ziti-sdk-golang/ziti/internal/edge_impl.(*edgeConn).Read: {connId=[1]} got buffer from queue 8852 bytes
    [   8.255]   DEBUG github.com/netfoundry/ziti-sdk-golang/ziti/internal/edge_impl.(*edgeConn).Read: {connId=[1]} read buffer = 32768 bytes
    100  8687  100  8687    0     0  18522      0 --:--:-- --:--:-- --:--:-- 18522
    [   8.269]    INFO github.com/netfoundry/ziti-edge/tunnel.myCopy: {dst-local=[:1] dst-remote=[wttr.ziti] src-remote=[127.0.0.1:52952] src-local=[127.0.0.1:8000]} stopping pipe
    [   8.269]   DEBUG github.com/netfoundry/ziti-sdk-golang/ziti/internal/edge_impl.(*edgeConn).Close: {connId=[1]} close: begin
    [   8.270]   DEBUG github.com/netfoundry/ziti-sdk-golang/ziti/edge.(*muxRemoveSinkEvent).Handle: {connId=[1]} Removed sink from mux. Current sink count: 0
    [   8.271]   DEBUG github.com/netfoundry/ziti-sdk-golang/ziti/edge.(*MsgMux).RemoveMsgSinkById: {connId=[1]} removed from msg mux
    [   8.273]   DEBUG github.com/netfoundry/ziti-sdk-golang/ziti/internal/edge_impl.(*edgeConn).Close: {connId=[1]} close: end

    c:\path\to\yubico-piv-tool-2.0.0\yubikey_windemo>curl -H "Host: wttr.in" http://localhost:9000 > "%HSM_DEST%\example_%EC_ID%.txt"
    [   8.298]   DEBUG github.com/netfoundry/ziti-foundation/channel2.(*classicDialer).Create [tls:local-edge-router:3022]: started
    % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                    Dload  Upload   Total   Spent    Left  Speed
      0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0[   8.379]   DEBUG github.com/netfoundry/ziti-foundation/transport/tls.Dial: server provided [2] certificates
    [   8.379]   DEBUG github.com/netfoundry/ziti-foundation/channel2.(*classicDialer).sendHello [u{classic}->i{}]: started
    [   8.380]   DEBUG github.com/netfoundry/ziti-foundation/channel2.(*classicDialer).sendHello [u{classic}->i{}]: exited
    [   8.381]   DEBUG github.com/netfoundry/ziti-foundation/channel2.(*classicDialer).Create [tls:local-edge-router:3022]: exited
    [   8.381]   DEBUG github.com/netfoundry/ziti-foundation/channel2.(*channelImpl).rxer [ch{ziti-sdk}->u{classic}->i{oWqn}]: started
    [   8.383]   DEBUG github.com/netfoundry/ziti-foundation/channel2.(*channelImpl).txer [ch{ziti-sdk}->u{classic}->i{oWqn}]: started
    [   8.384]   DEBUG github.com/netfoundry/ziti-sdk-golang/ziti/edge.(*muxAddSinkEvent).Handle: {connId=[1]} Added sink to mux. Current sink count: 1
    [   8.385]   DEBUG github.com/netfoundry/ziti-sdk-golang/ziti/edge.(*MsgMux).AddMsgSink: {connId=[1]} added to msg mux
    [   8.495]   DEBUG github.com/netfoundry/ziti-foundation/channel2.(*channelImpl).rxer [ch{ziti-sdk}->u{classic}->i{oWqn}]: waiter found for message. type [60784], sequence [1], replyFor [1]
    [   8.495]   DEBUG github.com/netfoundry/ziti-sdk-golang/ziti/internal/edge_impl.(*edgeConn).Connect: {connId=[1]} connected
    [   8.495]    INFO github.com/netfoundry/ziti-edge/tunnel.Run: {src-remote=[127.0.0.1:52954] src-local=[127.0.0.1:9000] dst-local=[:1] dst-remote=[wttr.ziti]} tunnel started
    [   8.496]   DEBUG github.com/netfoundry/ziti-sdk-golang/ziti/edge.(*MsgChannel).WriteTraced: {connId=[1] type=[EdgeDataType] chSeq=[-1] edgeSeq=[1]} writing 71 bytes
    [   8.496]   DEBUG github.com/netfoundry/ziti-sdk-golang/ziti/internal/edge_impl.(*edgeConn).Read: {connId=[1]} read buffer = 32768 bytes
    [   8.665]   DEBUG github.com/netfoundry/ziti-sdk-golang/ziti/edge.(*MsgEvent).Handle: {seq=[1] connId=[1]} handling received EdgeDataType
    [   8.666]   DEBUG github.com/netfoundry/ziti-sdk-golang/ziti/internal/edge_impl.(*edgeConn).Read: {connId=[1]} got buffer from queue 8852 bytes
    [   8.667]   DEBUG github.com/netfoundry/ziti-sdk-golang/ziti/internal/edge_impl.(*edgeConn).Read: {connId=[1]} read buffer = 32768 bytes
    100  8687  100  8687    0     0  23165      0 --:--:-- --:--:-- --:--:-- 24130
    [   8.674]    INFO github.com/netfoundry/ziti-edge/tunnel.myCopy: {src-remote=[127.0.0.1:52954] src-local=[127.0.0.1:9000] dst-local=[:1] dst-remote=[wttr.ziti]} stopping pipe
    [   8.674]   DEBUG github.com/netfoundry/ziti-sdk-golang/ziti/internal/edge_impl.(*edgeConn).Close: {connId=[1]} close: begin
    [   8.675]   DEBUG github.com/netfoundry/ziti-sdk-golang/ziti/edge.(*muxRemoveSinkEvent).Handle: {connId=[1]} Removed sink from mux. Current sink count: 0
    [   8.675]   DEBUG github.com/netfoundry/ziti-sdk-golang/ziti/edge.(*MsgMux).RemoveMsgSinkById: {connId=[1]} removed from msg mux
    [   8.676]   DEBUG github.com/netfoundry/ziti-sdk-golang/ziti/internal/edge_impl.(*edgeConn).Close: {connId=[1]} close: end

    c:\path\to\yubico-piv-tool-2.0.0\yubikey_windemo>[   9.271]   DEBUG github.com/netfoundry/ziti-sdk-golang/ziti/internal/edge_impl.(*edgeConn).Read: {connId=[1]} sequencer closed, closing connection
    [   9.271]   DEBUG github.com/netfoundry/ziti-sdk-golang/ziti/internal/edge_impl.(*edgeConn).Read: {connId=[1]} return EOF from closing/closed connection
    [   9.271]    INFO github.com/netfoundry/ziti-edge/tunnel.myCopy: {src-remote=[wttr.ziti] src-local=[:1] dst-local=[127.0.0.1:8000] dst-remote=[127.0.0.1:52952]} stopping pipe
    [   9.271]    INFO github.com/netfoundry/ziti-edge/tunnel.Run: {src-local=[127.0.0.1:8000] dst-local=[:1] dst-remote=[wttr.ziti] src-remote=[127.0.0.1:52952]} tunnel closed: 71 bytes sent; 8852 bytes received
    [   9.676]   DEBUG github.com/netfoundry/ziti-sdk-golang/ziti/internal/edge_impl.(*edgeConn).Read: {connId=[1]} sequencer closed, closing connection
    [   9.676]   DEBUG github.com/netfoundry/ziti-sdk-golang/ziti/internal/edge_impl.(*edgeConn).Read: {connId=[1]} return EOF from closing/closed connection
    [   9.676]    INFO github.com/netfoundry/ziti-edge/tunnel.myCopy: {src-remote=[wttr.ziti] src-local=[:1] dst-local=[127.0.0.1:9000] dst-remote=[127.0.0.1:52954]} stopping pipe
    [   9.677]    INFO github.com/netfoundry/ziti-edge/tunnel.Run: {src-remote=[127.0.0.1:52954] src-local=[127.0.0.1:9000] dst-local=[:1] dst-remote=[wttr.ziti]} tunnel closed: 71 bytes sent; 8852 bytes received
    REM set the codepage for the cmd prompt so that the output looks nice

    c:\path\to\yubico-piv-tool-2.0.0\yubikey_windemo>chcp 65001
    Active code page: 65001

    c:\path\to\yubico-piv-tool-2.0.0\yubikey_windemo>
    c:\path\to\yubico-piv-tool-2.0.0\yubikey_windemo>REM show the results in the console

    c:\path\to\yubico-piv-tool-2.0.0\yubikey_windemo>type "%HSM_DEST%\example_%RSA_ID%.txt"
    Weather report: Laurelton, United States

                  Overcast
          .--.     14..24 °F
      .-(    ).   ↘ 13 mph
      (___.__)__)  9 mi
                  0.0 in
                                                          ┌─────────────┐
    ┌─────────────���────────────────┬───────────────────────┤  Fri 14 Feb ├───────────────────────┬──────────────────────────────┐
    │            Morning           │             Noon      └──────┬──────┘     Evening           │             Night            │
    ├────────────��─────────────────┼──────────────────────────────┼──────────────────────────────┼──────────────────────────────┤
    │    \  /       Partly cloudy  │     \   /     Sunny          │     \   /     Clear          │     \   /     Clear          │
    │  _ /"".-.     6..19 °F       │      .-.      8..21 °F       │      .-.      8..19 °F       │      .-.      3..14 °F       │
    │    \_(   ).   ↘ 11-13 mph    │   ― (   ) ―   �� 11-13 mph    │   ― (   ) ―   ↗ 11-13 mph    │   ― (   ) ―   ↓ 8-13 mph     │
    │    /(___(__)  5 mi           │      `-’      6 mi           │      `-’      6 mi           │      `-’      6 mi           │
    │               0.0 in | 0%    │     /   \     0.0 in | 0%    │     /   \     0.0 in | 0%    │     /   \     0.0 in | 0%    │
    └──────────────────────────────┴──────────────────────────────┴──────────────────────────────┴──────────────────────���───────┘
                                                          ┌─────────────┐
    ┌──────────────────────────────┬───────────────────────┤  Sat 15 Feb ├───────────────────────┬────────────────────────���─────┐
    │            Morning           │             Noon      └──────┬──────┘     Evening           │             Night            │
    ├──────────────────────────────┼──────────────────────────────┼──────────────────────────────┼──────────────────────────────┤
    │     \   /     Sunny          │    \  /       Partly cloudy  │    \  /       Partly cloudy  │    \  /       Partly cloudy  │
    │      .-.      14..17 °F      │  _ /"".-.     17..24 °F      │  _ /"".-.     21..30 °F      │  _ /"".-.     17..24 °F      │
    │   ― (   ) ―   ↖ 3 mph        │    \_(   ).   ↑ 6-8 mph      │    \_(   ).   ↑ 9-14 mph     │    \_(   ).   ↑ 8-14 mph     │
    │      `-’      6 mi           │    /(___(__)  6 mi           │    /(___(__)  6 mi           │    /(___(__)  6 mi           │
    │     /   \     0.0 in | 0%    │               0.0 in | 0%    │               0.0 in | 0%    │               0.0 in | 0%    │
    └──────────────────────────────┴──────────────────────────────┴──────────────────────────────┴──────────────────────────────┘
                                                          ┌─────────────┐
    ┌──────────────────────────────���───────────────────────┤  Sun 16 Feb ├───────────────────────┬──────────────────────────────┐
    │            Morning           │             Noon      └──────┬──────┘     Evening           │             Night            │
    ├─────────────────────────────��┼──────────────────────────────┼──────────────────────────────┼──────────────────────────────┤
    │    \  /       Partly cloudy  │    \  /       Partly cloudy  │               Overcast       │               Overcast       │
    │  _ /"".-.     24..30 °F      │  _ /"".-.     35..39 °F      │      .--.     37..39 °F      │      .--.     32..33 °F      │
    │    \_(   ).   ↗ 4-7 mph      │    \_(   ).   ↗ 7-8 mph      │   .-(    ).   → 6-9 mph      │   .-(    ).   ↗ 3-8 mph      │
    │    /(___(__)  6 mi           │    /(___(__)  6 mi           │  (___.__)__)  6 mi           │  (___.__)__)  6 mi           │
    │               0.0 in | 0%    │               0.0 in | 0%    │               0.0 in | 0%    │               0.0 in | 0%    │
    └──────────────────────────────┴──────────────────────────────┴──────────────────────────────┴──────────────────────────────┘

    Follow @igor_chubin for wttr.in updates

    c:\path\to\yubico-piv-tool-2.0.0\yubikey_windemo>type "%HSM_DEST%\example_%EC_ID%.txt"
    Weather report: Laurelton, United States

                  Overcast
          .--.     14..24 °F
      .-(    ).   ↘ 13 mph
      (___.__)__)  9 mi
                  0.0 in
                                                          ┌─────────────┐
    ┌─────────────���────────────────┬───────────────────────┤  Fri 14 Feb ├───────────────────────┬──────────────────────────────┐
    │            Morning           │             Noon      └──────┬──────┘     Evening           │             Night            │
    ├────────────��─────────────────┼──────────────────────────────┼──────────────────────────────┼──────────────────────────────┤
    │    \  /       Partly cloudy  │     \   /     Sunny          │     \   /     Clear          │     \   /     Clear          │
    │  _ /"".-.     6..19 °F       │      .-.      8..21 °F       │      .-.      8..19 °F       │      .-.      3..14 °F       │
    │    \_(   ).   ↘ 11-13 mph    │   ― (   ) ―   �� 11-13 mph    │   ― (   ) ―   ↗ 11-13 mph    │   ― (   ) ―   ↓ 8-13 mph     │
    │    /(___(__)  5 mi           │      `-’      6 mi           │      `-’      6 mi           │      `-’      6 mi           │
    │               0.0 in | 0%    │     /   \     0.0 in | 0%    │     /   \     0.0 in | 0%    │     /   \     0.0 in | 0%    │
    └──────────────────────────────┴──────────────────────────────┴──────────────────────────────┴──────────────────────���───────┘
                                                          ┌─────────────┐
    ┌──────────────────────────────┬───────────────────────┤  Sat 15 Feb ├───────────────────────┬────────────────────────���─────┐
    │            Morning           │             Noon      └──────┬──────┘     Evening           │             Night            │
    ├──────────────────────────────┼──────────────────────────────┼──────────────────────────────┼──────────────────────────────┤
    │     \   /     Sunny          │    \  /       Partly cloudy  │    \  /       Partly cloudy  │    \  /       Partly cloudy  │
    │      .-.      14..17 °F      │  _ /"".-.     17..24 °F      │  _ /"".-.     21..30 °F      │  _ /"".-.     17..24 °F      │
    │   ― (   ) ―   ↖ 3 mph        │    \_(   ).   ↑ 6-8 mph      │    \_(   ).   ↑ 9-14 mph     │    \_(   ).   ↑ 8-14 mph     │
    │      `-’      6 mi           │    /(___(__)  6 mi           │    /(___(__)  6 mi           │    /(___(__)  6 mi           │
    │     /   \     0.0 in | 0%    │               0.0 in | 0%    │               0.0 in | 0%    │               0.0 in | 0%    │
    └──────────────────────────────┴──────────────────────────────┴──────────────────────────────┴──────────────────────────────┘
                                                          ┌─────────────┐
    ┌──────────────────────────────���───────────────────────┤  Sun 16 Feb ├───────────────────────┬──────────────────────────────┐
    │            Morning           │             Noon      └──────┬──────┘     Evening           │             Night            │
    ├─────────────────────────────��┼──────────────────────────────┼──────────────────────────────┼──────────────────────────────┤
    │    \  /       Partly cloudy  │    \  /       Partly cloudy  │               Overcast       │               Overcast       │
    │  _ /"".-.     24..30 °F      │  _ /"".-.     35..39 °F      │      .--.     37..39 °F      │      .--.     32..33 °F      │
    │    \_(   ).   ↗ 4-7 mph      │    \_(   ).   ↗ 7-8 mph      │   .-(    ).   → 6-9 mph      │   .-(    ).   ↗ 3-8 mph      │
    │    /(___(__)  6 mi           │    /(___(__)  6 mi           │  (___.__)__)  6 mi           │  (___.__)__)  6 mi           │
    │               0.0 in | 0%    │               0.0 in | 0%    │               0.0 in | 0%    │               0.0 in | 0%    │
    └──────────────────────────────┴──────────────────────────────┴──────────────────────────────┴──────────────────────────────┘

    Follow @igor_chubin for wttr.in updates

    c:\path\to\yubico-piv-tool-2.0.0\yubikey_windemo>REM ctrl-break or ctrl-pause to kill the tunnelers

    c:\path\to\yubico-piv-tool-2.0.0\yubikey_windemo>

    [  39.022]   DEBUG github.com/netfoundry/ziti-edge/tunnel/intercept.ServicePoller: caught signal interrupt
    [  39.015]   DEBUG github.com/netfoundry/ziti-edge/tunnel/intercept.ServicePoller: caught signal interrupt
    c:\path\to\yubico-piv-tool-2.0.0\yubikey_windemo>[  39.023]   DEBUG github.com/netfoundry/ziti-edge/tunnel/intercept.ServicePoller: caught signal interrupt
    [  39.017]   DEBUG github.com/netfoundry/ziti-edge/tunnel/intercept.ServicePoller: caught signal interrupt
    [  39.025]    INFO github.com/netfoundry/ziti-edge/tunnel/intercept.updateServices: stopping tunnel for unavailable service: wttr.ziti
    [  39.018]    INFO github.com/netfoundry/ziti-edge/tunnel/intercept.updateServices: stopping tunnel for unavailable service: wttr.ziti
    [  39.025]   ERROR github.com/netfoundry/ziti-edge/tunnel/intercept.updateServices: failed to stop intercepting: StopIntercepting not implemented by proxy interceptor
    [  39.019]   ERROR github.com/netfoundry/ziti-edge/tunnel/intercept.updateServices: failed to stop intercepting: StopIntercepting not implemented by proxy interceptor
    [  39.026]    INFO github.com/netfoundry/ziti-edge/tunnel/intercept.updateServices: stopping tunnel for unavailable service: netcat7256
    [  39.020]    INFO github.com/netfoundry/ziti-edge/tunnel/intercept.updateServices: stopping tunnel for unavailable service: netcat7256
    [  39.027]   ERROR github.com/netfoundry/ziti-edge/tunnel/intercept.updateServices: failed to stop intercepting: StopIntercepting not implemented by proxy interceptor
    [  39.020]   ERROR github.com/netfoundry/ziti-edge/tunnel/intercept.updateServices: failed to stop intercepting: StopIntercepting not implemented by proxy interceptor
    [  39.028]    INFO github.com/netfoundry/ziti-edge/tunnel/intercept/proxy.(*interceptor).Stop: stopping proxy interceptor
    [  39.021]    INFO github.com/netfoundry/ziti-edge/tunnel/intercept/proxy.(*interceptor).Stop: stopping proxy interceptor
    REM set the codepage for the cmd prompt so that the output looks nice

***
