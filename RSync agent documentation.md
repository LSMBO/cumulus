# RSync agent documentation

## Presentation

The RSync agent is a Flask REST API for Cumulus.

This module of Cumulus will transfer all the data to the server, using RSync to reduce the amount of transfer when it's possible.
The reason for a separate module, rather than transferring the data from the client, is to avoid parallel transfers, as well as to make sure the files are transferred from the best place in your local network. For instance, if your data are stored on a shared folder in the network, it is best to run the RSync agent directly on the server hosting that shared folder and transfer the data from there.

Another reason is that RSync requires a certificate to connect to the server, using a separate module avoids sharing that certificate with the users.


## Configuration

The file **cumulus_rsync.conf** contains the whole configuration. This module needs to know how the server is organized, in order to transfer the files to the correct locations.

* local.host: the IP address to listen to, usually 0.0.0.0 is fine
* local.port: the port to listen to
* storage.host: the IP address of the Cumulus server
* storage.port: the port of the Cumulus server
* storage.path: the root path for Cumulus on the server (default should be /cumulus)
* storage.user: the username to contact the server
* storage.public_key: the public SSH key to contact the server
* rsync.bin.path: the path to the rsync executable (necessary unless rsync is already in the PATH variable)
* ssh.bin.path: the path to the SSH executable
* refresh.rate: this is used to regularly check if there are files to transfer, the value is a number of seconds
* final.file: the file to send to indicate that the all the files for a job have been transferred, this file can be empty
* progress.file: the rsync log file, used to track the progress of the transfers
* queue.file: a SQLite file used to store the queue of files to transfer (this way, if the agent reboots, the transfers can restart automatically)
* remote.input.folder: the name of the subdirectory within the job folder where local files will be sent
* version: the version of the RSync agent
* survey.enabled: a boolean value to enable/disable the survey mode
* survey.time: the time when the survey would start, in the format HH:MM
* survey.depth: indicates how far we look within the directories
* survey.<name>.dir: the main directory to survey
* survey.<name>.regex: the rule to select which data to transfer
* survey.<name>.isfile: a boolean value to know if the data are files or folders


## Installation

1. Get the sources from Github
2. (Optional) Download RSync if you do not have it yet. Windows users can use https://itefix.net/cwrsync
3. Run package_cumulus_rsync_agent.bat to package the agent (this will create an executable file)
4. Adapt the configuration file in the packaged version (unless if you have configured it before)
5. Either run the executable file, or create a service to make sure it will be running automatically

#### Cumulus Service

The **service** subfolder has been created For Windows users. It contains scripts that use NSSM to create, edit, start and stop the service. Make sure to download NSSM from https://nssm.cc/ and to adjust the scripts if necessary.

Depending on your network configuration, you may want to create a service that is run by a user who has access to network paths. The Cumulus client will send network paths when it's possible, so it's important that the agent is able to access the files.

## Survey mode

A survey mode has been developped, but never tested in production mode so far.

This mode has been designed to watch over one or more directories, and to automatically transfer every files that appear in that directory. The RSync agent will only transfer the files that are less than 36 hours old, but at least 2 hours old.

The configuration requires three information for each directory: the path of the directory, a regular expression to determine which data to transfer, and a boolean to know if the data is a file or a folder.

## Routes

* "/": returns the version number of the RSync agent. It is also used by the client to check if the RSync agent can be reached
* "/send-rsync": handles a POST request to add files to the queue
* "/list-rsync": returns the list of shared files currently in the queue. Duplicates are removed from the list.
* "/cancel-rsync/<string:owner>/<int:job_id>": cancels the transfers to the server for the given job, returns the number of cancelled transfers
* "/progress-rsync/<string:owner>/<int:job_id>": returns a dictionnary with a list of files and their progress status; this list only contains the files that still have to be transferred