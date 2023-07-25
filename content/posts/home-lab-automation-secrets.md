---
title: "Secrets Manager for Home Lab Automation"
date: 2023-07-25T15:19:00-03:00
draft: true
---

One of the problems I found when working with my home lab is that I was lacking a solution for storing secrets. Even though there are open source solutions like Hashicorp vault to manage secrets, and managing it something requires maintenance and I didn't want to have the operational burden.  
I used to manage my password with a KeePass vault file. However that solution fell short when dealing with multiple devices like phones and multiple laptops. So I chose a password manager. I ended up choosing Bitwarden as a password manager, one of the reasons I chose this solution was that it is open source and if I ever want to host it myself I can migrate to the open source version and migration will bee seamless to a hosted solution. I still chose the cloud version of it. And at some point working on my labs I decided to use my password manager to store the secrets I need for my automations.  
Bitwarden offers a cli tool which is pretty handy for automations, it outputs secrets in json format which is pretty useful to be used in any language. I chose to keep things simple and use it on the shell and parse it with jq tool, one downside of the bitwarden CLI tool is that they don't offer a build for linux arm64 arch, so if you are running on a raspberry pi you will have to build the tool from source.  
This post is opinionated to Bitwarden, but the same concepts should work for other password managers CLI tools, off source depending on outputs a few things should be adapted to other password managers outputs.


# Logging in

Fist of all you will need an account on the Bitwarden Cloud Offering, or host Bitwarden yourself. However when working with automations you might want to enter you master password only once.  

Once having an account there are some operations related to log in, log out and keeping a sessions open so no passwords are needed for the rest of the session.  

Also something to have in mind is that Bitwarden does not store any of your passwords unencrypted on the cloud, which means that every time you log in or start a new session it downloads the full vault and decrypts it locally in your environment. If you log in but don't log out a copy of you vault is still stored locally on your machine so a good security practice is to log out whenever you are not needing access to your secrets.  

Something important is to remind that the vault is downloaded and synchronized at intervals to the the Bitwarden server, so if you recently added a secret on the phone app or desktop app, to make sure you have the latest secret its always good to synchronize the vault before pulling any secret.  

- Login
```bash
# Loging in to bitwarden
# We will fist print the help for logging in to see what options we have
$ bw login --help
Usage: bw login [options] [email] [password]

Log into a user account.

Options:
  --method <method>              Two-step login method.
  --code <code>                  Two-step login code.
  --sso                          Log in with Single-Sign On.
  --apikey                       Log in with an Api Key.
  --passwordenv <passwordenv>    Environment variable storing your password
  --passwordfile <passwordfile>  Path to a file containing your password as its first line
  --check                        Check login status.
  -h, --help                     display help for command

  Notes:

    See docs for valid `method` enum values.

    Pass `--raw` option to only return the session key.

  Examples:

    bw login
    bw login john@example.com myPassword321 --raw
    bw login john@example.com myPassword321 --method 1 --code 249213
    bw login --sso
```
As we can see there are multiple options and methods for logging in I will chose the simplest one for this example which is user and password, but if you can use 2 factor auth or SSO will make things more secure.

```bash
# Let's check first if we are not logged in
$ bw login --check 
You are not logged in.

# We can see we are not logged in so we are going to log in
# As I am doing it on an interactive terminal I will pass only the email address and password will be prompted
$ bw login francisoc@example.com
? Master password: [hidden]
You are logged in!

To unlock your vault, set your session key to the `BW_SESSION` environment variable. ex:
$ export BW_SESSION="rCyRhU35EVmU6/uNyUW2InaLZIgRUqs+95UpU+wps1GhI+UpLuwa1jqZkaOa/PbzyIO5lxfLhkbkZ1e6RU3ezQ=="
> $env:BW_SESSION="rCyRhU35EVmU6/uNyUW2InaLZIgRUqs+95UpU+wps1GhI+UpLuwa1jqZkaOa/PbzyIO5lxfLhkbkZ1e6RU3ezQ=="

You can also pass the session key to any command with the `--session` option. ex:
$ bw list items --session rCyRhU35EVmU6/uNyUW2InaLZIgRUqs+95UpU+wps1GhI+UpLuwa1jqZkaOa/PbzyIO5lxfLhkbkZ1e6RU3ezQ==

# Now let's check if we are logged in
$ bw login --check 
You are logged in!
```

