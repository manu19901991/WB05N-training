# STM32WB05N gatt server basic examples with minimal set of functions only in main.c

# add the below includes at the beginning of main.c in /* USER CODE BEGIN Includes *


The following header files are included for various functionalities related to STM32WB05N microcontroller and Bluetooth communication:

- `#include "stm32wb05n_hal.h"`: HAL (Hardware Abstraction Layer) for STM32WB05N.
- `#include "stm32wb05n_hal_aci.h"`: ACI (Application Control Interface) layer for HAL.
- `#include "stm32wb05n_hci_le.h"`: LE (Low Energy) layer of HCI (Host Controller Interface).
- `#include "stm32wb05n_gatt_aci.h"`: GATT (Generic Attribute Profile) ACI layer.
- `#include "stm32wb05n_gap_aci.h"`: GAP (Generic Access Profile) ACI layer.
- `#include "hci_const.h"`: Constants and macros for HCI.
- `#include "hci.h"`: Main header file for HCI functions.
- `#include "hci_tl.h"`: Transport Layer for HCI.


```c=
#include "stm32wb05n_hal.h"
#include "stm32wb05n_hal_aci.h"
#include "stm32wb05n_hci_le.h"
#include "stm32wb05n_gatt_aci.h"
#include "stm32wb05n_gap_aci.h"
#include "hci_const.h"
#include "hci.h"
#include "hci_tl.h"
```
---

# add the below includes at the beginning of main.c in * USER CODE BEGIN PTD */


The following macros and definitions are used for the STM32WB05N microcontroller and Bluetooth communication:

- `#define SENSOR_DEMO_NAME 'S','T','M','3','2','W','B','0','5'`: Defines the name for the our target as "STM32WB05".
- `#define BDADDR_SIZE 6`: Defines the Bluetooth device address size as 6 bytes.
- `#define CHAR_VALUE_LENGTH 20`: Defines the characteristic value length as 20 bytes.
- `#define HOST_TO_LE_16(buf, val) (((buf)[0] = (uint8_t) (val) ), \ ((buf)[1] = (uint8_t) (val>>8) ))`: Converts a 16-bit host value to little-endian format and stores it in the buffer (used to send notification, we will see it later)


```c=

#define SENSOR_DEMO_NAME   'S','T','M','3','2','W','B','0','5'
#define BDADDR_SIZE        6
#define CHAR_VALUE_LENGTH 20
#define HOST_TO_LE_16(buf, val)    (((buf)[0] =  (uint8_t) (val)    ) , \
                                    ((buf)[1] =  (uint8_t) (val>>8) ) )

```

# add the following variables

- `uint16_t sampleServHandle, TXCharHandle, RXCharHandle;`:
  - Handles for the sample service and its characteristics (TX and RX).

- Static variables for maintaining connection state:
  - `static uint16_t connection_handle;`: Handle for the active connection.
  - `static uint8_t set_connectable;`: Flag to set the device as connectable.
  - `static uint8_t connected;`: Flag to indicate connection status.
  - `static uint8_t pairing;`: Flag to indicate pairing process.
  - `static uint8_t paired;`: Flag to indicate if paired successfully.
  - `static uint8_t send_env;`: Flag to send data.
  - `static uint8_t reception;`: Flag to handle reception of data.

- Redundant handles for the sample service and its characteristics (TX and RX):
  - `uint16_t sampleServHandle, TXCharHandle, RXCharHandle;`.


```c=
/* USER CODE BEGIN PFP */
uint16_t sampleServHandle, TXCharHandle, RXCharHandle;
static uint16_t connection_handle;
static uint8_t set_connectable;
static uint8_t connected;
static uint8_t pairing;
static uint8_t paired;
static uint8_t send_env;
static uint8_t reception;
uint16_t sampleServHandle, TXCharHandle, RXCharHandle;
/* USER CODE END PFP */
```

# initialize HCI interface in user code begin 2

The following function initializes the Host Controller Interface (HCI) with the application event receiver callback:

