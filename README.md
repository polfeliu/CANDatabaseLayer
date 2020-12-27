# CANDatabaseLayer
*Warning!* This is a work in progress and it doesn't have proper testing done to it. Use at your own risk!

This Python module extracts information from CANdb Databases and generates handy C code for integrating it into embedded projects. It is environment agnostic, it can be integrated into any MCU in theory. Example files to integrate it into STM32 using HAL from ST are included in this repo.\
The candb library used for reading the dbc files ([cantools](https://cantools.readthedocs.io/en/latest/)) also accepts other formats but it hasn't been tested.
## Install
The best way right to use this module is to clone the repository and install the pipenv inside it.
```bash
git clone https://github.com/polfeliu/CANDatabaseLayer
cd CANDatabaseLayer
pip install pipenv #if you don't have pipenv...
pipenv install
pipenv shell
```
There you can edit the main.py and set the parameters as you need. \
Someday I may create a .exe file with pyinstaller but wrapping all the settings into it is too much work right now.


## Usage
To generate the files you can either edit the last part of main.py (that generates the test) or create a new python file and import the module from it.
When creating the can object you will choose the prefix that all the objects will have. All the prefixes in this readme are "CAN1".
```python
can = CANDatabaseLayer("CAN1") # create the object setting the prefix that will be used in all objects.
Handy if you have several networks

can.load('./test/CAN1.dbc')# Import database

src = r'./test/STM32TestProject/Core/Src/' #Where Source files should be copied to
hdr = r'./test/STM32TestProject/Core/Inc/' #Where Header files should be copied to

can.correctMinsMax() # Correct the minimums and maximums
can.process(node="Engine") #Generate for the Node Engine
can.process() # you can also generate all messages and signals without specifying the node

can.genFiles(srcfile=src, hdrfile=hdr);
can.reset() #you can reset the data to be able to generate a new Node in another place
    
shutil.copyfile(r'STM32CANCallbacks.c', src + r'STM32CANCallbacks.c') #Copy the STM32CANCallbacks
shutil.copyfile(r'STM32CANCallbacks.h', hdr + r'STM32CANCallbacks.h') #Copy the STM32CANCallbacks
```

The function process will extract the signals and frames of the database. The node parameter passed indicates which node is the code for, meaning only the frames and signals necessary will be in the code. Also if the frame is TX it will only have send() method, and if it is RX it will be receive(). Signals will always have getValue() methods and only setValue() methods if they are TX. \
Even though RX messages may have signals that are not RX declared in the database but are actually received they won't have an object to access them, forcing you to keep the database consistent if you want to access the signals ;). \
If no node is passed to the process function (``` can.process()```) the function will calculate all the frames signals and methods. 

### Settings
The generation library has a set of settings that are documented across this readme. They can be edited directly on the main.py or access them or after creating the instance

``` python
can.settings["CallbackLib"] = "STM32CANCallbacks" #name of the library that implementes the Callback functions for this bus
```

### Mins and Max
I've found that many databases tend to have the mins and maxs not calculated, because in the CANdb++ editor you have to click a button to update them according to the factors, offsets and datatype. \
To prevent mysterious errors when executing the program I programmed a cool optional feature: When you process a signal it checks that the mins and max are correct. If a signal fails the program halts and prints an error.\
The function correctMinsMax() can be called before processing the node and it simply recalculates all the mins and maxs to the cache.
To store the new mins and maxs you can call the function ``can.save("file")``.

### Filters
The library calculates a single hardware filter (filter and mask) for the RX messages of the node. This can be used to reduce the software load. The filters are declared in the header file

```c
#define CAN1_StandardFilter   0b00001100111
#define CAN1_StandardMask     0b11111111111
#define CAN1_ExtendedFilter   0b00000000000000000000000000000
#define CAN1_ExtendedMask     0b11111111111111111111111111111
```

In the STM32CANCallbacks.c files I've included an initialization of the CAN peripheral and Interaction Layer that includes the configuration of the bxCAN filter.

## Structure
The library generates DatabaseLayer.c and DatabaseLayer.h files with structures and functions for the following objects.

![alt text](https://github.com/polfeliu/CANDatabaseLayer/blob/main/diagram.png)

### Signals
The signal structure contains basic information of the signal provided by the database (length, byte_order, unit, initial_value, factor, offset, min, max) and own elements of the library:
* **value_type**: indicates in what structure the raw signal will be stored
* **raw**: stores the raw signal
* **getValue**: pointer to function that returns the physical value according to the factor and offset
* **setValue**: pointer to function that sets the raw value with the physical value passed according to the factor and offset

```c
struct CAN1sig_SleepInd_t CAN1sig_SleepInd = {
    .length = 1, 
    .byte_order = little_endian,
    .value_type = Tbool,
    .unit = "",
    .initial_value = 0,
    .factor = 1,
    .offset = 0,
    .min = 0,
    .max = 1,
    .raw = 0,
    .getValue = CAN1sig_SleepInd_getValue,
    .setValue = CAN1sig_SleepInd_setValue
};
```

#### getValue
```c
sleep = CAN1sig_SleepInd.getValue()
```
#### setValue
```c
sat = CAN1sig_SleepInd.setValue(34)
```
setValue checks if the value passed exceeds the signal maximum and minimum and saturates it accordingly. It also returns a boolean that informs if the value saturated.

*Warning!* As of now I have only tested signals with little_endian byte_orders and now. The results from using big_endian would be catastrofic, maybe the getValue and setValue could have a bit reversal feature.\
I assume you are using a little-endian architecture. Otherwise, since you seem to be looking for trouble, I recommend against using this library, as it can be **too** practical.

### Frames
Frames have similar structs with data from the database (ID, is_extended, DLC) and:
* **raw**: holds the raw message
* **receive**: unpacks the raw message and store the values of the signals to the signals raw parameter
* **send**:  packs all the signals to the frame raw parameter and calls CAN1_SendCallback() with the frame calculated (more on that later)
* **signals**: structures of the signals with start position inside the signal, a mask for the assembly of the frames and a pointer to the signal struct

The pointer can be used to access the signals of a message like that:
```c
CAN1_NM_Engine.signals.CAN1sig_SleepInd.signal->setValue(34);
```
```c
struct CAN1_NM_Engine_t CAN1_NM_Engine = {
    .ID = 0x51b, //dec: 1307
    .is_extended = false,
    .DLC = 4,
    .raw = 0,
    .receive = CAN1_NM_Engine_receive,
    .send = CAN1_NM_Engine_send,
    .signals = {
        .CAN1sig_SleepInd = {
            .signal = &CAN1sig_SleepInd,
            .startbit = 12,
            .mask = 0b1
        },
        
    }
};

```
send and receive methods can be used like that:
```c
CAN1_NM_Engine.send()

CAN1_NM_Engine.raw = data; #You must first update the raw value of the message with what you are receiving 
CAN1_NM_Engine.receive();
```

### CallBacks
#### Receive
The libray also defines a receive callback (*CAN1_ReceiveCallback*) that you can use to handle incoming messages and store it in the respective structures according to the ID.
When your driver receives a message it should call this function with the appropiate parameters. This function is driver agnostic and its stored in the DatabaseLayer file.
```c
void CAN1_ReceiveCallback(uint64_t data, uint32_t ID, bool is_extended, uint8_t DLC){
    if((ID > 0x1FFFFFFF) & is_extended){
        //invalid extended identifier
    }else if((ID > 0x7FF) & !is_extended){
        //invalid standard identifier
    }

    else if((ID==CAN1_NM_Engine.ID) & (is_extended==CAN1_NM_Engine.is_extended)){
        CAN1_NM_Engine.raw = data;
        CAN1_NM_Engine.receive();
    }

    else if((ID==CAN1_NM_Gateway_PowerTrain.ID) & (is_extended==CAN1_NM_Gateway_PowerTrain.is_extended)){
        CAN1_NM_Gateway_PowerTrain.raw = data;
        CAN1_NM_Gateway_PowerTrain.receive();
    }
```

In the STM32CANCallbacks files you can see and example on how this can be done for STM32 using HAL
#### Transmit
You should define a Transmit (*CAN1_SendCallback*) callback so that the messages are sent after being assembled. \
This function will be called with the *data, id, is_extended, DLC* parameters by each send() command.\
An example of this is also included in the STM32CANCallbacks files.

## Interaction Layer
Optionally, the Library can also generate FreeRTOS tasks for sending the messages. Right now the only option is cyclic, and it assumes that all messages have to be sent periodically according the Cyclic time set on the Database.\
The messages are grouped by cyclic times and a task is declared for each group.\
The Interaction Layer can be started by simply calling:
```c
CAN1_InteractionLayerInit()
```

