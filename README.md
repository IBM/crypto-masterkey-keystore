# crypto-masterkey-keystore
Custom KeyStore implementations for Master Keys  
Guidance for generating and managing application Master Keys  
Sample code for generating and managing application master keys  
----
## Custom KeyStores
----
## Guidance
Cryptographic keys (including TLS certificates) and passwords should be unique for each appliance instance and for each on-premises installation. If multiple customers can share the same appliance or server, each must have a unique set of keys and certificates.  Otherwise, every legitimate customer will have access to certificates/keys that allow her to spy on all other customers.

Keys, certs, and passwords should be established at installation time (first bootup for appliances). Either ask the installer to specify a password (for example, for an admin account password), or generate them dynamically (keystore passwords, cryptography keys, master key, etc.).  There should be no default passwords.  Customers can easily forget or neglect to change them.

Keys and passwords should not be stored in clear text in any configuration files or source code files.  If credentials must be stored, they must be **encrypted**.  Obfuscating or encoding the data alone (by XORing with some static byte array, or Base64 encoding, for example) is NOT enough. The key for such encryption should be generated dynamically at installation time. This is often termed the 'master key' because it is used to secure all other cryptographic keys and passwords.

Master keys should be generated using a key stretching algorithm like PBKDF2 from data gathered from multiple sources. At least one of these sources should be tied to the hardware on which the system is running.  Best practices would NOT store this master key anywhere.  It would be re-generated at each system startup from the fixed multiple data sources, some random, some machine-specific.

**NOTE:** Depending on the method by which you gather machine-specific data to create a master key, you may need to create a method whereby the master key can be recorded and placed in a safe place immediately after installation. If you do this, do not forget to also create a means whereby the system can be restarted from the master key if there is some sort of equipment failure.
----
## Samples
