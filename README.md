# ConnectWise Manage PSA webhook

This guide describes how to integrate a Zabbix 6.4 up to 7.4 installation with ConnectWise Manage PSA using the Zabbix webhook feature. It provides instructions for installing and configuring this ConnectWise Manage PSA webhook media type, its API authentication and an action in Zabbix.

Please note that recovery and update operations are supported only for trigger-based events.

This work is based on the example available in the ServiceNow webhook which is included with Zabbix

**Tested/Supported versions**: Zabbix 6.4.x, 7.0.16, 7.4.0, 7.4.1, 7.4.2

# Setting up ConnectWise

### 1. First, create an APImember user for creating incidents.

   - Under **System** -> **Members** -> **API Members**, create your new API member

### 2. Assign a role to this user which allows it to:

   - Create new tickets
   - Update ticket status
   - Add ticket notes
   - Set ticket resolution flag

### 3. Generate an API Keys set for the APIMember on its **API Keys** tab

   - Under **System** -> **Members** -> **API Members**, open your new API member
   - On the **API Keys** tab, create a new API key pair and copy these to use later in the Zabbix setup

### 4. Generate a ConnectWise Client ID to allow API interactivity

   - Go to https://developer.connectwise.com/ClientID
   - Click the Create new integration button and fill out the request details
   - Make sure to select Manage as your product
   - Wait for ConnectWise to provision the Client ID for you
   
# Setting up webhook in Zabbix

### 1. Import this webhook media type

   - Download the zbx_mediatype_cwpsa-6.4.yaml file from this repository
   - Under **Alerts** -> **Media types**, import the zbx_mediatype_cwpsa-6.4.yaml file.
   
### 2. Setup webhook Parameters

   - Open the newly added *ConnectWise Manage PSA* media type and set it up by filling the needed Paramter values

   - Review all the Parameter values which are prefilled with **\<placeholders\>** and fill with your values.
        - Parameters for Connectwise Priority and Company mappings are mostly optional (except the DEFAULT company client ID)

   - Parameters with names starting with **cwpsa_api_** are required for authentication
     - **cwpsa_api_clientid** is the Client ID string you generated from ConnectWise above
     - **cwpsa_api_user** is the concatenation of the APIMember's company name and public API key with a + symbol between
       (e.g. xyzcorp+7c21wqSxILBLqC98J)
     - **cwpsa_api_password** is the APIMember's private API key
     - **cwpsa_api_url** is the full path to the service tickets API on your ConnectWise Manage instance
       (e.g. https://yourcw.com/v4_6_release/apis/3.0/service/tickets/)

   - **cwpsa_serviceboard_name** must contain the exact name of the ConnectWise service board you wish to create tickets on
  
   - **cwpsa_status_new** must contain the exact name of the **status** you wish to use on this board for new tickets
  
   - **cwpsa_status_resolved** must contain the exact name of the **status** you wish to use for tickets which have had their Zabbix Problem *resolved*
  
   - **cwpsa_type** must contain the exact name of the **type** you wish to be applied to new tickets
  
   - **cwpsa_resolution_flag** is used to choose whether or not you wish to set the **Resolution** flag against the ticket note added when a ticket is updated upon Problem resolution
     (This value must be **true** or **false**)

   - Parameters starting with **cwpsa_cmpy_** map tickets to their correct companies in ConnectWise
      - The webhook selects the ConnectWise company ID at the start of the Zabbix Problem *alert subject*. It uses the longest ID that fits, so a shorter ID does not take a subject that starts with a longer one. The character after the ID must be the end of the subject, or not a letter, digit, or underscore. Matching ignores case.


      - **cwpsa_cmpy_DEFAULT** must be used and must be set to the ConnectWise company ID for the default company you wish to create tickets against (This is a Catchall scenario)
      - Each **cwpsa_cmpy_\<Company_ID\>** parameter is named with the ConnectWise company ID, and its value is that same ID


         - **cwpsa_cmpy_OxspringNetwork** has the value **OxspringNetwork**
         - A Zabbix Problem with an alert subject of "*OxspringNetwork - Monitored host down*" creates the ConnectWise ticket against that company

      - The media type includes one parameter for each active customer company exported from ConnectWise
    
   - Parameters starting with **cwpsa_priority_** are optionally used to manage the ConnectWise *priority* assigned to a ticket as it's created, depending on the Zabbix *severity*.
  
        - To enable priority mapping the **cwpsa_priority_is_used** Parameter must be set to **true** and if so:

           - Each of the **cwpsa_priority_for_??????** Parameters must have a value between 1 and 4 applied to map each to their respective ConnectWise priorities
         
           - Suggestions are listed in brackets in the **\<placeholder\>** text for each
                - Disaster and High mapped to *Priority 1 - Critical*
                - Average and Warning mapped to *Priority 2 - High*
                - Information mapped to *Priority 4 - Low*
                - Not Classified mapped to *Priority 3 - Medium*

### 3. Setup Zabbix user.
   
   - Create a new Zabbix user with the **ConnectWise Manage PSA webhook** as an enabled media
   - There is no need to use a valid **Send to** value as this is handled within the **ConnectWise Manage PSA webhook** config.
   - Make sure this user has User Group memberships to give it permission to monitor the required hosts
  
### 4. Setup custom trigger actions if using Connectwise company mappings

   - The Problem *alert subject* must start with the ConnectWise company ID, for example `OxspringNetwork - host down`. The webhook media **Messages Templates** config cannot inject that ID per host, so use trigger actions with custom operations whose alert subject starts with the company ID.

### 5. Review **Messages Templates** options to customise as you need

   - These configurations will only be used if you setup trigger action operations without custom messages

### 6. Enable and test

   - Enable the ConnectWise Manage PSA webhook and test
   - The quickest way I've found for running a repeatable test trigger is to create an item and trigger which watches for a file on the Zabbix server with 10s interval for quick response, then create and delete this file for trigger testing
        - A mostly up to date procedure for achieving this is documented at https://aaronsaray.com/2020/zabbix-test-notification/
