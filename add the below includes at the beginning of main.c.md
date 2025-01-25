---
title: add the below includes at the beginning of main.c

---

# add the below includes at the beginning of main.c in /* USER CODE BEGIN Includes *

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

```c=

#define SENSOR_DEMO_NAME   'S','T','M','3','2','W','B','0','5'
#define BDADDR_SIZE        6
#define CHAR_VALUE_LENGTH 20
#define HOST_TO_LE_16(buf, val)    (((buf)[0] =  (uint8_t) (val)    ) , \
                                    ((buf)[1] =  (uint8_t) (val>>8) ) )

```

# add the following variables

```c=
/* USER CODE BEGIN PFP */
 uint16_t sampleServHandle, TXCharHandle, RXCharHandle;
static  uint16_t connection_handle;
 static   uint8_t set_connectable;
  static  uint8_t connected;
  static  uint8_t pairing;
   static uint8_t paired;
  static uint8_t send_env;
  static uint8_t reception;
  uint16_t sampleServHandle, TXCharHandle, RXCharHandle;
/* USER CODE END PFP */
```

# initialize HCI interface in user code begin 2
```c=
hci_init(APP_UserEvtRx, NULL);
```

APP_UserEvtRx is the part in which the events coming from HCI are elaborated


# some variables
```c=
  uint8_t  ret;
    uint16_t service_handle, dev_name_char_handle, appearance_char_handle;
    uint8_t  device_name[] = {SENSOR_DEMO_NAME};
      uint8_t bdaddr[BDADDR_SIZE] = {0, 1, 2, 3, 4, 5}; // this is the mac address

```
# Create a protoype for APP_UserEvtRx

Put it in USER CODE BEGIN PFP
```c=
  void APP_UserEvtRx(void *pData);
```

# add the declaration in USER CODE BEGIN 4


```c=
void APP_UserEvtRx(void *pData)
{
  uint32_t i;

  hci_spi_pckt *hci_pckt = (hci_spi_pckt *)pData;

  if (hci_pckt->type == HCI_EVENT_PKT || hci_pckt->type == HCI_EVENT_EXT_PKT)
  {
    void *data;
    hci_event_pckt *event_pckt = (hci_event_pckt*)hci_pckt->data;

    if (hci_pckt->type == HCI_EVENT_PKT)
	{
      data = event_pckt->data;
    }
    else
	{
      hci_event_ext_pckt *event_pckt = (hci_event_ext_pckt*)hci_pckt->data;
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
          hci_vendor_specific_events_table[i].process((void *)blue_evt->data);
          break;
        }
      }
    }
    else
    {
      for (i = 0; i < (sizeof(hci_events_table)/sizeof(hci_events_table_type)); i++)
      {
        if (event_pckt->evt == hci_events_table[i].evt_code)
        {
          hci_events_table[i].process(data);
          break;
        }
      }
    }
  }
}

```

# Back to the main.c after HCI init

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

```c=

 ret = aci_gap_init(GAP_PERIPHERAL_ROLE, 0x00, 0x07, STATIC_RANDOM_ADDR, &service_handle, &dev_name_char_handle,
                            &appearance_char_handle);
         if (ret != BLE_STATUS_SUCCESS) {
        	 Error_Handler();
           return ret;
         }
         else {
        	 HAL_Delay(100);
         }

```


# upddate device name

```c=

  ret = aci_gatt_srv_write_handle_value_nwk(dev_name_char_handle + 1, 0, sizeof(device_name), device_name);
           if (ret != BLE_STATUS_SUCCESS) {
        	   Error_Handler();
             return ret;
           }
           else {
        	   HAL_Delay(100);
           }


```


# SET ADV PARAMS