- `hci_init(APP_UserEvtRx, NULL);`:
  - Initializes the HCI with the user event reception function `APP_UserEvtRx`.
  - The second parameter is set to `NULL`, indicating no additional parameters are passed.

APP_UserEvtRx is the part in which the events coming from HCI are elaborated

```c=
hci_init(APP_UserEvtRx, NULL);
```




# some variables

- `uint8_t ret;`: Return value for function calls.
- `uint16_t service_handle, dev_name_char_handle, appearance_char_handle;`:
  - `service_handle`: Handle for the Bluetooth service.
  - `dev_name_char_handle`: Handle for the device name characteristic.
  - `appearance_char_handle`: Handle for the appearance characteristic.

- `uint8_t device_name[] = {SENSOR_DEMO_NAME};`:
  - Array containing the device name "STM32WB05".

- `uint8_t bdaddr[BDADDR_SIZE] = {0, 1, 2, 3, 4, 5};`:
  - Array containing the Bluetooth device address (MAC address).

```c=
uint8_t ret;
uint16_t service_handle, dev_name_char_handle, appearance_char_handle;
uint8_t device_name[] = {SENSOR_DEMO_NAME};
uint8_t bdaddr[BDADDR_SIZE] = {0, 1, 2, 3, 4, 5}; // this is the mac address
```

# Create a protoype for APP_UserEvtRx

- `void APP_UserEvtRx(void *pData);`:
  - Function to receive and process user events.
  - The parameter `void *pData` is a pointer to the event data.

Put it in USER CODE BEGIN PFP
```c=
  void APP_UserEvtRx(void *pData);
```

# add the declaration in USER CODE BEGIN 4

**Function Implementation**:

- The function processes HCI (Host Controller Interface) packets received via UART:
  - Checks if the packet type is `HCI_EVENT_PKT` or `HCI_EVENT_EXT_PKT`.
  - Extracts the event packet from the HCI packet.
  - Processes the event based on its type:
    - If it's `EVT_LE_META_EVENT`, it processes LE Meta Events.
    - If it's `EVT_VENDOR`, it processes vendor-specific events.
    - Otherwise, it processes other HCI events.

```c=
void APP_UserEvtRx(void *pData)
{
  uint32_t i;

  hci_spi_pckt *hci_pckt = (hci_spi_pckt *)pData;

  if (hci_pckt->type == HCI_EVENT_PKT || hci_pckt->type == HCI_EVENT_EXT_PKT)
  {
    void *data;
    hci_event_pckt *event_pckt = (hci_event_pckt *)hci_pckt->data;

    if (hci_pckt->type == HCI_EVENT_PKT)
    {
      data = event_pckt->data;
    }
    else
    {
      hci_event_ext_pckt *event_pckt = (hci_event_ext_pckt *)hci_pckt->data;
      data = event_pckt->data;
    }

    if (event_pckt->evt == EVT_LE_META_EVENT)
    {
      evt_le_meta_event *evt = data;

      for (i = 0; i < (sizeof(hci_le_meta_events_table)/sizeof(hci_le_meta_events_table_type)); i++)
      {
        if (evt->subevent == hci_le_meta_events_table[i].evt_code)
        {
          hci_le_meta_events_table[i].process((void *)evt->data);
          break;
        }
      }
    }
    else if (event_pckt->evt == EVT_VENDOR)
    {
      evt_blue_aci *blue_evt = data;

      for (i = 0; i < (sizeof(hci_vendor_specific_events_table)/sizeof(hci_vendor_specific_events_table_type)); i++)
      {
        if (blue_evt->ecode == hci_vendor_specific_events_table[i].evt_code)
        {
          hci_vendor_specific_events_table[i].process((void *)

```

# Back to the main.c after HCI init


- `ret = hci_reset();`:
  - Resets the Host Controller Interface (HCI) and stores the return value in `ret`.

```c=
    ret = hci_reset();
    if(ret != 0) {
      Error_Handler();
  while(1);
    }
    else {

      HAL_Delay(2000);
    }

```




# SET TX POWER


The following code sets the transmit power level for the Bluetooth device:

