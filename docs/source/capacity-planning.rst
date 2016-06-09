Chapter 4: Capacity Planning Use Cases
**************************************

.. contents:: :local:

We are going to develop scripts for a couple of capacity planning use cases.  First use case is to identify the available port density and the unused transceivers in the network. Second use case is to identify the usage of hardware resources such as mac address, arp, route and tcam tables. We introduce Python’s jsonrpc and functions in this chapter. We also show you how to explore the Arista EOS commands and the json output via graphical user interface.

Port Capacity
=============

The goal of this script is to identify the number of unused ports and transceivers in the network. We are going to collect the information and report it in the following format.

.. code-block:: bash

  unused_ports =
  {

  switch_host_name: { switch_model: <switch_Model>,
  		                description: <switch description>,
  		                10G: {
  				                  interface_type: <Number of unused ports>,
  				                  interface_type: <Number of unused ports>
  			                   },
  		                40G: {
  				                  interface_type: <Number of unused ports>,
  				                  interface_type: <Number of unused ports>
  			                   }
                    },
  switch_host_name: { switch_model: <switch_Model>,
  		                description: <switch description>,
  		                10G: {
  				                  interface_type: <Number of unused ports>,
  				                  interface_type: <Number of unused ports>
  			                   },
  		                40G: {
  				                  interface_type: <Number of unused ports>,
  				                  interface_type: <Number of unused ports>
  			                   }
                    },
  }

Under each switch, we will report how many 10GE, 40GE or 100GE ports are available. There are different transceiver types available and we will identify the different types and save the information.

First we will write down the step by step commands to collect the data, second we will develop an algorithm and finally we will build the Python script using our framework.

Arista EOS Show Commands
------------------------

**Step 1:** Inventory

Model name and description provides additional insight when presenting unused ports. Show inventory command can be used to collect the model and switch description.

.. code-block:: bash

  Arista-switch#show inventory | json

      "systemInformation": {
          "name": "DCS-7050SX-128",
          "description": "96x 10Gb SFP+ + 8x 40Gb QSFP+ 2RU",
          "mfgDate": "2015-12-21",
          "hardwareRev": "02.00",
          "serialNum": "JPE14080459"
      },

**Step 2:** Commands to collect the unused interfaces

“notconnect” option in "show interfaces status" shows the ports configured as “no shutdown” but links are not up. “disabled” option shows the ports configured as “shutdown”.

