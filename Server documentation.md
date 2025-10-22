# Cumulus server documentation

## Presentation

The Cumulus server offers a Flask REST API to the clients, and manages the jobs. Each job is queued and launched when the requested resources are available. The server will create a new virtual machine for each job.


## Configuration

The file **cumulus.conf** contains the whole configuration. Cumulus is separated in three parts (server, client, rsync agent), and all parts need to configured accordingly.

* refresh.rate.in.seconds: the time in seconds to wait between each check for job status. The default value is 30 seconds, this allows the server to regularly check for new jobs without using too much resources
* data.max.age.in.days: the time in days before jobs and data are deleted from the server. The default value is 90 days, but the administrator should adjust it to prevent the partition to be full
* converter.raw.to.mzml: the command to run to convert .raw files to .mzML. Use variables %input_file% and %output_file% if needed.
    * Example: /usr/bin/mono /home/ubuntu/ThermoRawFileParser1.4.5/ThermoRawFileParser.exe -i "%input_file%" -b "%output_file%"
* converter.d.to.mzml: the command to run to convert .d folders to .mzML. Use variables %input_file% and %output_file% if needed (variable %uid_gid:<path>% can also be used to get the used/group id for a given path). 
    * Example: /usr/bin/docker run -u "%uid_gid:/cumulus%" -v /cumulus:/cumulus mfreitas/tdf2mzml tdf2mzml.py -i "%input_file%" -o "%output_file%"
* openstack.bin.path: the path to the openstack executable
* openstack.worker.username: the username used on the template worker
* openstack.worker.port: the port to connect to the workers
* openstack.snapshot.name: the name of the snapshot that will be used to create the workers
* openstack.volume.size.gb: the size in GB for the partition of the workers
* openstack.cert.key.name: the name of the public key that is recognized by openstack
* openstack.cloud.network: the name of the network that is recognized by openstack
* openstack.known.hosts.path: the path to the known_hosts file (usually $HOME/.ssh/known_hosts)
* openstack.cert.key.path: the path to the public hey
* local.host: the IP address to listen to, usually 0.0.0.0 is fine
* local.port: the port to listen to, make sure that your Cloud infrastructure allows this port
* storage.path: the root path for Cumulus (default should be /cumulus)
* storage.data.subpath: the name of the subfolder that will contain the shared files
* storage.jobs.subpath: the name of the subfolder that will contain the jobs
* storage.bin.subpath: the name of the subfolder that will contain the executables (or the github code)
* storage.temp.subpath: the name of the subfolder that will contain temporary files (usually when converting data to mzML)
* input.folder: the name of the subfolder in a job directory where the input files will be uploaded
* output.folder: the name of the subfolder in a job directory where the output files will be put
* temp.folder: the name of the subfolder in a job directory where the temp files will be put
* flavors.file.path: the name of the file containing the list of available flavors (it can also be a path relative to storage.bin.subpath)
* database.file.path: the name of the database where the jobs will be stored; the database is automatically created if it does not exist (it can also be a path relative to storage.bin.subpath)
* final.file: the name of the file that is sent by the RSync agent to indicate the end of the file transfer for a job
* version: the version number of the server
* client.min.version: the client's minimal version number accepted by the server


## Environment