- `ret = aci_hal_set_tx_power_level(0, 25);`:
  - Sets the transmit power level.
  - The first parameter `0` indicates the default power level.
  - The second parameter `25` specifies the power level in dBm (decibels relative to one milliwatt, conversion table is visible in the app note).
  - The return value `ret` is used to check for errors in setting the 

```c=
 ret = aci_hal_set_tx_power_level(0, 25);
       if (ret != BLE_STATUS_SUCCESS) {
         Error_Handler();
       }
       else {
         HAL_Delay(100);
       }
```

# Initialize GATT

# Initialize GATT Server

The following code initializes the Generic Attribute (GATT) server:

- `ret = aci_gatt_srv_init();`:
  - Initializes the GATT server.
  - The function sets up the necessary services and characteristics for the Bluetooth GATT server.
  - The return value `ret` is used to check for errors during the initialization process.

```c=
ret = aci_gatt_srv_init();
       if (ret != BLE_STATUS_SUCCESS) {
         Error_Handler();
         return ret;
       }
       else {
         HAL_Delay(100);
       }
```


# Initialize GAP

The following code initializes the Generic Access Profile (GAP) for the Bluetooth device:

- `ret = aci_gap_init(GAP_PERIPHERAL_ROLE, 0x00, 0x07, STATIC_RANDOM_ADDR, &service_handle, &dev_name_char_handle, &appearance_char_handle);`:
  - Initializes the GAP with the specified parameters:
    - `GAP_PERIPHERAL_ROLE`: The role of the device (Peripheral).
    - `0x00`: Privacy type.
    - `0x07`: Device name length.
    - `STATIC_RANDOM_ADDR`: Type of address.
    - `&service_handle`: Pointer to the service handle.
    - `&dev_name_char_handle`: Pointer to the device name characteristic handle.
    - `&appearance_char_handle`: Pointer to the appearance characteristic handle.
  - The return value `ret` is used to check for errors during initialization.

```c=

 ret = aci_gap_init(GAP_PERIPHERAL_ROLE, 0x00, 0x07, STATIC_RANDOM_ADDR, &service_handle, &dev_name_char_handle,
                   &appearance_char_handle);
if (ret != BLE_STATUS_SUCCESS) {
    Error_Handler();
    return ret;
} else {
    HAL_Delay(100);
}


```


# update device name

- `ret = aci_gatt_srv_write_handle_value_nwk(dev_name_char_handle + 1, 0, sizeof(device_name), device_name);`:
  - Writes the device name to the GATT server.
  - The function parameters are:
    - `dev_name_char_handle + 1`: Handle for the device name characteristic.
    - `0`: Offset value.
    - `sizeof(device_name)`: Size of the device name.
    - `device_name`: Pointer to the device name array.
  - The return value `ret` is used to check for errors during the write operation.

```c=

ret = aci_gatt_srv_write_handle_value_nwk(dev_name_char_handle + 1, 0, sizeof(device_name), device_name);
if (ret != BLE_STATUS_SUCCESS) {
    Error_Handler();
    return ret;
} else {
    HAL_Delay(100);
}



```


# SET ADV PARAMS

# Advertising Data

The following code sets up the advertising data for the Bluetooth device:

- `Advertising_Set_Parameters_t Advertising_Set_Parameters[1];`:
  - Declares an array of Advertising Set Parameters.

- `uint8_t adv_data[] = {...};`:
  - Array containing the advertising data:
    - `0x02, AD_TYPE_FLAGS, FLAG_BIT_LE_GENERAL_DISCOVERABLE_MODE | FLAG_BIT_BR_EDR_NOT_SUPPORTED`: Sets the advertising flags. The `2` indicates the length of the advertising flags data that follows.
    - `2, 0x0A, 0x00`: Sets the transmission power to 0 dBm. The `2` indicates the length of the transmission power data that follows.
    - `10, 0x09, SENSOR_DEMO_NAME`: Sets the complete name as "STM32WB05". The `10` indicates the length of the complete name data that follows (9 bytes for "STM32WB05" plus 1 byte for the type).
    - `13, 0xFF, 0x01`: Sets the SKD version. The `13` indicates the length of the SKD version data that follows.
    - `0x00`: Generic device.
    - `0x00`: Reserved byte.
    - `0xF4`: Reserved byte.
    - `0x00`: Reserved byte.
    - `0x00`: Reserved byte.
    - `bdaddr[5], bdaddr[4], bdaddr[3], bdaddr[2], bdaddr[1], bdaddr[0]`: Sets the BLE MAC address (starting from the most significant byte to the least significant byte).