In the previous snippet we logged in, however the output we have there it's not so good for automation as you need to either copy the session key into an environment variable or pass it as a flag. So we are going to do the same but using the `--raw` flag which gives us the session token in raw text to be used as we want in the automation.

```bash
# we log in and we get to stdout the session token
$ bw login francisco@example.com --raw
? Master password: [hidden]
ZuLcwNxNBHGcj9ScsyrqJ2DwqlcKMTeEuYWzN59w5N+AUz5Xi0b2ypQQS9o+5Tpa7jlez5B4sQRUm6ev4B5a7A==
```

Now we can get the session token and use it in our shell or scripts as we please. I found easy to use storing it on the `BW_SESSION` environment variable and export it in the current shell, so the session token is gone after closing the shell or it expires.

```bash
# we assign the token output to the BW_SESSION variable and export it so it's available to any child process of this shell which comes handy when running automations in terraform or ansible
$ export BW_SESSION=$(bw login francisco@example.com --raw)
? Master password: [hidden]

# now we output the variable (this is just to check, no need for this step once we know it's valueor print it)
$ echo $BW_SESSION 
oRWccGyrilYt4ov1rfGS+8Ea8Juihce+/azz83QCZHqj4vZIRGVtnGn+mishRxqvnBTELHwuVxRfrAXnecMPtg==

# Now let's check if we are logged in
$ bw login --check 
You are logged in!
```

Now you are logged in and session is saved to your shell to be used in automations without the need for password or tokens each time we call the `bw` command.  

But there is a catch. Once you are logged in, after a period of time the session expires and you need to fetch a new session token. There are multiple ways to do it and we will take a look at them in the following section.

```bash
# Check the status, since session has timed out we will find it in locked state
# all bw commends issued in locked state will ask for a password
$ bw status | jq                      
{
  "serverUrl": null,
  "lastSync": "2023-07-25T19:20:32.961Z",
  "userEmail": "francisco@example.com",
  "userId": "801A29B6-6026-4228-AD00-8DC87FD7C99E",
  "status": "locked"
}

# Order to be able to unlock we need to get a new session ID. 
# Using unlock (but we will record it on the session evironment variable)
$ export BW_SESSION=$(bw unlock --raw)                           
? Master password: [hidden]

# Then we can check the status and it will show as Unlocked
$ bw status | jq                      
{
  "serverUrl": null,
  "lastSync": "2023-07-25T19:20:35.961Z",
  "userEmail": "francisco@example.com",
  "userId": "801A29B6-6026-4228-AD00-8DC87FD7C99E",
  "status": "unlocked"
}

# NOTE, if you are using the cloud Bitwarden server URL is null otherwise if using the hosted version will show where you are logged in.
```

Now we have everything we need in order to operate on the shell logged in and unlocked to run our automations.



# Fetching secrets 

A benefit of the bw cli tool is that it outputs secret data in json format, then it can be parsed easily and manipulated any way you need it for automations needs. There is a command that is `bw list items` which will print out in json format ALL of the secrets stored in the vault. I highly discourage using this command unless you are in a safe environment because being used without care it might lead to leaking ALL secrets in the vault.  

Before fetching any secret, if the secret was recently added using a different method (phone app, Desktop app, etc) you need to issue a `bw sync` to get the latest vault.  

For this exercise I created a test user we will use from now on

```bash
# To get a specific secret we can use 'bw get item' like the following
$ bw get item 'My test login' | jq        
{
  "object": "item",
  "id": "6bbc548b-b5ef-4ec1-a883-b04a014a754a",
  "organizationId": null,
  "folderId": null,
  "type": 1,
  "reprompt": 0,
  "name": "My test login",
  "notes": "Notes on my test  secret",
  "favorite": false,
  "fields": [
    {
      "name": "APIKEY",
      "value": "MYAPIKEY12345667890",
      "type": 1,
      "linkedId": null
    },
    {
      "name": "TEST VALUE",
      "value": "NON SENSITIVE VALUE",
      "type": 0,
      "linkedId": null
    }
  ],
  "login": {
    "uris": [
      {
        "match": null,
        "uri": "https://example.com"
      }
    ],
    "username": "testuser",
    "password": "supersecretpassword1234566789",
    "totp": null,
    "passwordRevisionDate": "2023-07-25T20:07:18.461Z"
  },
  "collectionIds": [],
  "revisionDate": "2023-07-25T20:09:32.713Z",
  "creationDate": "2023-07-25T20:09:32.713Z",
  "deletedDate": null,
  "passwordHistory": [
    {
      "lastUsedDate": "2023-07-25T20:07:18.461Z",
      "password": "supersecretpassword123456"
    },
    {
      "lastUsedDate": "2023-07-25T20:06:56.359Z",
      "password": "supersecretpassword123"
    }
  ]
}
```