.. code-block:: bash
  :emphasize-lines: 10,11,25,26

  Arista-switch# show interfaces status notconnect disabled | json
  {
      "interfaceStatuses": {
          "Ethernet8": {
              "vlanInformation": {
                  "interfaceMode": "bridged",
                  "vlanId": 1,
                  "interfaceForwardingModel": "bridged"
              },
              "bandwidth": 10000000000,
              "interfaceType": "10GBASE-SR",
              "description": "",
              "autoNegotiateActive": false,
              "duplex": "duplexFull",
              "autoNegotigateActive": false,
              "linkStatus": "notconnect",
              "lineProtocolStatus": "notPresent"
          },
          "Ethernet9": {
              "vlanInformation": {
                  "interfaceMode": "bridged",
                  "vlanId": 1,
                  "interfaceForwardingModel": "bridged"
              },
              "bandwidth": 10000000000,
              "interfaceType": "Not Present",
              "description": "",
              "autoNegotiateActive": false,
              "duplex": "duplexFull",
              "autoNegotigateActive": false,
              "linkStatus": "notconnect",
              "lineProtocolStatus": "notPresent"
          },

Algorithm
---------

We are going to use the following algorithm to write the Python script.

#. Create an empty dictionary for unused_ports.
   **unused_ports = {}**

#. For each switch, create a {key: value} pair using the switch name as the key.
   **unused_ports[host_name] = {}**

#. For each switch, collect model and description, and update unused_ports dictionary.
   **unused_ports[host_name] = {“Model”: <model>, “Description”: <description> }**

#. For each switch, we will collect “show interfaces status notconnect disabled” output and parse through bandwidth and interface type values of each interface.

#. We will take the bandwidth value of each interface and check if there is an entry in the dictionary under the specific switch name. If there is no entry for that particular bandwidth, we will add a {key: value} pair entry for that bandwidth with the bandwidth as the key and an empty dictionary as the value.
   **unused_ports[host_name][bandwidth] = {}**

#. Next, we will look at the interface type of that interface. If there is no entry for that interface type within unused_ports[host_name][bandwidth], we will create one with a {key: value} pair. Key is the interface type and the value is an integer number “1”.
   **unused_ports[host_name][bandwidth][interfacetype] = 1**

#. If there is an entry for that interface type, we will increment the value by 1.
   **unused_ports[host_name][bandwidth][interfacetype] += 1**

Develop Script
--------------

**Prepare:** Create a new Python script using the framework we developed.

Open the IDLE and create a new script and save as unused_ports.py in your folder. Copy the code from section 1, 2, 3 from our framework and paste it in this new script.

::

  """
  Discover Unused Ports in the network
  """

  ### Section 1

  import pyeapi
  import getpass
  import pprint

  ### Section 2

  # Define file path and file names

  file_path = "/Users/anees/Google Drive/my-scripts/"
  file_name_switches = "switches.txt"
  file_switches = file_path + file_name_switches

  # Read the content of the file and save it in a List

  switches = []
  with open(file_switches) as readfile:
      for line in readfile:
          switches.append(line.strip())

  ### Section 3

  # Input Username and Password

  my_username = raw_input("Enter your username: ")
  my_password = getpass.getpass("Enter your password: ")

**Steps 1, 2 and 3:**
Since we are already familiar with pyeapi, dictionary and for loop, we can start section 4 with the usual for loop, empty dictionary and pyeapi commands which are required for the first three steps of our algorithm.

Step 1: Create an empty dictionary for unused_ports.

Step 2: For each switch, create a {key: value} pair using the switch name as the key.

Step 3: For each switch, collect model and description, and update unused_ports dictionary.

::

  ### Section 4

  # Create an Empty Dictionary
  unused_ports = {}
  errors = {}

  for switch in switches:
      try:
          # Define API Connection String
          node = pyeapi.connect(transport="https", host=switch, username=my_username, password=my_password, port=None)

          # collect the hostname of the switch and create an entry in the dictionary
          host_name = node.execute(["show hostname"])
          host_name_clean = str(host_name["result"][0]["hostname"])
          unused_ports[host_name_clean] = {}

          # Collect the model name and device description
          show_inventory = node.execute(["show inventory"])
          model = str(show_inventory["result"][0]["systemInformation"]["name"])
          description = str(show_inventory["result"][0]["systemInformation"]["description"])
          unused_ports[host_name_clean] = {"Model": model, "Description": description}


      except pyeapi.eapilib.ConnectionError:
          errors[switch] = "ConnectionError: unable to connect to eAPI"

      except pyeapi.eapilib.CommandError:
          errors[switch] = "CommandError: Check your EOS command syntax"


  ### Section 5

  if bool(errors):
      pprint.pprint(errors)

  if bool(unused_ports):
      pprint.pprint(unused_ports)

Dictionary structure for model and description from show inventory has been derived from command API explorer. So far we have used Python shell to explore the syntax. Another method is to use command API explorer. After you enable management api on the Arista switch, you can access the switch via your browser with the URL https://<IP-address>. It will prompt you for user name and password. After that you can type any EOS show command and click “submit POST request” to view the output in json format.

.. image:: images/ch04-pic1.png

Save and run the script from IDLE (Run → Run Module).

.. code-block:: bash

  >>> ================================ RESTART ================================
  >>>
  Enter your username: admin
  {'22sw2': {'Description': '96x 10Gb SFP+ + 8x 40Gb QSFP+ 2RU',
             'Model': 'DCS-7050SX-128'},
   '22sw35': {'Description': '64x QSFP+ 2RU', 'Model': 'DCS-7250QX-64'},
   '22sw37': {'Description': '64x QSFP+ 2RU', 'Model': 'DCS-7250QX-64'},
   '22sw4': {'Description': '96x 10Gb SFP+ + 8x 40Gb QSFP+ 2RU',
             'Model': 'DCS-7050SX-128'}}

**Step 4:** For each switch, we will collect “show interfaces status notconnect disabled” and parse through bandwidth and interface type values of each interface.

Before writing the script, we will identify the dictionary structure for bandwidth and interface type using command api explorer.

.. image:: images/ch04-pic2.png

.. code-block:: python
  :emphasize-lines: 23,24,25,27,28,29,30,31

  ### Section 4

  # Create an Empty Dictionary
  unused_ports = {}
  errors = {}

  for switch in switches:
      try:
          # Define API Connection String
          node = pyeapi.connect(transport="https", host=switch, username=my_username, password=my_password, port=None)

          # collect the hostname of the switch and create an entry in the dictionary
          host_name = node.execute(["show hostname"])
          host_name_clean = str(host_name["result"][0]["hostname"])
          unused_ports[host_name_clean] = {}

          # Collect the model name and device description
          show_inventory = node.execute(["show inventory"])
          model = str(show_inventory["result"][0]["systemInformation"]["name"])
          description = str(show_inventory["result"][0]["systemInformation"]["description"])
          unused_ports[host_name_clean] = {"Model": model, "Description": description}

          # Collect interfaces status
          sh_int_status = node.execute(["show interfaces status notconnect disabled"])
          sh_int_status_clean = sh_int_status["result"][0]["interfaceStatuses"]

          # Identify unused ports and categorize based on bandwidth and transceiver type
          for each_interface in sh_int_status_clean.keys():
              bandwidth = sh_int_status_clean[each_interface]["bandwidth"]
              bandwidth_GE = str(bandwidth / 1000000000) + "GE"
              interface_type = str(sh_int_status_clean[each_interface]["interfaceType"])

      except pyeapi.eapilib.ConnectionError:
          errors[switch] = "ConnectionError: unable to connect to eAPI"

      except pyeapi.eapilib.CommandError:
          errors[switch] = "CommandError: Check your EOS command syntax"


  ### Section 5

  if bool(errors):
      pprint.pprint(errors)

  if bool(unused_ports):
      pprint.pprint(unused_ports)

Save and run the script.

.. code-block:: bash

  >>> ================================ RESTART ================================
  >>>
  Enter your username: admin
  {'22sw2': {'Description': '96x 10Gb SFP+ + 8x 40Gb QSFP+ 2RU',
             'Model': 'DCS-7050SX-128'},
   '22sw35': {'Description': '64x QSFP+ 2RU', 'Model': 'DCS-7250QX-64'},
   '22sw37': {'Description': '64x QSFP+ 2RU', 'Model': 'DCS-7250QX-64'},
   '22sw4': {'Description': '96x 10Gb SFP+ + 8x 40Gb QSFP+ 2RU',
             'Model': 'DCS-7050SX-128'}}
  >>> bandwidth
  10000000000
  >>> bandwidth_GE
  '10GE'
  >>> interface_type
  'Not Present'

**Step 5:** We will take the bandwidth value of each interface and check if there is an entry in the dictionary under the specific switch name.

If there is no entry for that particular bandwidth, we will add a {key: value} pair entry for that bandwidth with the bandwidth as the key and an empty dictionary as the value.

.. code-block:: python
  :emphasize-lines: 33,34,35

  ### Section 4

  # Create an Empty Dictionary
  unused_ports = {}
  errors = {}

  for switch in switches:
      try:
          # Define API Connection String
          node = pyeapi.connect(transport="https", host=switch, username=my_username, password=my_password, port=None)

          # collect the hostname of the switch and create an entry in the dictionary
          host_name = node.execute(["show hostname"])
          host_name_clean = str(host_name["result"][0]["hostname"])
          unused_ports[host_name_clean] = {}

          # Collect the model name and device description
          show_inventory = node.execute(["show inventory"])
          model = str(show_inventory["result"][0]["systemInformation"]["name"])
          description = str(show_inventory["result"][0]["systemInformation"]["description"])
          unused_ports[host_name_clean] = {"Model": model, "Description": description}

          # Collect interfaces status
          sh_int_status = node.execute(["show interfaces status notconnect disabled"])
          sh_int_status_clean = sh_int_status["result"][0]["interfaceStatuses"]

          # Identify unused ports and categorize based on bandwidth and transceiver type
          for each_interface in sh_int_status_clean.keys():
              bandwidth = sh_int_status_clean[each_interface]["bandwidth"]
              bandwidth_GE = str(bandwidth / 1000000000) + "GE"
              interface_type = str(sh_int_status_clean[each_interface]["interfaceType"])

              # check for bandwidth entry and add it if not there
              if bandwidth_GE not in unused_ports[host_name_clean]:
                  unused_ports[host_name_clean][bandwidth_GE] = {}

      except pyeapi.eapilib.ConnectionError:
          errors[switch] = "ConnectionError: unable to connect to eAPI"

      except pyeapi.eapilib.CommandError:
          errors[switch] = "CommandError: Check your EOS command syntax"


  ### Section 5

  if bool(errors):
      pprint.pprint(errors)

  if bool(unused_ports):
      pprint.pprint(unused_ports)

Save and run the script.

.. code-block:: bash

  >>> ================================ RESTART ================================
  >>>
  Enter your username: admin
  {'22sw2': {'0GE': {},
             '10GE': {},
             '40GE': {},
             'Description': '96x 10Gb SFP+ + 8x 40Gb QSFP+ 2RU',
             'Model': 'DCS-7050SX-128'},
   '22sw35': {'0GE': {},
              '10GE': {},
              '40GE': {},
              'Description': '64x QSFP+ 2RU',
              'Model': 'DCS-7250QX-64'},
   '22sw37': {'0GE': {},
              '10GE': {},
              'Description': '64x QSFP+ 2RU',
              'Model': 'DCS-7250QX-64'},
   '22sw4': {'0GE': {},
             '10GE': {},
             '40GE': {},
             'Description': '96x 10Gb SFP+ + 8x 40Gb QSFP+ 2RU',
             'Model': 'DCS-7050SX-128'}}

The reason we are seeing 0 GE is because of the management interface. At the end of the script, we will add a couple of checks to skip management and dot1q sub interfaces.

.. code-block:: bash

  "Management2": {
        "vlanInformation": {
            "interfaceMode": "routed",
            "interfaceForwardingModel": "routed"
        },
        "bandwidth": 0,
        "interfaceType": "10/100/1000",
        "description": "",
        "autoNegotiateActive": true,
        "duplex": "duplexUnknown",
        "autoNegotigateActive": true,
        "linkStatus": "notconnect",
        "lineProtocolStatus": "down"
    },

**Step 6 and 7:** Next, we will look at the interface type of that interface. If there is no entry for that interface type within unused_ports[host_name][bandwidth], we will create one with a {key: value} pair. Key is the interface type and the value is an integer number “1”. If there is an entry for that interface type, we will increment the value by 1.

.. code-block:: python
  :emphasize-lines: 37,38,39,40,41

  ### Section 4

  # Create an Empty Dictionary
  unused_ports = {}
  errors = {}

  for switch in switches:
      try:
          # Define API Connection String
          node = pyeapi.connect(transport="https", host=switch, username=my_username, password=my_password, port=None)

          # collect the hostname of the switch and create an entry in the dictionary
          host_name = node.execute(["show hostname"])
          host_name_clean = str(host_name["result"][0]["hostname"])
          unused_ports[host_name_clean] = {}

          # Collect the model name and device description
          show_inventory = node.execute(["show inventory"])
          model = str(show_inventory["result"][0]["systemInformation"]["name"])
          description = str(show_inventory["result"][0]["systemInformation"]["description"])
          unused_ports[host_name_clean] = {"Model": model, "Description": description}

          # Collect interfaces status
          sh_int_status = node.execute(["show interfaces status notconnect disabled"])
          sh_int_status_clean = sh_int_status["result"][0]["interfaceStatuses"]

          # Identify unused ports and categorize based on bandwidth and transceiver type
          for each_interface in sh_int_status_clean.keys():
              bandwidth = sh_int_status_clean[each_interface]["bandwidth"]
              bandwidth_GE = str(bandwidth / 1000000000) + "GE"
              interface_type = str(sh_int_status_clean[each_interface]["interfaceType"])

              # check for bandwidth entry and add it if not there
              if bandwidth_GE not in unused_ports[host_name_clean]:
                  unused_ports[host_name_clean][bandwidth_GE] = {}

              # check for interface type and add it if not there
              if interface_type not in unused_ports[host_name_clean][bandwidth_GE]:
                  unused_ports[host_name_clean][bandwidth_GE][interface_type] = 1
              else:
                  unused_ports[host_name_clean][bandwidth_GE][interface_type] += 1

      except pyeapi.eapilib.ConnectionError:
          errors[switch] = "ConnectionError: unable to connect to eAPI"

      except pyeapi.eapilib.CommandError:
          errors[switch] = "CommandError: Check your EOS command syntax"


  ### Section 5

  if bool(errors):
      pprint.pprint(errors)

  if bool(unused_ports):
      pprint.pprint(unused_ports)

Save and run the script.

.. code-block:: bash

  >>> ================================ RESTART ================================
  >>>
  Enter your username: admin
  {'22sw2': {'0GE': {'10/100/1000': 1, 'N/A': 1},
             '10GE': {'10GBASE-CR': 1,
                      '10GBASE-SR': 1,
                      'Not Present': 90,
                      'dot1q-encapsulation': 3},
             '40GE': {'Not Present': 5},
             'Description': '96x 10Gb SFP+ + 8x 40Gb QSFP+ 2RU',
             'Model': 'DCS-7050SX-128'},
   '22sw35': {'0GE': {'10/100/1000': 1},
              '10GE': {'Not Present': 216},
              '40GE': {'Not Present': 1},
              'Description': '64x QSFP+ 2RU',
              'Model': 'DCS-7250QX-64'},
   '22sw37': {'0GE': {'10/100/1000': 1, 'N/A': 1},
              '10GE': {'40GBASE-CR4': 3, 'Not Present': 220},
              'Description': '64x QSFP+ 2RU',
              'Model': 'DCS-7250QX-64'},
   '22sw4': {'0GE': {'10/100/1000': 1},
             '10GE': {'10GBASE-CR': 1,
                      '10GBASE-SR': 1,
                      'Not Present': 90,
                      'dot1q-encapsulation': 3},
             '40GE': {'Not Present': 5},
             'Description': '96x 10Gb SFP+ + 8x 40Gb QSFP+ 2RU',
             'Model': 'DCS-7050SX-128'}}

**Step 8:** Exclude non physical and management interfaces in the unused ports list.

.. code-block:: python
  :emphasize-lines: 29

  ### Section 4

  # Create an Empty Dictionary
  unused_ports = {}
  errors = {}

  for switch in switches:
      try:
          # Define API Connection String
          node = pyeapi.connect(transport="https", host=switch, username=my_username, password=my_password, port=None)

          # collect the hostname of the switch and create an entry in the dictionary
          host_name = node.execute(["show hostname"])
          host_name_clean = str(host_name["result"][0]["hostname"])
          unused_ports[host_name_clean] = {}

          # Collect the model name and device description
          show_inventory = node.execute(["show inventory"])
          model = str(show_inventory["result"][0]["systemInformation"]["name"])
          description = str(show_inventory["result"][0]["systemInformation"]["description"])
          unused_ports[host_name_clean] = {"Model": model, "Description": description}

          # Collect interfaces status
          sh_int_status = node.execute(["show interfaces status notconnect disabled"])
          sh_int_status_clean = sh_int_status["result"][0]["interfaceStatuses"]

          # Identify unused ports and categorize based on bandwidth and transceiver type
          for each_interface in sh_int_status_clean.keys():
              if "." not in each_interface and "Ethernet" in each_interface:
                  bandwidth = sh_int_status_clean[each_interface]["bandwidth"]
                  bandwidth_GE = str(bandwidth / 1000000000) + "GE"
                  interface_type = str(sh_int_status_clean[each_interface]["interfaceType"])

                  # check for bandwidth entry and add it if not there
                  if bandwidth_GE not in unused_ports[host_name_clean]:
                      unused_ports[host_name_clean][bandwidth_GE] = {}

                  # check for interface type and add it if not there
                  if interface_type not in unused_ports[host_name_clean][bandwidth_GE]:
                      unused_ports[host_name_clean][bandwidth_GE][interface_type] = 1
                  else:
                      unused_ports[host_name_clean][bandwidth_GE][interface_type] += 1

      except pyeapi.eapilib.ConnectionError:
          errors[switch] = "ConnectionError: unable to connect to eAPI"

      except pyeapi.eapilib.CommandError:
          errors[switch] = "CommandError: Check your EOS command syntax"


  ### Section 5

  if bool(errors):
      pprint.pprint(errors)

  if bool(unused_ports):
      pprint.pprint(unused_ports)

Final Script
------------

Here is the final script that shows the inventory of unused ports and transceivers in the network.

::

  """
  Discover Unused Ports in the network
  """

  ### Section 1

  import pyeapi
  import getpass
  import pprint

  ### Section 2

  # Define file path and file names

  file_path = "/Users/anees/Google Drive/my-scripts/"
  file_name_switches = "switches.txt"
  file_switches = file_path + file_name_switches

  # Read the content of the file and save it in a List

  switches = []
  with open(file_switches) as readfile:
      for line in readfile:
          switches.append(line.strip())

  ### Section 3

  # Input Username and Password

  my_username = raw_input("Enter your username: ")
  my_password = getpass.getpass("Enter your password: ")

  ### Section 4

  # Create an Empty Dictionary
  unused_ports = {}
  errors = {}

  for switch in switches:
      try:
          # Define API Connection String
          node = pyeapi.connect(transport="https", host=switch, username=my_username, password=my_password, port=None)

          # collect the hostname of the switch and create an entry in the dictionary
          host_name = node.execute(["show hostname"])
          host_name_clean = str(host_name["result"][0]["hostname"])
          unused_ports[host_name_clean] = {}

          # Collect the model name and device description
          show_inventory = node.execute(["show inventory"])
          model = str(show_inventory["result"][0]["systemInformation"]["name"])
          description = str(show_inventory["result"][0]["systemInformation"]["description"])
          unused_ports[host_name_clean] = {"Model": model, "Description": description}

          # Collect interfaces status
          sh_int_status = node.execute(["show interfaces status notconnect disabled"])
          sh_int_status_clean = sh_int_status["result"][0]["interfaceStatuses"]

          # Identify unused ports and categorize based on bandwidth and transceiver type
          for each_interface in sh_int_status_clean.keys():
              if "." not in each_interface and "Ethernet" in each_interface:
                  bandwidth = sh_int_status_clean[each_interface]["bandwidth"]
                  bandwidth_GE = str(bandwidth / 1000000000) + "GE"
                  interface_type = str(sh_int_status_clean[each_interface]["interfaceType"])

                  # check for bandwidth entry and add it if not there
                  if bandwidth_GE not in unused_ports[host_name_clean]:
                      unused_ports[host_name_clean][bandwidth_GE] = {}

                  # check for interface type and add it if not there
                  if interface_type not in unused_ports[host_name_clean][bandwidth_GE]:
                      unused_ports[host_name_clean][bandwidth_GE][interface_type] = 1
                  else:
                      unused_ports[host_name_clean][bandwidth_GE][interface_type] += 1

      except pyeapi.eapilib.ConnectionError:
          errors[switch] = "ConnectionError: unable to connect to eAPI"

      except pyeapi.eapilib.CommandError:
          errors[switch] = "CommandError: Check your EOS command syntax"


  ### Section 5

  if bool(errors):
      pprint.pprint(errors)

  if bool(unused_ports):
      pprint.pprint(unused_ports)

Hardware Scalability Assessment
===============================

The goal of this script is to identify the usage of hardware resources such as mac address, arp, route and tcam tables. We are going to collect the information and report it in the following format.

.. code-block:: bash

  Verify_scale =
  {

  switch_host_name: { “MAC Scale”: <mac count>,
  		          “No. of VRFs”: <no. of VRFs>,
  		          “ARP Scale”: <arp count>,
            “Routing Scale”: <routes>,
  		          “TCAM Scale”: <tcam entries>,
                                   },
  switch_host_name: { “MAC Scale”: <mac count>,
  		          “No. of VRFs”: <no. of VRFs>,
  		          “ARP Scale”: <arp count>,
            “Routing Scale”: <routes>,
  		          “TCAM Scale”: <tcam entries>,
                                   },
  }

We are going to develop the scripts individually for each of these hardware resources using Python functions and then we will combine these functions in one script.

Mac Address Table Scale
-----------------------

Arista EOS Show Command
^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: bash
  :emphasize-lines: 5,10,15,20,25

  Arista-switch#show mac address-table count | json
  {
      "vlanCounts": {
          "115": {
              "dynamic": 2,
              "unicast": 1,
              "multicast": 0
          },
          "116": {
              "dynamic": 0,
              "unicast": 0,
              "multicast": 0
          },
          "4094": {
              "dynamic": 0,
              "unicast": 1,
              "multicast": 0
          },
          "1": {
              "dynamic": 5,
              "unicast": 0,
              "multicast": 0
          },
          "114": {
              "dynamic": 2,
              "unicast": 1,
              "multicast": 0
          },

Develop Script
^^^^^^^^^^^^^^

If you have observed all the scripts we have developed so far, we instantiate pyeapi object using connection parameters (IP address and authentication credentials) and using that object we execute various Arista EOS commands. In this script, we will instantiate the pyeapi object in the main script. Then we will create a function and define all the logic related to identifying mac address scale inside that function. First we will write this function to simply collect the mac address table count from the switch and print.

Open the IDLE, create a new script and save as mac_scale.py in your folder.

.. code-block:: python

  import pyeapi
  import pprint

  def mac_scale(node):
      mac = node.execute(["show mac address-table count"])
      pprint.pprint(mac)

  node = pyeapi.connect(transport="https", host="172.28.170.97", username="admin", password="admin", port=None)
  mac_scale(node)

Save and run the script.

.. code-block:: bash

  >>> ================================ RESTART ================================
  >>>
  {u'id': u'4439995728',
   u'jsonrpc': u'2.0',
   u'result': [{u'vlanCounts': {u'1': {u'dynamic': 5,
                                       u'multicast': 0,
                                       u'unicast': 0},
                                u'201': {u'dynamic': 0,
                                         u'multicast': 0,
                                         u'unicast': 1},
                                u'202': {u'dynamic': 0,
                                         u'multicast': 0,
                                         u'unicast': 1},
                                u'203': {u'dynamic': 0,
                                         u'multicast': 0,
                                         u'unicast': 1},

We can also pass the result back to the main script and we can print from the main script.

.. code-block:: python
  :emphasize-lines: 6,9,10

  import pyeapi
  import pprint

  def mac_scale(node):
      mac = node.execute(["show mac address-table count"])
      return mac

  node = pyeapi.connect(transport="https", host="172.28.170.97", username="admin", password="admin", port=None)
  mac = mac_scale(node)
  pprint.pprint(mac)

We can write the script to add the values of mac["result"][0]["vlanCounts"][vlan][ "dynamic"]  from each vlan. Let’s add this logic in the function and return the total mac count instead of the per vlan mac count.

.. code-block:: python
  :emphasize-lines: 6,7,8,9,10

  import pyeapi
  import pprint

  def mac_scale(node):
      show_mac = node.execute(["show mac address-table count"])
      show_mac_clean = show_mac["result"][0]["vlanCounts"]
      mac_count = 0
      for each_vlan in show_mac_clean.keys():
          mac_count += show_mac_clean[each_vlan]["dynamic"]
      return mac_count

  node = pyeapi.connect(transport="https", host="172.28.170.98", username="admin", password="admin", port=None)
  mac_count = mac_scale(node)
  pprint.pprint(mac_count)

VRF Scale
---------

Arista EOS Show Command
^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: bash
  :emphasize-lines: 2

  Arista-switch#show vrf | json
  % This is an unconverted command

  22sw4#show vrf
     Vrf        RD            Protocols       State             Interfaces
  ---------- ------------- --------------- -------------------- ---------------
     101        101:101       ipv4,ipv6       v4:routing,       Ethernet97.101,
                                              v6:no routing     Ethernet98.101,
                                                                Vlan101
     102        102:102       ipv4,ipv6       v4:routing,       Ethernet97.102,
                                              v6:no routing     Ethernet98.102,
                                                                Vlan102
     103        103:103       ipv4,ipv6       v4:routing,       Ethernet97.103,
                                              v6:no routing     Ethernet98.103,
                                                                Vlan103
     104        104:104       ipv4,ipv6       v4:routing,       Ethernet97.104,
                                              v6:no routing     Ethernet98.104,
                                                                Vlan104
     105        105:105       ipv4,ipv6       v4:routing,       Ethernet97.105,
                                              v6:no routing     Ethernet98.105,
                                                                Vlan105
     106        106:106       ipv4,ipv6       v4:routing,       Ethernet97.106,
                                              v6:no routing     Ethernet98.106,
                                                                Vlan106

As you can see, “show vrf” command is not converted to json format. We will explore how we can parse the required data for the commands that are not converted to json format.

Develop Script
^^^^^^^^^^^^^^

Open the IDLE, create a new script and save as vrf_scale.py in your folder.

.. code-block:: python

  import pyeapi

  node = pyeapi.connect(transport="https", host="172.28.170.98", username="admin", password="admin", port=None)

  show_vrf = node.execute(["show vrf"])

Save and run the script.

.. code-block:: bash
  :emphasize-lines: 11

  ================================ RESTART ================================
  >>>

  Traceback (most recent call last):
    File "/Users/anees/Google Drive/my-scripts/test.py", line 5, in <module>
      show_vrf = node.execute(["show vrf"])
    File "/Library/Python/2.7/site-packages/pyeapi/eapilib.py", line 464, in execute
      response = self.send(request)
    File "/Library/Python/2.7/site-packages/pyeapi/eapilib.py", line 385, in send
      raise CommandError(code, msg, command_error=err, output=out)
  CommandError: CLI command 1 of 1 'show vrf' failed: unconverted command

Since the command is not converted to json format, we have to collect the output using “text” format. For that we are going to use the Python module called jsonrpclib instead of Arista’s pyeapi module.

Install the jsonrpclib module using pip on your system.

.. code-block:: bash
  :caption: Apple Mac

  anees:~ anees$ pip install jsonrpclib

We will write a script using jsonrpc to collect the output for “show vrf” in “text” format.

.. code-block:: python
  :emphasize-lines: 5

  from jsonrpclib import Server

  node = Server("https://admin:admin@172.28.170.98/command-api")

  show_vrf = node.runCmds(1, ["show vrf"], "text")

Save and run the script.

.. code-block:: bash
  :emphasize-lines: 13

  >>> ================================ RESTART ================================
  >>>

  Traceback (most recent call last):
    File "/Users/anees/Google Drive/my-scripts/test.py", line 5, in <module>
      show_vrf = node.runCmds(1, ["show vrf"], "text")
    File "/Library/Python/2.7/site-packages/jsonrpclib/jsonrpc.py", line 288, in __call__

  !!! Output truncated

  "/System/Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7/ssl.py", line 808, in do_handshake
      self._sslobj.do_handshake()
  SSLError: [SSL: CERTIFICATE_VERIFY_FAILED] certificate verify failed (_ssl.c:590)

SSLError is due to the fact that certification verification is enabled by default from Python 2.7.9 onwards. For more information refer `PEP 476 <https://www.python.org/dev/peps/pep-0476/>`_. We will disable SSL verification to move forward with the script.

.. code-block:: python
  :emphasize-lines: 2,3

  from jsonrpclib import Server
  import ssl
  ssl._create_default_https_context = ssl._create_unverified_context


  node = Server("https://admin:admin@172.28.170.98/command-api")

  show_vrf = node.runCmds(1, ["show vrf"], "text")

Save and run the script.

.. code-block:: python

  >>> ================================ RESTART ================================
  >>>
  >>> show_vrf
  [{u'output': u'   Vrf        RD            Protocols       State             Interfaces      \n
                  ---------- ------------- --------------- -------------------- --------------- \n
                  101        101:101       ipv4,ipv6       v4:routing,          Ethernet97.101, \n
                                                              v6:no routing     Ethernet98.101, \n
                                                                                Vlan101         \n
                  102        102:102       ipv4,ipv6       v4:routing,       Ethernet97.102, \n
                                                              v6:no routing     Ethernet98.102, \n
                                                                                Vlan102         \n
                  103        103:103       ipv4,ipv6       v4:routing,       Ethernet97.103, \n
                                                             v6:no routing

We have to use Python’s string parsing modules such as splitlines() and split() to parse line by line and word by word to get the required field. This method is often called as screen scrapping.

As you can see from the “show vrf” output, it has some of the lines of data that are not necessary for our specific use case. This may make the parsing complicated as well. So we use EOS’s include option to filter only the required line and then we use Python’s string parsing modules to collect the necessary field.

.. code-block:: bash

  Arista-switch#show vrf | inc ipv4,ipv6
     101        101:101       ipv4,ipv6       v4:routing,       Ethernet97.101,
     102        102:102       ipv4,ipv6       v4:routing,       Ethernet97.102,
     103        103:103       ipv4,ipv6       v4:routing,       Ethernet97.103,
     104        104:104       ipv4,ipv6       v4:routing,       Ethernet97.104,
     105        105:105       ipv4,ipv6       v4:routing,       Ethernet97.105,
     106        106:106       ipv4,ipv6       v4:routing,       Ethernet97.106,

Let us update the vrf_scale.py with the EOS command with the filters.

.. code-block:: python
  :emphasize-lines: 7

  from jsonrpclib import Server
  import ssl
  ssl._create_default_https_context = ssl._create_unverified_context

  node = Server("https://admin:admin@172.28.170.98/command-api")

  show_vrf = node.runCmds(1, ["show vrf | include ipv4,ipv6"], "text")

Save and run the script. Then we are going to explore Python's string parsing functions to derive the list of VRFs.

.. code-block:: python

  >>> ================================ RESTART ================================
  >>>
  >>> show_vrf
  [{u'output': u'   101        101:101       ipv4,ipv6       v4:routing,
  Ethernet97.101, \n   102        102:102       ipv4,ipv6       v4:routing,
  Ethernet97.102, \n   103        103:103       ipv4,ipv6       v4:routing,
  Ethernet97.103, \n   104        104:104       ipv4,ipv6       v4:routing,
  Ethernet97.104, \n   105        105:105       ipv4,ipv6       v4:routing,
  Ethernet97.105, \n   106        106:106       ipv4,ipv6       v4:routing,
  Ethernet97.106, \n   107        107:107       ipv4,ipv6       v4:routing,
  Ethernet97.107, \n   108        108:108       ipv4,ipv6       v4:routing,
  Ethernet97.108, \n   109        109:109       ipv4,ipv6       v4:routing,
  Ethernet97.109, \n   110        110:110       ipv4,ipv6       v4:routing,
  Ethernet97.110, \n   111        111:111       ipv4,ipv6       v4:routing,
  Ethernet97.111, \n   112        112:112       ipv4,ipv6       v4:routing,
  Ethernet97.112, \n   113        113:113       ipv4,ipv6       v4:routing,
  Ethernet97.113, \n   114        114:114       ipv4,ipv6       v4:routing,
  Ethernet97.114, \n   115        115:115       ipv4,ipv6       v4:routing,
  Ethernet97.115, \n   mgmt       100:100       ipv4,ipv6       v4:no routing,
  Management1     \n'}]
  >>>
  >>> show_vrf[0]["output"]
  u'   101        101:101       ipv4,ipv6       v4:routing,       Ethernet97.101,
  \n   102        102:102       ipv4,ipv6       v4:routing,       Ethernet97.102,
  \n   103        103:103       ipv4,ipv6       v4:routing,       Ethernet97.103,
  \n   104        104:104       ipv4,ipv6       v4:routing,       Ethernet97.104,
  \n   105        105:105       ipv4,ipv6       v4:routing,       Ethernet97.105,
  \n   106        106:106       ipv4,ipv6       v4:routing,       Ethernet97.106,
  \n   107        107:107       ipv4,ipv6       v4:routing,       Ethernet97.107,
  \n   108        108:108       ipv4,ipv6       v4:routing,       Ethernet97.108,
  \n   109        109:109       ipv4,ipv6       v4:routing,       Ethernet97.109,
  \n   110        110:110       ipv4,ipv6       v4:routing,       Ethernet97.110,
  \n   111        111:111       ipv4,ipv6       v4:routing,       Ethernet97.111,
  \n   112        112:112       ipv4,ipv6       v4:routing,       Ethernet97.112,
  \n   113        113:113       ipv4,ipv6       v4:routing,       Ethernet97.113,
  \n   114        114:114       ipv4,ipv6       v4:routing,       Ethernet97.114,
  \n   115        115:115       ipv4,ipv6       v4:routing,       Ethernet97.115,
  \n   mgmt       100:100       ipv4,ipv6       v4:no routing,    Management1
  \n'
  >>> show_vrf_clean = show_vrf[0]["output"]
  >>>
  >>> show_vrf_clean.splitlines()
  [u'   101        101:101       ipv4,ipv6       v4:routing,
  Ethernet97.101, ', u'   102        102:102       ipv4,ipv6       v4:routing,
  Ethernet97.102, ', u'   103        103:103       ipv4,ipv6       v4:routing,
  Ethernet97.103, ', u'   104        104:104       ipv4,ipv6       v4:routing,
  Ethernet97.104, ', u'   105        105:105       ipv4,ipv6       v4:routing,
  Ethernet97.105, ', u'   106        106:106       ipv4,ipv6       v4:routing,
  Ethernet97.106, ', u'   107        107:107       ipv4,ipv6       v4:routing,
  Ethernet97.107, ', u'   108        108:108       ipv4,ipv6       v4:routing,
  Ethernet97.108, ', u'   109        109:109       ipv4,ipv6       v4:routing,
  Ethernet97.109, ', u'   110        110:110       ipv4,ipv6       v4:routing,
  Ethernet97.110, ', u'   111        111:111       ipv4,ipv6       v4:routing,
  Ethernet97.111, ', u'   112        112:112       ipv4,ipv6       v4:routing,
  Ethernet97.112, ', u'   113        113:113       ipv4,ipv6       v4:routing,
  Ethernet97.113, ', u'   114        114:114       ipv4,ipv6       v4:routing,
  Ethernet97.114, ', u'   115        115:115       ipv4,ipv6       v4:routing,
  Ethernet97.115, ', u'   mgmt       100:100       ipv4,ipv6       v4:no routing,
  Management1     ']
  >>>
  >>> type(show_vrf_clean.splitlines())
  <type 'list'>
  >>>
  >>> for each_line in show_vrf_clean.splitlines():
  	print each_line


     101        101:101       ipv4,ipv6       v4:routing,       Ethernet97.101,
     102        102:102       ipv4,ipv6       v4:routing,       Ethernet97.102,
     103        103:103       ipv4,ipv6       v4:routing,       Ethernet97.103,
     104        104:104       ipv4,ipv6       v4:routing,       Ethernet97.104,
     105        105:105       ipv4,ipv6       v4:routing,       Ethernet97.105,
     106        106:106       ipv4,ipv6       v4:routing,       Ethernet97.106,
     107        107:107       ipv4,ipv6       v4:routing,       Ethernet97.107,
     108        108:108       ipv4,ipv6       v4:routing,       Ethernet97.108,
     109        109:109       ipv4,ipv6       v4:routing,       Ethernet97.109,
     110        110:110       ipv4,ipv6       v4:routing,       Ethernet97.110,
     111        111:111       ipv4,ipv6       v4:routing,       Ethernet97.111,
     112        112:112       ipv4,ipv6       v4:routing,       Ethernet97.112,
     113        113:113       ipv4,ipv6       v4:routing,       Ethernet97.113,
     114        114:114       ipv4,ipv6       v4:routing,       Ethernet97.114,
     115        115:115       ipv4,ipv6       v4:routing,       Ethernet97.115,
     mgmt       100:100       ipv4,ipv6       v4:no routing,    Management1

  >>> each_line
  u'   mgmt       100:100       ipv4,ipv6       v4:no routing,    Management1     '
  >>>
  >>> each_line.split()
  [u'mgmt', u'100:100', u'ipv4,ipv6', u'v4:no', u'routing,', u'Management1']

  >>> each_line.split()[0]
  u'mgmt'

  >>> str(each_line.split()[0])
  'mgmt'

  >>> for each_line in show_vrf_clean.splitlines():
  	print str(each_line.split()[0])

  101
  102
  103
  104
  105
  106
  107
  108
  109
  110
  111
  112
  113
  114
  115
  mgmt

Now we have an idea on how to use jsonrpc to collect the show output in “text” format and parse the output using Python’s string processing modules.

Let’s update our script with a function for VRF scale.

.. code-block:: python
  :emphasize-lines: 11,14,20

  import pprint
  from jsonrpclib import Server
  import ssl
  ssl._create_default_https_context = ssl._create_unverified_context

  def vrf_scale(node):
      show_vrf = node.runCmds(1, ["show vrf | include ipv4,ipv6"], "text")
      show_vrf_clean = show_vrf[0]["output"]

      # Create a list to store the VRFs
      vrfs = []
      for line in show_vrf_clean.splitlines():
          fields = line.split()
          vrfs.append(fields[0])
      return vrfs

  node = Server("https://admin:admin@172.28.170.97/command-api")
  vrfs = vrf_scale(node)

  print ("Number of VRFs: %s" %(len(vrfs)))
  print("VRFs List")
  pprint.pprint(vrfs)

Save and run the script.

.. code-block:: bash

  >>> ================================ RESTART ================================
  >>>
  Number of VRFs: 16
  VRFs List
  [u'101',
   u'102',
   u'103',
   u'104',
   u'105',
   u'106',
   u'107',
   u'108',
   u'109',
   u'110',
   u'111',
   u'112',
   u'113',
   u'114',
   u'115',
   u'mgmt']

ARP Scale
---------

In VRF environment, we have to calculate the number of ARP entries per VRF and add them to derive the total number of ARP entries.

Arista EOS Show Command
^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: bash
 :emphasize-lines: 3,12

  Arista-switch# show ip arp vrf 101 summary | json
  {
      "dynamicEntries": 3,
      "ipV4Neighbors": [],
      "notLearnedEntries": 0,
      "totalEntries": 3,
      "staticEntries": 0
  }

  Arista-switch# show ip arp vrf 107 summary | json
  {
      "dynamicEntries": 4,
      "ipV4Neighbors": [],
      "notLearnedEntries": 0,
      "totalEntries": 4,
      "staticEntries": 0
  }

Develop Script
^^^^^^^^^^^^^^

Since we already wrote a function called vrf_scale to identify the VRF names, we can use that function to get the list of VRFs and then we can write a program to identify the ARP scale using that list.

Create a new Python script from IDLE and save it as arp_scale.py.

.. code-block:: python
  :emphasize-lines: 15,16,17,18,27

  import pprint
  from jsonrpclib import Server
  import ssl
  ssl._create_default_https_context = ssl._create_unverified_context

  def vrf_scale(node):
      show_vrf = node.runCmds(1, ["show vrf | include ipv4,ipv6"], "text")
      show_vrf_clean = show_vrf[0]["output"]
      vrfs = []
      for line in show_vrf_clean.splitlines():
          fields = line.split()
          vrfs.append(fields[0])
      return vrfs

  def arp_scale(node, vrfs):
      for each_vrf in vrfs:
          show_arp = node.runCmds(1, ["show ip arp vrf " + each_vrf + " summary"])
          print show_arp[0]["dynamicEntries"]

  # Define Connection Attributes
  node = Server("https://admin:admin@172.28.170.97/command-api")

  # Call VRF function and get the VRF names
  vrfs = vrf_scale(node)

  # Call ARP function
  arp_scale(node, vrfs)

Save and run the script.

.. code-block:: bash

  >>> ================================ RESTART ================================
  >>>
  2
  2
  2
  2
  2
  2
  2
  2
  2
  2
  2
  2
  2
  2
  2
  3

Let us complete the script by adding these number of ARP entries per VRF and return only the total number of ARP entries from the function.

.. code-block:: python
  :emphasize-lines: 16,19,20,29,31

  import pprint
  from jsonrpclib import Server
  import ssl
  ssl._create_default_https_context = ssl._create_unverified_context

  def vrf_scale(node):
      show_vrf = node.runCmds(1, ["show vrf | include ipv4,ipv6"], "text")
      show_vrf_clean = show_vrf[0]["output"]
      vrfs = []
      for line in show_vrf_clean.splitlines():
          fields = line.split()
          vrfs.append(fields[0])
      return vrfs

  def arp_scale(node, vrfs):
      arp_count = 0
      for each_vrf in vrfs:
          show_arp = node.runCmds(1, ["show ip arp vrf " + each_vrf + " summary"])
          arp_count += show_arp[0]["dynamicEntries"]
      return arp_count

  # Define Connection Attributes
  node = Server("https://admin:admin@172.28.170.97/command-api")

  # Call VRF function and get the VRF names
  vrfs = vrf_scale(node)

  # Call ARP function
  arp_count = arp_scale(node, vrfs)

  print ("Total Number of ARP entries %s" % (arp_count))

Save and run the script.

.. code-block:: bash

  >>> ================================ RESTART ================================
  >>>
  Total Number of ARP entries 33

Routing Scale
-------------

In VRF environment, we have to calculate number of routes per VRF and add them to derive the total number of routes.

Arista EOS Show Command
^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: bash
  :emphasize-lines: 9,25

  Arista-switch# show ip route vrf 101 summary | json
  {
      "maskLen": {
          "24": 626,
          "8": 2,
          "32": 207609,
          "31": 6
      },
      "totalRoutes": 208243,
      "staticNexthopGroup": 0,
      "bgpCounts": {
          "bgpExternal": 208229,
          "bgpInternal": 0,
          "bgpTotal": 208229
      },

  Arista-switch# show ip route vrf 102 summary | json
  {
      "maskLen": {
          "24": 626,
          "8": 2,
          "32": 9,
          "31": 6
      },
      "totalRoutes": 643,
      "staticNexthopGroup": 0,
      "bgpCounts": {
          "bgpExternal": 629,
          "bgpInternal": 0,
          "bgpTotal": 629
      },

Develop Script
^^^^^^^^^^^^^^

We are going to use the same logic as arp_scale.py script to calculate the route scale. We use vrf_scale function to get the list of VRFs and then by using that list we will write a function for calculating route scale.

Create a new Python script from IDLE and save it as route_scale.py.

.. code-block:: python
  :emphasize-lines: 15,16,17,18,19,20,29,31

  import pprint
  from jsonrpclib import Server
  import ssl
  ssl._create_default_https_context = ssl._create_unverified_context

  def vrf_scale(node):
      show_vrf = node.runCmds(1, ["show vrf | include ipv4,ipv6"], "text")
      show_vrf_clean = show_vrf[0]["output"]
      vrfs = []
      for line in show_vrf_clean.splitlines():
          fields = line.split()
          vrfs.append(fields[0])
      return vrfs

  def route_scale(node, vrfs):
      route_count = 0
      for each_vrf in vrfs:
          show_route = node.runCmds(1, ["show ip route vrf " + each_vrf + " summary"])
          route_count += show_route[0]["totalRoutes"]
      return route_count

  # Define Connection Attributes
  node = Server("https://admin:admin@172.28.170.97/command-api")

  # Call VRF function and get the VRF names
  vrfs = vrf_scale(node)

  # Call Route function
  route_count = route_scale(node, vrfs)

  print ("Total Number of IPv4 routes %s" % (route_count))

Save and run the script.

.. code-block:: bash

  >>> ================================ RESTART ================================
  >>>
  Total Number of IPv4 routes 234

TCAM Scale
----------

Arista EOS Show Command
^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: bash
  :emphasize-lines: 2,6,8,13,19

  Arista-switch# show platform trident tcam | json
  % This is an unconverted command

  Arista-switch# show platform trident tcam
  === TCAM summary for switch Linecard0/0 ===
  TCAM group 20 uses 1 entry and can use up to 767 more.
    MLAG uses 1 entries.
  TCAM group 10 uses 38 entries and can use up to 1754 more.
    Mlag control traffic uses 4 entries.
    CVX traffic uses 6 entries.
    L3 Control Priority uses 20 entries.
    IGMP Snooping Flooding uses 8 entries.
  TCAM group 11 uses 58 entries and can use up to 1734 more.
    ACL Management uses 10 entries.
    L2 Control Priority uses 10 entries.
    Storm Control Management uses 2 entries.
    ARP Inspection uses 1 entries.
    L3 Routing uses 35 entries.
  TCAM group 43 uses 1 entry and can use up to 767 more.
    Vxlan EFP uses 1 entries.

We will collect the "show platform" output in "text" format and parse through the string to collect the number of TCAM entries. We can pipe the show output to receive only the interesting lines.

.. code-block:: bash

  Arista-switch# show platform trident tcam | include TCAM group
  TCAM group 20 uses 1 entry and can use up to 767 more.
  TCAM group 10 uses 38 entries and can use up to 1754 more.
  TCAM group 11 uses 58 entries and can use up to 1734 more.
  TCAM group 43 uses 1 entry and can use up to 767 more.

Develop Script
^^^^^^^^^^^^^^

Create a new Python script from IDLE and save it as tcam_scale.py.

.. code-block:: python

  from jsonrpclib import Server
  import ssl
  ssl._create_default_https_context = ssl._create_unverified_context

  node = Server("https://admin:admin@172.28.170.98/command-api")

  show_tcam = node.runCmds(1, ["show platform trident tcam | include TCAM group"], "text")

Save and run the script.

.. code-block:: bash
  :emphasize-lines: 7

  >> ================================ RESTART ================================
  >>>

  Traceback (most recent call last):
    File "/Users/anees/Google Drive/my-scripts/test.py", line 8, in <module>
      show_tcam = node.runCmds(1, ["show platform trident tcam | include TCAM group"], "text")
  ProtocolError: (1002, u"CLI command 1 of 1 'show platform trident tcam | include TCAM group' failed: invalid command")

We need to understand why this command fails when we are running using jsonrpc. Let’s open the command-api explorer for this switch and test the command.

.. image:: images/ch04-pic3.png

The above output shows that the command should run from privileged mode. So we need to send the command “enable” in front of “show platform trident tcam” command.

.. image:: images/ch04-pic4.png

Let’s update the tcam_scale.py script to run the “show platform trident tcam” from privileged mode. Then we will explore the options to strip out the interesting data.

.. code-block:: python
  :emphasize-lines: 7

  from jsonrpclib import Server
  import ssl
  ssl._create_default_https_context = ssl._create_unverified_context

  node = Server("https://admin:admin@172.28.170.98/command-api")

  show_tcam = node.runCmds(1, ["enable", "show platform trident tcam | include TCAM group"], "text")

Save and run the script.

.. code-block:: bash

  >>> ================================ RESTART ================================
  >>>
  >>> show_tcam
  [{u'output': u''}, {u'output': u'TCAM group 20 uses 1 entry and can use up to 767 more.\nTCAM group 10 uses 38 entries and can use up to 1754 more.\nTCAM group 11 uses 58 entries and can use up to 1734 more.\nTCAM group 43 uses 1 entry and can use up to 767 more.\n'}]
  >>>
  >>> show_tcam[1]
  {u'output': u'TCAM group 20 uses 1 entry and can use up to 767 more.\nTCAM group 10 uses 38 entries and can use up to 1754 more.\nTCAM group 11 uses 58 entries and can use up to 1734 more.\nTCAM group 43 uses 1 entry and can use up to 767 more.\n'}
  >>>
  >>> show_tcam[1]["output"]
  u'TCAM group 20 uses 1 entry and can use up to 767 more.\nTCAM group 10 uses 38 entries and can use up to 1754 more.\nTCAM group 11 uses 58 entries and can use up to 1734 more.\nTCAM group 43 uses 1 entry and can use up to 767 more.\n'
  >>>
  >>> show_tcam[1]["output"].splitlines()
  [u'TCAM group 20 uses 1 entry and can use up to 767 more.', u'TCAM group 10 uses 38 entries and can use up to 1754 more.', u'TCAM group 11 uses 58 entries and can use up to 1734 more.', u'TCAM group 43 uses 1 entry and can use up to 767 more.']
  >>>
  >>> for each_line in show_tcam[1]["output"].splitlines():
  	     print each_line

  TCAM group 20 uses 1 entry and can use up to 767 more.
  TCAM group 10 uses 38 entries and can use up to 1754 more.
  TCAM group 11 uses 58 entries and can use up to 1734 more.
  TCAM group 43 uses 1 entry and can use up to 767 more.
  >>>
  >>> each_line
  u'TCAM group 43 uses 1 entry and can use up to 767 more.'
  >>>
  >>> each_line.split()
  [u'TCAM', u'group', u'43', u'uses', u'1', u'entry', u'and', u'can', u'use', u'up', u'to', u'767', u'more.']
  >>>
  >>> each_line.split()[4]
  u'1'
  >>> for each_line in show_tcam[1]["output"].splitlines():
  	     print each_line.split()[4]

  1
  38
  58
  1
  >>> tcam_count = 0
  >>>
  >>> for each_line in show_tcam[1]["output"].splitlines():
  	     tcam_count += each_line.split()[4]

  Traceback (most recent call last):
    File "<pyshell#49>", line 2, in <module>
      tcam_count += each_line.split()[4]
  TypeError: unsupported operand type(s) for +=: 'int' and 'unicode'
  >>>
  >>> for each_line in show_tcam[1]["output"].splitlines():
  	     tcam_count += int(each_line.split()[4])

  >>> tcam_count
  98

Let’s update the tcam_scale.py script with a Python function.

.. code-block:: python

  from jsonrpclib import Server
  import ssl
  ssl._create_default_https_context = ssl._create_unverified_context

  def tcam_scale(node):
      tcam_count = 0
      show_tcam = node.runCmds(1, ["enable", "show platform trident tcam | include TCAM group"], "text")

      for each_line in show_tcam[1]["output"].splitlines():
          tcam_count += int(each_line.split()[4])

      return tcam_count

  # Define Connection Attributes
  node = Server("https://admin:admin@172.28.170.97/command-api")

  # Call tcam scale function
  tcam_count = tcam_scale(node)

  print ("Total Number of TCAM Entries used is %s" % (tcam_count))

Hardware Scale
--------------

We will consolidate all the five scalability use cases in one script. Since we used jsonrpc for most of the scalability use cases, we will convert the mac scalability function to use jsonrpc instead of pyeapi. We will also add a function to find the hostname of the switch for storing the hardware scale under the switch name.

Create a new Python script from IDLE and save it as hardware_scale.py.

.. code-block:: python

  import pprint
  from jsonrpclib import Server
  import ssl
  ssl._create_default_https_context = ssl._create_unverified_context

  def mac_scale(node):
      show_mac = node.runCmds(1, ["show mac address-table count"])
      show_mac_clean = show_mac[0]["vlanCounts"]
      mac_count = 0
      for each_vlan in show_mac_clean.keys():
          mac_count += show_mac_clean[each_vlan]["dynamic"]
      return mac_count

  def vrf_scale(node):
      show_vrf = node.runCmds(1, ["show vrf | include ipv4,ipv6"], "text")
      show_vrf_clean = show_vrf[0]["output"]
      vrfs = []
      for line in show_vrf_clean.splitlines():
          fields = line.split()
          vrfs.append(fields[0])
      return vrfs

  def arp_scale(node, vrfs):
      arp_count = 0
      for each_vrf in vrfs:
          show_arp = node.runCmds(1, ["show ip arp vrf " + each_vrf + " summary"])
          arp_count += show_arp[0]["dynamicEntries"]
      return arp_count

  def route_scale(node, vrfs):
      route_count = 0
      for each_vrf in vrfs:
          show_route = node.runCmds(1, ["show ip route vrf " + each_vrf + " summary"])
          route_count += show_route[0]["totalRoutes"]
      return route_count

  def tcam_scale(node):
      tcam_count = 0
      show_tcam = node.runCmds(1, ["enable", "show platform trident tcam | include TCAM group"], "text")
      for each_line in show_tcam[1]["output"].splitlines():
          tcam_count += int(each_line.split()[4])
      return tcam_count

  def hostname(node):
      host_name = node.runCmds(1, ["show hostname"])
      host_name_clean = str(host_name[0]["hostname"])
      return host_name_clean


  # Define Connection Attributes for jsonrpc
  node = Server("https://admin:admin@172.28.170.97/command-api")

  # Call the hardware scale functions
  name = hostname(node)
  mac_count = mac_scale(node)
  vrfs = vrf_scale(node)
  arp_count = arp_scale(node, vrfs)
  route_count = route_scale(node, vrfs)
  tcam_count = tcam_scale(node)

  # Store the values in a dictionary
  verify_scale = {}
  verify_scale[name] = {"MAC Scale": mac_count,
                        "Number of VRFs": len(vrfs),
                        "Number of ARP Entries": arp_count,
                        "Number of Routes": route_count,
                        "Number of TCAM entries used": tcam_count
                        }

  # Print the result
  pprint.pprint(verify_scale)

Save and the run the script.

.. code-block:: bash

  >>> ================================ RESTART ================================
  >>>
  {'22sw2': {'MAC Scale': 5,
             'Number of ARP Entries': 33,
             'Number of Routes': 234,
             'Number of TCAM entries used': 98,
             'Number of VRFs': 16}}

Now we have the logic for the script to find the hardware scale. Next we will integrate this script with our framework. We will start with section 1, 2 and 3 of the script framework.

.. code-block:: python

  """
  Script for hardware scalability assessment
  """

  ### Section 1

  import pprint
  from jsonrpclib import Server
  import getpass
  import ssl
  ssl._create_default_https_context = ssl._create_unverified_context

  ### Section 2

  # Define file path and file names

  file_path = "/Users/anees/Google Drive/my-scripts/"
  file_name_switches = "switches.txt"
  file_switches = file_path + file_name_switches

  # Read the content of the file and save it in a List

  switches = []
  with open(file_switches) as readfile:
      for line in readfile:
          switches.append(line.strip())

  ### Section 3

  # Input Username and Password

  my_username = raw_input("Enter your username: ")
  my_password = getpass.getpass("Enter your password: ")

Let's add all the scalability assessment functions in section 4A.

.. code-block:: python

  ## Section 4A - Define Hardware Scalability Assessment Functions

  def mac_scale(node):
      show_mac = node.runCmds(1, ["show mac address-table count"])
      show_mac_clean = show_mac[0]["vlanCounts"]
      mac_count = 0
      for each_vlan in show_mac_clean.keys():
          mac_count += show_mac_clean[each_vlan]["dynamic"]
      return mac_count

  def vrf_scale(node):
      show_vrf = node.runCmds(1, ["show vrf | include ipv4,ipv6"], "text")
      show_vrf_clean = show_vrf[0]["output"]
      vrfs = []
      for line in show_vrf_clean.splitlines():
          fields = line.split()
          vrfs.append(fields[0])
      return vrfs

  def arp_scale(node, vrfs):
      arp_count = 0
      for each_vrf in vrfs:
          show_arp = node.runCmds(1, ["show ip arp vrf " + each_vrf + " summary"])
          arp_count += show_arp[0]["dynamicEntries"]
      return arp_count

  def route_scale(node, vrfs):
      route_count = 0
      for each_vrf in vrfs:
          show_route = node.runCmds(1, ["show ip route vrf " + each_vrf + " summary"])
          route_count += show_route[0]["totalRoutes"]
      return route_count

  def tcam_scale(node):
      tcam_count = 0
      show_tcam = node.runCmds(1, ["enable", "show platform trident tcam | include TCAM group"], "text")
      for each_line in show_tcam[1]["output"].splitlines():
          tcam_count += int(each_line.split()[4])
      return tcam_count

  def hostname(node):
      host_name = node.runCmds(1, ["show hostname"])
      host_name_clean = str(host_name[0]["hostname"])
      return host_name_clean

Our script for hardware scalability assessment is written for a single switch and we hard coded switch IP, username and password in the script. We need to redefine the jsonrpc connection object using variables.

We are going to rewrite this statement

.. code-block:: python

  node = Server("https://admin:admin@172.28.170.97/command-api")

with variables as below.

.. code-block:: python

  node = Server("https://"+my_username+":"+my_password+"@"+switch+"/command-api")

Since we need to run this main script for all the switches, we will create a for loop and place the main script that calls all the functions within the for loop.

.. code-block:: python

  ### Section 4B - Main Script

  verify_scale = {}

  for switch in switches:

      # Define Connection Attributes for jsonrpc
      node = Server("https://"+my_username+":"+my_password+"@"+switch+"/command-api")

      # Call the hardware scale functions
      name = hostname(node)
      mac_count = mac_scale(node)
      vrfs = vrf_scale(node)
      arp_count = arp_scale(node, vrfs)
      route_count = route_scale(node, vrfs)
      tcam_count = tcam_scale(node)

      # Store the values in a dictionary

      verify_scale[name] = {"MAC Scale": mac_count,
                            "Number of VRFs": len(vrfs),
                            "Number of ARP Entries": arp_count,
                            "Number of Routes": route_count,
                            "Number of TCAM entries used": tcam_count
                            }

Finally we will print the result in the section 5 using pprint module.

.. code-block:: python

  ### Section 5

  # Print the result
  pprint.pprint(verify_scale)

Save and run the script.

.. code-block:: bash

  >>> ================================ RESTART ================================
  >>>
  Enter your username: admin
  {'22sw2': {'MAC Scale': 5,
             'Number of ARP Entries': 33,
             'Number of Routes': 234,
             'Number of TCAM entries used': 98,
             'Number of VRFs': 16},
   '22sw35': {'MAC Scale': 0,
              'Number of ARP Entries': 2050,
              'Number of Routes': 6307,
              'Number of TCAM entries used': 510,
              'Number of VRFs': 17},
   '22sw37': {'MAC Scale': 0,
              'Number of ARP Entries': 2046,
              'Number of Routes': 6273,
              'Number of TCAM entries used': 510,
              'Number of VRFs': 18}}

Exception Handling
^^^^^^^^^^^^^^^^^^

One final feature we need to add in our script is exception handling. Without the exception handling, the script will terminate even if one switch is not accessible or any one EOS command is incorrect. Our goal is to save the error in a variable and continue executing the program for other switches or EOS commands. Also our goal is to differentiate if it is a connectivity (or access) issue or incorrect EOS command.

To understand the syntax of output error for inaccessible switch and incorrect EOS commands, we will test the script for those errors. We create a small script called jsonexception.py with incorrect EOS command “show hostname1”.

.. code-block:: python

  import pprint
  from jsonrpclib import Server
  import ssl
  ssl._create_default_https_context = ssl._create_unverified_context

  node = Server("https://admin:admin@172.28.170.97/command-api")
  host_name = node.runCmds(1, ["show hostname1"])

Save and run the script.

.. code-block:: bash
  :emphasize-lines: 7

  >>> ================================ RESTART ================================
  >>>

  Traceback (most recent call last):
    File "/Users/anees/jsonexception.py", line 9, in <module>
      host_name = node.runCmds(1, ["show hostname1"])
  ProtocolError: (1002, u"CLI command 1 of 1 'show hostname1' failed: invalid command")

  >>> import jsonrpclib
  >>> dir(jsonrpclib)
  ['Config', 'Fault', 'History', 'MultiCall', 'ProtocolError', 'Server', '__builtins__', '__doc__', '__file__', '__name__', '__package__', '__path__', 'config', 'dumps', 'history', 'jsonclass', 'jsonrpc', 'loads']

Now we will update the script to handle this exception message "ProtocolError".

.. code-block:: python
  :emphasize-lines: 6,10,11

  import pprint
  from jsonrpclib import Server, ProtocolError
  import ssl
  ssl._create_default_https_context = ssl._create_unverified_context

  try:
      node = Server("https://admin:admin@172.28.170.97/command-api")
      host_name = node.runCmds(1, ["show hostname1"])

  except ProtocolError as e:
      print ("Invalid EOS Command %s" % (e))

Save and run the script.

.. code-block:: bash

  >>> ================================ RESTART ================================
  >>>
  Invalid EOS Command (1002, u"CLI command 1 of 1 'show hostname1' failed: invalid command")

In this example, 172.28.170.98 switch is accessible but username and password are not admin/admin.

.. code-block:: python
  :emphasize-lines: 8

  import pprint
  from jsonrpclib import Server, ProtocolError

  import ssl
  ssl._create_default_https_context = ssl._create_unverified_context

  try:
      node = Server("https://admin:admin@172.28.170.98/command-api")
      host_name = node.runCmds(1, ["show hostname"])

  except ProtocolError as e:
      print ("Invalid EOS Command %s" % (e))

Save and run the script.

.. code-block:: bash

  >>> ================================ RESTART ================================
  >>>

  Traceback (most recent call last):
    File "/Users/anees/jsonexception.py", line 7, in <module>
      host_name = node.runCmds(1, ["show hostname"])
    File "/Library/Python/2.7/site-packages/jsonrpclib/jsonrpc.py", line 288, in __call__
      return self.__send(self.__name, args)
    File "/Library/Python/2.7/site-packages/jsonrpclib/jsonrpc.py", line 237, in _request
      response = self._run_request(request)
    File "/Library/Python/2.7/site-packages/jsonrpclib/jsonrpc.py", line 255, in _run_request
      verbose=self.__verbose
    File "/System/Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7/xmlrpclib.py", line 1280, in request
      return self.single_request(host, handler, request_body, verbose)
    File "/System/Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7/xmlrpclib.py", line 1328, in single_request
      response.msg,
  ProtocolError: <ProtocolError for admin:admin@172.28.170.98/command-api: 401 Unauthorized>

As you can see, "except ProtocolError" does not catch this exception of incorrect authentication and the script fails. We will create a except clause to catch all the other error messages.

.. code-block:: python
  :emphasize-lines: 14,15

  import pprint
  from jsonrpclib import Server, ProtocolError

  import ssl
  ssl._create_default_https_context = ssl._create_unverified_context

  try:
      node = Server("https://admin:admin@172.28.170.98/command-api")
      host_name = node.runCmds(1, ["show hostname"])

  except ProtocolError as e:
      print ("Invalid EOS Command %s" % (e))

  except:
      print ("eAPI Connection Error")

Save and run the script.

.. code-block:: bash

  >>> ================================ RESTART ================================
  >>>
  eAPI Connection Error

Let’s update the section 1 & 4B of the script with the exception handling method.

.. code-block:: python
  :emphasize-lines: 4,14,35,36,38,39

  ### Section 1

  import pprint
  from jsonrpclib import Server, ProtocolError
  import getpass
  import ssl
  ssl._create_default_https_context = ssl._create_unverified_context

  ### Section 4B - Main Program

  verify_scale = {}

  for switch in switches:
      try:
          # Define Connection Attributes for jsonrpc
          node = Server("https://"+my_username+":"+my_password+"@"+switch+"/command-api")

          # Call the hardware scale functions
          name = hostname(node)
          mac_count = mac_scale(node)
          vrfs = vrf_scale(node)
          arp_count = arp_scale(node, vrfs)
          route_count = route_scale(node, vrfs)
          tcam_count = tcam_scale(node)

          # Store the values in a dictionary

          verify_scale[name] = {"MAC Scale": mac_count,
                                "Number of VRFs": len(vrfs),
                                "Number of ARP Entries": arp_count,
                                "Number of Routes": route_count,
                                "Number of TCAM entries used": tcam_count
                                }

      except ProtocolError as e:
          verify_scale[switch] = "Invalid EOS Command" + str(e)

      except:
          verify_scale[switch] = "eAPI Connection Error"

Now let us test the script. On the switches.txt file, we are going to add an IP address  “172.28.170.98” which we know that having an authentication issue.

.. code-block:: bash

  172.28.170.97
  172.28.170.98
  172.28.170.115
  172.28.170.114

Save and run the script.

.. code-block:: bash

  >>> ================================ RESTART ================================
  >>>
  Enter your username: admin
  {'172.28.170.98': 'eAPI Connection Error',
   '22sw2': {'MAC Scale': 5,
             'Number of ARP Entries': 33,
             'Number of Routes': 234,
             'Number of TCAM entries used': 98,
             'Number of VRFs': 16},
   '22sw35': {'MAC Scale': 0,
              'Number of ARP Entries': 2050,
              'Number of Routes': 6307,
              'Number of TCAM entries used': 510,
              'Number of VRFs': 17},
   '22sw37': {'MAC Scale': 0,
              'Number of ARP Entries': 2046,
              'Number of Routes': 6273,
              'Number of TCAM entries used': 510,
              'Number of VRFs': 18}}

Final Script:
-------------

.. code-block:: python

  """
  Script for hardware scalability assessment
  """

  ### Section 1

  import pprint
  from jsonrpclib import Server, ProtocolError
  import getpass
  import ssl
  ssl._create_default_https_context = ssl._create_unverified_context

  ### Section 2

  # Define file path and file names

  file_path = "/Users/anees/Google Drive/my-scripts/"
  file_name_switches = "switches.txt"
  file_switches = file_path + file_name_switches

  # Read the content of the file and save it in a List

  switches = []
  with open(file_switches) as readfile:
      for line in readfile:
          switches.append(line.strip())

  ### Section 3

  # Input Username and Password

  my_username = raw_input("Enter your username: ")
  my_password = getpass.getpass("Enter your password: ")

  ### Section 4A - Define Hardware Scale Functions

  def mac_scale(node):
      show_mac = node.runCmds(1, ["show mac address-table count"])
      show_mac_clean = show_mac[0]["vlanCounts"]
      mac_count = 0
      for each_vlan in show_mac_clean.keys():
          mac_count += show_mac_clean[each_vlan]["dynamic"]
      return mac_count

  def vrf_scale(node):
      show_vrf = node.runCmds(1, ["show vrf | include ipv4,ipv6"], "text")
      show_vrf_clean = show_vrf[0]["output"]
      vrfs = []
      for line in show_vrf_clean.splitlines():
          fields = line.split()
          vrfs.append(fields[0])
      return vrfs

  def arp_scale(node, vrfs):
      arp_count = 0
      for each_vrf in vrfs:
          show_arp = node.runCmds(1, ["show ip arp vrf " + each_vrf + " summary"])
          arp_count += show_arp[0]["dynamicEntries"]
      return arp_count

  def route_scale(node, vrfs):
      route_count = 0
      for each_vrf in vrfs:
          show_route = node.runCmds(1, ["show ip route vrf " + each_vrf + " summary"])
          route_count += show_route[0]["totalRoutes"]
      return route_count

  def tcam_scale(node):
      tcam_count = 0
      show_tcam = node.runCmds(1, ["enable", "show platform trident tcam | include TCAM group"], "text")
      for each_line in show_tcam[1]["output"].splitlines():
          tcam_count += int(each_line.split()[4])
      return tcam_count

  def hostname(node):
      host_name = node.runCmds(1, ["show hostname"])
      host_name_clean = str(host_name[0]["hostname"])
      return host_name_clean

  ### Section 4B - Main Program

  verify_scale = {}

  for switch in switches:
      try:
          # Define Connection Attributes for jsonrpc
          node = Server("https://"+my_username+":"+my_password+"@"+switch+"/command-api")

          # Call the hardware scale functions
          name = hostname(node)
          mac_count = mac_scale(node)
          vrfs = vrf_scale(node)
          arp_count = arp_scale(node, vrfs)
          route_count = route_scale(node, vrfs)
          tcam_count = tcam_scale(node)

          # Store the values in a dictionary

          verify_scale[name] = {"MAC Scale": mac_count,
                                "Number of VRFs": len(vrfs),
                                "Number of ARP Entries": arp_count,
                                "Number of Routes": route_count,
                                "Number of TCAM entries used": tcam_count
                                }

      except ProtocolError as e:
          verify_scale[switch] = "Invalid EOS Command" + str(e)

      except:
          verify_scale[switch] = "eAPI Connection Error"

  ### Section 5

  # Print the result
  pprint.pprint(verify_scale)
