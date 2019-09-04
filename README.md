# crypto-masterkey-keystore

This repository provides an inexpensive means to protect cryptographic master keys (key encrypting keys, KEKs) in a way that is resistant to some of the most common remote file exfiltration attacks.

## Contents
[Guidance for generating and managing application Master Keys](#guidance)  
[Custom KeyStore implementations for Master Keys](#custom-keystores)  
[Sample code for generating and managing application master keys](#samples)



## Guidance
First, a few definitions:
<dl>
  <dt>login passwords</dt>
  <dd>A password or passphrase used by a <b>human</b> to access a front-end system. Store as a randomly-salted hash.</dd>
  <dt>credentials</dt>
  <dd>A username and password pair used by a <b>system</b> to access other resources. Must be encrypted.</dd>
  <dt>Data Encrypting Key (DEK)</dt>
  <dd>A cryptographic key used to encrypt data. This data may be anything, including cryptographic keys.</dd>
  <dt>Key Encrypting Key (KEK) or "Master Key"<dt>
  <dd>The top-level cryptographic key used to protect a Data Encrypting Key</dd>
</dl>

The guidance given here deals primarily with __credentials__ rather than with ordinary user login passwords.

1. Cryptographic keys (including TLS certificates) and passwords should be *unique* for each appliance instance and for each on-premises installation.  If multiple customers can share the same appliance or server, each must have a unique set of keys and certificates.  Otherwise, every legitimate customer will have access to certificates/keys that allow her to spy on all other customers.

1. Keys, certs, and passwords should be established at installation time (first bootup for appliances).  Either ask the installer to specify a password (for example, for an admin account password), or generate them dynamically (keystore passwords, cryptography keys, master key, etc.).  There should be no default passwords.  Customers can easily forget or neglect to change them.

1. Keys and credentials should not be stored in clear text in any configuration files or source code files.  If these must be stored, they must be __encrypted__.  Obfuscating or encoding the data alone (by XORing with some static byte array, or Base64 encoding, for example) is _not_ enough. Keys and certificates should be stored in a key store, protected by a long random passphrase or encrypted by a strong randomly generated encryption key.

1. These random keys and passphrases should be generated dynamically at installation time, and they should, themselves, be protected by a randomly generated key.  This is often termed the 'master key' because it is used to secure all other cryptographic keys and passwords.

1. Master keys should be generated using a key stretching algorithm like HKDF from data gathered from multiple sources.  At least one of these sources may be tied to the hardware on which the system is running.  Best practices would NOT store this master key _anywhere_.  It would be re-generated at each system startup from the fixed multiple data sources, some random, some machine-specific. 

1. Note that depending on the method by which you gather machine-specific data to create a master key, you may need to create a method whereby the master key can be recorded and placed in a safe place immediately after installation. If you do this, do not forget to also create a means whereby the system can be restarted from the master key if there is some sort of equipment failure.


## Custom KeyStores

To make the storage and retrieval of a Master Key simpler, we have provided a couple of implementations of the standard Java `KeyStore` class. These key stores hold exactly __one__ `SecretKey`, which you then can use to securely encrypt a Data Encryption Key, which in turn protects all other credentials, keys, and certificates.  We recommend this two-tier approach so that the Master Key can easily be backed up, restored, and replaced in case of a breach, without needing to touch any other encrypted data.  

The Master Key (loaded and stored via our KeyStore implementations) is used to securely encrypt the Data Encryption Key in a file (this way you have complete understanding and control of the encryption of this important key). The Data Encryption Key is then used to generate any needed passphrases or cryptography keys, which are used to protect credentials, private keys, or other secret encryption keys.  To replace the Master Key, you decrypt the Data Encryption Key using the old Master Key, generate a new Master Key, then encrypt the Data Encryption Key (without changing it) with the _new_ Master Key and save the encrypted result.

## Samples
### Initialize Master Key System
![image of Master Key Initialization](https://github.com/IBM/crypto-masterkey-keystore/blob/master/common/images/MasterKeyInit.PNG)
Initialize the system at install time.
+ Create and save the Master Key material. There are a couple of possible approaches:
  + Use Java's `KeyGenerator` to create a strong AES encryption key, then use our `KeyStore` to save it in a known folder. 
  Be sure to secure the folder's permissions so only the application has permission to enter the folder.
  + Using a secure random number generator, save random bytes into a series of files with (Base64-encoded) random names, and set the least significant nybble of their timestamps to random data. Save the files in a known protected folder.  Sorting the files in alphabetical order, digest their contents, names, and random timestamp nybbles to create a random Master Key.
+ Back up the Master Key value to a file. Use password based encryption to protect it, and have the user keep it __offline__.
+ Generate a secure random Data Encryption Key (DEK) and use the Master Key (KEK in diagram above) to encrypt and save it. You can now wipe the Master Key from memory -- it is only used to encrypt/decrypt the Data Encryption Key.
+ Use the DEK and hard-coded alias names to generate any needed KeyStore passphrases, passphrases for each entry in each KeyStore, and encryption keys to encrypt any secret data (like credentials) stored in configuration files.

Because all passphrases and encryption keys can be re-generated dynamically from just the DEK and the correct alias, __none__ need to be stored in the system at all.
  ```java
  import java.security.NoSuchAlgorithmException;
  import javax.crypto.KeyGenerator;
  import javax.crypto.SecretKey;
  import java.security.KeyStore;
  import SecretFolderKeyStoreParameter;
  
  public class InitMasterKey {
    
    public static Folder  getInstallConfigFolder() {  // re-write for your app
      return new Folder("./config");
    }
    public static SecretKey generateMasterKey() {
      KeyGenerator keyGen = null;
      try {
        keyGen = KeyGenerator.getInstance("AES");  // AES key generator
      } catch (NoSuchAlgorithmException e) {
        e.printStackTrace();
      }
      keyGen.init(256);  // 256 bit key
      SecretKey masterKey = keyGen.generateKey();
      return masterKey;
    }
    public static void storeMasterKey(File configFolder, SecretKey masterKey) {
      File keyStoreFolder = new File(configFolder.getRoot(), "keyStore");
      KeyStore ks = KeyStore.getInstance("SecretFolderKeyStore";
      SecretFolderKeyStoreParameter param = new SecretFolderKeyStoreParameter(keyStoreFolder);
      ks.load(param);
      ks.setKeyEntry("masterKey", masterKey, null, null);
      ks.store(param);
    }
    public static void main(String args[]) {
      SecretKey masterKey = generateMasterKey();
      storeMasterKey(getInstallConfigFolder(), masterKey);
      // Prompt installer for password, seve KEK in pwd-protected file. Warn user to move it OFFLINE.
      backupMasterKey( getUserPassword(), getBackupFile() );
      // Create and encrypt DEK using Master Key. Save encrypted DEK in config folder
      SecretKey dek = generateAndStoreDataKey(getInstallConfigFolder(), masterKey);
      masterKey.destroy();
      // derive english passphrase from DEK and alias. phrase is at least 256 bits of entropy.
      char [] ksPwd = SecretGen.generatePassphrase(dek, "keyStorePassphrase", 256);
      // init and fill app's keystore with needed secrets. Use dek to derive additional passphrases as needed.
      buildAppKeyStore(getInstallConfigFolder(), ksPwd, dek);
      dek.destroy();
      Arrays.fill(ksPwd, ' ');  // wipe pwd from memory
    }
  ```
### Use Master Key System at Runtime
![image of Master Key Used at Runtime](https://github.com/IBM/crypto-masterkey-keystore/blob/master/common/images/MasterKeyReadWrite.png)
Use the system at run time.  
+ At system start-up, recover or re-generate the Master Key.  The approaches are:
  + Use our `KeyStore` to retrieve the Master Key from the known folder.
  + Open the known folder, sort the key material files in alphabetical order, then digest contents, names, and timestamps as before to re-generate the Master Key.
+ Use the Master Key to decrypt the Data Encrypting Key (DEK).  Wipe the Master Key from memory.
+ When a key or passphrase is needed, re-generate it using the DEK and the proper alias.
```java
    public static void main(String args[]) {
      // uses KeyStore api to retrieve the Master Key
      SecretKey masterKey = getMasterKey(getInstallConfigFolder());
      // Use Master Key to decrypt and return DEK
      SecretKey dek = getDataKey(getInstallConfigFolder(), masterKey);
      masterKey.destroy();  // clear sensitive data from memory
      masterKey = null;
      // Use dek and alias to derive an english passphrase having at least 256 bits of entropy
      char[] keystorePassphrase = SecretGen.generatePassphrase(dek, "keyStorePassphrase", 256); 
      // Load app's keystore from config folder. It is protected with a strong passphrase
      KeyStore baseStore = getAppKeyStore(getInstallConfigFolder(), keystorePassphrase);
      Arrays.fill( keystorePassphrse, ' ');
      // ... runtime use of secrets in app keystore ... //
      dek.destroy();
      dek = null;
    }
```
### Restore a Lost Master Key
![image of Restoring Master Key](https://github.com/IBM/crypto-masterkey-keystore/blob/master/common/images/MasterKeyRestore.PNG)
Restore a lost Master Key.
+ With the system offline, use the password protected backup of the Master Key to decrypt the Data Encryption Key (DEK).
+ Use the same process used at initialization to create a __new__ Master Key. Wipe all traces of the old Master Key material, and old backup, to prevent confusion.
+ Create a new backup of the new Master Key. Again, this must be kept securely _offline_.
+ Encrypt the existing DEK with the new Master Key and save the newly encrypted DEK. Delete the old encrypted DEK file to avoid confusion.
+ The system will can now be started normally.
