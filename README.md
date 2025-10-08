# Setting up Python/Java/.NET Azure Functions development environment on Apple M1/M2 (arm64)

## Summary
At the time of writing, Azure Functions support local development for .NET and Java on arm64 devices but there is no support for Python yet (supposed to happen imminently).

That means that for Python we must set up x86 emulation on arm64 using Apple Rosetta (should be already installed on your Mac)

Based on what I've seen and read, the most elegant solution for our purposes is to install two versions of Homebrew locally. One version for arm64 and one for x86_64. That way, the arm64 version of Homebrew will be responsible for packages that work natively on my Mac M2, while the x86_64 version will contain the emulated ones. This is convenient because we can easily move packages around to a different architecture based on their state of support and performance. Homebrew stores the local packages in different locations depending on your architecture so both can coexist without the risk of collision. 
For arm64, Homebrew will sit in /opt/homebrew/bin/brew and the packes in /opt/homebrew
For x86_64, Homebrew will sit in /usr/local/bin/brew and the packes in /usr/local

Using this approach, we can install development packages for .NET and Java using arm64 version of Homebrew and for Python using the x86_64 version.


### My Environment
* Apple Silicon M2
* OS X Ventura 13.5.1
* Visual Studio Code 1.8.2


## Setting up Python local development

**Note:** We will be installing Python version 3.9 and Azure Functions Core Tools version 4.


1. Open Terminal window and switch to x86_64 architecture:

```console
/usr/bin/arch -x86_64 /bin/zsh —-login
```

2. Verify that the architecture was switched to x86_64 (result should be i386):
```console
arch
```

3. Install Homebrew for emulated packages. The installer will figure out automatically that we're running in x86_64 and will adjust the packages path to /usr/local/bin/brew
```console
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

4. Install Python and Azure Functions Core Tools
```console
/usr/local/bin/brew install python@3.9
/usr/local/bin/brew tap azure/functions
/usr/local/bin/brew install azure-functions-core-tools@4
```
**Note:** It appears that in some cases, azure-functions-core-tools library, is missing the `Arm64` directory inside `/usr/local/Cellar/azure-functions-core-tools@4/YOUR_AZURE_FUNCTION_CORE_TOOLS_VERSION/workers/python/3.9/OSX/` folder. 
This will cause the function to fail when you run your code with the following error: `File DefaultWorkerPath: /usr/local/Cellar/azure-functions-core-tools@4/YOUR_AZURE_FUNCTION_CORE_TOOLS_VERSION/workers/python/3.9/OSX/Arm64/worker.py does not exist`. If that folder is missing, create a symlink:
```console
cd /opt/homebrew/Cellar/azure-functions-core-tools@4/4.3.0/workers/python/3.9/OSX/
ln -s X64 Arm64
```

5. Open a new Terminal window (stay in arm64) and install Homebrew for native packages. You can type `arch` command again to veirfy that you're in the right architecture
```console
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

6. For convenience, we point our env variables in `.zprofile` to different locations based on the architecture of the Terminal window. This way we don't have to put the whole path every time we want to call a different version of a command. We also change the Terminal window title to include the architecture so we know exactly what Terminal we're working in.    Add the following to your `~/.zprofile` file

**Note:** change the azure-function-core-tools alias version number to whatever you've installed in step 3 (here I have 4.0.5312).

```console
if [ $(arch) = "i386" ]; then
    alias python3="/usr/local/bin/python3.9"
    alias brew='/usr/local/bin/brew'
    alias pyenv="arch -x86_64 pyenv"
    alias func="/usr/local/Cellar/azure-functions-core-tools@4/4.0.5312/func"

    # change Terminal window title
    echo -n -e "\033]0;Rosetta i386\007"
else
    # add amd64 homebrew/bin to path (this will also export a bunch of env variables needed by Homebrew
    eval "$(/opt/homebrew/bin/brew shellenv)"

    # change Terminal window title
    echo -n -e "\033]0;zsh arm64\007"
fi
```

7. Restart the `.zprofile` and validate the aliases
```console
source .zprofile
which python3
which brew
which func 
```

8. Most of our development for Python Azure Functions is done in Visual Studio Code. That means that we need a x86_64 Terminal version in there too. Launch VS Code and open the Command Palette by pressing Cmd+Shift+P, select Preferences: Open User Settings (JSON), and add the following JSON to your configuration:
```console
"terminal.integrated.profiles.osx": {
    "rosetta": {
        "path": "arch",
        "args": ["-x86_64", "zsh", "-l"],
        "overrideName": true
    }
}
```
Once you've finished this step, you should see an extra profile for terminal window in VS Code ("rosetta" and "zsh" which are basically x86_64 and arm64)

