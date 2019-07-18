# How to "AWS Creds"
###### Be secure, dummies

## Principles

1. Use AWS CLI credentials with MFA
2. Use a credentials file to make programmatic access easier
3. Use the helper functions to keep things simple across linux and windows
4. Rotate your access keys regularly


## Configure tools and functions
The following instructions are for _**linux**_, but include some features that allow the generation of temporary credentials within windows as well.


### Install tlx
[TLX](https://pypi.org/project/tlx/) is a really basic python wrapper around boto3 sessions with a cli command.

`pip install tlx`


### Create your AWS Credentials file
Create/edit your credentials file: `vim ~/.aws/credentials`


#### ~/.aws/credentials
```
    [default]
    aws_access_key_id = <access_key>
    aws_secret_access_key = <secret_key>

    [account_name_dev]
    source_profile = default
    role_arn = arn:aws:iam::xxxxxxxxxxxx:role/<role_name>
    mfa_serial = arn:aws:iam::xxxxxxxxxxxx:mfa/<iam_user_name>

    [account_name_prod]
    source_profile = default
    role_arn = arn:aws:iam::xxxxxxxxxxxx:role/<role_name>
    mfa_serial = arn:aws:iam::xxxxxxxxxxxx:mfa/<iam_user_name>
```

### Install AWS IAM Key Rotator
Clone [Rhyeals key rotator](https://github.com/rhyeal/aws-rotate-iam-keys), and symlink it into `/usr/bin`.
```
    git clone https://github.com/rhyeal/aws-rotate-iam-keys.git
    sudo ln -s /home/<user>/path/to/repo/aws-rotate-iam-keys/src/bin/aws-rotate-iam-keys /usr/bin/aws-rotate-iam-keys
```


### Create Helper functions
Add the helper functions to your `.bashrc` (or `.zshrc` etc.)

#### ~/.bashrc
```
# Funcs
  gac_for_win()
      {
          get-aws-creds -p $@ | sed -r 's/export /$env:/; s/=(.*)/="\1"/g; s/unset (.*)/$env:\1=""/'
      }

  gac ()
      {
          for i in "$( get-aws-creds -p $@ --quiet)";
          do
              eval "${i}";
          done
      }
```
Refresh your shell with: `source ~/.bashrc`


## Get Credentials
### TLX
To get credentials manually using tlx, run this command, and enter your MFA token when prompted.
```
    user@host:~> get-aws-creds -p account_name_dev
    Enter MFA code for arn:aws:iam::xxxxxxxxxxxx:mfa/user.name:
    Keys and token for profile: 'account_name_dev'
    Paste the following into your shell:

    export AWS_ACCESS_KEY_ID=<access_key>
    export AWS_SECRET_ACCESS_KEY=<secret_key>
    export AWS_SESSION_TOKEN=<session_token>
    user@host:~> _
```

### Automatically set the credentials variables using the `gac ()` helper
```
    user@host:~> gac account_name_dev
    Enter MFA code for arn:aws:iam::xxxxxxxxxxxx:mfa/user.name:
    user@host:~> _
```
The variables are set automatically, and no feedback is printed unless there is an error.

### Export a set of variables for use on windows powershell with `gac_for_win ()`
```
    user@host:~> gac_for_win account_name_dev
    Enter MFA code for arn:aws:iam::xxxxxxxxxxxx:mfa/user.name:
    Keys and token for profile: 'account_name_dev'
    Paste the following into your shell:

    $env:AWS_ACCESS_KEY_ID="<access_key>" 
    $env:AWS_SECRET_ACCESS_KEY="<secret_key>" 
    $env:AWS_SESSION_TOKEN="<session_token>"
    user@host:~> _
```
These statements can be copied out of your linux session, and into a powershell window (e.g. VSCode terminal) to set credentials.

## Rotate AWS Access Keys
```
    user@host:~> aws-rotate-iam-keys --profile default
    Making new access key
    Updating profile: default
    Made new key <access_key>
    Key rotated
    user@host:~> _
```
**TODO:** Find a `cron` solution that will queue up a command to run when the computer is next awake, otherwise your linux session needs to be always running.


## AWS Credential Helper for CodeCommit
The AWS CLI for CodeCommit includes a [credential helper](https://docs.aws.amazon.com/codecommit/latest/userguide/setting-up-https-unixes.html) that wraps git calls and supports MFA. It requires the credentials file to be set up (which we have done above). The helper is smart enough to understand a codecommit remote, vs say GitHub and behaves appropriately.
It will prompt for an MFA token if required.

This command will setup the helper for a single repo. To apply the helper across the board, use `--global`

#### Setting credential.helper
```
    cd /path/to/repo/
    git config --local credential.helper '!aws --profile account_name_dev codecommit credential-helper $@'
```

#### Switching credential.helper profiles
A handy trick is to use the back-search of your bash shell to find the command, and then edit the profile name.
```
    user@host~/repo> <ctrl-R>
    user@host~/repo> git config --local credential.helper '!aws --profile account_profile_prod codecommit credential-helper $@'
    bck-i-search: credential.he_
```

**TODO:** Credential helper...helper. I haven't yet figured out how to get it to apply only to a specific remote. It applies to all configured remotes.

e.g. You could add two CodeCommit remotes, in different accounts that require a different IAM profile to access. Using the approach above, you'll have to switch the credential helper to use one profile, and `git push remote1`, then switch it again to the other profile, and `git push remote2`.
