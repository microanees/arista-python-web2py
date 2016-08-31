Chapter 9: Web2Py - Troubleshooting Use Cases
*********************************************

.. contents:: :local:

In this chapter, we will create functions and views for the tasks in the "Network-wide Troubleshooting" category. The purpose of these tasks is to perform some of the common network troubleshooting steps across the network using the tool.

.. image:: images/ch09-pic1.png

We are going to create three tasks (Data Plane Packet Drops, Control Plane Drops and Last 10 Sev1-3 log messages) in this section. Each task will have one or more functions in the default controller and a corresponding view (.html file).

Data Plane Packet Drops
=======================

We are going to write a function packet_drops() in the default controller of our web2py application Arista_EOS_tool. We have already written a Python script to discover network-wide packet drops in the earlier chapter in this book. We are going to reuse that script here. The core logic is going to be the same. The only change here is to receive the input with web2py form. The input fields are username, password, site and role. We are going to create a drop-down menu to show the sites and roles so that the users don’t have to type them.

.. image:: images/ch09-pic2.png

Algorithm
---------

We are going to use the below algorithm to write the script.

#. Create a New Function and a View for this task "Packet Drops".
#. Display a form to get username, password, site and role.
#. Build Switch IP Address list.
#. Reuse the packet drops Python script and returns the result to the view.

Develop Script
--------------

Step 1: Create a New Function and a View for this task "Packet Drops"
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Go to admin interface using the url https://<web-server>/admin/default/index

Arista_EOS_Tool: Manage --> Edit

.. image:: images/ch09-pic3.png

Controllers: default.py --> Edit

.. image:: images/ch09-pic4.png

Create a new function packet_drops()

.. code-block:: python

  def packet_drops():
      return dict()

Create a View for the function packet_drops

Click the "files toggle" on the top left

Click Create and select views from the drop down window

Provide the file name with path --> default/packet_drops.html

.. image:: images/ch09-pic5.png

We are going to create a default view with the header "Packet Drops". Save this file.

.. code-block:: html

  {{extend 'layout.html'}}
  <h1>Packet Drops</h1>
  {{=BEAUTIFY(response._vars)}}

Step 2: Display a form to get username, password, site and role
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

We know how to create a form with the static fields such as username and password using SQLFORM.factory().

.. code-block:: python

  form = SQLFORM.factory(
      Field('username', requires=IS_NOT_EMPTY()),
      Field('password', 'password', requires=IS_NOT_EMPTY()))

We need to create the fields for site and role with a drop-down menu. The SQLFORM.factory() provides the option using IS_IN_SET() function to display drop-down menu. The drop-down menu can list the content of Python's data structure called set. First we will look at SQLFORM.factory()'s option for drop-down menu, and then we will spend some time in the Python interpreter to understand what is set.

.. code-block:: python

  form = SQLFORM.factory(
      Field('username', requires=IS_NOT_EMPTY()),
      Field('password', 'password', requires=IS_NOT_EMPTY()),
      Field('site', requires=IS_IN_SET(sites)),
      Field('role', requires=IS_IN_SET(roles)))

The "site" and "role" fields are going to show the drop-down menus with the list of sites and roles stored in the set "sites" and "roles" respectively. Now we will explore Python's data structure set. We will create a list with duplicate entries, and then we will convert the list as set.

.. code-block:: bash

  anees:~ anees$ python
  >>>
  >>> sites = ["All", "sjc", "atl", "lax", "sjc"]
  >>>
  >>> sites
  ['All', 'sjc', 'atl', 'lax', 'sjc']
  >>>
  >>> type(sites)
  <type 'list'>
  >>>
  >>> sites = set(["All", "sjc", "atl", "lax", "sjc"])
  >>>
  >>> sites
  set(['sjc', 'All', 'lax', 'atl'])
  >>>
  >>> type(sites)
  <type 'set'>

Before defining the SQLFORM.factory(), we need to build the set sites and roles from the inventory.json file. We will read this file and pull site and role for each switch and add it to the set sites and roles. Since sites and roles are sets, we don’t have to worry about the duplicate entries of site and role from the inventory.json file.

Let’s start building this portion of the script step by step.

.. code-block:: python

  def packet_drops():
      # Read the inventory.json file into a dictionary
      with open(file) as readfile:
          inventory = json.load(readfile)

      # Define set sites and roles with a single item "All"
      sites = set(["All"])
      roles = set(["All"])

      # Read site and role of each switch and add it to the set
      for each_switch in inventory.keys():
          sites.add(inventory[each_switch]["site"])
          roles.add(inventory[each_switch]["role"])

      return dict(sites=sites, roles=roles)