The server has been designed to work on the [SCIGNE Cloud at IPHC](https://scigne.fr/), and will need some adjustments if you want to make it run on another platform.

In our configuration, we have been attributed a dedicated network in which we can use all the resources of the quota. We have also used a certificate to create the main virtual machine (called the **controller**).

The controller is the VM where the server will be installed. It's also where the conversions from raw/d to mzML will be made, which means that this VM should have enough resources to support that.

Cumulus requires a second virtual machine, called the **template worker**. This VM contains the apps that will be called by the jobs, and a 500 GB partition where a Linux 24.04 is installed. A snapshot of this partition is created, and used to generate the real workers (who will have much more resources). The template worker allows the administrator to install an app, verify that it's fully functional, before using it in production.

In addition to the controller and the template worker, a large partition should be created and mounted on the controller. This partition should be used to store all the content of Cumulus, and be shared with NFS with the template worker. That way, each job will have access to the shared partition. The point of that shared partition is to avoid transferring data from the server to the workers, as well as the output from the workers to the server. This partition should be large enough to store a significant amount of raw data.

**Note** that the partition size and the resources allocated to the controller, the template worker and the workers have been determined based on the quota available for our network. As an example, this is the resources we have on our main instance:

|  | CPU | RAM | DISK |
| ----- | ----- | ----- | ----- |
| Allowed quota | 256 | 1000 GB | 16100 |
| Controller | 16 | 64 GB | 40 GB |
| NFS partition | - | - | 10000 GB |
| Template worker | 4 | 8 GB | 500 GB |

The rest of the resources will be used by the workers. Openstack uses **flavors** to determine the set of resources that can be attributed to a virtual machine. This part is more detailed in the [corresponding section](#flavors--strategies).

## Installation

There is no automated installation for the moment, but a [script](https://github.com/LSMBO/cumulus_server/blob/main/scripts/install_cumulus_server.sh) is provided to guide an administrator into setting up a Cumulus server. **Important: ** this script is not meant to be executed, it's meant as a guideline to install the dependencies, configure the openstack client, clone the Github repository, etc.

Cumulus has been designed so that the clients and the RSync agent only need to contact the controller. This means that the controller is the only VM who must have a public IP address. This also means that the controller must be secure. The administrator must configure the firewall (at least to allow the port used in local.port). In case of a use from an entity with a single outside IP address, the controller's firewall may also be configured to only accept requests from this particular IP address.


## Flavors & strategies

Openstack uses flavors to determine the resources to attribute at the creation of a virtual machine. Each flavor has a number of CPU, a size of memory and a size for the disk space.

In Cumulus, the approach is to let the administrator select a few flavors that will allow the best mix of workers. The administrator defines a weight for each strategy, and a **total weight limit**. The administrator has to make sure that no combination of strategies within the limit will never take more than the resources allowed in the quota.

In the original Cumulus instance on the SCIGNE Cloud, the following strategies have been selected, with a maximum weight of 7 :
| Strategy | CPU | RAM | DISK | Weight |
| ----- | ----- | ----- | ----- | ----- |
| m1.4xlarge | 32 | 64 GB | 500 GB | 1 |
| m1.8xlarge-16xmem | 64 | 256 GB | 500 GB | 2 |
| m1.16xlarge-28xmem | 128 | 448 GB | 500 GB | 4 |

Note that the original flavors did not offer that amount of disk space. The template worker has been installed on a volume of 500 GB, and a snapshot of this volume is used to create the other workers.


## Other daemons

The Cumulus server offers a Flask REST API, with a [list of routes](#routes), and runs three different daemons:

* The **main daemon** watches over the PENDING jobs to start them, and watches over the RUNNING jobs to check how and when they end.
* The **converter daemon** watches over the PENDING jobs to check if there are input files that need to be converted. The server will get the list of input files, search in the settings if they need to be converted, and convert those who are not yet converted.
* The **cleaning daemon** will remove the oldest files based on the values in the configuration file. This daemon will regularly archive the old jobs, and delete the old shared data files (unless if they appear in PENDING jobs).


## Database description

The server uses a simple SQLite database to store the content of the jobs. The database is automatically created if the file defined in the configuration file is not found.

The schema of the database is the following:

| Field | Type | Required | Description |
| ----- | ---- | -------- | ----------- |
| id | integer | ✅ | Primary key and autoincremented |
| owner | text | ✅ | Name of the creator of the job |
| app_name | text | ✅ | Unique name of the selected app |
| strategy | text | ✅ | Unique name of the selected strategy |
| description | text | ✅ | Description given by the user (may be an empty text) |
| settings | text | ✅ | A JSON represetation of the settings sent by the user |
| status | text | ✅ | The current status of the job ([see the list of status](#list-of-possible-status)) |
| host | text | ❌ | Deprecated, used to contain the IP address of the worker VM. The information are now stored in the file **.cumulus.host** in the job folder |
| creation_date | integer | ❌ | A timestamp representing the date of the creation of the job |
| start_date | integer | ❌ | A timestamp representing the date of the beginning of the job |
| end_date | integer | ❌ | A timestamp representing the date of the end of the job |
| stdout | text | ❌ | Deprecated, used to contain the text of STDOUT. How the entire log is in the file **.cumulus.log** in the job folder |
| stderr | text | ❌ | Deprecated, used to contain the text of STDOUT. How the entire log is in the file **.cumulus.log** in the job folder |
| job_dir | text | ❌ | The path of the job directory |
| start_after_id | integer | ❌ | *Work in progress* |
| workflow_name | text | ❌ | *Work in progress* |
| last_modified | integer | ❌ | A timestamp corresponding to the last time the job was updated |

## Routes

* "/start" (POST)

    Creates a pending job and its associated directory, then returns the job ID and directory.

    The app's name, strategy and parameters selected by the user are stored in the request's form.

* "/joblist/<int:job_id>/<int:number>/"

    Retrieves the latest jobs. The jobs returned only contain a very limited amount of information, just enough to fill the job list in the GUI. The job correspond to the ID given as a parameter will be filled entirely, with the log content and the list of output files as well.

    The function behind this route is the same as the one used in **/search**

* "/search" (POST)

    Retrieves a list of jobs corresponding to a series of criteria provided in the request's form. The jobs returned only contain a very limited amount of information, just enough to fill the job list in the GUI. The job correspond to the ID given as a parameter will be filled entirely, with the log content and the list of output files as well.

    The function behind this route is the same as the one used in **/joblist**

* "/cancel/<string:owner>/<int:job_id>"

    Sends a signal to the server to cancel a job. This will only work if the job is not already finished, and if the user making the request is the owner of the job.

* "/delete/<string:owner>/<int:job_id>"

    Sends a signal to the server to delete a job. This will only work if the job is already finished, and if the user making the request is the owner of the job.

    **Warning**: Deleting a job will permanently delete the job folder and the entry in the database.

* "/getresults/<string:owner>/<int:job_id>/<path:file_name>"

    Retrieves and sends a result file for a given job if the requesting user is the owner.

* "/getfile/<string:owner>/<path:file_name>"

    Retrieves and sends a shared file, from the shared data folder.

* "/getfilecontent" (POST)

    When a user wants to download a list of shared files, this route has to be called first to retrieve the entire content of the file (as it may be a folder). The user will then have to call *get_file* for each file.

* "/info"

    Returns information about the available flavors, as a JSON response.

* "/apps"

    Returns a JSON response containing a list of available applications. The response is an array of XML strings, allowing the client to extract the necessary information.

* "/storage"

    Returns a JSON response containing the list of file names and their sizes in the shared data folder.

* "/diskusage"

    Returns the current disk usage statistics as a JSON response. The response contains the total, used, and free disk space for the storage path specified in the configuration.

* "/fail" (POST)

    Handles a job failure request by extracting the job ID and error message from the request form,	logging the failure, and marking the job as failed using the utility function.

	This route is used in the RSync agent, when some input files are not available. It allows the agent to report a failure without wasting time transferring files that cannot be processed.

* "/config"

    Returns the current configuration as a JSON response. This route is used to check if the server can be reached, to retrieve the current configuration and to check its version number.

    Note that only the server configuration relevant to the client is returned.


## Job life cycle

A job goes through several steps after its creation. There are globally three main stages over the life of a job: before its execution, during its execution, and after its execution.

### List of possible status

* **PENDING**: Status for the jobs that have just been created. Jobs are stored in a queue, waiting to be started. The sequence to start new jobs should correspond to their order of creation, but the job can be delayed if the selected strategy requires a large amount of resources.
* **PREPARING**: When the job starts, the server will have to execute some tasks before it can properly launch the job. These tasks involve the creation of the worker node on the Cloud, and eventually the conversion of the RAW data to the mzML format.
* **RUNNING**: This status indicates that the app of the job is actively running with the user's set of parameters. Users can follow the job progress in the Logs tab.
* **DONE**: Status for jobs who ended in success. Users can download the output in the Data tab.
* **CANCELLED**: This status can only be obtained when the user chooses to cancel a job. A user cannot cancel somebody else's job.
* **FAILED**: Status for jobs who ended in failure. The log may help to see what went wrong.
* **ARCHIVED**: The jobs are archived after a number of days (90 by default, it can be changed on the server). Once archived, the output data are deleted but the parameters and logs remain. The status will either ARCHIVED\_DONE, ARCHIVED\_CANCELLED or ARCHIVED\_FAILED, depending on the original status.
* **PAUSED**: When the server is stopped, all the jobs with the status PREPARING are paused. When the server restarts, the paused jobs can restart 

### Detailed life cycle

1. Creation of the job
    1. The user selects the app, the strategy and fills in the settings, then clicks on **Start job**.
    2. The client sends a POST request to the server, calling the "/start" route. At the same time, the client sends a request to the RSync agent to upload the input files to the server.
    3. The server creates an entry in the database and the job folder. The job is put in the queue with the status **PENDING**.
    4. The server runs the pending jobs as they come, within the limitations of the resources. A job with a heavy strategy may have to wait until the resources are available.
    5. During that time, the RSync agent is transferring the files to the server, one by one, finishing with an empty file to signify the end of the process.
    6. The server will simultaneously check for files to convert to mzML. A separated thread searches for all the pending jobs and checks if they need mzML files. If they do, and the mzML files are not yet there, the conversion will start automatically. Only one mzML conversion can be done at the same time. The log of the conversion is redirected to the job's log file.
    7. When all the files are present, converted, and the resources are available, the server will pick the job and set its status to **PREPARING**.
2. Execution of the job
    1. Creation of the worker node, based on the strategy. The server will generate a volume based on the snapshot of the template worker, then a virtual machine using the volume. At every step, the server waits until the newly created item is available. This step can be time-consuming.
    2. The server will generate the script and eventually the configuration file, based on the settings selected by the user. The files will be placed in the job folder, and kept after the job is done.
    3. The mzML conversion may still be running at this point, the server will wait as long as necessary if it is the case.
    4. Finally, the server can set the status of the job to **RUNNING** and send a SSH request to start the job on the remote worker node. The job folder and the data folder are on a NFS shared drive, so the server only needs to send the job folder as an argument.
    5. [The script that is called by the server](https://github.com/LSMBO/cumulus_server/blob/main/scripts/start_job.sh) will do several things:
        1. Copy the job folder to the worker node
        2. Create an empty file **.cumulus.alive** in the job folder, this file's last modification date will be used to let the serve know if the job is still alive or not.
        3. Set the active directory to the **output** folder, to make sure that all the files generated locally by the app will be returned to the user.
        4. Execute the job's script and redirect STDOUT and STDERR in the log file. The script is executed in the background.
        5. Infinite loop while the PID is still running. In this loop, the log is copied to the server (so the user can follow the progress), the file .cumulus.alive is updated, and the resources currently used are saved.
        6. When the script has finished (either success or error, it does not matter at this stage), the content of the output folder is copied back to the server. Along with the final version of the log file.
        7. Finally, the file .cumulus.alive is renamed .cumulus.stop to indicate to the server that the process has stopped properly.
    6. During the execution of the previous step, the server will keep an eye on the RUNNING jobs and determine which are still running based on two simple criteria: the file .cumulus.alive exists, and it has been modified since the last verification. Otherwise, the server considers the following options:
        1. If the file .cumulus.alive exists but its modification time has not changed, the job gets the status **FAILED**.
        2. If the file .cumulus.alive does not exist, and neither the file .cumulus.stop, the job gets the status **FAILED**.
        3. If the file .cumulus.stop exists and the **end_tag** is not found in the log file, the job gets the status **FAILED**.
        4. Otherwise, the job gets the status **DONE**.
    7. Another possibility is that the user called the "/cancel" route. In that case, the job's status is **CANCELLED**, the server sends a SSH request to the worker node to kill the process, and the worker is destroyed.
3. After the job ends
    1. Once the job ends, either with status DONE, FAILED or CANCELLED, the server will record the time in the database, and destroy the worker node where the job was executed.
    2. The jobs will remain available for a certain amount of time, before being archived. The archiving of a job consists in deleting the output folder, and prepending the status with "ARCHIVED_". The jobs with the status **DONE** will then become **ARCHIVED_DONE** and so on.



## How to add new apps

1. Install the app on the template worker

    The worker nodes are created from a clone of the template worker, so this is where the apps must be installed and tested.

    The administrator can install and configure the apps as they choose, they have to install the dependencies as well. It is advised to give the template worker a partition large enough to contain all the apps and possibly different versions of them. Docker images can also be used.

    The template worker does not need a lot of resources, but it has to be enough to make sure that the apps installed are actually working.

2. Update the VM snapshot

    Once the desired apps are installed, and the template worker has been cleaned of temporary/useless data, it's time to put it in production.

    The administrator just has to recreate a snapshot of the current volume. The name of the snapshot has to match the name given in the [configuration file](#configuration).

    The administrator can use [this script](https://github.com/LSMBO/cumulus_server/blob/main/scripts/create_template_snapshot.sh) to update the snapshot. The script creates a back-up of the previous snapshot before creating the new snapshot.

3. Create an XML file

    The final part of the procedure is to create the XML file that will contain all the information for the client to generate the settings page, and for the server to call the app with the selected settings.

    The rules to create the XML files are listed in a [XSD file stored in the Cumulus client](https://github.com/LSMBO/cumulus-client/blob/main/server/apps.xsd) (so the validity of the apps can be checked). The [apps currently available](https://github.com/LSMBO/cumulus_server/tree/main/apps) can also be used as examples to follow.

    The XML file has to start with a \<tool> element containing the general information about the tool (ie. name, version, url, description, etc.). This element will then contain one or more \<section> elements, in which a series of *parameter* elements can be added. The parameters can be of different types:
    * **\<select>:** a dropdown list
    * **\<checklist>:** a dropdown list with checkboxes
    * **\<keyvalues>:** a table with textboxes for a key and a value on each row
    * **\<checkbox>:** a checkbox
    * **\<string>:** a textbox
    * **\<number>:** a textbox for numbers
    * **\<range>:** two textboxes for numbers, usually to determine a parameter between a minimum and a maximum
    * **\<filelist>:** a file browser, the client will produce a different display if *multiple* is true or false

    The parameters all share a series of basic attributes, such as the name, label, tooltip text, etc. Each parameter also has specific attributes, for instance \<number> and \<range> have *min* and *max* attributes, when \<string> does not.

    The XDS also allows \<conditional> elements within the sections. A conditional contains a primary parameter, and one or more \<when> elements that can contain a list of parameters. The value of the primary parameter will determine which \<when> element is visible. Note that for the moment, Cumulus only considers \<select>, \<filelist>, \<checkbox>, \<number> and \<string> as primary parameter.

    The main tool, and each parameter, have an optional **command** attribute. This attribute gives the server the knowledge of what to add to the command line to take this parameter into account. An alternative is to use the **convert_config_to** tool attribute, this will generate a configuration file in addition to the command line; the supported format are YAML and JSON. This administrator can use some variables with this attribute:
    * **%value%** will be replaced by the value of the parameter
    * **%value2%** will be replaced by the value of the parameter in the context of a range parameter
    * **%key%** will be replaced by the key in the context of a keyvalue parameter
    * **%nb_threads%** will be replaced by the number of CPU available with the selected strategy
    * **%config-file%** will be replaced by the name of the job's configuration file 
    * **%input_dir%** will be replaced by the name of the job's input directory
    * **%output_dir%** will be replaced by the name of the job's output directory

    Note that the generation of the configuration files have been realized based on the apps we wanted to work with. It may not work directly with every apps. In the context of a configuration file, the **name** attribute of the parameters is used to determine the hierarchy of the parameter, using the dot notation. For instance, the name *app.feature.param* would be transformed into the followning JSON equivalent: `{ "app": { "feature": { "param": "value" } } }`

    Finally, the \<tool> contains a **end_tag** attribute that will be used to determine whether the app has finished successfully or not. This tag has to be carefully chosen.

    The schema structure is described in the [XSD documentation](/XSD%20documentation.md).