```c=

Advertising_Set_Parameters_t Advertising_Set_Parameters[1];
uint8_t adv_data[] =
{
  0x02, AD_TYPE_FLAGS, FLAG_BIT_LE_GENERAL_DISCOVERABLE_MODE | FLAG_BIT_BR_EDR_NOT_SUPPORTED,
  2, 0x0A, 0x00, /* Transmission Power (0 dBm) */
  10, 0x09, SENSOR_DEMO_NAME,  /* Complete Name */
  13, 0xFF, 0x01, /* SKD version */
  0x00, /* generic device */
  0x00,
  0xF4, 
  0x00, /*  */
  0x00, /*  */
  bdaddr[5], /* BLE MAC start - MSB first - */
  bdaddr[4],
  bdaddr[3],
  bdaddr[2],
  bdaddr[1],
  bdaddr[0]  /* BLE MAC stop */
};



```

# Start advertising


# Advertising Configuration and Data

The following code configures and sets the advertising parameters and data for the Bluetooth device:

- `ret = aci_gap_set_advertising_configuration(...)`:
  - Configures the advertising parameters for the Bluetooth device.
  - The parameters include:
    - `GAP_MODE_GENERAL_DISCOVERABLE`: Sets the device in general discoverable mode.
    - `ADV_PROP_CONNECTABLE | ADV_PROP_SCANNABLE | ADV_PROP_LEGACY`: Advertising properties.
    - `ADV_INTERV_MIN`, `ADV_INTERV_MAX`: Minimum and maximum advertising intervals.
    - `ADV_CH_ALL`: Advertising channels.
    - `STATIC_RANDOM_ADDR`: Type of address.
    - `NULL`: No peer address.
    - `ADV_NO_WHITE_LIST_USE`: Do not use the white list.
    - `0`: Transmission power (0 dBm).
    - `LE_1M_PHY`: Primary advertising PHY.
    - `0`: Number of advertising event skips.
    - `LE_1M_PHY`: Secondary advertising PHY (not used with legacy advertising).
    - `0`: Advertising SID.
    - `0`: No scan request notifications.
 
- `ret = aci_gap_set_advertising_data_nwk(...)`:
  - Sets the advertising data for the Bluetooth device.
  - The parameters include:
    - `0`: Advertising handle.
    - `ADV_COMPLETE_DATA`: Data type.
    - `sizeof(adv_data)`: Size of the advertising data.
    - `adv_data`: Pointer to the advertising data array.
 
 - `Advertising_Set_Parameters[0].Advertising_Handle = 0`:
  - Sets the advertising handle.

- `Advertising_Set_Parameters[0].Duration = 0`:
  - Sets the advertising duration.

- `Advertising_Set_Parameters[0].Max_Extended_Advertising_Events = 0`:
  - Sets the maximum number of extended advertising events.

- `ret = aci_gap_set_advertising_enable(...)`:
  - Enables advertising with the specified parameters.
 