Test your script using the URL https://<web-server>/Arista_EOS_Tool/default/packet_drops.

.. image:: images/ch09-pic6.png

Let’s add the form and validate the display of the form.

.. code-block:: python

  def packet_drops():
      # Read the inventory.json file into a dictionary
      with open(file) as readfile:
          inventory = json.load(readfile)

      # Define set sites and roles with a single item "All"
      sites = set(["All"])
      roles = set(["All"])

      # Read site and role of each switch and add it to the set
      for each_switch in inventory.keys():
          sites.add(inventory[each_switch]["site"])
          roles.add(inventory[each_switch]["role"])

      # Create a form using the sets
      form = SQLFORM.factory(
          Field('username', requires=IS_NOT_EMPTY()),
          Field('password', 'password', requires=IS_NOT_EMPTY()),
          Field('site', requires=IS_IN_SET(sites)),
          Field('role', requires=IS_IN_SET(roles),))

      return dict(form=form)

Test your script using the URL https://<web-server>/Arista_EOS_Tool/default/packet_drops.

.. image:: images/ch09-pic7.png

Step 3: Build Switch IP Address list
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Our goal is to derive the list of switch IP addresses from the inventory.json file based on the selection of site and role by the user. One important point to remember here is that we provide an option "All" in the site and role field of the form. So we need to write an algorithm to derive the switch IP addresses. Let us look at the inventory of the switches before writing the algorithm.

.. image:: images/ch09-pic8.png

Here is the algorithm we are going to use to build the list of switch IP addresses.

* Create an empty list called switches = [ ]
* If the site = "All" and the role = "All", we need all the switch IP addresses from the inventory.json file. So we will assign inventory.keys() to the list switches.
* If both the site and role are not “All”, we have to parse through site and role of each switch against the input selected by the user.

  - If the site = “All” and the role = (user selected), compare the role of each switch against the role selected by the user.
  - If the site = (user selected) and the role = "All compare the site of each switch against the site selected by the user.
  - If the site = (user selected), role = (user selected), compare the site and role of each switch against the input selected by the user.

Let's update the packet_drops() function to derive switch ip addresses after the user submitted the form.

.. code-block:: python
  :emphasize-lines: 22-45

  def packet_drops():
      # Read the inventory.json file into a dictionary
      with open(file) as readfile:
          inventory = json.load(readfile)

      # Define set sites and roles with a single item "All"
      sites = set(["All"])
      roles = set(["All"])

      # Read site and role of each switch and add it to the set
      for each_switch in inventory.keys():
          sites.add(inventory[each_switch]["site"])
          roles.add(inventory[each_switch]["role"])

      # Create a form using the sets
      form = SQLFORM.factory(
          Field('username', requires=IS_NOT_EMPTY()),
          Field('password', 'password', requires=IS_NOT_EMPTY()),
          Field('site', requires=IS_IN_SET(sites)),
          Field('role', requires=IS_IN_SET(roles),))

      if form.process().accepted:
          # Create a switch IP addresses list from site and role selection
          switches = []

          """ compare the site and role between user selected and the values in
          inventory """

          if form.vars.site == "All" and form.vars.role == "All":
              switches = inventory.keys()
          else:
              for each_switch in inventory.keys():
                  each_site = inventory[each_switch]["site"]
                  each_role = inventory[each_switch]["role"]
                  if form.vars.site == "All" and form.vars.role == each_role:
                      switches.append(each_switch)
                  elif (form.vars.site == each_site and
                          form.vars.role == "All"):
                      switches.append(each_switch)
                  elif (form.vars.site == each_site and
                          form.vars.role == each_role):
                      switches.append(each_switch)

          # Return the switches list to the view for verification purpose
          return dict(switches=switches)

      return dict(form=form)

Test your script using the URL https://<web-server>/Arista_EOS_Tool/default/packet_drops. Make sure you see the expected switch IP addresses list by selecting various combinations of sites and roles.

.. image:: images/ch09-pic9.png

Step 4: Reuse the packet drops Python script
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

We will use the packet drops Python script which we wrote in chapter 3. The only thing you should change in that script is to change the username and password variable in the pyeapi.connect() function. You should use form.vars.username and form.vars.password for username and password variable.

