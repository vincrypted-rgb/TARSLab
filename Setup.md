# Guided Instruction for using the **JPCERT Tool Analysis Result Sheet**

>[!TIP!] What is the **JPCERT Tool Analysis Result Sheet?**
> The JPCERT Tool Analysis Result Sheet (sometimes reffered to as just Tool Analysis) is a list, created by **Japan Computer Emergency Response Team**, which is more specifically known as the JPCERT Coordination Center, or JPCERT/CC. This list shows the log results after the use of 49 different **Windows** tools. The Tool Analysis Result Sheet is most commonly used by **blue teamers, incident responders, and threat hunters looking for lateral movement inside of a system or network.**

> The [Github for the Tool Analysis Result Sheet](https://jpcertcc.github.io/ToolAnalysisResultSheet/) has multiple linked reports and more information regarding the background and the uses of it that this lab may not cover.

## Lab Goals

The goal of this lab is to introduce the **JPCERT Tool Analysis Result Sheet** and its uses, as well as establish critical blue team thinking that allows you to apply the same process to unknown proceesses.

## In this lab, you will:

- Review detailed event logs and compare them with the Result Sheet.
- Determine what process was run and what might be the implications of that.
- Learn why Tool Analysis is used in incident response, threat hunting, and so on.
- Understand why the Tool Analysis method is useful and learn how to apply the thinking in other areas.
- Use the Tool Analysis concept to determine what a process is, despite it not being on the sheet.

## You will need:

- A **WINDOWS** VM.
- [JPCERT Tool Analysis Result Sheet](https://jpcertcc.github.io/ToolAnalysisResultSheet/) <-- Click to access the website.
- A little bit of imagination

>[!NOTE]
>The way this lab has been set up, you will briefly be acting as an attacker. In order for Event Viewer to have logs to review, something has to happen on the device. All "attacker" steps will be marked as such. However, the aforementioned "bit of imagination" comes into play when you get into the "defender" stage. In a real incident, it's not often that the incident response team is going to have a step by step guide of how to carry out the exact commands for the attack on their system. Try to imagine taking on a new role without that past information.

## SETUP

In the official Tool Analysis documentation, as well as in this lab, it is recommended that you have **Audit Policy Enabled** and **Sysmon installed**.
The steps for that are as follows:

### Sysmon: 

Access a web browser inside your virtual machine. It doesn't particularly matter which one, but Google Chrome or Microsoft Edge will be the easiest since they come pre-installed on the Antisyphon VMs.

Search for **Sysmon**

Click on **this one**:

Download the file. It should come as a .zip.

When that's finished, unzip the files. This is done one of two ways:
- Seclect the zip folder, and in the upper navigation bar, click **extract all**
- Right click the zip folder, then in the pop up menu, click **extract all**

Still in your VM, close your file explorer and go back to the browser. We're going to install a configuration file for sysmon so that it's more effective. This file will allow us to set up specific rules and filter in only the kind of events we need.

Search for either **"Sysmon Configuration File"** or **"SwiftOnSecurity sysmon"**. Both will get you to the same page, the second is more specific.

Click on **sysmonconfig-export.xml**

On the right side of the screen, you will see a download button. Hovering over it will display the text *Download raw file*. Click this.

>[!CAUTION]
>You **MUST** download the configuration file inside of the folder that you extracted earlier!! Otherwise installation will fail and you'll have to move the file! (It's not that much work, but still...) When it prompts you with where you want to download the file, look at your downloaded files. You should see **Sysmon.zip** and then **Sysmon**. (If you renamed the folder when you extracted the files, you'll see whatever you named it) Do **not** put it in the zip folder. Put it with the extracted files.

Once it's downloaded, you're done with the browser. Open Command Prompt as an administrator. 

Navigate to the directory that you installed the configuration file in (with the extracted files as well).

>[!TIP]
>```dir``` - lists all of the available files and subfolders in the directory you're in.
>```cd``` - without an argument, it prints what directory you're currently in, similar to pwd in powershell
>```cd [directory name]``` - change directory. Moves you into the directory that you specify. It is case sensitive.
>```cd ..``` - goes back one directory. If you were in C:/Users/Administrator/Downloads/Sysmon and typed cd .. you would be moved to C:/Users/Administrator/Downloads.
>```whoami``` - Just in case you forget, it prints the name of the user that you're under as you work. (More helpful when switching between accounts)

Install Sysmon with this command:

```cmd
.\sysmon64.exe -accepteula -i sysmonconfig-export.xml
```
>[!NOTE]
>If you changed the name of your sysmon config file, replace "sysmonconfig-export.xml" with the name of your file.

If it installs properly, you should get this message:

If it does not install properly, you will get this message, or something similar to it:

If that happens, check your syntax, capitalization, spelling, file names, and that you're in the right directory.

To double check that it's working properly, search for **event viewer** in the windows search bar. (Or, if you prefer Windows+R, eventvwr.msc works too.)

It might take a moment to load, but navigate through the following menus: Applications and Services Logs > Microsoft > Windows > Sysmon. It will show as operational and be generating logs. If it doesn't show up in the list of applications in the Windows folder, either refresh/close and reopen Event Viewer, or your installation failed. 

### Enabling Audit Policy

There are two methods to doing this. The first is through Group Policy Object settings. The second is through PowerShell. 

>[!NOTE]
> The JPCERT Report wasn't very clear on what specific parts of the Audit Policy they had enabled. There are some basic recommendations in this lab as the bare minimum, but you are welcome to enable more than are shown here.

**Group Policy Object Method**

Open Run. (Windows Key + R, Windows Key + X and then click Run in the menu, right click the start menu button and then click run in the pop-up menu, search for it with windows search, and so on.)

```run
gpedit.msc
```
This will bring you here.

Underneath Local **Computer** Policy, click on *Windows settings*, then *security settings*.

>[!TIP]
>Typing ```secpol.msc``` into the Run window will bring you straight to the security settings. However, you will need access to the *Administrative Templates* section later on, which is why gpedit is the suggested route.

Click on **Advanced Audit Policy Configuration**

Expand the list by clicking on **System Audit Policies - Local Group**

At this point, you can click into each subsection and decide what you want to have on. When clicking into each audit policy, the Explain tab can be helpful for understanding what the policy will be logging and also the volume of logs that policy will create. 

The **absolute *bare minimum***:

- Detailed Tracking: Audit Process Creation - Success/Failure
- Detailed Tracking: Audit Process Termination - Success/Failure.

Make sure to apply your changes in each menu before leaving the window, as it won't save.

>[!TIP]
>Turn on the **Include Command Line Policy**. Enabling this feature gives us a more detailed look at processes started via command prompt, which might be more likely from the perspective of someone remotely connected via WinRM or RDP but using the command line.
>
>You can enable it by going to *Administrative Templates* (underneath Windows settings up top. You may not see it if you used secpol). Click *System*, then *Audit Process Creation*. Change it to Enabled and click Apply.

### Final Setup

Last thing to do before we're ready to continue with the "attack" phase is to increase the Log sizes. This is recommended in the Tool Analysis report in order to not lose data.

Open Event Viewer.

Expand Windows Logs.

Right click on Security (or whichever log you want to change, but Audit Policies log to security). Click on properties.

Next to Maximum Log Size (KB), enter the new size. The absolute **MAXIMUM** you can have is 4,194,240 KB. (4GB)

Select "Overwrite as needed (oldest events first)". This allows for continuous logging. Click Apply.

Then, repeat this process for the Sysmon log. The path is: *Applications and Services Logs > Microsoft > Windows > Sysmon > Operational.*

Right click on operational. Increase the log size as desired. Overwrite as needed (oldest events first). Apply. 

## ATTACKER PHASE
