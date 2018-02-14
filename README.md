# PSTCollector
Powershell script to find, gather and assist in the upload of .PST files to Office 365

Developed by: Appleoddity

Requirements:
  1) All collected systems need to be running Windows 7 or newer
  2) CollectorMaster.ps1 needs to run Powershell 5 or newer
  3) All computers must be part of a domain
  4) CollectMaster should be run with Domain Admin privileges or equivalent
  5) PSEXEC is required and needs to be downloaded separately.
  
Usage:
  PSTCollector consists of three components. The CollectorMaster that controls the collection process and is run from a central computer, and the CollectorAgent which is pushed to each workstation on the network to find, collect, and remove PST files. PSEXEC is additionaly used by CollectorMaster to start the CollectorAgent in the SYSTEM context of each workstation. PSEXEC will need to be downloaded separately. 
  
  Start by creating a file share that has read-only access provided to all domain users and domain computers. Preferably "Everyone" or "Authenticated Users" will work. Place both the `CollectorAgent.ps1` and the `CollectorMaster.ps1` scripts in the newly created file share. Obtain the `PSEXEC.EXE` file by downloading and decmopressing PSTOOLS (https://docs.microsoft.com/en-us/sysinternals/downloads/psexec) then place `PSEXEC.EXE` in the same folder with the Agent and Master.
  
  Create an additional file share where PST files will be collected, this is the CollectPath specified when the CollectorMaster is run. It is important to pay close attention to the permissions created on the collection share to avoid giving unauthorized users access to PST files. The permissions on the collection share should be setup as follows:
  - Sharing permissiongs:
      - Grant EVERYONE, FULL CONTROL
  - NTFS Permissions
      - Grant Administrators and SYSTEM, FULL CONTROL to This folder, subfolders and files
      - Grant CREATOR OWNER, FULL CONTROL to Subfolders and files only
      - Grant Authenticated Users the following special permissions on This folder only:
        - Traverse folder / execute file
        - List folder / read data
        - Create folders / append data
        
        
  You're now ready to begin collecting PST files. The PSTCollector tool has 3 different modes that it can run in: FIND; COLLECT; and REMOVE. The tool must run in order FIND, then COLLECT, then REMOVE. You can start the tool in REMOVE mode, but it will intelligently work it's way through FIND then COLLECT then REMOVE. If there are any errors along the way, it will not proceed. It is recommended to do each step separately in large deployments so you can closely monitor it's progress. **Do not REMOVE PST files until you know they are collected and imported properly AND you have removed the PST files from each user's Outlook. See the note below on the additional script I included for this.**
  
  The PSTCollector tool can collect PST files from domain joined workstations, or from network file shares. When using the tool, you can specify multiple locations to collect as either an OU path (i.e. `OU=COMPUTERS,DC=DOMAIN,DC=LOCAL`) or file share (i.e. `\\fileserver\UserData`). The CollectorAgent runs on the CollectorMaster system when collecting from a file share, and will run in the same user context of the user that started the CollectorMaster script. That user must have full access to the entire file share location for things to work properly.
  
  The CollectorAgent works exclusively in the C:\PSTCollector folder of each workstation. It will save a log and configuration XML file with the name of the JOB you specified. The XML file can be tweaked to change the behavior of the script the next time it is run. Similarly to the CollectorAgent, the CollectorMaster saves a log and configuration XML file to the root of the Collection share which is a compilation of all the individual logs and config files generated by the Agent. Again, modifying the XML file is a convenient way to change the behavior of the tool the next time it is run.
  
  Why would you want to chagne the behavior of the tool? For instance, if you run the tool against an entire organizational unit and some systems in that organizational unit are old, or will never be online, you can tell PSTCollector to skip that machine by changing it's status to VOID in the XML file. You can change any location or file status to VOID to get the PSTCollector scripts to ignore that particular location or file. To ignore particular files you will have to modify the XML file on the actual workstation. To skip locations you will modify the XML file on the collection share. The log file and configuration XML files can be used to closely monitor the progress and behavior of the scripts.
  
  To start a collection you will run the CollectorMaster.ps1 script as Domain Admin (or equivalent) on a machine that has access to all workstations and can continue uninterrupted. Run the tool using the full UNC path to the file share created earlier. This is how the tool know where to grab the agent and push it to workstations. This tool can intelligently resume its progress, and can be run as many times as necessary. However, the same JOBNAME and COLLECTPATH must be used for each run, or it will not work. Locations can be changed between each run. Powershell 5 must be available on the Master system. If you like the defaults, you don't have to specify any command line switches, instead you will be prompted for them when you run the tool. Otherwise, the CollectorMaster supports the following command line switches:
  - `Mode` <mode>            #REQUIRED - The mode to run the collection in (FIND, COLLECT, or REMOVE)
  - `JobName` <jobname>      #REQUIRED - Give the job a descriptive name. All subsequent runs MUST use the same JobName.
  - `Locations` <locations>  #REQUIRED - Specify locations, separated by commas, as an OU path, or network file path. See above.
  - `CollectPath` <path>     #REQUIRED - Specify the location to the Collection share. i.e. \\fileserver\PSTCollection
  - `ConfigPath` <path>      #OPTIONAL - Specify where the Master configuration file will be saved. It defaults to CollectPath.
  - `ForceRestart`           #OPTIONAL - Specifying this switch will wipe out the existing configuration and start over new. CAREFUL!
  - `Noping`                 #OPTIONAL - This tells PSTCollector to not attempt to ping workstations to see if it is online, before trying                                        to collect from it. Without this switch offline machines are skipped much faster.
  - `Throttlelimit`          #OPTIONAL - Set the limit of active collector threads that can run at the same time. Default is 25.
  - `NoSkipCommon`           #OPTIONAL - Tell the collector to check ALL folders on the hard drive for PST files. Without this switch, the                                         collector will improve performance by skipping folders we know don't usually have PST files.                                             This includes: both Program Files folders, Windows, System Volume Info, and Recycle Bins.
  - `IsArchive`              #OPTIONAL - When creating the import template for Office 365, it will tell Office 365 to import the PST files                                         in to the user's archive mailbox instead of their main mailbox. Default is TRUE.
        
  Example:
    `\\fileserver\scripts\PSTCollector\CollectorMaster.ps1 -Mode FIND -JobName PSTCollect -Locations "OU=Computers,DC=Domain,DC=Local","\\fileserver\homefolders" -CollectPath \\fileserver\PSTCollection`
    
    
Things to Note:
  - Use the configuration XML files to keep track of your progress. I recommend viewing and editing them in Notepad++.
  - The PSTCollector tool creates a CSV export file during the "COLLECT" phase that can be used as the Azure import template for Office 365.
  - The actual upload to Azure and import of the PST files to Office 365 is done by Microsoft provided tools. Preferably 'AzCopy.'
  - PSTCollector tries to match the Office 365 mailbox to the PST file by looking up the e-mail address of the owner of the PST file in active directory. Users must have the user e-mail address field populated in AD for this to work. If the lookup fails for any reason, the PST file will be mapped to the email address of the "Administrator" in AD.
  - You should review and correct mapping errors in the master configuration XML file prior to running the last COLLECT phase. The owner of the file should be updated if it is inaccurate.
  - You should modify the Agent and Master configuration XML files as necessary, especially using the VOID status to tell the collector to skip problematic or un-necessary locations and files.
  - The Office 365 import process supports a maximum of 250 PST files for one job at this time. If you have more than 250 PST files you will have to split your exported CSV template in to multiple files and jobs.
  - All PST file copies are done using the built in RoboCopy in Windows.
  - The CollectorAgent has an undocumented parameter `ipg` which can be modified in code. This is the interpacket gap time used by RoboCopy. By default, the value is 1ms and the Agent throttles the PST copy so it doesn't overwhelm your network.
  - **I've included an additional .vbs script that can be run as a login script or scheduled task in the `user` context of each machine to uninstall the PST file from Outlook BEFORE you use the PSTCollector to `REMOVE` the file. If you `REMOVE` the file before it is removed from Outlook, the user will get an error in Outlook.**
  