.. code-block:: python
  :emphasize-lines: 44-105

  def packet_drops():
      # Read the inventory.json file into a dictionary
      with open(file) as readfile:
          inventory = json.load(readfile)

      # Define set sites and roles with a single item "All"
      sites = set(["All"])
      roles = set(["All"])

      # Read site and role of each switch and add it to the set
      for each_switch in inventory.keys():
          sites.add(inventory[each_switch]["site"])
          roles.add(inventory[each_switch]["role"])

      # Create a form using the sets
      form = SQLFORM.factory(
          Field('username', requires=IS_NOT_EMPTY()),
          Field('password', 'password', requires=IS_NOT_EMPTY()),
          Field('site', requires=IS_IN_SET(sites)),
          Field('role', requires=IS_IN_SET(roles),))

      if form.process().accepted:
          # Create a switch IP addresses list from site and role selection
          switches = []

          """ compare the site and role between user selected and the values in
          inventory """

          if form.vars.site == "All" and form.vars.role == "All":
              switches = inventory.keys()
          else:
              for each_switch in inventory.keys():
                  each_site = inventory[each_switch]["site"]
                  each_role = inventory[each_switch]["role"]
                  if form.vars.site == "All" and form.vars.role == each_role:
                      switches.append(each_switch)
                  elif (form.vars.site == each_site and
                          form.vars.role == "All"):
                      switches.append(each_switch)
                  elif (form.vars.site == each_site and
                          form.vars.role == each_role):
                      switches.append(each_switch)

          # Packet Drops Script
          interface_drops = {}
          errors = {}
          for switch in switches:
              try:
                  # Define API Connection String
                  node = pyeapi.connect(transport="https", host=switch,
                                        username=form.vars.username,
                                        password=form.vars.password,
                                        port=None)
                  # Execute the desired command
                  interface_counters = node.execute(
                      ["show interfaces counters discards"])
                  interface_counters_clean = interface_counters[
                      "result"][0]["interfaces"]
                  host_name = node.execute(["show hostname"])
                  host_name_clean = str(host_name["result"][0]["hostname"])
                  interface_drops[host_name_clean] = {}

                  for interface in interface_counters_clean.keys():
                      if "Port-Channel" not in interface:
                          if interface_counters_clean[interface]["inDiscards"] or interface_counters_clean[interface]["outDiscards"] != 0:
                              interface_drops[host_name_clean][interface] = {}
                              show_interface = node.execute(
                                  ["show interfaces " + str(interface)])
                              show_interface_clean = show_interface["result"][0][
                                  "interfaces"][interface]["interfaceCounters"]
                              interface_drops[host_name_clean][interface]["Interface Status"] = show_interface[
                                  "result"][0]["interfaces"][interface]["interfaceStatus"]
                              interface_drops[host_name_clean][interface]["Line Protocol Status"] = show_interface[
                                  "result"][0]["interfaces"][interface]["lineProtocolStatus"]

                              if interface_counters_clean[interface]["inDiscards"] != 0:
                                  interface_drops[host_name_clean][
                                      interface]["inDiscards"] = {}
                                  interface_drops[host_name_clean][interface]["inDiscards"][
                                      "Total Discards"] = interface_counters_clean[interface]["inDiscards"]
                                  interface_drops[host_name_clean][interface]["inDiscards"][
                                      "Input Errors"] = show_interface_clean["inputErrorsDetail"]

                              if interface_counters_clean[interface]["outDiscards"] != 0:
                                  interface_drops[host_name_clean][
                                      interface]["outDiscards"] = {}
                                  interface_drops[host_name_clean][interface]["outDiscards"][
                                      "Total Discards"] = interface_counters_clean[interface]["outDiscards"]
                                  interface_drops[host_name_clean][interface]["outDiscards"][
                                      "Output Errors"] = show_interface_clean["outputErrorsDetail"]

              except pyeapi.eapilib.ConnectionError:
                  errors[switch] = "ConnectionError: unable to connect to eAPI"

              except pyeapi.eapilib.CommandError:
                  errors[switch] = "CommandError: Check your EOS command syntax"

              if not interface_drops[host_name_clean]:
                  del interface_drops[host_name_clean]

          # Return the switches list to the view for verification purpose
          return dict(errors=errors, interface_drops=interface_drops)

      return dict(form=form)

Test your script using the URL https://<web-server>/Arista_EOS_Tool/default/packet_drops

.. image:: images/ch09-pic10.png

Script Enhancements
-------------------

At this point, we know how to port our scripts to web2py. To summarize, we have three sections in the script.

#. Web2Py form to receive user input.
#. Build Switch List based on the user input.
#. The core logic for your Network Use case.

