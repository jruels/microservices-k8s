# Setup Student VM's

# Clone the lab repo in VS Code

### Step 1: Clone the repo

1. Open Visual Studio Code
2. In Visual Studio Code, click **Clone Repository** and paste `https://github.com/jruels/microservices-k8s.git`
3. Hit **Enter**, and in the pop-up window, browse to `C:\Users\tekstudent\Downloads\repos`
4. Click **Select as repository destination**
5. When prompted to open the cloned repo, choose **Open**.
6. After opening the folder, click the third icon in the left toolbar for source control. Next to **changes**, click the ellipses (three dots) and choose **pull**.

# Configure AWS credentials

### **Step 1: Log into the AWS Console**

1. In a browser, log into the [AWS Console](https://console.aws.amazon.com/) using the credentials in the spreadsheet below.
    * [Cloud credentials](https://docs.google.com/spreadsheets/d/1gTV6btPeIyyXylRkDn2_LNbWkf9BGU6wsi5eIb-ynLY/edit?gid=2103659978#gid=2103659978)
2. Search for IAM in the search bar.
3. Click **IAM**
4.Click **Users**.
5. Click **autodev-admin**.
6. Click **Security Credentials**.
7. Scroll down to **Access Keys**
   1. Click **Actions** -> and **Deactivate** and **Delete** any existing keys.
8. Click **Create access key**
9. Select **Command Line Interface (CLI)**. 
10. Check the confirmation box at the bottom of the page and click **Next**.
11. Skip the description and click **Create access key**.
12. **IMPORTANT:** Copy the **Access key** and **Secret access key** and save them somewhere. You can optionally download the `csv` file for easy reference. 

---

### **Step 2: Use AWS Configure**

1. In the Visual Studio Code Terminal run: 

   ```
   aws configure
   ```

2. Supply the required information.
   * Credentials 
   * Region = `us-west-1`
3. Select the default option for the remaining options.

---

### **Step 3: Test AWS CLI**

1. Confirm that `aws` can use the credentials.

   ```
   aws sts get-caller-identity
   ```

2. You should see your account information returned.


## Set up a remote SSH session in Visual Studio Code.   

### Create the SSH configuration file.

On the left sidebar, click the icon that looks like a computer with a connection icon.

In the Remote Explorer, hover your mouse cursor over **SSH**, click on the gear icon (⚙️) in the top right corner, and select the top option: `C:\Users\tekstudent\.ssh\config` This will open the SSH configuration file in a new editor tab.

### Add the SSH configuration for the lab server.

Add the following lines to the SSH configuration file, replacing `<IP of server from the spreadsheet>` with the actual IP address of your server and `<Path to the cloned lab directory/keys/lab.pem>` with the correct path to the `lab.pem` file in your lab directory.

**PROTIP**: Right-click the `lab.pem` in the Visual Studio Code Explorer and click `Copy Path`. Paste it below as the value for `IdentifyFile`

```plaintext
Host ubuntu
  HostName <IP of server from the spreadsheet>
  IdentityFile <Path to the cloned lab directory/keys/lab.pem>
  User ubuntu
```

### Save the SSH configuration file.

Save the changes to the SSH configuration file and close it.

### Connect to the lab server.

1. In the Remote Explorer, you should now see the entry for the ubuntu server under "SSH Targets."
2. Click on the entry to connect.
3. Visual Studio Code will open a new window connected to the server.
4. You can now open a terminal in this new window and run commands on the server.

### Create a working directory

In Visual Studio Code, you can create a new folder or file as if it was on your local machine.
Click **Open Folder** and select `/home/ubuntu`.
In future labs, you can create a directory for each lab.


## Congratulations

Congratulations, you've successfully configured your machine.
