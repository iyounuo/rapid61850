# Rapid-prototyping protection schemes with IEC 61850 #

The goal of this software is to automatically generate C/C++ code which reads and writes GOOSE and Sampled Value packets. Any valid IEC 61850 Substation Configuration Description (SCD) file, describing GOOSE and/or SV communications, can be used as the input. The output code is lightweight and platform-independent, so it can run on a variety of devices, including low-cost microcontrollers and the Raspberry Pi. It's ideal for rapid-prototyping new power system protection, control, and automation systems that require communications.

This readme file describes how to set up the software, and its basic use.

*The code is meant to be a proof of concept, and is highly experimental. It has not been tested on many SCD files. Some features may be incomplete.*

<img style="float:right" src="http://personal.strath.ac.uk/steven.m.blair/mbed-cropped.png" />

## Features ##

 - Implements sending and receiving GOOSE and Sampled Value packets
 - Lightweight and fast, and suitable for low-cost microcontrollers and the Raspberry Pi
 - Platform-independent, and any C/C++ compiler should work
 - Performs validation of the SCD file, and reports any problems
 - Can optionally support fixed-length GOOSE encoding, which reduces GOOSE encoding time by approximately 50%
 - Supports initialisation of data type values, and instance-specific values
 - Simple API. The platform can be used in two ways:
   - As part of a native C/C++ program. This approach would be used where deterministic real-time performance is important, or where the network interface is custom (such as on a microcontroller). It also works well with the Qt C++ GUI framework.
   - As part of a Python or Java program. This approach uses additional C code (with winpcap/libpcap) to automatically handle the communications and data model, with [SWIG](http://www.swig.org) wrappers to link to a Python or Java program. All the communications is handled behind the scenes. It is useful for any application where sub-millisecond performance is not needed, because it offers the comfort and convenience of writing your control logic code in a high-level language.
 - An experimental JSON-based implementation of the IEC 61850 ACSI (in short, the spec for comms). A very lightweight HTTP/HTTPS stack makes the rapid61850 data model self-describing and accessible on-demand. This is a significantly simplified alternative to the MMS protocol. The use of JSON as the data format is easily supported by several programming languages, and especially JavaScript-based web apps.
 - Open source, under the GPL 2

You can read more about the motivation and benefits of the project [here](http://strathprints.strath.ac.uk/43427/1/S_Blair_Rapid_IEC_61850_preprint.pdf).

## Installation ##

This process has been tested on Windows and Ubuntu, but other Linux flavours and OS X should work too. Most steps only need to be completed once.

The software requires Eclipse, with the Eclipse Modeling Framework (EMF). Java Emitter Templates (JET) is needed for development, but not to run the code. It's easiest to start with the version of Eclipse that comes with the Modeling Tools bundle (see here: http://www.eclipse.org/downloads/). (If you are planning on using the Python or Java interfaces on Windows, it is best to use the 32-bit versions of Eclipse, and the JDK.)

There are two source code trees: `emf` (in Java), and `c` (obviously written in C). Each should be a separate project in Eclipse. The Java `emf` project directory is organised as follows:

 - `src/`
   - `rapid61850/`: code that does the bulk of the conversion from an SCD file to C code. The class `Main` contains the `main()` function for the project, and contains the filename for the input SCD file.
     - `templates/`: template classes that are generated by JET.
   - `ch/`: the EMF Java model implementation. These files are all automatically generated by EMF, but are included in the repo for convenience.
 - `model/`: the IEC 61850 XML Schema files. EMF uses these to generate the model.
 - `templates/`: the template source files used by JET.

### EMF import process ###

 1. Start Eclipse, with the Workspace set to the root of the repository directory, e.g., `/home/user/rapid61850` on Linux.
 2. Create an "EMF Project" called "emf", at the location of the repository code.
 3. Select "XML Schema" as the Model Importer type. Select all the IEC 61850 XML Schema documents in the `emf/model` directory.
 4. Select the three root packages that are imported (although, only `scl` is used). Click "Finish". This will re-generate some files in `emf/model`: scl.ecore, lcoordinates.ecore, lmaintenance.ecore, and SCL.genmodel.
 5. Create a new project of type "Convert Projects to JET Projects", and select the `emf` project. For the `emf` project, go to Project Properties > JET Settings, and set Template Containers to "templates", and Source Container to "src". Delete the `rapid61850/templates` directory in the root of `emf` that was created before JET was configured correctly.
 6. Open `SCL.genmodel` and right-click on the root of the model tree. Select "Show Properties View" and ensure that "Compliance Level" is set to "6.0". Right-click on the root again and select "Generate Model Code". This should re-generate the model implementation files (in the `emf/src/ch` directory), and set up the project properly for using the generated code.
 7. Two additional JAR libraries must be included for the project to compile. In the Project Properties for `emf`, go to Java Build Path > Libraries. Click on "Add External JARs..." and find `com.ibm.icu_4.4.2.v20110823.jar` and `org.eclipse.emf.query_1.2.100.v200903190031.jar` (or similar versions). These should be located in the "plugins" directory within the Eclipse installation.

### C code project example ###

An example SCD file and a `main.c` file are provided. Many of the other C files are generated automatically. For the C code to compile with Eclipse, you should:

 - If you plan to use the native, low-level C/C++ interface (as shown in [the next section](https://github.com/stevenblair/rapid61850#using-the-code-with-a-new-scd-file)), exclude the two `interface*.c` files from the build in Eclipse: right-click on the files > "Resource Configurations" > "Exclude from Build...", and then choose "Release" or "Debug" or another build. Also, exclude the `main_SV_LE.c` file, which provides an example implementation of IEC 61850-9-2LE Sampled Values, using `scd_LE.xml` as the SCD file. Otherwise, if using the high-level interfaces, exclude the existing `main.c` and `main_SV_LE.c` files.
 - Install MinGW and add `C:\MinGW\bin;` to `PATH` in the Project Properties > C/C++ Build > Environment options. (Other compilers should work too.)
 - In Project Properties > C/C++ Build > Settings > GCC Compiler Includes, set `"${workspace_loc:/${ProjName}/Include}"` as an include path.
 - In Project Properties > C/C++ Build > Settings > MinGW C Linker, add `wpcap` and `ws2_32` (assuming you are using Windows) to "Libraries" and add `"${workspace_loc:/${ProjName}/Lib}"` and `"C:\MinGW\lib"` to "Library search path".
   - With Linux, use `pcap` instead of `wpcap`, and just add `"${workspace_loc:/${ProjName}/Lib}"` to the  "Library search path".
 - The WinPcap library files and header files (from http://www.winpcap.org/devel.htm) have been included in the repository for convenience. The PC must also have the WinPcap driver installed (either by installing Wireshark, or from http://www.winpcap.org/install/default.htm).
   - With Ubuntu, libpcap can be installed using `sudo apt-get install libpcap-dev`.
   - Remember that, on Linux, **libpcap needs to run as root**, so either start Eclipse or run the compiled binary from the Terminal with `sudo`. Alternatively, you can grant the binary the [capability to access the network interface](http://packetlife.net/blog/2010/mar/19/sniffing-wireshark-non-root-user/) using: `sudo setcap cap_net_raw,cap_net_admin=eip /path_to_project/rapid61850/c/Release/c`.


## Using the code with a new SCD file ##

First, open the file `Main.java`. In the `Main` class, set the value of `SCD_FILENAME` to the filename of the SCD file. The SCD file should be in the same directory as the `Main.java` file. Run the Java project to generate the C implementation. **If the SCD parser complains, ensure that the first two lines of the SCD file exactly match those from the example `scd.xml` in the repository.** It's usually best to refresh the C project in Eclipse, to ensure that Eclipse knows about the new or modified files.

A basic C `main()` function will look something like:

```C
#include "iec61850.h"

int length = 0;
unsigned char buffer[2048] = {0};

int main() {
    initialise_iec61850();                                         // initialise all data structures

    // send GOOSE packet
    E1Q1SB1.S1.C1.TVTRa_1.Vol.instMag.f = 1.024;                   // set a value that appears in the dataset used by the "ItlPositions" GOOSE Control
    length = E1Q1SB1.S1.C1.LN0.ItlPositions.send(buffer, 1, 512);  // generate a goose packet, and store the bytes in "buffer"
    send_ethernet_packet(buffer, length);                          // platform-specific call to send an Ethernet packet


    // in another IED...


    // receive GOOSE or SV packet
    length = recv_ethernet_packet(buffer);                         // platform-specific call to receive an Ethernet packet
    gse_sv_packet_filter(buffer, length);                          // deals with any GOOSE or SV dataset that is able to be processed

    // read value that was updated by the packet (it will equal 1.024)
    float inputValue = D1Q1SB4.S1.C1.RSYNa_1.gse_inputs_ItlPositions.E1Q1SB1_C1_Positions.C1_TVTR_1_Vol_instMag.f;

    return 0;
}
```

The data structures used for generating GOOSE and SV packets are stored within `LN0`. GOOSE packets are generated by calling the appropriate `send(buffer, statusChange, timeAllowedToLive)` function, where `statusChange` should be `1` if any value in the dataset has changed, and where `timeAllowedToLive` is the time in milliseconds for the receiver to wait for the next re-transmission. SV packets are sent by calling the `update(buffer)` function, which returns `0` if the next ASDU was written, but other ASDUs are free. It returns the size of the packet when all ASDUs have been written (and `buffer` contains the packet data). Clearly, a real implementation might include the use of platform-specific timers, interrupts and callbacks, where needed.

The generated C code implements all IEDs specified in the SCD file. You can use the code to emulate the communications between several IEDs, or just use one IED's implementation.

### Set the local MAC address ###

In `ctypes.h`, set `LOCAL_MAC_ADDRESS_VALUE` to the local network interface's MAC address. This must be done for each physical device.

### Callbacks after a dataset is decoded ###

Callbacks should be set up in the form:

```C
void SVcallbackFunction(CTYPE_INT16U smpCnt) {
    ;
}

void GSEcallbackFunction(CTYPE_INT32U timeAllowedToLive, CTYPE_TIMESTAMP T, CTYPE_INT32U stNum, CTYPE_INT32U sqNum) {
    ;
}

//...

D1Q1SB4.S1.C1.exampleMMXU_1.sv_inputs_rmxuCB.datasetDecodeDone = &SVcallbackFunction;
D1Q1SB4.S1.C1.RSYNa_1.gse_inputs_ItlPositions.datasetDecodeDone = &GSEcallbackFunction;
```

where `D1Q1SB4.S1.C1.exampleMMXU_1` is a Logical Node defined in `datatypes.h` (and `ied.h`). `rmxuCB` is the name of the `SampledValueControl`, in a different IED, which sent the SV packets. After being initialised, the callback function will be executed after this dataset is successfully decoded, to allow the LN to deal with the new data. For example, by default, only one packet of data is saved for each GSE or SV Control - and is overwritten when a new packet arrives. Therefore, it may be useful to use the callback to log the data to a separate memory buffer.

### Fixed-length GOOSE encoding ###

To enable fixed-length GOOSE encoding, in `ctypes.h` set the value of `GOOSE_FIXED_SIZE` to `1`. Otherwise, it should have a value of `0`. This can only be enabled globally for all GOOSE encoding, rather than on a per Control basis.

### Platform-specific options ###

All platform-specific options can be edited in `ctypes.h` or `ctypes.c`. For example, for a big endian platform, change:

```C
#define LITTLE_ENDIAN       1
```

to:

```C
#define LITTLE_ENDIAN       0
```

All `CTYPE_*` definitions must map to local datatypes of the correct size and sign.

In `ctypes.c`, the basic library function `memcpy()` is used to copy bytes in order (according to platform endianness), and `reversememcpy()` copies the bytes of multi-byte data types in reverse order (for converting between endianness). Although these will work, they can be replaced with platform-specific alternatives for better performance.

The value of `TIMESTAMP_SUPPORTED` should be set to `0`, unless generating timestamps has been implemented for your platform. An implementation for Windows has been included by default.

## Using the JSON interface ##

*This functionality is highly experimental. Several data types have not been fully tested yet. There is only support for Windows at present, but Linux and OS X will be supported. At the moment, it will be difficult to use the JSON interface on an embedded platform.*

An "index" of the data model provided by rapid61850 is generated automatically. This fully exposes the data model, including all meta data (such as data types and functional constraints). A JavaScript object notation (JSON) interface has been specified for implementing the IEC 61850 abstract communication service interface (ACSI), and the JSON interface is exposed via HTTP (or HTTPS).

[Mongoose](https://github.com/cesanta/mongoose), which is embedded in the repository, provides a simple and lightweight web server. A new thread is spawned for each IED; this allows multiple IEDs to be tested together from a single application. (Note: no locking has been implemented for the data model.)

### Building the code ###

 1. In the C project build settings, add `"${workspace_loc:/${ProjName}/src}"` as an include path. The ensures the JSON code can access the other header files.
 2. 

### Using SSL to encrypt all connections ###

 1. Install OpenSSL for your operating system.
 2. In the C project build settings:
   a. define the symbol `USE_SSL`
   b. link to the library `ssl32`
   b. add the linker search path to the OpenSSL `bin` directory (e.g., `"C:\OpenSSL-Win32\bin"` on Windows)
 3. Ensure that the SSL certificate (ssl_cert.pem) is in the appropriate directory: typically at the root of the `C` directory if running from Eclipse. WARNING: the included certificate file is for testing only. Generate or purchase a new certificate for production purposes.
 4. If you wish to use HTTP authentication, set `USE_HTTP_AUTH` to `1` in `json.h`. Create your password file called `htpasswd.txt`, in the same directory as the SSL certificate. Mongoose (as well as various web sites) can be used to help create the MD5 hash: see `main_json.c`.

## Using the Python or Java interfaces ##

So far, this readme has described how to use the native C/C++ interface. It's also possible to use [SWIG](http://www.swig.org/) to automatically generate wrappers for high-level languages from C/C++ header files. At the moment, Python and Java interfaces on Windows and Linux have been tested, but other languages (such as C#, Lua, Perl, Ruby, etc.) should work too. (You can also use the high-level interface from C/C++ too, but not alongside the native interface.)

Four C files, with filenames `interface*`, are generated along with the rest of the GOOSE/SV code. These files, and the SWIG interface file `rapid61850.i`, are used as the input to SWIG. They contain functions to start a (platform-dependent) network interface using winpcap/libpcap, and functions to send GOOSE or SV packets using that network interface. All of the interaction with pcap is done in C, and is hidden by the interface given to SWIG.

Note that this interface can also be used within a C/C++ application - this is shown in the example `main.c` file, if `HIGH_LEVEL_INTERFACE` is defined as `1`. If your are not using this high-level interface, and are using the plain C interface, you may need to exclude the two `interface*.c` files from the build in Eclipse.

### Building on Windows ###

If using MinGW as the C compiler (as described above), this process is significantly simpler if the 32-bit versions of Eclipse and the JDK are used. The following instructions assume this. It's also assumed that the Python or Java application exists within a directory at the same level as the `emf` and `c` directories.

 - Generate the C code for your SCD file, as described above.
 - [Download](http://www.swig.org/download.html) `swigwin`, which is a pre-compiled binary of SWIG for Windows. Once unzipped, there are two options for using this:
   - Add the location of `swig.exe` to the Windows `PATH` environment variable.
   - Or, copy the contents of the swigwin directory (i.e., copy `swig.exe` *and* all the sub-folders) to the `c/src` directory. You will need to tell Eclipse to exclude these directories from the build.
 - Create the directory for your Python or Java program called, for example, `python_interface` or `java_interface`. You may wish to make this an Eclipse PyDev or Java project.
 - Open a command prompt at the `c/src` directory, and run SWIG using one of the following commands:

    For Python:

        swig -python -outdir ..\..\python_interface rapid61850.i

      For Java:

        swig -java -outdir ..\..\java_interface rapid61850.i

The following subsections explain how to change the compiler settings for the `c` project to generate a dynamic library, instead of an executable. This differs for Python and Java. It may be helpful to create different build configurations in Eclipse if you need to use more than one of the C/C++, Python, or Java interfaces. You may also need to exclude the existing `main.c` file from any Python or Java builds.

#### Python interface C compiler settings ####

 - In C/C++ Build > Settings > Build Artifact:
   - set Artifact Type to `Shared Library`
   - set Artifact name to `rapid61850`
   - set Artifact extension to `pyd`
   - set Output prefix to `_`
 - In C/C++ Build > Settings > Tool Settings > Includes, use the following Include Paths (**adjust these to match the exact version and location of Python on your system**):
   - `"C:\Python27"`
   - `"C:\Python27\include"`
   - `"C:\Python27\Lib"`
   - `"${workspace_loc:/${ProjName}/Include}"`
 - In C/C++ Build > Settings > Tool Settings > Libraries, use the following Libraries (-l):
   - `wpcap`, `python27`, and `ws2_32`. (Again, adjust `python27` to the correct version.)
 - In C/C++ Build > Settings > Tool Settings > Libraries, use the following Libraries search paths (-L):
   - `"${workspace_loc:/${ProjName}/Lib}"`
   - `"C:\Python27\libs"`
 - Build the C project, and copy the `_rapid61850.pyd` file from the Release folder to the `python_interface` project directory.
 - Create and run your Python code, e.g.:

    ```python
    import rapid61850
    from rapid61850 import *

    rapid61850.start()

    rapid61850.interface_gse_send_D1Q1SB4_C1_MMXUResult(1, 512)     # send GOOSE packet

    rapid61850.cvar.E1Q1SB1.S1.C1.LPHDa_1.Mod.stVal = MOD_ON_1      # interact with IED data model
    print rapid61850.cvar.E1Q1SB1.S1.C1.LPHDa_1.Mod.stVal
    ```

    Note that all C global variables appear within `rapid61850.cvar`.

#### Java interface C compiler settings ####

 - In C/C++ Build > Settings > Build Artifact:
   - set Artifact Type to `Shared Library`
   - set Artifact name to `rapid61850`
   - set Artifact extension to `dll`
   - leave Output prefix blank
 - In C/C++ Build > Settings > Tool Settings > Includes, use the following Include Paths (**adjust these to match the exact version and location of Java on your system**):
   - `"C:\Program Files (x86)\Java\jdk1.7.0_03\include"`
   - `"C:\Program Files (x86)\Java\jdk1.7.0_03\include\win32"`
   - `"${workspace_loc:/${ProjName}/Include}"`
 - In C/C++ Build > Settings > Tool Settings > Libraries, use the following Library search path (-L):
   - `"${workspace_loc:/${ProjName}/Lib}"`
 - In In C/C++ Build > Settings > Tool Settings > Miscellaneous, add `-Wl,-add-stdcall-alias` to the Linker flags
 - Build the C project, and copy the `rapid61850.dll` file from the Release folder to the `java_interface` project directory.
 - Create your Java code, e.g.:

    ```java
    public class Main {
        static {
            System.loadLibrary("rapid61850");
        }

        public static void main(String[] args) {
            rapid61850.start();

            System.out.println(rapid61850.interface_gse_send_E1Q1SB1_C1_Performance(1, 512));                 // send GOOSE packet
    
            rapid61850.getE1Q1SB1().getS1().getC1().getMMXUa_1().getMod().setStVal(Mod.MOD_ON_1);             // interact with IED data model
            System.out.println(rapid61850.getE1Q1SB1().getS1().getC1().getMMXUa_1().getMod().getStVal());
        }
    }
    ```
 - To run the Java program, you first need to specify the path to the native library. In Project Properties > Java Build Path > Libraries, expand the "JRE System Library" tree and select "Native library location". Click on "Edit..." and enter the project name (e.g., `java_interface`) as the Location path. Note that if the interface changes, such as due to changes in the SCD file, then all `.java` and `.class` files generated by SWIG should be deleted before a new dynamic library is compiled and used by the Java program.


### Building on Linux ###

On Linux, it's easier to create the Python or Java interface with the Terminal, rather than with Eclipse. The steps below assume Ubuntu (and have only been tested on 11.10 64-bit), so it may differ on other distributions. **Note that because the following scripts compile all *.c files, the example `main*.c` files may need to be deleted from the C source code directory.**

Install the following packages:

```sh
sudo apt-get install libpcap-dev
sudo apt-get install swig
sudo apt-get install build-essential
sudo apt-get install python2.7         # other Python versions should be ok
sudo apt-get install openjdk-6-jdk
```
<!--are there any more?-->

Open a Terminal at the `rapid61850/c/src` directory.

#### Python ####

```sh
# attempt to clean up any previous files
rm *.o *.so *_wrap.c rapid61850.py rapid61850.pyc

# run SWIG, output goes in current directory
swig -python rapid61850.i

# compile and link the C library
gcc -fPIC -c *.c -I/usr/include/python2.7
gcc -shared *.o -lpcap -o _rapid61850.so

# run Python. sudo is needed for the network interface
sudo python2.7

# example Python program:
>>> import rapid61850
>>> rapid61850.start()
>>> print rapid61850.interface_gse_send_D1Q1SB4_C1_MMXUResult(1, 512)
332
>>> exit()
```

#### Java ####

```sh
# attempt to clean up any previous files
rm *.o *.so *_wrap.c java/*.class java/*.java

mkdir java    # only needed once

# run SWIG, and put the .java files (there will be a lot) in the "java" sub-directory
swig -java -outdir java rapid61850.i

# compile and link the C library
gcc -fPIC -c *.c -I/usr/lib/jvm/java-6-openjdk/include -I/usr/lib/jvm/java-6-openjdk/include/linux
ld -G *.o -lpcap -o librapid61850.so

# compile all .java files, including the sample program
javac -d java/ java/*.java ../../java_interface/Main.java

# run the sample Java program. sudo is needed for the network interface
cd java
sudo java -Djava.library.path=/home/steven/rapid61850/c/src/ Main    # this path must be set correctly
```

## Software documentation ##

### Validation ###

`SCDValidator.java` performs semantic validation of the SCD file, prior to code generation. It checks for the following constraints:

 - Where necessary, the names of IEDs, logical nodes, data sets and data types are unique.
 - Each Control instance has matching DataSet and ControlBlock instances.
 - Each logical node “Input” has a corresponding source in a data set (typically in another IED).
 - No circular sub-data object (SDO) references occur.
 - Data attributes, basic data attributes and SDOs must map to valid types that exist in the SCD file.
 - Data type definitions appear in a hierarchical order, that will result in valid generated code.

As shown in `Main.java`, the validation process is separate from the code generation process. Therefore, it's possible to reuse the validation process in other software, if needed.

The validation process extensively uses the [EMF Model Query](http://help.eclipse.org/galileo/index.jsp?topic=/org.eclipse.emf.query.doc/tutorials/queryTutorial.html) framework for searching and filtering SCD data. It uses a SQL-like syntax. For example, to find all IEDs in the SCD document object `root`:

```java
public void checkForDuplicateNames(DocumentRoot root) {
    // describe a condition: is the object an IED (called `TIED` in EMF)?
    final EObjectCondition isIED = new EObjectTypeRelationCondition(
        SclPackage.eINSTANCE.getTIED()
    );
    
    // build and execute a query
    IQueryResult iedResult = new SELECT(
        new FROM(root),
        new WHERE(isIED)
    ).execute();

    // loop through results
    for (Object o : iedResult) {
        TIED ied = (TIED) o;
        String iedName = ied.getName();

        // ...
    }
}
```

### Augmented IEC 61850-6 SCL model ###

`SCDAdditionalMappings.java` uses hash maps to explicitly link parts of the Java representation of the SCL model. This greatly simplifies the code generation process. (In the SCL, these links are implicit and are achieved by string-matching.) Each mapping is as follows:

 - Each DAI is mapped to an AbstractDataAttribute, which has the sub-type of either DA or BDA, which defines the type of the DAI value.
 - Each Control, which has the sub-types GSEControl and SampledValueControl, is mapped to the matching DataSet for convenience.
 - ExtRefs are mapped to all DataSets, typically in other IEDs, which satisfy the ExtRef.
 - Instances of BaseElement, which is the super-class of DO, DA, and BDA, are mapped to Strings which contain pre-calculated C code.
 - FCDAs are mapped to:
   - The data item to which the FCDA refers, which may be a DO, DA, or BDA (all of which are sub-classes of BaseElement). Therefore, the type of the FCDA can be inferred.
   - The LN instance which contains the source data.
  - A String which is the unique name of the FCDA used in C code generation.

### Code generation ###

The following UML class diagram illustrates how a generic representation of a C file is used by the code generation process:

<img src="http://personal.strath.ac.uk/steven.m.blair/CFile-UML.png" />

Java Emitter Template (JET) files (`CSourceTemplate` and `CHeaderTemplate`) are used to define the generic structure of C source and header files. This approach allows several header files, which specify function prototypes, to be generated automatically from the `CSource` objects.

## Notes and possible features ##

 - Some data types are not supported yet. However, the main *useful* data types (integer, floating-point, and boolean) are supported.
 - FCDAs and ExtRefs cannot use the syntax `vector.mag.f` as values for data object or data attribute references.
 - Data types cannot contain arrays.
 - According to [the standard](http://www.tissues.iec61850.com/tissue.mspx?issueid=579), SV datasets should only contain primitive data types, and not constructed types. However, because SV encoding involves fixed-length value fields, it is always possible to reconstruct the data, if encoded and decoded consistently. Therefore, this library will allow constructed types to be encoded in SV packets. Semantically, SV datasets should only contain data values that have been sampled at the specified sampling rate. Again, for practicality, this library allows any DA or DO to be used in SV datasets.
 - Support for trigger options is not implemented at present.