```c=

                                   Advertising_Set_Parameters_t Advertising_Set_Parameters[1];
                                      uint8_t adv_data[] =
                                  {
                                    0x02, AD_TYPE_FLAGS, FLAG_BIT_LE_GENERAL_DISCOVERABLE_MODE | FLAG_BIT_BR_EDR_NOT_SUPPORTED,
                                    2, 0x0A, 0x00, /* Trasmission Power (0 dBm) */
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
                                                               }
                                                               else {
                                                            	   HAL_Delay(100);
                                                               }

                                  ret = aci_gap_set_advertising_data_nwk(0, ADV_COMPLETE_DATA, sizeof(adv_data), adv_data);

                                  if (ret != BLE_STATUS_SUCCESS) {
                                                            	   Error_Handler();
                                                                 return ret;
                                                               }
                                                               else {
                                                            	   HAL_Delay(100);
                                                               }



                                  Advertising_Set_Parameters[0].Advertising_Handle = 0;
                                  Advertising_Set_Parameters[0].Duration = 0;
                                  Advertising_Set_Parameters[0].Max_Extended_Advertising_Events = 0;

                                  /* enable advertising */
                                  ret = aci_gap_set_advertising_enable(ENABLE, 1, Advertising_Set_Parameters);

                                  if (ret != BLE_STATUS_SUCCESS) {
                                                            	   Error_Handler();
                                                                 return ret;
                                                               }
                                                               else {
                                                            	   HAL_Delay(100);
                                                               }    
 ```                
 
 # here we add service an characteristic
 
 
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
                                                  } ;

                                                  STM32WB_memcpy(&char_uuid.Char_UUID_128, charUuidTX, 16);
                                                  ret =  aci_gatt_srv_add_char_nwk(sampleServHandle, UUID_TYPE_128, &char_uuid, CHAR_VALUE_LENGTH, CHAR_PROP_NOTIFY, ATTR_PERMISSION_NONE, 0,
                                                                                   16, 1, &TXCharHandle);
                                                  if (ret != BLE_STATUS_SUCCESS)  {
                                                 	 Error_Handler();
                                                  } ;

                                                  STM32WB_memcpy(&char_uuid.Char_UUID_128, charUuidRX, 16);
                                                  ret =  aci_gatt_srv_add_char_nwk(sampleServHandle, UUID_TYPE_128, &char_uuid, CHAR_VALUE_LENGTH, CHAR_PROP_WRITE | CHAR_PROP_WRITE_WITHOUT_RESP, ATTR_PERMISSION_NONE, GATT_NOTIFY_ATTRIBUTE_WRITE,
                                                                                   16, 1, &RXCharHandle);
                                                  if (ret != BLE_STATUS_SUCCESS) {
                                                 	 Error_Handler();
                                                  };



 ```
 
 
 # add this below uint_8 baddr line 133 USER CODE BEGIN2
 
 
 ```c=
Service_UUID_t service_uuid;
Char_UUID_t char_uuid;
```

# Now we add a while loop to process HCI and manage the events callback

Inside While(1) add the following

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
When en event will change the value to send_env and reception, a buffer will be filled and a value notified with random numbers

```c=
static void User_Process(void)
{


	  uint8_t ret;


	  //when notification is enabled on the first characteristic, i get a periodic update of the variable
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


	  if (reception)

	  {

	  	   HAL_Delay(2000);
	  	    }

}

```


# add the handling of hci event in user code begin 4




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

void hci_disconnection_complete_event(uint8_t  Status,
                                      uint16_t Connection_Handle,
                                      uint8_t  Reason)
{
	uint8_t bdaddr[6];
	uint8_t ret;
  connected = FALSE;
  pairing   = FALSE;
  paired    = FALSE;

  /* Make the device connectable again */
  set_connectable = TRUE;
  connection_handle = 0;

  Advertising_Set_Parameters_t Advertising_Set_Parameters[1];

                   uint8_t adv_data[] =
                   {
                     0x02, AD_TYPE_FLAGS, FLAG_BIT_LE_GENERAL_DISCOVERABLE_MODE | FLAG_BIT_BR_EDR_NOT_SUPPORTED,
                     2, 0x0A, 0x00, /* Trasmission Power (0 dBm) */
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



                   if (ret != BLE_STATUS_SUCCESS)
                     {
                  	    Error_Handler();
                     }
                     else
                     {
                  	   HAL_Delay(100);
                     }

                   ret = aci_gap_set_advertising_data_nwk(0, ADV_COMPLETE_DATA, sizeof(adv_data), adv_data);
                     if (ret != BLE_STATUS_SUCCESS)
                     {
                  	   Error_Handler();                   }
                     else
                     {
                  	   HAL_Delay(100);                   }


                     Advertising_Set_Parameters[0].Advertising_Handle = 0;
                     Advertising_Set_Parameters[0].Duration = 0;
                     Advertising_Set_Parameters[0].Max_Extended_Advertising_Events = 0;

                     /* enable advertising */
                     ret = aci_gap_set_advertising_enable(ENABLE, 1, Advertising_Set_Parameters);
                     if (ret != BLE_STATUS_SUCCESS)
                     {
                  	   Error_Handler();
                       while (1);
                     }
                     else
                     {
                  	   HAL_Delay(100);
                     }
}


void Attribute_Modified_Request_CB(uint16_t Connection_Handle, uint16_t attr_handle,
                                   uint8_t data_length, uint8_t *att_data)
{
  if (attr_handle == TXCharHandle + 2)
  {
    if (att_data[0] == 1)
	{
      send_env = TRUE;
    }
	else if (att_data[0] == 0)
	{
      send_env = FALSE;
    }
  }

  if(attr_handle == RXCharHandle + 1)
    {

	  reception = TRUE;
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