The first two sections are going to be common for the most use cases. We can create functions for these two sections so that we can re-use them for all our use cases. We will also create a function for discovering the packet drops which will improve the readability and functionality of the script.

Step 1: Create a function to create a web2py form
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

We will create a function called eos_form to build a web2py form. We will call this form from the packet_drops function.

.. code-block:: python

  def eos_form():
      # Read the inventory.json file into a dictionary
      with open(file) as readfile:
          inventory = json.load(readfile)

      # Define set sites and roles with a single item "All"
      sites = set(["All"])
      roles = set(["All"])

      # Read site and role of each switch and add it to the set
      for each_switch in inventory.keys():
          sites.add(inventory[each_switch]["site"])
          roles.add(inventory[each_switch]["role"])

      # Create a form using the sets
      form = SQLFORM.factory(
          Field('username', requires=IS_NOT_EMPTY()),
          Field('password', 'password', requires=IS_NOT_EMPTY()),
          Field('site', requires=IS_IN_SET(sites)),
          Field('role', requires=IS_IN_SET(roles),))

      return form


  def packet_drops():
      # Build Form
      form = eos_form()

      if form.process().accepted:
          # Create a switch IP addresses list from site and role selection
          switches = []

          """ compare the site and role between user selected and the values in
          inventory """

Step 2: Create a function to derive the list of switches
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

We will create a function switches_list to derive the list of switch IP addresses. We will call this function from the packet_drops function. When we call, we need to pass the user input so that switches_list function derives the list from the user provides values for site and role.

.. code-block:: python

  def switches_list(form_vars):
      # Read the inventory
      with open(file) as readfile:
          inventory = json.load(readfile)

      # Define an empty list
      switches = []

      """ compare the site and role between user selected and the values in
            inventory """
      if form_vars.site == "All" and form_vars.role == "All":
          switches = inventory.keys()
      else:
          for each_switch in inventory.keys():
              each_site = inventory[each_switch]["site"]
              each_role = inventory[each_switch]["role"]
              if form_vars.site == "All" and form_vars.role == each_role:
                  switches.append(each_switch)
              elif form_vars.site == each_site and form_vars.role == "All":
                  switches.append(each_switch)
              elif form_vars.site == each_site and form_vars.role == each_role:
                  switches.append(each_switch)

      return switches


  def packet_drops():
      # Build Form
      form = eos_form()

      if form.process().accepted:
          # Create a switch IP addresses list from site and role selection
          switches = switches_list(form.vars)

          # Packet Drops Script
          interface_drops = {}
          errors = {}
          for switch in switches:

Step 3: Create a function for discovering interface drops
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

We will write a function for discovering interface drops and we will call this function from packet_drops function.

.. code-block:: python

  def discover_packet_drops(node):
      # Define empty dictionary
      int_drops = {}

      # Execute the desired command
      int_counters_raw = node.execute(["show interfaces counters discards"])
      int_counters = int_counters_raw["result"][0]["interfaces"]

      for interface in int_counters.keys():
          if "Port-Channel" not in interface:
              if (int_counters[interface]["inDiscards"] or
                      int_counters[interface]["outDiscards"] != 0):
                  int_drops[interface] = {}
                  show_int_raw = node.execute(
                      ["show interfaces " + str(interface)])
                  show_int = show_int_raw["result"][0][
                      "interfaces"][interface]["interfaceCounters"]
                  int_drops[interface]["Interface Status"] = show_int_raw[
                      "result"][0]["interfaces"][interface]["interfaceStatus"]
                  int_drops[interface]["Line Protocol Status"] = show_int_raw[
                      "result"][0]["interfaces"][interface]["lineProtocolStatus"]

                  if int_counters[interface]["inDiscards"] != 0:
                      int_drops[interface]["inDiscards"] = {}
                      int_drops[interface]["inDiscards"][
                          "Total Discards"] = int_counters[interface]["inDiscards"]
                      int_drops[interface]["inDiscards"][
                          "Input Errors"] = show_int["inputErrorsDetail"]

                  if int_counters[interface]["outDiscards"] != 0:
                      int_drops[interface]["outDiscards"] = {}
                      int_drops[interface]["outDiscards"][
                          "Total Discards"] = int_counters[interface]["outDiscards"]
                      int_drops[interface]["outDiscards"][
                          "Output Errors"] = show_int["outputErrorsDetail"]
      return int_drops


  def host_name(node):
      host_name = node.execute(["show hostname"])
      host_name_clean = str(host_name["result"][0]["hostname"])
      return host_name_clean


  def packet_drops():
      # Build Form
      form = eos_form()

      if form.process().accepted:
          # Create a switch IP addresses list from site and role selection
          switches = switches_list(form.vars)

          # Packet Drops Script
          interface_drops = {}
          errors = {}
          for switch in switches:
              try:
                  # Define API Connection String
                  node = pyeapi.connect(transport="https", host=switch,
                                        username=form.vars.username,
                                        password=form.vars.password,
                                        port=None)

                  # Identify the switch hostname for reporting purpose
                  switchname = host_name(node)

                  # Discover Interface Packet Drops
                  interface_drops[switchname] = discover_packet_drops(node)

              except pyeapi.eapilib.ConnectionError:
                  errors[switch] = "ConnectionError: unable to connect to eAPI"

              except pyeapi.eapilib.CommandError:
                  errors[switch] = "CommandError: Check your EOS command syntax"

              if not interface_drops[switchname]:
                  del interface_drops[switchname]

          # Return the switches list to the view for verification purpose
          return dict(errors=errors, interface_drops=interface_drops)

      return dict(form=form)

