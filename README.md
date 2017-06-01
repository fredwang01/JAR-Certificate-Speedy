"# JAR-Certificate-Speedy" 

JAR archive file extends ZIP archive file format. JAR archive contain META-INF files which can be used to verify the integrity and certificates of entry in the archive. We can usually retrieve the certificates in the JAR archive using the following code fragment:
```
public byte[] getCerts(File file) {
        JarFile jarFile = null;
        InputStream inputStream = null;
        try {
            jarFile = new JarFile(file);
            Enumeration<JarEntry> enumEntries = jarFile.entries();
            while (enumEntries.hasMoreElements()) {
                JarEntry jarEntry = enumEntries.nextElement();
                if (jarEntry.isDirectory()) {
                    continue;
                }

                if (!jarEntry.getName().startsWith("META-INF/")) {
                    inputStream = jarFile.getInputStream(jarEntry);
                    byte[] buffer = new byte[4096];
                    while (inputStream.read(buffer) >= 0) {
                    }

                    Certificate[] certificates = jarEntry.getCertificates();
                    if (certificates == null || certificates.length == 0) {
                        Log.e(TAG, "empty cert.");
                        return new byte[0];
                    }
                    X509Certificate cert = (X509Certificate) certificates[0];
                    return cert.getEncoded();
                }
            }
            return new byte[0];
        } catch (Exception e) {
            e.printStackTrace();
            return new byte[0];
        } finally {
            if (jarFile != null) {
                try {
                    jarFile.close();
                } catch (Exception e) {
                }
            }
            if (inputStream != null) {
                try {
                    inputStream.close();
                } catch (Exception e) {
                }
            }
        }
    }
```

But before retrieving the certificates, we should read fully the JAR entry no matter if we use the contents. What on earth happened to the JAR entry reading include:
* Parse the RSA(DSA/EC) file to retrieve the certificates, the data structure of certificates is described as `ASN1` usually.
* Verify the integrity of MF file and each entry in it with the SF file.
* Read the JAR entry, usually need decompress from the archive, and calculate the digest of the contents of JAR entry readed fully in order to verify the entry integrity with the corresponding digest in the MF file.

As we see above, if you just want to retrieve the certificates from JAR archive and speedy the process, we can optimize some unnecessary implementation. Because each entry in the archive signs with the same certificate(chain) on Android platform, we can get the certificates using any entry in the archive(usually the first one) which must satisfy the requirements:
* Can not use directory entry in the archive.
* Can not use any META-INF files in the archive.

The reason can be found out in the following code(`JarVerifier.java`):
```
    VerifierEntry initEntry(String name) {
        // If no manifest is present by the time an entry is found,
        // verification cannot occur. If no signature files have
        // been found, do not verify.
        if (man == null || signatures.size() == 0) {
            return null;
        }

        Attributes attributes = man.getAttributes(name);
        // entry has no digest
        if (attributes == null) {
            return null;
        }
        ...
    }
```
Because the directory entry and META-INF files are not described in the MF file, here `Attributes attributes = man.getAttributes(name);`  get `null` , which will result in the common ZIP InputStream instead of JarFileInputStream which can retrieve the certificates.

In my custom implementation, we can retrieve the certificates by directly calling `Certificate[] certificates = jarEntry.getCertificates();`, without any MF verify, JAR entry read and verify, and checking the entry type.

In short, if we only read entry contents, it is better to use ZIP API instead of JAR API.

By the way, when constructing a JarFile object like `new JarFile(path)` or a ZipFile object like `new ZipFile(path)`, all entries meta data in the central directory readed and cached. Please refer to the following code(`ZipFile.java`):
```
private void readCentralDir() throws IOException {
    ...
    /*
     * Seek to the first CDE and read all entries.
     */
    rafs = new RAFStream(mRaf, centralDirOffset);
    bin = new BufferedInputStream(rafs, 4096);
    for (int i = 0; i < numEntries; i++) {
        ZipEntry newEntry = new ZipEntry(ler, bin);
        mEntries.put(newEntry.getName(), newEntry);
    }
}
```
The advantage of this implementation is that it is mush faster to retrieve the entry meta data. But if there are too many entries in the archive and the name of entries is too long, this implementation may be a big memory burden.









