# Nexus 3 with Self Signed Cert

This was done on a MacOS laptop, but the process should work the same with most Linux distros.

## 1. Download and untar.

* Download the Nexus 3 tar file.  Move it to a directory of your choice and untar it. This should produce two new directories:  `nexus-3-<version>` and `sonatype-work`
* Rename 'nexus-3-<version>` to `nexus3` for simplicity.
* Create the following variables to keep the next few steps organized:
    * Determine your current directory: `pwd`
    * `install_dir=<current dir>/nexus3`
    * `data_dir=<current dir>/sonatype-work/nexus3`
    
## 2. Start Nexus 3

You need to start Nexus in order for a number of default files and dirs to be created in the "data" directory.

`$install_dir/bin/nexus run`

Once Nexus finishes starting, you can try to access it at port 8081, or just stop it with `ctrl+c`.

## 3. Generate a Self-Signed certificate.

For my test, I added an entry to my `hosts` file to that `mynexus.ca` would resolve to my own computer.
If you already have DNS configured, you won't need to do this, simply use the DNS address you have setup for your machine.

* Go to `$data_dir/etc/ssl`.  If this folder doesn't yet exist, create it.
* Run the following command, but be sure to change the `alias`, `CN`, and `DNS` entries to match your domain name.  Leave the password as `password`.

```
keytool -genkeypair -keystore keystore.jks -storepass password -alias mynexus.ca \
 -keyalg RSA -keysize 2048 -validity 5000 -keypass password \
 -dname 'CN=*.mynexus.ca, OU=Sonatype, O=Sonatype, L=Unspecified, ST=Unspecified, C=US' \
 -ext 'SAN=DNS:mynexus.ca,DNS:www.mynexus.ca'
```

This will create a new file named `keystore.jks` in your ssl directory.

## 4. Configure Nexus 3 for SSL

Edit the file `nexus.properties` found in `$data_dir/etc`.

You can keep all of the existing properties commented out and append the following properties to the file:

```
application-port-ssl=8443
nexus-args=${jetty.etc}/jetty.xml,${jetty.etc}/jetty-http.xml,${jetty.etc}/jetty-https.xml,${jetty.etc}/jetty-requestlog.xml
ssl.etc=${karaf.data}/etc/ssl
```

Next, edit the file `$install_dir/etc/jetty/jetty-https.xml`

Add a `certAlias` element to the `sslContextFactory` block.  Make sure you use the alias you used when you created your keystore.  The config will look like this:

```
...
  <New id="sslContextFactory" class="org.eclipse.jetty.util.ssl.SslContextFactory$Server">
    <Set name="certAlias">mynexus.ca</Set>
    <Set name="KeyStorePath"><Property name="ssl.etc"/>/keystore.jks</Set>
...
```

## Start Nexus

Start Nexus again.  This time, when Nexus starts it will also be available at:
`https://<your domain>:8443`

Remember to use `https` and add port `8443`.  Since it is a self-signed cert, you will have to accept the warning if using FireFox, or type "thisisunsafe" to view Nexus with Chrome.  See [see this reference here](https://medium.com/@dblazeski/chrome-bypass-net-err-cert-invalid-for-development-daefae43eb12).

If you `curl` Nexus, you will have to add `--insecure` due to the self-signed cert.