Final Script
------------

The final script with all the functions is shown below.

.. code-block:: python

  def eos_form():
      # Read the inventory.json file into a dictionary
      with open(file) as readfile:
          inventory = json.load(readfile)

      # Define set sites and roles with a single item "All"
      sites = set(["All"])
      roles = set(["All"])

      # Read site and role of each switch and add it to the set
      for each_switch in inventory.keys():
          sites.add(inventory[each_switch]["site"])
          roles.add(inventory[each_switch]["role"])

      # Create a form using the sets
      form = SQLFORM.factory(
          Field('username', requires=IS_NOT_EMPTY()),
          Field('password', 'password', requires=IS_NOT_EMPTY()),
          Field('site', requires=IS_IN_SET(sites)),
          Field('role', requires=IS_IN_SET(roles),))

      return form


  def switches_list(form_vars):
      # Read the inventory
      with open(file) as readfile:
          inventory = json.load(readfile)

      # Define an empty list
      switches = []

      """ compare the site and role between user selected and the values in
            inventory """
      if form_vars.site == "All" and form_vars.role == "All":
          switches = inventory.keys()
      else:
          for each_switch in inventory.keys():
              each_site = inventory[each_switch]["site"]
              each_role = inventory[each_switch]["role"]
              if form_vars.site == "All" and form_vars.role == each_role:
                  switches.append(each_switch)
              elif form_vars.site == each_site and form_vars.role == "All":
                  switches.append(each_switch)
              elif form_vars.site == each_site and form_vars.role == each_role:
                  switches.append(each_switch)

      return switches


  def discover_packet_drops(node):
      # Define empty dictionary
      int_drops = {}

      # Execute the desired command
      int_counters_raw = node.execute(["show interfaces counters discards"])
      int_counters = int_counters_raw["result"][0]["interfaces"]

      for interface in int_counters.keys():
          if "Port-Channel" not in interface:
              if (int_counters[interface]["inDiscards"] or
                      int_counters[interface]["outDiscards"] != 0):
                  int_drops[interface] = {}
                  show_int_raw = node.execute(
                      ["show interfaces " + str(interface)])
                  show_int = show_int_raw["result"][0][
                      "interfaces"][interface]["interfaceCounters"]
                  int_drops[interface]["Interface Status"] = show_int_raw[
                      "result"][0]["interfaces"][interface]["interfaceStatus"]
                  int_drops[interface]["Line Protocol Status"] = show_int_raw[
                      "result"][0]["interfaces"][interface]["lineProtocolStatus"]

                  if int_counters[interface]["inDiscards"] != 0:
                      int_drops[interface]["inDiscards"] = {}
                      int_drops[interface]["inDiscards"][
                          "Total Discards"] = int_counters[interface]["inDiscards"]
                      int_drops[interface]["inDiscards"][
                          "Input Errors"] = show_int["inputErrorsDetail"]

                  if int_counters[interface]["outDiscards"] != 0:
                      int_drops[interface]["outDiscards"] = {}
                      int_drops[interface]["outDiscards"][
                          "Total Discards"] = int_counters[interface]["outDiscards"]
                      int_drops[interface]["outDiscards"][
                          "Output Errors"] = show_int["outputErrorsDetail"]
      return int_drops


  def host_name(node):
      host_name = node.execute(["show hostname"])
      host_name_clean = str(host_name["result"][0]["hostname"])
      return host_name_clean


  def packet_drops():
      # Build Form
      form = eos_form()

      if form.process().accepted:
          # Create a switch IP addresses list from site and role selection
          switches = switches_list(form.vars)

          # Packet Drops Script
          interface_drops = {}
          errors = {}
          for switch in switches:
              try:
                  # Define API Connection String
                  node = pyeapi.connect(transport="https", host=switch,
                                        username=form.vars.username,
                                        password=form.vars.password,
                                        port=None)

                  # Identify the switch hostname for reporting purpose
                  switchname = host_name(node)

                  # Discover Interface Packet Drops
                  interface_drops[switchname] = discover_packet_drops(node)

              except pyeapi.eapilib.ConnectionError:
                  errors[switch] = "ConnectionError: unable to connect to eAPI"

              except pyeapi.eapilib.CommandError:
                  errors[switch] = "CommandError: Check your EOS command syntax"

              if not interface_drops[switchname]:
                  del interface_drops[switchname]

          # Return the switches list to the view for verification purpose
          return dict(errors=errors, interface_drops=interface_drops)

      return dict(form=form)

