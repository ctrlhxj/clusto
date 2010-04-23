##################################
  Getting Started
##################################

:Release: |version|
:Date: |today|

Installation
------------

From debian package
~~~~~~~~~~~~~~~~~~~
Add the following to your /etc/apt/sources.list::

 deb http://mirrors.digg.com/digg digg-public-lenny main contrib non-free

Update the index and install clusto::

 # aptitude update
 # aptitude install clusto

From source
~~~~~~~~~~~
::

 $ git clone git://github.com/digg/clusto.git
 $ cd clusto
 $ python setup.py build

As root::

 # python setup.py install
 # mkdir /etc/clusto /var/lib/clusto
 # cp conf/* /etc/clusto/
 # cp contrib/* /var/lib/clusto/

You may need to install additional python libraries for certain features of clusto to function properly.

- SQLAlchemy: http://www.sqlalchemy.org/
- setuptools: http://peak.telecommunity.com/
- scapy: http://www.secdev.org/projects/scapy/doc/
- ipython: http://ipython.scipy.org/moin/
- IPy: http://c0re.23.nu/c0de/IPy/
- MySQLdb: http://sourceforge.net/projects/mysql-python/
- libvirt: http://libvirt.org/
- simplejson: http://code.google.com/p/simplejson/

Configuration
-------------

Clusto is built on top of SQLAlchemy and therefore should work with any backend database available to that library. For the same of simplicity, the following documentation assumes you are using a MySQL backend.

Create a MySQL database
~~~~~~~~~~~~~~~~~~~~~~~
::

 # aptitude install mysql-server
 # mysql -u root
 mysql> CREATE DATABASE clusto;
 mysql> GRANT ALL PRIVILEGES ON clusto.* TO 'clustouser'@'localhost' IDENTIFIED BY 'clustopass';
 mysql> FLUSH PRIVILEGES;

/etc/clusto/clusto.conf
~~~~~~~~~~~~~~~~~~~~~~~
::

 [clusto]
 dsn = mysql://clustouser:clustopass@127.0.0.1/clusto

Populating clusto
-----------------
In order to use clusto, you'll have to make it aware of your server environment. Every environment is built differently and may require different approaches to the discovery of servers and usage of clusto's features.

IPManagers for your subnets
~~~~~~~~~~~~~~~~~~~~~~~~~~~

SimpleEntityNameManager
~~~~~~~~~~~~~~~~~~~~~~~

Creating a datacenter
~~~~~~~~~~~~~~~~~~~~~
Clusto comes ready to use with Basic drivers for a variety of entities and devices most systems engineers are familiar with. These Basic drivers can be used directly if only basic functionality is desired, but are intended to be subclassed and overridden with different variables and method. It is considered a best practice to create subclasses in case you need to customize a driver later on.
This is an example of a custom datacenter driver representing a colocation facility. (src/clusto/drivers/locations/datacenters/equinixdatacenter.py)::

 from basicdatacenter import BasicDatacenter

 class EquinixDatacenter(BasicDatacenter):
 	'''
	Equinix datacenter driver
	'''

	_driver_name = 'equinixdatacenter'

Add the following to src/clusto/drivers/locations/datacenters/__init__.py::

 from equinixdatacenter import *

This subclass is an example of a simple wrapper that does not provide additional functionality beyond distinguishing this type of datacenter from another one.

At clusto's core, it's a tool for gluing together other services, protocols, and logic in a data-centric way. An example of such usage would be a method that opens a ticket with the colo provider for a remote hands action. Add the following to equinixdatacenter.py::

	from smtplib import SMTP
 	def remote_hands(self, message, priority='low', mail_server='mail.example.com', mail_from='clusto@example.com', mail_to='remotehands@datacenter.net'):
		msg = 'From: %s\r\nTo: %s\r\nSubject: Remote hands request: %s priority\r\n\r\n%s' % (mail_from, mail_to, priority, request)
		server = SMTP(mail_server)
		server.sendmail(mail_from, mail_to, request)
		server.quit()

Using clusto-shell we can instantiate our new datacenter and test the remote_hands method::

 from clusto.drivers import EquinixDatacenter

 datacenter = EquinixDatacenter('eqix-sv3')
 datacenter.remote_hands('Turn on power for new racks in my cage', priority='high')

Adding racks to the datacenter
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Rack factories are fairly specific to each organization's environment... Common rack layouts are uncommon. For these types of highly-customized features, you'll need to modify the files in /var/lib/clusto to suit your needs. Clusto ships with some examples of the files used by Digg.

Take a look at /var/lib/clusto/rackfactory.py to get an idea of how to programatically define rack layouts.

Continuing our example from above, let's assume the remote hands at the datacenter plugged in your new rack and closed the ticket. Now you want to be able to manipulate the rack in clusto and start adding servers to it. Let's go back to clusto-shell::

 from clusto.drivers import APCRack

 rack = APCRack('sv3-001')
 rack.set_attr(key='racklayout', value='201001')
 datacenter = get_by_name('eqix-sv3')
 datacenter.insert(rack)

That's all you need to do to create a new rack instance and insert it into a datacenter. The set_attr for rack layout will be explained in the following section.

Rack factory
~~~~~~~~~~~~
If you have more than one rack using the same layout of devices and connections between them, clusto can ease a lot of the data entry involved with creating and populating new racks with custom RackFactory classes::

 from rackfactory import get_factory

 factory = get_factory('sv3-001')
 factory.connect_ports()

The get_factory call will get the 'sv3-001' rack instance from clusto and lookup an attribute of key='rackfactory' to determine which factory class should be used to fill in the rest of the information. The __init__ method of Digg201001RackFactory also creates network, console, and power switch instances with names based on the name of the rack.

The connect_ports method ensures that this rack is in the datacenter, that the network, console, and power instances are in this rack, and that their ports are all connected as intended. This gives us the basic structure of everything that will be identical across all racks with this layout.

SNMP trap listener
~~~~~~~~~~~~~~~~~~
One of the services that ships with clusto called clusto-snmptrapd, will listen for SNMP traps from Cisco switches implementing the CISCO-MAC-NOTIFICATION-MIB which can be enabled with the following configuration on supported IOS releases::

 snmp-server enable traps mac-notification change
 snmp-server host <ip> version 2c <community>  mac-notification
 mac address-table notification change
 !
 interface <int>
   snmp trap mac-notification change added

When clusto-snmptrapd receives one of these traps, it gathers the IP of the switch, the port number, the VLAN that learned the MAC, and the MAC itself. It then queries the clusto database for the switch IP and checks for an attribute of key='snmp', subkey='discovery', value=1... If this attribute is set on the switch or any of it's parents, the daemon checks to see if there's an object connected to the switch port in clusto. If not, then we assume this is a new server and create a new PenguinServer object with a name generated from a SimpleEntityNameManager called 'servernames'. The daemon then gets the switch's parent rack, gets the rack factory for that rack, then calls add_server(server, switchport) on the rack factory instance.

It's a bit of a complicated process, but the end result is that new servers get added to clusto automatically as soon as they appear on the network without any human intervention aside from creating the rack instance.