```c=
ret = aci_gap_set_advertising_configuration(0, GAP_MODE_GENERAL_DISCOVERABLE,
                                            ADV_PROP_CONNECTABLE | ADV_PROP_SCANNABLE | ADV_PROP_LEGACY,
                                            ADV_INTERV_MIN,
                                            ADV_INTERV_MAX,
                                            ADV_CH_ALL,
                                            STATIC_RANDOM_ADDR, NULL,
                                            ADV_NO_WHITE_LIST_USE,
                                            0, /* 0 dBm */
                                            LE_1M_PHY, /* Primary advertising PHY */
                                            0, /* 0 skips */
                                            LE_1M_PHY, /* Secondary advertising PHY. Not used with legacy advertising. */
                                            0, /* SID */
                                            0 /* No scan request notifications */);

if (ret != BLE_STATUS_SUCCESS) {
    Error_Handler();
    return ret;
} else {
    HAL_Delay(100);
}

ret = aci_gap_set_advertising_data_nwk(0, ADV_COMPLETE_DATA, sizeof(adv
                                               }    
 ```                
 
 # here we add service an characteristic
 
 # GATT Service and Characteristic Initialization

The following code sets up and initializes a GATT service and its characteristics for the Bluetooth device:

- `uint8_t max_attribute_records = 1 + 3 + 2;`:
  - Sets the maximum number of attribute records.

- UUIDs:
  - The following UUIDs are used:
    - `D973F2E0-B19E-11E2-9E96-0800200C9A66`
    - `D973F2E1-B19E-11E2-9E96-0800200C9A66`
    - `D973F2E2-B19E-11E2-9E96-0800200C9A66`

- `const uint8_t uuid[16] = {...};`:
  - Array containing the UUID for the GATT service.

- `const uint8_t charUuidTX[16] = {...};`:
  - Array containing the UUID for the TX characteristic.

- `const uint8_t charUuidRX[16] = {...};`:
  - Array containing the UUID for the RX characteristic.

- `STM32WB_memcpy(&service_uuid.Service_UUID_128, uuid, 16);`:
  - Copies the service UUID to the service UUID structure.

- `aci_gatt_srv_add_service_nwk(UUID_TYPE_128, &service_uuid, PRIMARY_SERVICE, max_attribute_records, &sampleServHandle);`:
  - Adds a primary service to the GATT server.
  - The parameters include the UUID type, service UUID, service type, maximum attribute records, and a pointer to the service handle.

- `STM32WB_memcpy(&char_uuid.Char_UUID_128, charUuidTX, 16);`:
  - Copies the TX characteristic UUID to the characteristic UUID structure.

- `aci_gatt_srv_add_char_nwk(sampleServHandle, UUID_TYPE_128, &char_uuid, CHAR_VALUE_LENGTH, CHAR_PROP_NOTIFY, ATTR_PERMISSION_NONE, 0, 16, 1, &TXCharHandle);`:
  - Adds a characteristic to the GATT server for TX.
  - The parameters include the service handle, UUID type, characteristic UUID, value length, properties, permissions, GATT notify attribute write, and a pointer to the characteristic handle.

- `STM32WB_memcpy(&char_uuid.Char_UUID_128, charUuidRX, 16);`:
  - Copies the RX characteristic UUID to the characteristic UUID structure.