Control Plane Drops
===================

This is going to be a very simple script. We have already written functions for form and switch list. We have also written script for control plane drops in the chapter 3.

We will write a new function called discover_copp_drops() in default.py and re-use the script for the control plane drops that we wrote in the chapter 3. Then we will create another function called controlplane_drops() in default.py to build our use-case by using all the three functions.

.. code-block:: python

  def discover_copp_drops(node):
      # Define empty dictionary
      copp_drops = {}

      # Execute the desired command
      copp_counters_raw = node.execute(
          ["show policy-map interface control-plane copp-system-policy"])
      copp_counters = copp_counters_raw["result"][0][
          "policyMaps"]["copp-system-policy"]["classMaps"]

      # parse through each copp system class map and discover non drop counters
      for each_copp_class in copp_counters.keys():
          if (copp_counters[each_copp_class]["intfPacketCounters"]
                  ["dropPackets"] != 0):
              copp_drops[each_copp_class] = {}
              copp_drops[each_copp_class]["Drop Packets"] = copp_counters[
                  each_copp_class]["intfPacketCounters"]["dropPackets"]

      return copp_drops


  def controlplane_drops():
      # Build Form
      form = eos_form()

      if form.process().accepted:
          # Create a switch IP addresses list from site and role selection
          switches = switches_list(form.vars)

          # CoPP Drops Script
          copp_drops = {}
          errors = {}
          for switch in switches:
              try:
                  # Define API Connection String
                  node = pyeapi.connect(transport="https", host=switch,
                                        username=form.vars.username,
                                        password=form.vars.password,
                                        port=None)

                  # Identify the switch hostname for reporting purpose
                  switchname = host_name(node)

                  # Discover CoPP Drops
                  copp_drops[switchname] = discover_copp_drops(node)

              except pyeapi.eapilib.ConnectionError:
                  errors[switch] = "ConnectionError: unable to connect to eAPI"

              except pyeapi.eapilib.CommandError:
                  errors[switch] = "CommandError: Check your EOS command syntax"

              if not copp_drops[switchname]:
                  del copp_drops[switchname]

          # Return the switches list to the view for verification purpose
          return dict(errors=errors, copp_drops=copp_drops)

      return dict(form=form)

Create a View for the function controlplane_drops

Click the "files toggle" on the top left

Click Create and select views from the drop down window

Provide the file name with path --> default/controlplane_drops.html

We are going to create a default view with the header "Control Plane Drops". Save this file.

.. code-block:: html

  {{extend 'layout.html'}}
  <h1>Control Plane Drops</h1>
  {{=BEAUTIFY(response._vars)}}

Test your script using the URL https://<web-server>/Arista_EOS_Tool/default/controlplane_drops

.. image:: images/ch09-pic11.png

Last 10 Sev 1-3 Log Messages
============================

The goal of this script is to collect the last 10 severity 1-3 messages from the switches. It will be helpful when you are troubleshooting a network-wide problem and quickly run this command and look at the severity 1-3 logs on the switches within the network. We have not written a script for this logic previously in this book. So we will first explore the logic through IDLE and then we will port the logic into this tool.

Let’s login to an Arista switch and explore the required EOS commands.