From the output if that command we can get several data. However I will focus on the ones that can be relevant to automation.  

- id: it's the unique identification (UUID) of the secret. This might be needed in certain use cases.

- name: That is the name of the secret we saved. Note this name is not unique so we might have duplicates here.

- type: The type of secret stored, it's a numeric value and mapping is: 1 login, 2 note, 3 credit Card , 4 Identity. We will focus on 1 and 2 as they have different fields but others can be used if needed.

- notes: This is freeform plan text field to save notes on a secret. This can be useful to store for example a public key however there are other type of secret that can be used for keys. and we will review it later on this post

- login: Is a collection of objects for the login, I will focus on the ones relevant for automations
  - username: login username 
  - password: login password
  - uris: a list of uris related to the project, might come handy for automating. Beware is a list that can hold multiple items
  - totp: 2 factor auth code. If login has 2 factor auth will be shown here. However I discourage from using totp and user/pass credentials within the same app. It's not secure

- fields: It's a list of the custom fields set up. For example if you need user and password set in the same secret as an api key. The api key can be saved in a custom field as a hidden text. Since this is freeform I will explain a bit but not entirely as it is custom data. Each custom field in the list will have their own fields. Also will focus on the ones I consider relevant
  - name: name of the field, you will use this name to identify in on the list
  - value: the value of the field
  - type: type of the field which can be 1 as hidden text, 0 plain text. I will not cover other types, but this 2 ones should cover any automation needs. If it's sensitive data use hidden and if it's non sensitive use 0, like some parameter that is needed for the automation.


I haven't found a use for automation for other of the values, but they are there to be used and there could be more uses to the data stored on the secret.  


### Warning when fetching secrets by name

When fetching a secret, if possible use the secret ID which is unique. When fetching using text between quotes like the previous example 'My test login', will search in the vault for all secrets which start with that text so there will be use cases were it will return multiple objects.  
To prove this I cloned the secret and got another one with a name 'My test login - Clone'. Then get item in this case will return and error printing the 2 objects that match that start of the text. So be careful with naming which might break automations or use IDs which are safer. Downside of using the IDs is that they are not shown in the apps so you need to manually query it first to know the ID.  

Here is an example:

```bash
# this query will result in more then one object matching
$ bw get item 'My test login'                          
More than one result was found. Try getting a specific object by `id` instead. The following objects were found:
6bbc548b-b5ef-4ec1-a883-b04a014a754a
915a56a8-be14-4636-a220-b04a014c3680

# If we get both items by id parsing the name with jq
$ bw get item 6bbc548b-b5ef-4ec1-a883-b04a014a754a | jq -r .name
My test login

$ bw get item 915a56a8-be14-4636-a220-b04a014c3680 | jq -r .name
My test login - Clone
```

As we can see in the example both secrets start with the same string so that is why the error was printed since fetching by name is very limited.



# Parsing a secret

When getting a secret depending on the info you want to get, there are different ways to fetch the data. Let's look at `bw get` documentation

```bash
$ bw get --help
Usage: bw get [options] <object> <id>

Get an object from the vault.

Arguments:
  object                             Valid objects are: item, username, password, uri, totp, notes, exposed, attachment, folder, collection, org-collection, organization, template, fingerprint, send
  id                                 Search term or object's globally unique `id`.

Options:
  --itemid <itemid>                  Attachment's item id.
  --output <output>                  Output directory or filename for attachment.
  --organizationid <organizationid>  Organization id for an organization object.
  -h, --help                         display help for command

  If raw output is specified and no output filename or directory is given for
  an attachment query, the attachment content is written to stdout.

  Examples:

    bw get item 99ee88d2-6046-4ea7-92c2-acac464b1412
    bw get password https://google.com
    bw get totp google.com
    bw get notes google.com
    bw get exposed yahoo.com
    bw get attachment b857igwl1dzrs2 --itemid 99ee88d2-6046-4ea7-92c2-acac464b1412 --output ./photo.jpg
    bw get attachment photo.jpg --itemid 99ee88d2-6046-4ea7-92c2-acac464b1412 --raw
    bw get folder email
    bw get template folder
```

