# Encrypting Secrets using WSO2 Secure Vault

WSO2 Micro Integrator can use secrets with static origins as well as dynamic origins in configurations. This applies to secrets in server-level configurations as well as configurations within integration solutions (synapse configurations).

It is not recommended to use plain-text values for your sensitive data in your configurations. Therefore, you can encrypt the plain-text secrets using the secure vault implementation that is built in to the Micro Integrator. 

The Micro Integrator is shipped with the **Cipher Tool**, which uses the secure vault implementation and encrypts the secrets you specify. Running the **Cipher Tool** also ensures that the secrets are enabled in the Micro Integrator environment. You can then refer the encrypted secrets from anywhere in your server configurations or synapse configurations.

Note that you can customize the default secure vault configurations in the product by implementing a new secret repository, call back handler, etc.

Follow the steps given below to encrypt secrets.

## Step 1: Defining secrets

### Static secrets

Static secrets are sensitive data that are specified directly in configurations. The secret can be a plain-text value or the alias of an encrypted value.

You must first list the plain-text secrets in the `deployment.toml` file under the `secrets` header. Note that we have specified an <b>alias</b> for the secret followed by the actual plain-text secret.

```toml
[secrets]
server_secret = "[secret_1]"
synapse_secret = "[secret_2]"
```

### Dynamic secrets

Dynamic secrets are specified in configurations as environment variables, system properties, Docker secrets, or Kubernetes secrets. The actual secrets is then encrypted using the WSO2 API Controller, **apictl** and injected to the environment.

1.  First, list the dynamic secrets in the `deployment.toml` file under the `[secrets]` section. However, unlike for static secrets, specify the secret value as an environment variable or system property.

    !!! Note
        In this example, `dynamic_secret` is a placeholder for the secret. You will use this placeholder as the secret's alias when you encrypt the plain-text secret using the apictl (in the next step).

    === "Environment Variable"
        ```toml
        [secrets]
        server_secret = "$env{dynamic_secret}"
        ```

    === "System Property"
        ```toml
        [secrets]
        server_secret = "$sys{dynamic_secret}"
        ``` 

2.  Now, encrypt a plain-text secret for the `dynamic_secret` alias by using the WSO2 API Controller. For more information, see [Encrypting Secrets with CTL]({{base_path}}/install-and-setup/setup/api-controller/encrypting-secrets-with-ctl)

## Step 2: Running the Cipher Tool

Running the Cipher Tool will first encrypt any [static secrets](#static-secrets) defined in the `[secrets]` section, and then enable all the secrets (static as well as dynamic) in the environment.

### In a VM environment

In a <b>VM environment</b>, you need to manually run the Cipher Tool as follows:

!!! Tip
    If you are using **Windows**, you need to have [Ant](http://ant.apache.org/) installed before using the Cipher Tool.

1.  Open a terminal, navigate to the `<MI_HOME>/bin/` directory.
2.  Execute the `-Dconfigure` command with the cipher tool script as shown below.

    === "On Linux"
        ```bash
        ./ciphertool.sh -Dconfigure
        ```

    === "On Windows"
        ```bash
        ./ciphertool.bat -Dconfigure
        ```

### In a Kubernetes environment

In a <b>Kubernetes environment</b>, you don't need to manually run the Cipher tool. Follow the steps given below.

1. Open your Integration Project in WSO2 Integration Studio, which contains all the integration artifacts and the Kubernetes Exporter.
2. Open the `pom.xml` of the Kubernetes Exporter module and select the <b>Enable Cipher Tool</b> check box as show below.

    <img src="{{base_path}}/assets/img/integrate/k8s_deployment/enable-cipher-tool-in-k8s.png">

3.  When you build the Docker image from your Kubernetes exporter, the secrets will get encrypted and enabled in the environment.

If you specified any [static secrets](#static-secrets), go back to the `deployment.toml` file and see that the secrets are encrypted.

```toml
[secrets]
keystore_password = "encrypted_pass_3"
key_password = "encrypted_pass_4"
truststore_password = "encrypted_pass_5"
```

## Step 3: Accessing secrets

### In server configurations

You can refer an encrypted secret in your server configurations by using the `$secret{alias}` function in place of the plain-text secret as shown below. Replace `alias` with the secret's alias that is defined under the `[secrets]` section.

!!! Note
    You can use encrypted [static secrets](#static-secrets) as well as [dynamic secrets](#dynamic-secrets).

```toml
[keystore.primary]
password = "$secret{server_secret}"
alias = "$secret{server_secret}"
key_password = "$secret{server_secret}"  

[truststore]                  
password = "$secret{server_secret}"
```

### In synapse configurations

You can refer an encrypted secret in your synapse configurations by using the **vault lookup** function (`{wso2:vault-lookup('alias')`) in place of the plain-text secret as shown below. Replace `alias` with the secret's alias that is defined under the `[secrets]` section.

!!! Note
    You can only use encrypted [static secrets](#static-secrets) here.

```xml
<log level="custom">
  <property expression="wso2:vault-lookup('synapse_secret')" name="secret"/>
</log>
```

## Step 4: Populating dynamic secrets

If you have defined [dynamic secrets in your configurations](#dynamic-secrets), you must populate the secret into the relevant environment as required. 

### In a VM environment

For an instance, in the case of environment variables, you can populate them with the export command as follows:

```bash
export env_carbon_sec=<ENCRYPTED_VALUE>
```

### In a Kubernetes environment

If you are in a Kubernetes enviroment, you should have generated a .yaml file with the encrypted secrets using the WSO2 API Controller, **apictl**. See [defining dynamic secrets](#dynamic-secrets).

### Start server as a background job

If you start the Micro Integrator as a background job, you will not be able to provide password values on the command line. Therefore, you must start the server in "daemon" mode as explained below.

1. Create a new file in the MI_HOME directory. The file should be named according to your OS as explained below.

    * For Linux: The file name should be `password-tmp`.
    * For Windows: The file name should be `password-tmp.txt`.

    !!! Note
        When you start the server, the keystore password will be picked from this new file. Note that this file is automatically deleted from the file system after the server starts. Therefore, the admin has to create a new text file every time the server starts.

        Alternatively, if you want to retain the password file after the server starts, the file should be named as follows:

        * For Linux: The file name should be `password-persist`
        * For Windows: The file name should be `password-persist.txt`

2. Add the primary keystore password (which is "wso2carbon" by default) to the new file and save. By default, the password provider assumes that both private key and keystore passwords are the same. If not, the private key password must be entered in the second line of the file.

3. Now, start the server as a background process by running the following command.
   ```bash
   ./micro-integrator.sh start
   ```