.. code-block:: console

  Switch# show logging | json
  % This is an unconverted command

  switch# show logging 10 errors

  Jan  9 10:55:57 22sw2 ribd-vrf-102[27895]: %BGP-3-NOTIFICATION: sent to neighbor 10.10.202.11 (AS 99) 4/0 (Hold Timer Expired Error/Unspecified) 0 bytes

  Jan  9 10:55:57 22sw2 ribd-vrf-111[28249]: %BGP-3-NOTIFICATION: sent to neighbor 10.10.211.11 (AS 99) 4/0 (Hold Timer Expired Error/Unspecified) 0 bytes

Before looking at the algorithm, let’s take a look at the basic Python script to pull the logs from the switch. Since the show log is not json converted, we will use the jsonrpclib with “text” format, and we parse the data using the string parsing methods.

.. code-block:: python

  import pprint
  from jsonrpclib import Server
  import ssl
  ssl._create_default_https_context = ssl._create_unverified_context

  node = Server("https://admin:admin@172.28.170.97/command-api")

  show_log = node.runCmds(1, ["show logging 10 errors"], "text")
  show_log_clean = show_log[0]["output"]

  pprint.pprint(show_log_clean)

Save and run this script.

.. code-block:: console

  >>> ================================ RESTART ================================
  >>>
  u'Jan  9 10:55:57 22sw2 ribd-vrf-102[27895]: %BGP-3-NOTIFICATION: sent to neighbor 10.10.202.11 (AS 99) 4/0 (Hold Timer Expired Error/Unspecified) 0 bytes\nJan  9 10:55:57 22sw2 ribd-vrf-111[28249]: %BGP-3-NOTIFICATION: sent to neighbor 10.10.211.11 (AS 99) 4/0 (Hold Timer Expired Error/Unspecified) 0 bytes\n'

Algorithm
---------

We will build the script directly from web2py editor. We are going to write a function switch_logs() in the default controller of our web2py application Arista_EOS_tool. We are going to use the below algorithm to write the script.

1. Create a New Function and a View for this task “show logs."
2. Use the eos_form() and switches_list() function to display form.
3. Write the core logic of the script

  a) Create a list for the "logging" levels 0 to 3, which are emergencies, alerts, critical and errors.
  b) For each "logging" level within the list, collect the show logging 10 <logging_level> output.
  c) For each logging level, parse through the required output and store under the corresponding switch entry within the dictionary.

4. Return the dictionary show_log to the view.

Develop Script
--------------

Step 1: Create a New Function and a View for this task “show logs"
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Create a new function switch_logs() in the default.py controller.

.. code-block:: python

  def switch_logs():
      return dict()

Create a view switch_logs.html under views/default folder.

.. code-block:: html

  {{extend 'layout.html'}}
  <h1>Display Logs with Severity 0 to 3 </h1>
  {{=BEAUTIFY(response._vars)}}

Step 2: Use the eos_form() and switches_list() function to display form
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

We have written two functions eos_form() and switches_list() in the previous use case. We will use those functions with this script to display form and derive IP addresses of the switches from the user selection.

.. code-block:: python

  def switch_logs():
      # Build Form
      form = eos_form()

      if form.process().accepted:
          # Create a switch IP addresses list from site and role selection
          switches = switches_list(form.vars)

      return dict(form=form)

Step 3: Write the core logic of the script
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Since we are going to use jsonrpclib, we need to install that module in the Linux system where we are hosting the web2py application.

.. code-block:: bash

  anees@ubuntu-web2py:~$ sudo pip2 install jsonrpclib
  Downloading/unpacking jsonrpclib
    Downloading jsonrpclib-0.1.7.tar.gz
    Running setup.py (path:/tmp/pip_build_root/jsonrpclib/setup.py) egg_info for package jsonrpclib

  Installing collected packages: jsonrpclib
    Running setup.py install for jsonrpclib

  Successfully installed jsonrpclib
  Cleaning up...

The algorithm for the core logic is shown below.

a) Create a list for the "logging" levels 0 to 3, which are emergencies, alerts, critical and errors.
b) For each "logging" level within the list, collect the show logging 10 <logging_level> output.
c) For each logging level, parse through the required output and store under the corresponding switch entry within the dictionary.