From the documentation we have a couple of objects that can be fetched directly without any parsing of json, which make our lives much easier.  
Objects that can be fetched that I found relevant for automations are username, password, uri, totp, and notes.  
We will see a couple of examples here.

```bash
# Get username
$ bw get username 6bbc548b-b5ef-4ec1-a883-b04a014a754a
testuser

# Get password
$ bw get password 6bbc548b-b5ef-4ec1-a883-b04a014a754a
supersecretpassword1234566789

# get URI. NOTE: of there is more than 1 URI it will return the 1st one
# If there are more and there is a need to fetch all of them get item needs to be used
# and parse the json output
$ bw get uri 6bbc548b-b5ef-4ec1-a883-b04a014a754a
https://example.com

# get the notes of the secret
$ bw get notes 6bbc548b-b5ef-4ec1-a883-b04a014a754a
Notes on my test  secret


# Also you can fetch the cretendials to environment variables.
export MY_USERNAME=$(bw get username 6bbc548b-b5ef-4ec1-a883-b04a014a754a)
export MY_PASSWORD=$(bw get password 6bbc548b-b5ef-4ec1-a883-b04a014a754a)
export MY_URI=$(bw get uri 6bbc548b-b5ef-4ec1-a883-b04a014a754a)
export MY_notes=$(bw get NOTES 6bbc548b-b5ef-4ec1-a883-b04a014a754a)
```

### Getting custom information

The information we fetched so far is very straightforward; however, when using custom fields or  we need to parse more than 1 URI. We can use the `bw get item` which returns all the information for a secret in json format, and then We will be parsing it with jq tool.  
If we look at the previous example

```bash
# To get a specific secret we can use 'bw get item' like the following
$ bw get item 'My test login' | jq        
{
  "object": "item",
  "id": "6bbc548b-b5ef-4ec1-a883-b04a014a754a",
  "organizationId": null,
  "folderId": null,
  "type": 1,
  "reprompt": 0,
  "name": "My test login",
  "notes": "Notes on my test  secret",
  "favorite": false,
  "fields": [
    {
      "name": "APIKEY",
      "value": "MYAPIKEY12345667890",
      "type": 1,
      "linkedId": null
    },
    {
      "name": "TEST VALUE",
      "value": "NON SENSITIVE VALUE",
      "type": 0,
      "linkedId": null
    }
  ],
  "login": {
    "uris": [
      {
        "match": null,
        "uri": "https://example.com"
      }
    ],
    "username": "testuser",
    "password": "supersecretpassword1234566789",
    "totp": null,
    "passwordRevisionDate": "2023-07-25T20:07:18.461Z"
  },
  "collectionIds": [],
  "revisionDate": "2023-07-25T20:09:32.713Z",
  "creationDate": "2023-07-25T20:09:32.713Z",
  "deletedDate": null,
  "passwordHistory": [
    {
      "lastUsedDate": "2023-07-25T20:07:18.461Z",
      "password": "supersecretpassword123456"
    },
    {
      "lastUsedDate": "2023-07-25T20:06:56.359Z",
      "password": "supersecretpassword123"
    }
  ]
}
```

We can get the information in the following fashion looking at the fields we have 2 custom fields with names "APIKEY" and  "TEST VALUE". In order to successfully get those values we can get them using some jq tricks.

```bash

# get the field with name APIKEY
$ bw get item 'My test login' | jq -r '.fields[] | select (.name == "APIKEY") .value'
MYAPIKEY12345667890

# get the field with name TEST VALUE
$ bw get item 'My test login' | jq -r '.fields[] | select (.name == "TEST VALUE") .value'
NON SENSITIVE VALUE


# Assign those values to environment variables to be used further in automations
export MY_APIKEY=$(bw get item 'My test login' | jq -r '.fields[] | select (.name == "APIKEY") .value')
export MY_PARAMETER=$(bw get item 'My test login' | jq -r '.fields[] | select (.name == "TEST VALUE") .value')
```

### Getting text from NOTE type

Even though we can use login type and the notes field for some use cases, using the secure note type fits better the use case for secrets that require a big text field. A good use case is for storing Private and Public Keys for x509 keys or certificate chains, etc. Personally I use it to keep my SSH private keys (in case I loose my laptop) and also for storing SSH Pub Keys, which are not sensitive secrets but I fetch them from the same place for automation purposes.  