- `aci_gatt_srv_add_char_nwk(sampleServHandle, UUID_TYPE_128, &char_uuid, CHAR_VALUE_LENGTH, CHAR_PROP_WRITE | CHAR_PROP_WRITE_WITHOUT_RESP, ATTR_PERMISSION_NONE, GATT_NOTIFY_ATTRIBUTE_WRITE, 16, 1, &RXCharHandle);`:
  - Adds a characteristic to the GATT server for RX.
  - The parameters include the service handle, UUID type, characteristic UUID, value length, properties, permissions, GATT notify attribute write, and a pointer to the characteristic handle.
 
 
 ```c=

uint8_t max_attribute_records = 1 + 3 + 2;

/*
UUIDs:
D973F2E0-B19E-11E2-9E96-0800200C9A66
D973F2E1-B19E-11E2-9E96-0800200C9A66
D973F2E2-B19E-11E2-9E96-0800200C9A66
*/

const uint8_t uuid[16] = {0x66, 0x9a, 0x0c, 0x20, 0x00, 0x08, 0x96, 0x9e, 0xe2, 0x11, 0x9e, 0xb1, 0xe0, 0xf2, 0x73, 0xd9};
const uint8_t charUuidTX[16] = {0x66, 0x9a, 0x0c, 0x20, 0x00, 0x08, 0x96, 0x9e, 0xe2, 0x11, 0x9e, 0xb1, 0xe1, 0xf2, 0x73, 0xd9};
const uint8_t charUuidRX[16] = {0x66, 0x9a, 0x0c, 0x20, 0x00, 0x08, 0x96, 0x9e, 0xe2, 0x11, 0x9e, 0xb1, 0xe2, 0xf2, 0x73, 0xd9};
STM32WB_memcpy(&service_uuid.Service_UUID_128, uuid, 16);

ret = aci_gatt_srv_add_service_nwk(UUID_TYPE_128, &service_uuid, PRIMARY_SERVICE, max_attribute_records, &sampleServHandle);
if (ret != BLE_STATUS_SUCCESS)  {
    Error_Handler();
}

STM32WB_memcpy(&char_uuid.Char_UUID_128, charUuidTX, 16);
ret =  aci_gatt_srv_add_char_nwk(sampleServHandle, UUID_TYPE_128, &char_uuid, CHAR_VALUE_LENGTH, CHAR_PROP_NOTIFY, ATTR_PERMISSION_NONE, 0,
                                 16, 1, &TXCharHandle);
if (ret != BLE_STATUS_SUCCESS)  {
    Error_Handler();
}

STM32WB_memcpy(&char_uuid.Char_UUID_128, charUuidRX, 16);
ret =  aci_gatt_srv_add_char_nwk(sampleServHandle, UUID_TYPE_128, &char_uuid, CHAR_VALUE_LENGTH, CHAR_PROP_WRITE | CHAR_PROP_WRITE_WITHOUT_RESP, ATTR_PERMISSION_NONE, GATT_NOTIFY_ATTRIBUTE_WRITE,
                                 16, 1, &RXCharHandle);
if (ret != BLE_STATUS_SUCCESS) {
    Error_Handler();
}



 ```
 
 
 # add this below uint_8 baddr line 133 USER CODE BEGIN2
 
 The following variables are used to store the UUIDs for the GATT service and characteristics:

- `Service_UUID_t service_uuid;`:
  - Declares a variable `service_uuid` of type `Service_UUID_t`.
  - This variable is used to store the UUID of the GATT service.

- `Char_UUID_t char_uuid;`:
  - Declares a variable `char_uuid` of type `Char_UUID_t`.
  - This variable is used to store the UUID of a GATT characteristic.
 
 
 ```c=
Service_UUID_t service_uuid;
Char_UUID_t char_uuid;
```

# Now we add a while loop to process HCI and manage the events callback

Inside While(1) add the following

- `hci_user_evt_proc();`:
  - Processes the Host Controller Interface (HCI) user events.
  - This function handles any events generated by the Bluetooth stack and processes them accordingly.

- `User_Process();`:
  - Handles user-defined processing.
  - This function contains any custom code or logic defined by the user for their specific application.

```c=
hci_user_evt_proc();
User_Process();
```

# Declare the user_process function

Put this in user code begin PFP
```c=
static void User_Process(void);
```


# Add the body of the function
In user code begin 4 at the bottom of main.c
When en event will change the value to send_env and reception, a buffer will be filled and a value notified with random numbers.

## Notification Handling

- `if (send_env)`:
  - Checks if notifications are enabled on the first characteristic.
  - If enabled, a periodic update of the variable is performed.
  - A buffer `buff[8]` is filled with random values, seeded by the current time.
  - The `HOST_TO_LE_16(buff, HAL_GetTick() >> 3);` function is called to update the buffer with environmental data.
  - The `aci_gatt_srv_notify` function is called to send a notification with the buffer data.
  - A delay of 1000 milliseconds is introduced using `HAL_Delay(1000);`.

## Reception Handling

- `if (reception)`:
  - Checks if the reception condition is met.
  - If met, toggles the state of GPIO pin 7 on port C using `HAL_GPIO_TogglePin(GPIOC, GPIO_PIN_7);`.