.. code-block:: python
  :emphasize-lines: 4

  # Import the neccessary python modules
  import pyeapi
  import json
  from jsonrpclib import Server, ProtocolError


  def hostname(node):
      host_name = node.runCmds(1, ["show hostname"])
      host_name_clean = str(host_name[0]["hostname"])
      return host_name_clean


  def collect_logs(node):
      # Define an empty list
      show_log = []

      # Create a list for the logging levels 0 to 3
      severity_level = ["emergencies", "alerts", "critical", "errors"]

      # For each logging level listed in the list, collect the show log
      for each_sev_level in severity_level:
          show_log_cli_raw = node.runCmds(
              1, ["show logging 10 " + each_sev_level], "text")
          show_log_cli = show_log_cli_raw[0]["output"]
          show_log.append(show_log_cli)

      return show_log


  def switch_logs():
      form = eos_form()
      if form.process().accepted:
          form_vars = form.vars
          switches = switches_list(form_vars)

          # Define dictionaries
          show_log = {}

          for switch in switches:
              node = Server("https://" + form_vars.username + ":" +
                            form_vars.password + "@" + switch +
                            "/command-api")

              # Collect hostname for reporting purpose
              name = hostname(node)

              # Collect the logs
              show_log[name] = collect_logs(node)

          return dict(show_log=show_log)

      return dict(form=form)

Test your script using the URL https://<web-server>/Arista_EOS_Tool/default/switch_logs.

.. image:: images/ch09-pic12.png

We are going to add three things to complete this script

#. Add Exception Handing
#. Display log entry one per line for clean view of the output
#. Add a logic to delete the switch entries if there are no logs.

.. code-block:: python
  :emphasize-lines: 13-14,27,30,41-43,45-46,48-49,51

  def collect_logs(node):
      # Define an empty list
      show_log = []

      # Create a list for the logging levels 0 to 3
      severity_level = ["emergencies", "alerts", "critical", "errors"]

      # For each logging level listed in the list, collect the show log
      for each_sev_level in severity_level:
          show_log_cli_raw = node.runCmds(
              1, ["show logging 10 "+each_sev_level], "text")
          show_log_cli = show_log_cli_raw[0]["output"]
          for line in show_log_cli.splitlines():
              show_log.append(line)

      return show_log


  def switch_logs():
      form = eos_form()
      if form.process().accepted:
          form_vars = form.vars
          switches = switches_list(form_vars)

          # Define dictionaries
          show_log = {}
          errors = {}

          for switch in switches:
              try:
                  node = Server("https://" + form_vars.username + ":" +
                                form_vars.password + "@" + switch +
                                "/command-api")

                  # Collect hostname for reporting purpose
                  name = hostname(node)

                  # Collect the logs
                  show_log[name] = collect_logs(node)

                  # If there are no logs, delete the entry for the switch
                  if not show_log[name]:
                      del show_log[name]

              except ProtocolError as e:
                  errors[switch] = "Invalid EOS Command" + str(e)

              except:
                  errors[switch] = "eAPI Connection Error"

          return dict(errors=errors, show_log=show_log)

      return dict(form=form)

Test your script using the URL https://<web-server>/Arista_EOS_Tool/default/switch_logs.

.. image:: images/ch09-pic13.png

Final Script
------------

The final script with all the functions is shown below.

.. code-block:: python

  # Import the neccessary python modules
  from jsonrpclib import Server, ProtocolError


  def hostname(node):
      host_name = node.runCmds(1, ["show hostname"])
      host_name_clean = str(host_name[0]["hostname"])
      return host_name_clean


  def collect_logs(node):
      # Define an empty list
      show_log = []

      # Create a list for the logging levels 0 to 3
      severity_level = ["emergencies", "alerts", "critical", "errors"]

      # For each logging level listed in the list, collect the show log
      for each_sev_level in severity_level:
          show_log_cli_raw = node.runCmds(
              1, ["show logging 10 "+each_sev_level], "text")
          show_log_cli = show_log_cli_raw[0]["output"]
          for line in show_log_cli.splitlines():
              show_log.append(line)

      return show_log


  def switch_logs():
      form = eos_form()
      if form.process().accepted:
          form_vars = form.vars
          switches = switches_list(form_vars)

          # Define dictionaries
          show_log = {}
          errors = {}

          for switch in switches:
              try:
                  node = Server("https://" + form_vars.username + ":" +
                                form_vars.password + "@" + switch +
                                "/command-api")

                  # Collect hostname for reporting purpose
                  name = hostname(node)

                  # Collect the logs
                  show_log[name] = collect_logs(node)

                  # If there are no logs, delete the entry for the switch
                  if not show_log[name]:
                      del show_log[name]

              except ProtocolError as e:
                  errors[switch] = "Invalid EOS Command" + str(e)

              except:
                  errors[switch] = "eAPI Connection Error"

          return dict(errors=errors, show_log=show_log)

      return dict(form=form)
