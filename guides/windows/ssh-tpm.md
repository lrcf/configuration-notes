# SSH on TPM (Windows)

This guide will cover:
* How to create a new personal certificate on TPM
* How to install `putty-cac` and `wsl-ssh-pageant` (necessary for authentication)
* Export of the public key of the certificate to `ssh-rsa` using Pageant

> ***⚠️ The official TPM provider <u>does not</u> support `Curve25519` and there’ve been instances of [broken implementation of generating RSA keys on TPM chips in the past.](https://www.bleepingcomputer.com/news/security/tpm-chipsets-generate-insecure-rsa-keys-multiple-vendors-affected/)** Before proceeding, always make sure your system and drivers are up to date.*
> 
> *That being said, hardware authentication shall still be prefered over software authentication. In majority of cases, you’ll be more secure by authenticating using TPM but always consider the attack factor when deciding between TPM and file-based authentication.*
>
> *From solely security standpoint (disregarding convenience), it’s recommended to use a hardware key that supports stronger cryptography instead of an insecure TPM chip.*

*You will need admin permissions to successfully issue a certificate. Pro version of Windows might be required.*

*This guide assumes you have [Chocolatey](https://chocolatey.org/) installed on your machine.*

---

### 0. Check if TPM is enabled

1. Open `tpm.msc` (you will be asked for admin permissions)
2. The window should have the following messages:
	* **Status** — “The TPM is ready for use.”
	* **TPM Manufacturer Version** — should have a version listed.

> For your security, you can opt to clear your TPM using the **“Clear TPM…”** option located under the **“Actions”** window on the right side. A computer restart will be required.

If the system reports that the TPM is not ready or enabled, you might have an option to enable the TPM in computer board (BIOS/UEFI) menu.

Instructions for accessing the computer board menu are different for each computer, in most cases, you can access it by pressing `Esc` or `Del` and following the instructions there.

Modern processors by major brands (Intel and AMD) have support for TPM embedded on the processor chip itself.

### 1. Create a TPM smart card

Open PowerShell <u>**as administrator**</u> and type in the following command:

```bat
tpmvscmgr create `
	/name "<computer name> TPM VSC" `
	/pin prompt `
	/adminkey random `
	/generate `
	/attestation AIK_AND_CERT
```

* **`/name`** — This name will be displayed under _Device Manager._ (`devmgmt.msc`)
* **`/attestation AIK_AND_CERT`** — Your smart card will be attestated by Microsoft that your chip is *truly* hardware-bound. *(requires internet connection)*
	* Alternatively, `/attestation AIK_ONLY` can be used in place of the argument.


#### Additional resources:
* [`tpmvscmgr` command reference](https://docs.microsoft.com/en-us/windows-server/administration/windows-commands/tpmvscmgr)
* [Further documentation for `tpmvscmgr`](https://docs.microsoft.com/en-us/windows/security/identity-protection/virtual-smart-cards/virtual-smart-card-tpmvscmgr)

### 2. Generate a new SSH certificate

> *It’s always recommended to generate a new SSH certificate instead of re-using an existing one.* <!-- TODO: However if using a new certificate isn’t an option for you, here’s how you can store an existing certificate to TPM. -->

Open PowerShell <u>**as administrator**</u> and type in the following command:

```powershell
New-SelfSignedCertificate `
	-Subject "CN=<display name>,O=<computer name>" `
	-KeyAlgorithm RSA -KeyLength 4096 `
	-Provider "Microsoft Smart Card Key Storage Provider" `
	-CertStoreLocation "Cert:\CurrentUser\My"
```

* **`-Subject`** — This defines a certificate subject (in this case, a user of the computer).
	* **`CN`** (`CommonName`) — Name of the certificate holder.
	* **`O`** (`Organisation`) — Name of the organisation (in this case, the computer).
* **`-KeyAlgorithm RSA -KeyLength 4096`**
	* Supported algorithms depend on the TPM chip embedded on your computer.
	* If creation of the certificate fails with the `SCARD_E_UNSUPPORTED_FEATURE` error code, the TPM chip on your computer <u>doesn’t support 4096-bit RSA.</u> You’ll have to generate a 2048-bit RSA key by replacing `4096` with `2048`.
	* As of December 2020, there has been no known implementation of `Curve25519` on TPM. ([source](https://twitter.com/mjg59/status/1341146141220454400))
* **`-Provider`** — This string ensures the key will be stored in the smart card on TPM we just created.
* **`-CertStoreLocation`** — Specifies that this certificate will be created for _the current user_ and stored under _current user._ For security reasons, creating and storing a key for each user of the machine is strongly recommended.

After successful creation of your certificate, you can view your freshly-created certificate in the Certificate Manager:
1. Open Certificate Manager (`certmgr.msc`)
2. On the left-side panel, go to **“Personal” → “Certificates”**
3. On the right-side panel, you should now see your certificate for you to inspect.

#### Additional resources:
* [List of supported algorithms for the TPM provider](https://docs.microsoft.com/en-us/windows/win32/seccertenroll/cng-key-storage-providers#microsoft-smart-card-key-storage-provider) — **Note:** Not all algorithms will be supported on *your machine.*

### 3. Install `putty-cac` and the bridge

Open PowerShell <u>**as administrator**</u> and type in the following command:

```bat
choco install putty-cac wsl-ssh-pageant-gui
```

puTTY will create items on your start menu. You can freely remove them by following these steps:
1. Open the start menu.
2. Search for **puTTY** on your start menu list and right click on the item.
3. Select **“More” → “Open file location”**
4. Remove the undesired shortcuts and folders located under `C:\ProgramData\Microsoft\Windows\Start Menu\Programs\`.

#### Additional resources:
* `putty-cac` ([Chocolatey](https://community.chocolatey.org/packages/putty-cac), [GitHub](https://github.com/NoMoreFood/putty-cac))
* `wsl-ssh-pageant-gui` ([Chocolatey](https://community.chocolatey.org/packages/wsl-ssh-pageant-gui))
	* On GitHub as [`wsl-ssh-pageant`](https://github.com/benpye/wsl-ssh-pageant)

<h3 id="#4-configure-to-start-pageant-and-the-bridge-on-sign-in">4. Configure to start Pageant and the bridge on sign in</h3>

1. Open `taskschd.msc` (you will be asked for admin permissions)
2. Under the **“Actions”** window on the right side, click on **“Create Task…”**
3. A new window will appear, configure the task with the following settings:
	* **“General”** tab:
		* **Name** — Set to whatever you like, for easier identification, you can set it to “Pageant”.
		* Select **“Run only when the user is logged on”**
		* <u>Make sure **“Run with highest privileges”** is unselected.</u>
		* **“Configure for”** — “Windows 10”
	* **“Triggers”** tab:
		* Click on the **“New…”** on the bottom side of the window.
		* **“Begin the Task”** — “At log on”
		* **“Any user”**
		* **“Enabled”** under “Advanced settings”
		* Click on **“OK”**.
	* **“Actions”** tab:
		* Click on the **“New…”** on the bottom side of the window.
		* **“Action”** — “Start a program”
		* **“Program/script”** — `C:\ProgramData\chocolatey\bin\wsl-ssh-pageant-gui.exe -systray -winssh ssh-pageant`
		* You will be asked whether the program you specified has arguments, click on **“Yes”**.
		* Repeat the steps above, but this time for `"C:\Program Files\PuTTY\pageant.exe"`.
	* **“Conditions”** tab:
		* <u>**Disable**</u> all conditions.
	* **“Settings”** tab – check <u>only</u> the following checkboxes:
		* **“Allow task to be run on demand”**
		* **“If the running task does not end when requested, force it to stop”**
		* On the bottom menu, select **“Do not start a new instance”**.
		* <u>Leave everything else unchecked.</u>
4. Click on **“OK”**.
5. If you have any instances of Pageant or the bridge opened, <u>close them.</u>
6. Once the task has been created, select the task and under the “Selected Item” menu, click on **“Run”**.

### 5. Configure Pageant to accept smart card certificates

*Pageant and the bridge should be now running if you have started [the scheduled task.](#4-configure-to-start-pageant-and-the-bridge-on-sign-in)*

1. On the system tray menu, right click on the Pageant icon.
2. Enable the following options:
	* **“Autoload Certs”**
	* **“Remember Certs”**
	* **“Filter: Smart Card Logon Certs”**
	* **“Filter: No Expired Certs”**

### 6. Import certificate to Pageant and export as `ssh-rsa`

*Pageant and the bridge should be now running if you have started [the scheduled task.](#4-configure-to-start-pageant-and-the-bridge-on-sign-in)*

1. Once again on the Pageant menu, click on **“View Keys & Certs”**.
2. Select your certificate here.
	* The certificate should have been automatically added by Pageant. If not, add it by clicking on the **“Add CAPI Cert”** button and selecting the certificate from the window.
3. Click on **“Copy To Clipboard”** to copy the `ssh-rsa` representation of your certificate to clipboard.
4. Add the public key to `~/.ssh/authorized_keys` of the target machines or to your favourite services.
	* Here’s a quick link to manage the keys on your GitHub profile: [`https://github.com/settings/keys`](https://github.com/settings/keys)

### 7. Configure Pageant for Windows’s OpenSSH

Open PowerShell <u>**as administrator**</u> and type in the following command:

```powershell
[System.Environment]::SetEnvironmentVariable('SSH_AUTH_SOCK', '\\.\pipe\ssh-pageant', 'Machine'); refreshenv
```

* This will set the `SSH_AUTH_SOCK` environment variable to `\\.\pipe\ssh-pageant` for the whole *machine* and refresh the session’s environment variables.

Now create `%userprofile%\.ssh\config` with the following content:

```
Host *
	IdentityAgent SSH_AUTH_SOCK
```

### Testing the configuration

Now to test your configuration, try to sign in to your target machine using `ssh <target>`. (In case of GitHub: `ssh git@github.com` – you will see your username in the output.)

If the authentication is successful, then congratulations, you’ve done everything right and now you are successfully signing in using the SSH key stored on your machine’s TPM.