```c=
static void User_Process(void)
{
   
 // When notification is enabled on the first characteristic, I get a periodic update of the variable
    if (send_env) {
        uint8_t buff[8];

        // Seed the random number generator
        srand((unsigned int)time(NULL));

        // Fill the buffer with random values
        for (int i = 0; i < 8; i++) {
            buff[i] = (uint8_t)(rand() % 256);
        }

        // Environmental_Update logic directly in the main function
        HOST_TO_LE_16(buff, HAL_GetTick() >> 3);

        ret = aci_gatt_srv_notify(connection_handle, TXCharHandle + 1, GATT_NOTIFICATION, 8, buff);

        HAL_Delay(1000);
    }

    if (reception) {
         HAL_GPIO_TogglePin(GPIOC, GPIO_PIN_7);
    }
}


```


# add the handling of hci event in user code begin 4

# Function Descriptions

- `hci_le_connection_complete_event`:
  - This function is called when a LE connection is completed.
  - It sets the `connected`, `pairing`, and `paired` flags to `TRUE`.
  - It also sets the `connection_handle` to the provided `Connection_Handle`.

- `hci_le_enhanced_connection_complete_event`:
  - This function is called when a LE enhanced connection is completed.
  - It internally calls `hci_le_connection_complete_event` to handle the connection completion.

- `hci_disconnection_complete_event`:
  - This function is called when a disconnection is completed.
  - It sets the `connected`, `pairing`, and `paired` flags to `FALSE`.
  - It sets the `set_connectable` flag to `TRUE` to make the device connectable again.
  - It reconfigures and enables advertising to make the device discoverable and connectable.

- `Attribute_Modified_Request_CB`:
  - This callback function is called when an attribute is modified.
  - It checks if the `TX` characteristic has been modified and sets the `send_env` flag accordingly.
  - It also checks if the `RX` characteristic has been modified and toggles the state of GPIO pin 7 on port C if the condition is met.

- `aci_gatt_srv_attribute_modified_event`:
  - This function is called when an attribute modification event occurs.
  - It calls the `Attribute_Modified_Request_CB` function to handle the attribute modification.

---

```c
void hci_le_connection_complete_event(uint8_t Status,
                                      uint16_t Connection_Handle,
                                      uint8_t Role,
                                      uint8_t Peer_Address_Type,
                                      uint8_t Peer_Address[6],
                                      uint16_t Conn_Interval,
                                      uint16_t Conn_Latency,
                                      uint16_t Supervision_Timeout,
                                      uint8_t Master_Clock_Accuracy)
{
  connected = TRUE;
  pairing = TRUE;
  paired = TRUE;
  connection_handle = Connection_Handle;
}

void hci_le_enhanced_connection_complete_event(uint8_t Status,



```c=


void hci_le_connection_complete_event(uint8_t Status,
                                      uint16_t Connection_Handle,
                                      uint8_t Role,
                                      uint8_t Peer_Address_Type,
                                      uint8_t Peer_Address[6],
                                      uint16_t Conn_Interval,
                                      uint16_t Conn_Latency,
                                      uint16_t Supervision_Timeout,
                                      uint8_t Master_Clock_Accuracy)
{
  connected = TRUE;
  pairing = TRUE;
  paired = TRUE;
  connection_handle = Connection_Handle;
}

void hci_le_enhanced_connection_complete_event(uint8_t Status,
                                               uint16_t Connection_Handle,
                                               uint8_t Role,
                                               uint8_t Peer_Address_Type,
                                               uint8_t Peer_Address[6],
                                               uint8_t Local_Resolvable_Private_Address[6],
                                               uint8_t Peer_Resolvable_Private_Address[6],
                                               uint16_t Conn_Interval,
                                               uint16_t Conn_Latency,
                                               uint16_t Supervision_Timeout,
                                               uint8_t Master_Clock_Accuracy)
{
  hci_le_connection_complete_event(Status,
                                   Connection_Handle,
                                   Role,
                                   Peer_Address_Type,
                                   Peer_Address,
                                   Conn_Interval,
                                   Conn_Latency,
                                   Supervision_Timeout,
                                   Master_Clock_Accuracy);
}

