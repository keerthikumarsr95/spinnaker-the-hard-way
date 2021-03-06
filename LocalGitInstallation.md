# Spinnaker LocalGit Installation

I created this document after following the document titled ["Getting Set Up for Spinnaker Development"](https://www.spinnaker.io/guides/developer/getting-set-up/). Without this document, and help from Spinnaker Core member(s), let's just say none of this would be possible

## My laptop Details

- Installed on my Mac running macOS Sierra (10.12.6)
- MacBook Pro (15" 2016); 16 GB RAM; 2.6 GHz Intel Core i7

## Key File/Folder Locations

- **~/.hal**
> This is the folder where `hal` stores its configuration(s) for the Spinnaker deployment on your laptop.
- **~/.spinnaker**
> This is the folder that your Spinnaker microservices read from. `hal deploy apply` generates files in this folder for the individual microservices.
- **~/dev/spinnaker**
> This is the folder where we will be forking and cloning all the Spinnaker microservices. `hal` command will create additional folders here: 
> - `logs` where the log and error files for each microservice that's running is stored
> - `scripts` where the start/stop scripts for each microservice is stored
> -`*.pid` files for each microservice that's running
- **/opt/halyard**, **/opt/spinnaker**
> `hal` is installed in `/opt/halyard` folder. `/opt/spinnaker` folder consists of a `config` folder with two files that looks like configuration files (`halyard-user` and `halyard.yml`). _Why use two folders?_ Looks like a technical debt to me :wink:

## Assumptions

- You have a GitHub account
- You have configured your laptop to access repos in GitHub over SSH. References: [1](https://help.github.com/articles/adding-a-new-ssh-key-to-your-github-account/) and [2](https://help.github.com/articles/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent/)
- You have `gcloud` installed and available on your `$PATH`. Also, make sure it is configured to interact with your GCP account. Here's a quick 3 step installation process on a Mac:

```bash
curl https://sdk.cloud.google.com | bash
exec -l $SHELL
gcloud init
```

I used Google Cloud SDK version `219.0.1` and `gsutil` version `4.34`

## Core Tools

- **git**
> `brew install git` (Version running on my laptop: 2.19.0)
- **curl**
> `brew install curl; 7.61.1` (Version running on my laptop: 7.61.1)
- **netcat**
> `brew install netcat; 0.7.1` (Version running on my laptop: 0.7.1)
- **redis-server**
> `brew install redis; brew services start redis` (Version running on my laptop: 4.0.11)
- **java**
> Downloaded directly from Oracle (Version running on my laptop: 1.8.0_31)
- **node**

```bash
curl -o- https://raw.githubusercontent.com/creationix/nvm/v0.33.11/install.sh | bash
# Follow instructions at end of script to add nvm to ~/.bash_rc
  
nvm install v8.9.0 
# Version running on my laptop: 8.9.0
```

- **yarn**

```bash
npm install -g yarn
# Version running on my laptop: 1.9.4
```

Make sure all the pieces are installed by running the following:

```bash
{
echo "---gcloud---"
gcloud config list
echo "---git---"
git --version
echo "---curl---"
curl --version
echo "---netcat---"
netcat -V
echo "---redis-server---"
/usr/local/Cellar/redis/4.0.11/bin/redis-server -v
echo "---java---"
java -version
echo "---node---"
node -v
echo "---yarn---"
yarn -v
}
```

If you have an existing Redis server, run the following command to flush the database:

```bash
shell-prompt>redis-cli
127.0.0.1:6379> FLUSHDB ASYNC
OK
127.0.0.1:6379> KEYS *
(empty list or set)
127.0.0.1:6379> exit
```

## Install Halyard

```bash
cd ~/
curl -O https://raw.githubusercontent.com/spinnaker/halyard/master/install/macos/InstallHalyard.sh
sudo bash InstallHalyard.sh
rm InstallHalyard.sh
```

## Set up Storage Service

If you have used GCS already as the Storage provider, you may want to clean up before setting up your new local setup:

```bash
gsutil ls gs:// | grep -i spin
gsutil rm -r gs://<spin-bucket-name>
```

Time to make sure you are ready to interact with Google Cloud Storage. Here are the commands I ran to get things setup.

```bash
{
SERVICE_ACCOUNT_NAME='spinnaker-gce-account'
SERVICE_ACCOUNT_DEST='/Users/anasharm/.config/gcloud/spinnaker-gce-account.json'

# If the Service Account already exist, running the command again will simply throw an error. Ignore it
gcloud iam service-accounts create $SERVICE_ACCOUNT_NAME --display-name $SERVICE_ACCOUNT_NAME
SA_EMAIL=$(gcloud iam service-accounts list --filter="displayName:$SERVICE_ACCOUNT_NAME" --format='value(email)')
PROJECT=$(gcloud info --format='value(config.project)')

# Run this command and look for "spinnaker-gce-account" as a member of the project with a role of "roles/storage.admin"
# gcloud projects get-iam-policy $PROJECT | grep -i --context=2 "roles/storage.admin"
# If it does, you can skip this next command. However, running the command will do no harm
gcloud projects add-iam-policy-binding $PROJECT --role roles/storage.admin --member serviceAccount:$SA_EMAIL
gcloud iam service-accounts keys create $SERVICE_ACCOUNT_DEST --iam-account $SA_EMAIL
BUCKET_LOCATION=us
}
```

If everything looks good, run the following commands to configure Storage Service. Here's an interesting observation. Prior to running the following command, `~/.hal` folder had no `config` file. There is no `~/.spinnaker` folder created at this stage either. After running it, `~/.hal` folder is populated with `config` file, a `default` profile folder with an empty `config` folder inside of it. This command adds the GCS details under `persistentStorage > gcs` section of the `config` file. Bottom line, as you run the `hal config` commands, you are essentially building up the configuration of your local Spinnaker install

```bash
hal config storage gcs edit --project $PROJECT --bucket-location $BUCKET_LOCATION --json-path $SERVICE_ACCOUNT_DEST
```

Running the command below adds a single line to the `~/.hal/config` file: `persistentStoreType: gcs` under `persistentStorage` as shown below:

![Persistent Storage GCS](https://s3.amazonaws.com/us-east-1-anand-files/spinnaker-localgit-install/gcs-storage.png)

```bash
hal config storage edit --type gcs
```

## Set up Cloud Provider (Kubernetes)

Considering that running all Spinnaker microservices on a machine with 16GB RAM is in and of itself a stretch, trying to run the Cloud Provider locally would pretty much kill the laptop. So, this is where we all have a choice: Pick the Cloud Provider that is most convenient for your use case. Ideally, point to an instance of Kubernetes that you already have access to.

[Download rtp-learn.yml - Cisco/CoDE team members ONLY](https://gitscm.cisco.com/projects/NERDS/repos/spinnaker-localgit/browse/kube-config/rtp-learn.yaml)

I prefer to not use the `~/.kube/config` file on my laptop, as that tends to change a lot. At least for me it does. So I tend to create a separate configuration file within `~/.kube/` folder (say, `spinnaker.yaml`) with the relevant Kubernetes cluster entry (or entries) that your local Spinnaker would deploy applications to.

```bash
export KUBECONFIG="/Users/anasharm/.kube/rtp-learn.yaml"
hal config provider kubernetes account add rtp-learn-admin --provider-version v2 --context $(kubectl config current-context) --kubeconfig-file "/Users/anasharm/.kube/rtp-learn.yaml"
hal config provider kubernetes enable
```

After running these commands shown above, the `~/.hal/config` file had two changes: kubernetes enabled was set to "true" (not shown in the screenshot below) and kubernetes account `rtp-learn-admin` was added with the details provided as part of the command line

![Kubernetes v2 Account](https://s3.amazonaws.com/us-east-1-anand-files/spinnaker-localgit-install/kubernetes-v2-1.png)

Not entirely sure why the document titled [Kubernetes Provider V2](https://www.spinnaker.io/setup/install/providers/kubernetes-v2/) says the following command is also necessary as part of provisioning a Kubernetes Cloud provider. Anyways, the result of running the above command was that the `~/.hal/config` changed whereby a new line `artifacts: true` was added under the `features:` section.

```bash
hal config features edit --artifacts true
```

![Kubernetes v2 Account](https://s3.amazonaws.com/us-east-1-anand-files/spinnaker-localgit-install/kubernetes-v2-2.png)

## Fork Spinnaker Microservices

The goal of this section is the following:

- Fork all the Spinnaker Microservices into your personal GitHub account

Here are all the Spinnaker Microservices that are documented in [Spinnaker Reference Docs](https://www.spinnaker.io/reference/architecture/#spinnaker-microservices). 

**OPTION 1:**

Open each of the links below in a separate tab and fork the repo into your personal repo:

- [Deck](https://github.com/spinnaker/deck)
- [Gate](https://github.com/spinnaker/gate)
- [Orca](https://github.com/spinnaker/orca)
- [Clouddriver](https://github.com/spinnaker/clouddriver)
- [Front50](https://github.com/spinnaker/front50)
- [Rosco](https://github.com/spinnaker/rosco)
- [Igor](https://github.com/spinnaker/igor)
- [Echo](https://github.com/spinnaker/echo)
- [Fiat](https://github.com/spinnaker/fiat)
- [Kayenta](https://github.com/spinnaker/kayenta)
- [Halyard](https://github.com/spinnaker/halyard)


**OPTION 2:**

Run the [Setup Forks](spin-scripts/setup-forks.sh) script. However, the script assumes that you have the following on your latop:

- [Hub CLI](https://hub.github.com/)
- Configure `hub` CLI so that it knows your GitHub username and GitHub Personal Token

## Configure Spinnaker Deployment 

Configure `hal` to use the 'LocalGit' install type, as opposed to the default 'LocalDebian' type.

```bash
hal config deploy edit --type localgit --git-origin-user=indrayam
```

![LocalGit Deployment Type](https://s3.amazonaws.com/us-east-1-anand-files/spinnaker-localgit-install/localgittype-1.png)

Configure Spinnaker to update/install version `branch:upstream/master`. When we run `hal deploy apply`, deploy this version of Spinnaker microservice. Notice that since this is a local deployment, you are not specifying a version (like `1.9.5`). Instead, you are asking `hal` to get the latest version on the `master` branch

```bash
hal config version edit --version branch:upstream/master
```

![Spinnaker Version](https://s3.amazonaws.com/us-east-1-anand-files/spinnaker-localgit-install/localgittype-2.png)

## Configure Fiat (OPTIONAL)

Run the following commands to enable Fiat Authn and Enable LDAP as the authentication source

```bash
hal config security authn ldap edit --user-dn-pattern=".." --url="..."
hal config security authn ldap enable
```

Here's what the `~/.hal/config` changes looked like after running the command `hal config security authn ldap edit...`

![Fiat LDAP Edit Changes](https://s3.amazonaws.com/us-east-1-anand-files/spinnaker-localgit-install/fiat-ldap-3.png)

[hal config security authn ldap edit command - Cisco/CoDE team members ONLY](https://gitscm.cisco.com/projects/NERDS/repos/spinnaker-localgit/browse/hal-commands/ldap-authc.sh)

If you would like to customize the LDAP settings and `hal` does not seem to be co-operating, after the final `hal deploy apply` command is run (see below), feel free to create `~/.spinnaker/fiat-local.yml` file to selectively override configuration values in `~/.spinnaker/fiat.yml`

[Download fiat-local.yml - Cisco/CoDE team members ONLY](https://gitscm.cisco.com/projects/NERDS/repos/spinnaker-localgit/browse/spinnaker-local-config/fiat-local.yml)

An interesting trivia: After running `hal config security authn ldap enable`, the `~/.hal/config` file had these changes (or not):

- Set `authn > ldap > enabled: true`
- Set `authn > enabled > true`
- It did not do anything to `features > auth: false` or `features > fiat: false` settings

![Fiat LDAP Enabled Changes](https://s3.amazonaws.com/us-east-1-anand-files/spinnaker-localgit-install/fiat-ldap-4.png)

## Configure Gate (OPTIONAL)

Run the following commands to enable Gate to use LDAP as Role Provider.

```bash
hal config security authz ldap edit \
    --url ldap://...:.../dc=abc,dc=com \
    --manager-dn '...' \
    --manager-password \
    --user-dn-pattern cn={0},ou=... \
    --group-search-base OU=...,dc=abc,dc=com \
    --group-search-filter "(member{0})" \
    --group-role-attributes cn
hal config security authz enable
```

[hal config security authz ldap edit command - Cisco/CoDE team members ONLY](https://gitscm.cisco.com/projects/NERDS/repos/spinnaker-localgit/browse/hal-commands/ldap-authz.sh)

![Gate Changes Part 1](https://s3.amazonaws.com/us-east-1-anand-files/spinnaker-localgit-install/gate-ldap-1.png)

![Gate Changes Part 2](https://s3.amazonaws.com/us-east-1-anand-files/spinnaker-localgit-install/gate-ldap-2.png)

If you would like to customize the LDAP settings and `hal` does not seem to be co-operating, after the final `hal deploy apply` command is run (see below),  create `~/.spinnaker/gate-local.yml` file to selectively override configuration values in `~/.spinnaker/gate.yml`

[Download gate-local.yml - Cisco/CoDE team members ONLY](https://gitscm.cisco.com/projects/NERDS/repos/spinnaker-localgit/browse/spinnaker-local-config/gate-local.yml)

## Review and Deploy

Before you run the following command, just to be on the safe side, do the following:

- Run the following command. Why? I am not sure, but I do remember seeing a permission error the first time I ran this command on my Mac. Anyways, it does not hurt.
- If you're on a Mac with 16 GB of RAM, it may be a good idea to have the Activity Monitor running to see how your CPU usage spikes
- Also, turn off absolutely everything that you really DO NOT need. I even turned off my Docker daemon.

```bash
sudo chown -R <userid>:staff /opt/halyard
```

You're now ready to review the configurations defined in `~/.hal/config` and apply it by running

```bash
hal deploy apply --service-names clouddriver front50 deck gate fiat orca
#Or
hal deploy apply --service-names clouddriver front50 gate fiat orca
# followed by
hal deploy apply --service-names deck
```

Things that happens, assuming everything goes well:

- `~/.spinnaker/` folder will get a single YAML file per Microservice that is started
- `~/dev/spinnaker/` folder will get populated by one folder per Microservice. This is where `hal` clones your forked Spinnaker repositories (from your GitHub account) and sets up `upstream` git remote to point to the respective Microservice repo under `spinnaker` GitHub organization
- `~/dev/spinnaker/` folder also gets `scripts` and `logs` folder. You can individually stop and start microservices using the scripts inside the `scripts` folder. However, it is a pain to do it one at a time. You might find the scripts linked to in the section below a bit more useful
- `deck.err` file under `~/dev/spinnaker/logs` folder gets insanely large and seems to hog all your laptop's CPU as `deck` is starting up. Not sure why. Takes a good 10 mins or so before `deck` startup is complete and the `deck.err` stops growing in size. In my case, it stopped after it had grown from 0 to **2.1 MB** in size! 

## Immediate Next Steps

First things first:

- Download all [Spin Scripts](spin-scripts/) folder under `/tmp/spin-scripts`
- Run [Stop All Microservices](spin-scripts/stop-all.sh) to stop all the Spinnaker microservices. Once this command successfully completes, you will have your laptop back :wink: You're welcome!

Feel free to re-run the script. It would not hurt. If you see messages complaining about individual microservices not running or such, don't worry. Our first goal is to get them all stopped!

Open the "Activity Monitor" and make sure that you do not see Spinnaker microservices related processes. Most of them will have their Process Name as "java". If you see a few, select and double-click each. You should an entry for the `halyard` daemon and another one for the `gradle` daemon. If you see others and you know that these are not Spinnaker microservices, you are all set.

Finaly step: Quickly peruse through the Spinnaker microservice configurations under `~/.spinnaker`

## Pick your Spinnaker Microservices

Everyone is interested in learning or working with a few Spinnaker microservice(s). Mine are:

- deck
- gate
- clouddriver
- fiat
- front50
- igor

Customize [Start Core Services](spin-scripts/start-core.sh) shell script to suit your needs. Make sure you do the same for the [Stop Core Services](spin-scripts/stop-core.sh) script

When it all looks good, start the core services:

```bash
/tmp/start-core.sh
```

## Time to Read/Write Code

If you want to review code and learn from the Spinnaker "Gods" of code, I would encourage using an editor like [Sublime Text](https://www.sublimetext.com/3). After installing the editor (assuming you do not already have it installed), install the [Package Control](https://packagecontrol.io/installation) plugin which makes installing Sublime Text plugins trivial.

Install [ProjectManager](https://github.com/randy3k/ProjectManager) plugin and then create a Project called `spinnaker-dev` with the following Project configuration:

```bash
{
    "folders":
    [
        {
            "path": "~/.hal"
        },
        {
            "path": "~/.spinnaker"
        },
        {
            "path": "~/.spinnaker-local"
        },
        {
            "path": "~/workspace/spinnaker-the-hard-way/spin-scripts"
        },
        {
            "path": "~/dev/spinnaker/halyard"
        },       
        {
            "path": "~/dev/spinnaker/clouddriver"
        },
        {
            "path": "~/dev/spinnaker/deck"
        },
        {
            "path": "~/dev/spinnaker/echo"
        },
        {
            "path": "~/dev/spinnaker/fiat"
        },
        {
            "path": "~/dev/spinnaker/front50"
        },
        {
            "path": "~/dev/spinnaker/gate"
        },
        {
            "path": "~/dev/spinnaker/igor"
        },
        {
            "path": "~/dev/spinnaker/kayenta"
        },
        {
            "path": "~/dev/spinnaker/orca"
        },
        {
            "path": "~/dev/spinnaker/rosco"
        },
        {
            "path": "~/dev/spinnaker/scripts"
        },
    ]
}
```

Once ready, you will be able to search across all microservices using simple keyboard shortcuts like `Cmd+P`. I cannot say enough about how easy it is to breeze through code in Sublime Text

If you would like to dive into writing Java software, you probably want to use IntelliJ. Repeat these steps for each Microservice that you want to work with. Remember, IntelliJ is itself a Java software and has the potential of being a memory hog. You've been warned :wink:

```bash
cd ~/dev/spinnaker/<microservice>/
./gradlew idea
```

If something goes wrong, run the following to clean all IntelliJ related files/folders:

```bash
git clean -dnxf -e '*.iml' -e '*.ipr' -e '*.iws'
git clean -dxf -e '*.iml' -e '*.ipr' -e '*.iws'
```

## Useful Commands/Hacks

- `hal deploy apply --service-names deck, fiat, gate, clouddriver, orca, front50`

## Maximum number of files that MacOS can open

[Source: Is there a fix for the “Too many open files in system” error on OS X 10.7.1?](https://superuser.com/a/1171028)

Run:

`launchctl limit maxfiles`



For OS X Sierra (10.12.X), the default value seems to be low. 256 by default.

1. Create a file at /Library/LaunchDaemons/limit.maxfiles.plist and paste the following in (feel free to change the two numbers (which are the soft and hard limits, respectively):

```bash
<?xml version="1.0" encoding="UTF-8"?>  
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"  
        "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">  
  <dict>
    <key>Label</key>
    <string>limit.maxfiles</string>
    <key>ProgramArguments</key>
    <array>
      <string>launchctl</string>
      <string>limit</string>
      <string>maxfiles</string>
      <string>64000</string>
      <string>524288</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
    <key>ServiceIPC</key>
    <false/>
  </dict>
</plist>
```

2. Change the owner of your new file:

`sudo chown root:wheel /Library/LaunchDaemons/limit.maxfiles.plist`

3. Load these new settings:

`sudo launchctl load -w /Library/LaunchDaemons/limit.maxfiles.plist`

4. Finally, check that the limits are correct:

`launchctl limit maxfiles`

5. Check your ulimit settings. I am not sure if we need to increase those as well, or will the System Settings done in the steps above will be enough

```bash

ulimit -a
ulimit -n 8192
```