Not all object types support fetching the same data as the login object so we will be using `bw get item` and parse the json object for those use cases.


```bash
# fetch an object of type SECURE NOTE
{
  "object": "item",
  "id": "F59847A4-06EE-4791-B063-F762A0622B90",
  "organizationId": null,
  "folderId": "5716E44F-08A1-4745-92E5-30F1DDECE38A",
  "type": 2,
  "reprompt": 0,
  "name": "SSH Pub Key francisco@test",
  "notes": "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQCmoMcP8PIzEVANAIEwXd0B9RBIq8Zj+XrcoVlIsTph1it1iWnRQm6GYFd+s/TkqT6kfYHGOa2tGCOM5N10co0NP1sHrPVsWFZaY3tGU1+5nrLb09csk/XjByNjpp045iRVR4MKtD7lFhHHfhdB3Y7vYp+VqVIIWesmmomQwTg10TcHE+QWM44CmUwk25D3SsEFRVt3h2mlUysqlRrakzEhz8OxXOSkKuTDGMsyR5K4TTlFp0gB3pNYmbDbcKdKcrm4bOHGdoMieoqcGPF9SBO8V/tP/Pl4bzxtYNjf8sew8HNWHlmLJ5Yxwjsmed1P2nCY2Bsu1/idVgTV6fgS1lGiqAO+angKSPkcroUF8tKA8ti+U8kqnWQybur8MHJHezruv31VXeoM9BGnZ+kARTcsTwV7BIEXTjBXWHR6oIgjUK9FDc3zALM/nEZqvysD+Ybf2C1dMpjx+dnxndyiomzYutuP0C4497VioHZmLQKMvdevVRrlTUXUobbmaC65xLc= francisco@test",
  "favorite": false,
  "secureNote": {
    "type": 0
  },
  "collectionIds": [],
  "revisionDate": "2022-10-25T20:25:24.376Z",
  "creationDate": "2022-10-25T20:25:24.376Z",
  "deletedDate": null
}
```

As we can see these type of objects have less information and the main piece of information needed is the notes field. Fortunately the notes field is on the root of the json object so it is easy to parse.

```bash

# Get the public SSH key from a secure note
$ bw get item 'SSH Pub Key francisco@test' | jq -r .notes
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQCmoMcP8PIzEVANAIEwXd0B9RBIq8Zj+XrcoVlIsTph1it1iWnRQm6GYFd+s/TkqT6kfYHGOa2tGCOM5N10co0NP1sHrPVsWFZaY3tGU1+5nrLb09csk/XjByNjpp045iRVR4MKtD7lFhHHfhdB3Y7vYp+VqVIIWesmmomQwTg10TcHE+QWM44CmUwk25D3SsEFRVt3h2mlUysqlRrakzEhz8OxXOSkKuTDGMsyR5K4TTlFp0gB3pNYmbDbcKdKcrm4bOHGdoMieoqcGPF9SBO8V/tP/Pl4bzxtYNjf8sew8HNWHlmLJ5Yxwjsmed1P2nCY2Bsu1/idVgTV6fgS1lGiqAO+angKSPkcroUF8tKA8ti+U8kqnWQybur8MHJHezruv31VXeoM9BGnZ+kARTcsTwV7BIEXTjBXWHR6oIgjUK9FDc3zALM/nEZqvysD+Ybf2C1dMpjx+dnxndyiomzYutuP0C4497VioHZmLQKMvdevVRrlTUXUobbmaC65xLc= francisco@test

# Assign it to an environment variable

export SSH_PUB_KEY=$(bw get item 'SSH Pub Key francisco@test' | jq -r .notes)

```


# Common Use Cases

When working with automation tools like terraform, pulumi or ansible. Or using the cloud providers CLI tools like aws, google azure, digital ocean, vultr, etc. Fetching the credentials to a special environment variable its all you need for access. This is a somewhat safer than saving the credentials to a local file. As the environment variables scope is to the shell they are running, they are stored in memory and when the shell process is done the variables disappear.  
Most of this tools can read credentials in their credentials chain from environment variables.  

Some of the credentials that I use the most are to create resources in the cloud. By doing so either using terraform or pulumi. The providers for the cloud providers always have a way to setting credentials with environment variables.  
Following I will list some of the ones I use the most but just examples how this approach can be used.  

When dealing with api keys I tend to use the password field to store the api keys in a login object, which is easy to fetch

### Getting credentials in ansible task.
An example is using the lookup function to get env vars into ansible. We will take the SSH Pub key from previous example


