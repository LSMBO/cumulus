# Cumulus

Cumulus is a three-part tool whose purpose is to run software on a Cloud, without requiring the users to have any computational knowledge. Cumulus is made of three elements: 
* [A client](https://github.com/LSMBO/cumulus-client) with a graphical user interface developped in Javascript using ElectronJS.
* [A server application](https://github.com/LSMBO/cumulus_server), running on the Cloud. The server will dispatch the jobs, monitor them, and store everything in a database. The server is developped in Python and provides a Flast REST API for the client and the agent.
* [A separate agent](https://github.com/LSMBO/cumulus_rsync) who manages the transfer of the files from client to server. The agent is developped in Python and provides a Flask REST API for the client.

Cumulus has been developped to work around the [SCIGNE Cloud at IPHC](https://scigne.fr/), it may require some modifications to work on other Clouds, depending on how the virtual machines are organized. The virtual machines available in Cumulus are set up like this:
* A controller with 16VCPU, 64GB RAM, 20GB drive, this is where cumulus-server is running. This VM is the only one requiring a public IP address.
* A 9TB storage unit, mounted as a NFS shared drive on every VM so the content is shared with the same path. This is where the data, the jobs and apps will be stored.
* A template worker node with 4VCPU and 8GB RAM, 500GB drive, this is where the apps are installed and tested. When a modification is made here (new app, new version, dependency update, etc.), a snapshot is generated and will be used to create the real worker nodes.
* At most 7 worker nodes, each being a copy of the template worker node. These VM are created on the fly and destroyed once the job is finished. The resources for these VM depend on the flavor selected by the user.

The virtual machines on the Cloud are all running with Ubuntu 24.04, Cumulus has only been tested there so it's possible that some scripts may not work on a different Linux distribution.

The purpose of the RSync Agent is to provide a single queue to manage the transfer of large files. Cumulus has been developped to run applications dealing with mass spectrometry data, which are often between 1 and 10GB. Cumulus has been developped for a use in a work environment with a dedicated network, where data are stored on a server, and users are running apps on their own sessions. In that context, it is more effective to transfer data from the server directly, rather than from each user's session. The queue is stored in a local database, so it does not disappear if the agent has to be restarted.
The agent has been tested on a Windows server, and uses [cwRsync](https://www.itefix.net/cwrsync), but it should work on a Linux server using a local RSync command. Scripts to create a Windows service are provided.