![alt text](https://learn.microsoft.com/en-ca/azure/azure-functions/media/functions-develop-vs-code/vs-code-rosetta.png)

9. In VS Code we can set `rosetta` Terminal as default (in case someone spends most of their time with Python Azure Functions). Launch VS Code and open the Command Palette by pressing Cmd+Shift+P, select Preferences: Open User Settings (JSON), and add the following JSON to your configuration: 
```console
"terminal.integrated.defaultProfile.osx": "rosetta"
```
10. For existing Python Azure functions, the `venv` has to be recreated using x86_64 `python + azure-functions-core-tools`. 
Open an existing function in VS Code and recreate the `venv` in terminal (make sure it's `rosetta` terminal) after removing the old one
```console
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```
You should now be able to run your Python Azure function in VS Code (make sure it's `rosetta` terminal).

11. <strong>This is optional</strong>. I found myself switching between architectures in Terminal quite often and with a few windows open, it became difficult to keep track of what version of Homebrew I was working with. I wrote a little assistive launcher using applescript in Automator to help with that. Basically it's a app icon to launch a Terminal and set it to x86_64 architecture automatically.

**Note:** The launcher is attached to the repo but here is the code just in case:
```console
if application "Terminal" is running then
	tell application "Terminal"
		# do script without "in window" will open a new window        
		do script "env /usr/bin/arch -x86_64 /bin/zsh"
		activate
	end tell
else
	tell application "Terminal"
		# window 1 is guaranteed to be recently opened window        
		do script "env /usr/bin/arch -x86_64 /bin/zsh" in window 1
		activate
	end tell
end if

delay 1
tell application "System Events"
	keystroke "source ~/.zprofile" & return
end tell
```


## Setting up .NET local development

Azure Functions support local development for .NET, we'll install the SDK using arm64 version of Homebrew. 

1. Open a new Terminal window (stay in arm64) and install .NET SDK from Homebrew
**Note:** I'm installing version 6.0.413. For other versions check: https://github.com/isen-ng/homebrew-dotnet-sdk-versions

```console
brew tap isen-ng/dotnet-sdk-versions
brew install --cask dotnet-sdk6-0-400
```

2. Symlink .NET SDK to /usr/loca/bin
```console
sudo ln -s /usr/local/share/dotnet/dotnet /usr/local/bin/
```

You should now be able to run your Python Azure function in VS Code (make sure it's `zsh` terminal).

## Setting up Java local development

Even though there is a native version of openJDK 17 for arm64, I went with x86_64 mainly because I've tried both and found some quirks with arm64 version (everything still works but minor code changes were needed to get rid of some warnings which wasn't the case with x86_64 version).

**Note:** We're going with version 17 of openJDK because it's the highest version Microsoft supports for Azure functions.

1. Open a new Terminal window (in x86_64) and install openJDK 17 from Homebrew
```console
brew install openjdk@17
```

2. Symlink openJDK to /usr/loca/bin
```console
sudo ln -sfn /usr/local/opt/openjdk@17/libexec/openjdk.jdk /Library/Java/JavaVirtualMachines/openjdk-17.jdk
```

3. Set JAVA_HOME in your `.zprofile`
```console
export JAVA_HOME="/Library/Java/JavaVirtualMachines/openjdk-17.jdk/Contents/Home/"
```

4. Point VS Code to your new Java location.
Launch VS Code and open the Command Palette by pressing Cmd+Shift+P, select Preferences: Open User Settings (JSON), and add the following JSON to your configuration:
```console
"java.jdt.ls.java.home": "/Library/Java/JavaVirtualMachines/openjdk-17.jdk/Contents/Home"
```

You should now be able to run your Python Azure function in VS Code (make sure it's `rosetta` terminal).

**Note:** Should you decide to try the native openJDK, stay in arm64 Terminal and run the same steps except for step number 2 since openJDK will be installed into a different location. Replace it with this:
```console
sudo ln -sfn /opt/homebrew/opt/openjdk@17/libexec/openjdk.jdk /Library/Java/JavaVirtualMachines/openjdk-17.jdk
```

## Building and running Docker images/containers

Go to Docker Settings and make sure the following options are enabled (basically says that when we build a non amd64 docker image, we should use the Rosetta emulator):

	Settings->General->Use Virtualization framework
	Settings->Features in Development->Use Rosetta for x86/amd64 emulation on Apple Silicon

#### Building Docker image
```console
docker build -f ./Dockerfile --platform=linux/amd64 -t function-name-docker-image:latest .
```

#### Running Docker image
```console
docker run --platform linux/amd64 --name function-name-docker-container -p 80:80 -it function-name-docker-image:latest	
```

## Oracle Client + cx_Oracle
We need to install the Oracle client in x86_64 and then the cx_Oracle extension module in venv of our project.

1. Pick a version and download the Oracle client for Mac from: https://www.oracle.com/ca-en/database/technologies/instant-client/macos-intel-x86-downloads.html

Make sure you download the `Basic Package(ZIP), SQL*Plus Package (ZIP) and SDK Package (ZIP)`. Unzip the files and move the contents of all three folders into one new singe folder.

2. Update your .zprofile and point the env virables to the new folder where Oracle client files sit.
My client is sitting in: `/Users/my_user/Oracle_client/19.8.0.0.0/`. I've set it up this way in case I want to try another client version later on. 

**Note:** In .zprofile, make sure these variables are set within the i386 block (the IF statement in the .zprofile that checks which architecture we're running)

**Note:** Point `TNS_ADMIN` to wherever your `tnsnames.ora` file is at.

```console
export ORACLE_BASE=/Users/slava/Projects/Oracle_client
export ORACLE_HOME=$ORACLE_BASE/19.8.0.0.0
export PATH=$ORACLE_HOME:$PATH
export DYLD_LIBRARY_PATH=$ORACLE_HOME
export OCI_LIB_DIR=$ORACLE_HOME
export OCI_INC_DIR=$ORACLE_HOME/sdk/include
export TNS_ADMIN=$ORACLE_BASE/admin/network  
```
3. Test SQLPlus in Terminal (x86_64)
```console
sqlplus "user/pass@(DESCRIPTION=(ADDRESS=(PROTOCOL=TCP)(Host=hostname.network)(Port=1521))(CONNECT_DATA=(SID=remote_SID)))"
```

4. Start new Python project, create venv and install cx_Oracle.
```console
python3 -m venv .venv
source .venv/bin/activate
pip install cx_Oracle
```

## Spark development

Coming soon...