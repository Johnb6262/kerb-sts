# AWS KERBEROS STS
Based on the ADFS-CLI script [originally posted by Quint Van Deman](https://blogs.aws.amazon.com/security/post/Tx1LDN0UBGJJ26Q/How-to-Implement-Federated-API-and-CLI-Access-Using-SAML-2-0-and-AD-FS).

## Overview
This script provides a seamless mechanism for federating the AWS CLI. When properly configured, this script allows a user to get a short lived (1 hour) set of credentials for each authorized role.

The script leverages Kerberos and a [SAML-compatible IdP](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_create_saml.html) to avoid any need for the user to enter
an AD domain password, or provide AWS credentials. However, users can also
authenticate using NTLM with their username and password or with a Kerberos keytab.

> See [our security policy](SECURITY.md) for handling of security-related matters.

## Configuration
Kerb-STS looks for configuration in the ~/.kerb-sts/config.json file. This file contains the following fields: 

Field | Required? | Description
:--- |:--- |:---
idp_url | Yes | URL where the SAML authentication requests are sent
adfs_url | No | **deprecated** URL where the SAML authentication requests are sent
region | Yes | Region for AWS credentials
kerb_domain | No | Domain name used for the Kerberos GSS exchange. This is set to the domain name of `idp_url` by default
preferred_auth_type | No | Type of authentication to use when a username and password is submitted. Acceptable values: `ntlm` (default), `kerberos`

Users can generate this file with Kerb-STS:
```
kerb-sts --configure
```
This will prompt the user for those values and then serialize the configuration. Users
can also manually create the configuration file. A sample for AD FS is demonstrated below:
```
{
  "region": "us-east-1",
  "idp_url": "https://sample.domain.com/adfs/ls/IdpInitiatedSignOn.aspx?loginToRp=urn:amazon:webservices"
}
```
Users can override either of the configured values on the command line.

## Installation
* *Note: Python 2.7.10 is the minimal version supported*
* *Note: All platforms have been tested with both Python 2.7 and 3.5*

### OSX
* *Note: If you are using El Capitan or Sierra, refer to the subsequent OSX section*

0. sudo easy_install pip
1. sudo pip install kerb-sts

### OSX - El Capitan
* *Note: El Capitan forces the version of some modules which directly interfere with kerb-sts. In order to
get it to work users need to either use a version of Python that was not included with the OS or need
to follow these instructions which leverage virtual environments.*

0. sudo easy_install pip virtualenv
1. virtualenv ~/kerb-sts
2. source ~/kerb-sts/bin/activate
3. sudo pip install kerb-sts --ignore-installed six
4. deactivate
5. sudo ln -s ~/kerb-sts/bin/kerb-sts /usr/local/bin/kerb-sts

### MacOS Sierra
0. You will need to update your version of Python to 2.7.12+; Homebrew is the easiest method.
1. You will also need to install/update the XCODE Development Extensions
  1a. sudo xcode-select install
2. You can then just run sudo pip install kerb-sts

### Windows
0. Install [Python](https://www.python.org/downloads/)
1. Ensure python and python/scripts are on the PATH
2. Install pywin32 from [mhammond/pywin32](https://github.com/mhammond/pywin32/releases). Follow the instructions to ensure you get the correct version.
3. pip install kerb-sts

### Ubuntu
0. sudo apt-get update
1. sudo apt-get install -y krb5-kdc libkrb5-dev python-setuptools python3-pip
2. sudo pip install kerb-sts

## Usage
If the install went smoothly `kerb-sts` should be on your path. There are a lot of configuration options.
The best way to discover them is to check out the help statement.
```
kerb-sts --help
```

#### Default Role
The script allows users to specify an AWS IAM role that will be set as the default IAM role in
the credentials file.
```
kerb-sts -r [iam-role-to-assume]
```
All subsequent AWS CLI commands will use this role by default.

Additionally, all available roles will be added as named profiles to the credentials file.
Users can then leverage the default role or use the AWS_DEFAULT_PROFILE environment variable to
select a specific role/profile. You can find more information about the credentials file
in the [AWS Documentation](http://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-started.html#cli-config-files).

#### Only Role
If you have a lot of roles available, it may take a while to process all available roles.
In this case, you can use the `-o` option and only the specified role will be processed.
It will also be treated as the default role (`-r` option).

#### Daemon
By passing in a `--daemon` flag, the script will continue running and update the credentials file every
half hour. The refresh time can be set with the `--refresh` argument, but remember
the tokens only last for one hour.
```
kerb-sts -r iam-role-to-assume --daemon
```

#### Authenticating with Username/Password
```
kerb-sts -u username -p 'password' -d DOMAIN
```
Based on the value of `preferred_auth_type` in your config file, this call with authenticate using NTLM, or will create a kerberos ticket for the inputted user and authenticate.

#### Keytab
This script allows users to generate Kerberos tokens with Kerberos keytabs. Keytabs
are private key files that are signed with the user's name, domain, and password.
You can generate a keytab by running:
```
ktutil -k username.keytab add -p username@DOMAIN.COM -e arcfour-hmac-md5 -V 1
```
Users can use the keytab to authenticate with Kerberos by running:
```
kinit -kt username.keytab username@DOMAIN.COM
```
Keytabs allow users to authenticate without their password. The keytab is signed with the password however, so
when a password is updated the keytab must likewise be updated.
They can then be used with kerb-sts to generate temporary tokens:
```
kerb-sts --key username.keytab -u username -d DOMAIN.COM
```

#### Credential File
The default location for the AWS credentials file is ~/.aws/credentials. Users are also able to specify
a different location for the credentials generated.
```
kerb-sts -c ./aws-credentials
```

## Troubleshooting
#### Kerberos
If you are having issues authenticating with Kerberos, make sure you can run `kinit`. This should prompt you for
your password and then login successfully. You can view your current Kerberos tickets with `klist`. If you want to
ensure Kerberos is working properly you can delete all of your tickets with `kdestroy -A` and then try to get another
ticket issued by running `kinit`.

## Building a Distribution
### Python
The easiest way to install and distribute kerb-sts is using a wheel.
A distribution can be built by running:
```
python setup.py bdist_wheel
```
That should output a .whl file in the dist directory which can be installed with pip.

### Windows EXE
Kerb-STS can also be built into a standalone executable with Python bundled to ease installation.
```
python setup.py install
pip install pyinstaller
pyinstaller --onfile kerb_sts/__main__.py
```
This will produce a dist/\_\_main__.exe which can then be renamed/run as a standalone exe.

## Development
The recommended way to install locally from source is to use a virtual environment. From the root
of the kerb-sts source code directory run:

0. pip install virtualenv
1. virtualenv venv
2. source venv/bin/activate
3. python setup.py install
4. python kerb_sts/