```bash
export SSH_PUB_KEY=$(bw get item 'SSH Pub Key francisco@test' | jq -r .notes)
```

Then use in ansible
```yaml
- name: Print SSH Pub Key
  ansible.builtin.debug:
    msg: "'{{ lookup('ansible.builtin.env', 'SSH_PUB_KEY') }}'"
```


### Using environment variables in terraform

Using environment variables for credentials will depend on the terraform providers being used so I will go through the different ones I use for setting cloud services credentials

#### AWS provider
Most AWS services will not use just an API key but will use Access key ID and Secret Access key, AWS credential chain will fetch them from AWS_ACCESS_KEY_ID and AWS_SECRET_ACCESS_KEY respectively. I follow the practice to store the key id in the username field which is non sensitive and the secret access key in the password field.  
Fetching credentials for AWS stored in a secret would look like.

```bash
export AWS_ACCESS_KEY_ID=$(bw get item 'my AWS credential' | jq -r .login.username)
export AWS_SECRET_ACCESS_KEY=$(bw get item 'my AWS credential' | jq -r .login.password)
```
Then the terraform provider or the aws cli will pick it up. So making it safe as no one sees the credentials unless you print them from the environment.


#### Cloudflare provider

 Similar to AWS but the only use CLOUDFLARE_API_TOKEN variable for the provider. So we can store that api token in the password field of a secret and the provider will pick it up.

 Example:
 ```bash
 export CLOUDFLARE_API_TOKEN=$(bw get item 'my cloudflare api token'| jq -r .login.password)
 ```


 #### Other providers

 Similar to cloudflare, other providers like digital ocean, Google Cloud, vultr, etc would use an env or set of env variables to get credentials. Please refer to the providers documentation for the format of env vars to be stored in the secret.

 - Digital Ocean env vars: `DIGITALOCEAN_TOKEN` or `DIGITALOCEAN_ACCESS_TOKEN` 
 - Google Cloud:  `GOOGLE_CREDENTIALS`, `GOOGLE_CLOUD_KEYFILE_JSON`, `GCLOUD_KEYFILE_JSON`
 - vultr: `VULTR_API_KEY`



# Example
After explaining multiple ways of getting credentials from password manager for automating home labs secrets I will show an example.  
Context: The following could be used in a terraform automation for creating a VM in vultr cloud provider and setting DNS records hosted in Cloudflare and using an AWS S3 or compatible backend to store terraform state. For this use case we would need vulr, cloudflare and S3 credentials. DISCLAIMED terraform code is out of the scope of this blog post and will probably be written in a future blog post. Also credential names are invented for the example purposes.  
Getting credentials for the above use case. This could be done on the shell or scripted for making it easier to manage

```bash
# Loging to BW to get Session token
export BW_SESSION=$(bw login francisco@example.com --raw)
# If already logged in unlock to get the session token
export BW_SESSION=$(bw unlock --raw)

# Get the cloud provider api key
export VULTR_API_KEY=$(bw get item 'vultr api key'| jq -r .login.password)

# Get S3 storage Backendn keys
export AWS_ACCESS_KEY_ID=$(bw get item 'S3 terraform key' | jq -r .login.username)
export AWS_SECRET_ACCESS_KEY=$(bw get item 'S3 terraform key' | jq -r .login.password)

# Get cloudflare api token
export CLOUDFLARE_API_TOKEN=$(bw get item 'Cloudflare API token'| jq -r .login.password)

```

After setting all environment variables we would be all set for running the terraform stack described in the context of the example.


# Conclusion

Using a password manager can be handy for those hobbyists who run home labs or cloud resources for personal purposes. In my case I was already using Bitwarden Password manager for personal secrets management so I integrated to my personal projects, without incurring in extra costs or setting up something else which requires maintenance.  
A password manager could be used in an enterprise environment for automation purposes using shared secrets among users, however the level of control over the secrets probably is not the best fit as there are better solutions for enterprise environments like AWS Secrets manager, Google cloud Secret manager, Hashicorp vault in either enterprise version or open source, or Azure Key Vault. These solutions offer features better suited for enterprise use and finer grained control over secrets.  
Back to personal use I am very happy with the solution, as I used to copy the passwords or a hardcoded files for environment variables or just kept them hardcoded in the code. This solutions allows me to keep the secrets safe outside of the code and files and it's ephemeral to runtime whenever used.