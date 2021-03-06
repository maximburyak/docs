﻿#Encryption

The encryption bundle introduces data encryption to RavenFS. The encryption is fully transparent for the end-user, it encrypts content of all files.
Metadata of stored files are not encrypted, likewise [configuration items](../../configurations). Indexes in RavenFS keep only those values so they aren't encrypted too.

## Configuration

If you want to setup a new file system with encryption bundle using the Studio, then please refer to [this](../../studio/how-to/how-to-setup-encryption) page.

The possible configuration options are:   

* **Raven/Encryption/Algorithm** with [AssemblyQualifiedName](http://msdn.microsoft.com/en-us/library/system.type.assemblyqualifiedname.aspx) as a value. Additionally provided type must be a subclass of [SymmetricAlgorithm](http://msdn.microsoft.com/en-us/library/system.security.cryptography.symmetricalgorithm.aspx) from `System.Security.Cryptography` namespace and must not be an abstract class    
* **Raven/Encryption/Key** a key used for encryption purposes with minimum length of 8 characters, base64 encoded   
* **Raven/Encryption/KeyBitsPreference** the preferred encryption key size in bits  

### Global configuration

All configuration settings can be setup server-wide by adding them to server configuration file.

### File system configuration

All settings can be overridden per file system during the file system creation process.

{CODE encryption_1@FileSystem\Server\Bundles\Encryption.cs /}

Above example demonstrates how to create `EncryptedFS` with active encryption and with non-default encryption algorithm.

{NOTE All encryption settings can only be provided when a file system is being created. Changing them later will cause a file system malfunction. /}

## Encryption key management

There are two types of configuration: server-wide configuration, which is usually located at `Raven.Server.exe.config / web.config` file and a file system specific configuration, which is located at `<system>` database. For the config file, we provide support for encrypting the file using [DPAPI](http://en.wikipedia.org/wiki/Data_Protection_API), using the standard .NET config file encryption system. For the file system specific values, we provide our own support for encrypting the values using DPAPI.

So, as the consequences of the above:    

*	Your files are encrypted when they are on a disk using strong encryption.    
*	You can use a a server wide or a file system specific key for the encryption.   
*	Your encryption key is guarded using DPAPI.   
*	The data is safely encrypted on a disk, and the OS guarantees that no one can access the encryption key.   

{NOTE It is your responsibility to backup the encryption key, as there is no way to recover data without it. /}

## Encryption & Backups

{DANGER:Important}

By default, backup of the encrypted file system **contains the encryption key (`Raven/Encryption/Key`) as a plain text** in `Filesystem.Document` file. This is required to allow to restore the backup on a different machine. 

To not include any confidential file system settings, please issue a backup request manually with filled file system document.

### Backup & Restore

1. Issue a backup request with empty `SecuredSettings` (where encryption configuration is placed) in [FileSystemDocument](../../../glossary/file-system-document):

{CODE-BLOCK:json}
curl -X POST "http://localhost:8080/fs/NorthwindFS/admin/backup?incremental=false" \
-d "{\"BackupLocation\":\"c:\\temp\\backup\\NorthwindFS\",\"FileSystemDocument\":{\"SecuredSettings\":{},\"Settings\":{\"Raven/ActiveBundles\": \"Encryption\"},\"Disabled\":false,\"Id\":null}}"
{CODE-BLOCK/}

2. Notice that `Filesystem.Document` found in `c:\temp\backup\NorthwindFS\` contains exactly the same information that you send in your request, which means that your file system cannot be restored until you specify encryption keys in this document.

3. After filling `Raven/Encryption/Key`, `Raven/Encryption/Algorithm` and `Raven/Encryption/KeyBitsPreference` in your `Filesystem.Document` you can issue the restore request:

{CODE-BLOCK:json}
curl -X POST "http://localhost:8080/admin/fs/restore" \
-d "{\"FilesystemName\":\"NewNorthwindFS\",\"BackupLocation\":\"c:\\temp\\backup\\NorthwindFS\",\"IndexesLocation\":null,\"RestoreStartTimeout\":null,\"FilesystemLocation\":\"~\\FileSystem\\NewNorthwindFS\",\"Defrag\":false,\"JournalsLocation\":null}"
{CODE-BLOCK/}

{DANGER/}

## Related articles

- [Studio : How to setup file encryption?](../../studio/how-to/how-to-setup-encryption)