void hci_disconnection_complete_event(uint8_t Status,
                                      uint16_t Connection_Handle,
                                      uint8_t Reason)
{
  uint8_t bdaddr[6];
  uint8_t ret;
  connected = FALSE;
  pairing = FALSE;
  paired = FALSE;

  /* Make the device connectable again */
  set_connectable = TRUE;
  connection_handle = 0;

  Advertising_Set_Parameters_t Advertising_Set_Parameters[1];
  uint8_t adv_data[] =
  {
    0x02, AD_TYPE_FLAGS, FLAG_BIT_LE_GENERAL_DISCOVERABLE_MODE | FLAG_BIT_BR_EDR_NOT_SUPPORTED,
    2, 0x0A, 0x00, /* Transmission Power (0 dBm) */
    10, 0x09, SENSOR_DEMO_NAME,  /* Complete Name */
    13, 0xFF, 0x01, /* SKD version */
    0x00, /* generic device */
    0x00,
    0xF4, /* ACC+Gyro+Mag 0xE0 | 0x04 Temp | 0x10 Pressure */
    0x00, /*  */
    0x00, /*  */
    bdaddr[5], /* BLE MAC start - MSB first - */
    bdaddr[4],
    bdaddr[3],
    bdaddr[2],
    bdaddr[1],
    bdaddr[0]  /* BLE MAC stop */
  };

  adv_data[23] |= 0x01; /* Sensor Fusion */

  ret = aci_gap_set_advertising_configuration(0, GAP_MODE_GENERAL_DISCOVERABLE,
                                              ADV_PROP_CONNECTABLE | ADV_PROP_SCANNABLE | ADV_PROP_LEGACY,
                                              ADV_INTERV_MIN,
                                              ADV_INTERV_MAX,
                                              ADV_CH_ALL,
                                              STATIC_RANDOM_ADDR, NULL,
                                              ADV_NO_WHITE_LIST_USE,
                                              0, /* 0 dBm */
                                              LE_1M_PHY, /* Primary advertising PHY */
                                              0, /* 0 skips */
                                              LE_1M_PHY, /* Secondary advertising PHY. Not used with legacy advertising. */
                                              0, /* SID */
                                              0 /* No scan request notifications */);

  if (ret != BLE_STATUS_SUCCESS) {
    Error_Handler();
  } else {
    HAL_Delay(100);
  }

  ret = aci_gap_set_advertising_data_nwk(0, ADV_COMPLETE_DATA, sizeof(adv_data), adv_data);
  if (ret != BLE_STATUS_SUCCESS) {
    Error_Handler();
  } else {
    HAL_Delay(100);
  }

  Advertising_Set_Parameters[0].Advertising_Handle = 0;
  Advertising_Set_Parameters[0].Duration = 0;
  Advertising_Set_Parameters[0].Max_Extended_Advertising_Events = 0;

  /* Enable advertising */
  ret = aci_gap_set_advertising_enable(ENABLE, 1, Advertising_Set_Parameters);
  if (ret != BLE_STATUS_SUCCESS) {
    Error_Handler();
    while (1);
  } else {
    HAL_Delay(100);
  }
}

void Attribute_Modified_Request_CB(uint16_t Connection_Handle, uint16_t attr_handle,
                                   uint8_t data_length, uint8_t *att_data)
{
  if (attr_handle == TXCharHandle + 2) {
    if (att_data[0] == 1) {
      send_env = TRUE;
    } else if (att_data[0] == 0) {
      send_env = FALSE;
    }
  }

  if (attr_handle == RXCharHandle + 1) {
    reception = TRUE;
    HAL_GPIO_TogglePin(GPIOC, GPIO_PIN_7);
  }
}

void aci_gatt_srv_attribute_modified_event(uint16_t Connection_Handle,
                                           uint16_t Attr_Handle,
                                           uint16_t Attr_Data_Length,
                                           uint8_t Attr_Data[])
{
  Attribute_Modified_Request_CB(Connection_Handle, Attr_Handle, Attr_Data_Length, Attr_Data);
}


```
