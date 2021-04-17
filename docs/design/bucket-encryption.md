# Bucket Encryption

## Old support for bucket-level encryption

### create_encrypted_bucket.js

The previous way of encrypting objects in buckets is by using a script
to create the bucket like:

```
docker exec scality-s3 node bin/create_encrypted_bucket.js -h 10.10.61.61 -p 8000 -a XIJG9OTSMLDDIEWX21HT -k BTNMVTDBujqpTUEVYVYtfhxExCWy3KvDlad4hmwC -b my-encrypted-bucket
```

The script is creating a bucket through the REST API, passing an extra
proprietary header "x-amz-scal-server-side-encryption: AES256" to
enable encryption using a newly created bucket master key stored in
the KMS server, and referenced from the bucket metadata.

It's also possible with the existing REST API to specify an existing
master key ID to use by providing the extra
"x-amz-scal-server-side-encryption-aws-kms-key-id" header, although
the create_encrypted_bucket.js script does not provide this option.

We plan to keep the compatibility with those headers for now but the
compatibility will be removed eventually in favor of the bucket
encryption API.

## Bucket Encryption API

We now support bucket encryption using the standard S3 APIs, to enable
or disable encryption on existing buckets:

- [PutBucketEncryption](https://docs.aws.amazon.com/AmazonS3/latest/API/API_PutBucketEncryption.html)

- [GetBucketEncryption](https://docs.aws.amazon.com/AmazonS3/latest/API/API_GetBucketEncryption.html)

- [DeleteBucketEncryption](https://docs.aws.amazon.com/AmazonS3/latest/API/API_DeleteBucketEncryption.html)

### PutBucketEncryption

We support the following rule to enable encryption automatically on
all objects in the bucket:

```
<ServerSideEncryptionConfiguration xmlns="http://s3.amazonaws.com/doc/2006-03-01/">
   <Rule>
      <ApplyServerSideEncryptionByDefault>
         <SSEAlgorithm>AES256</SSEAlgorithm>
      </ApplyServerSideEncryptionByDefault>
   </Rule>
   ...
</ServerSideEncryptionConfiguration>
```

Alternatively, we also allow specifying a master key ID to use for the
bucket that will be sent when contacting the KMS via the
KMSMasterKeyID tag, instead of letting S3C generate one automatically:

```
<ServerSideEncryptionConfiguration xmlns="http://s3.amazonaws.com/doc/2006-03-01/">
   <Rule>
      <ApplyServerSideEncryptionByDefault>
         <SSEAlgorithm>aws:kms</SSEAlgorithm>
         <KMSMasterKeyID>abcdef</KMSMasterKeyID>
      </ApplyServerSideEncryptionByDefault>
   </Rule>
   ...
</ServerSideEncryptionConfiguration>
```

**Note**: only one `<Rule>` entry is allowed in the server-side
encryption configuration.

**Note**: the `<BucketKeyEnabled>` rule option is not supported.

#### Implementation of PutBucketEncryption Regarding Bucket Metadata

* If the `serverSideEncryption` does not exist, we create it and fill
  with the relevant attributes (see [Internal Bucket
  Metadata](#internal-bucket-metadata))

* If the `serverSideEncryption` exists, we set its `mandatory`
  attribute to `true`, and possibly set the `algorithm` to the new
  configured algorithm, and if it is `aws:kms`, the **KMSMasterKeyId**
  also gets stored in the `configuredMasterKeyId` attribute. We do not
  change the other attributes, notably the `masterKeyId` is not
  replaced by a new one.

### GetBucketEncryption

Returns the bucket encryption configuration, as specified by a
previous PutBucketEncryption.

**Note**: due to the bucket metadata format preservation, it should
  also be returning an existing bucket-level encryption configuration
  for buckets created with the legacy
  `x-amz-scal-server-side-encryption` header, that did not get an
  explicit configuration applied via PutBucketEncryption.

#### Implementation of GetBucketEncryption Regarding Bucket Metadata

See [Internal Bucket Metadata](#internal-bucket-metadata)

The implementation reads the fields stored in the
`serverSideEncryption` bucket metadata attribute, and returns the
formatted XML response filled with the needed information.

* If there is no `serverSideEncryption` attribute, or if the
  `serverSideEncryption` attribute does not contain the
  `mandatory: true` field, return
  `ServerSideEncryptionConfigurationNotFoundError`.

### DeleteBucketEncryption

Deletes the existing bucket encryption configuration, removing any
default bucket encryption policy for the bucket hence storing new
objects unencrypted unless the requests provide the header
`x-amz-server-side-encryption`.

Internally, some of the bucket encryption metadata will be kept but
the `mandatory` flag is removed (see [Internal
Metadata](#internal-metadata) section).

#### Implementation of DeleteBucketEncryption Regarding Bucket Metadata

See [Internal Bucket Metadata](#internal-bucket-metadata)

When the bucket encryption configuration is deleted, we don't entirely
delete the `serverSideEncryption` section, because we want to keep the
existing parameters, notably the `masterKeyId`. Instead, we drop the
`mandatory` field and `configuredMasterKeyId` (if set), and keep the
other fields.

Deleting a bucket encryption configuration that does not exist
(i.e. no `serverSideEncryption` section or its `mandatory` field is
unset) returns a silent success to the client.

## Application Logic for Object Encryption

The following steps determine when and how to apply encryption to a
particular object PUT request:

* Does the request contain the `x-amz-server-side-encryption` header?

  * **yes**: encrypt object with the algorithm value of
    `x-amz-server-side-encryption` and master key ID
    `x-amz-server-side-encryption-aws-kms-key-id` if it is set and the
    algorithm value is `aws:kms`, or the bucket managed key under
    `serverSideEncryption.masterKeyId` otherwise

  * **no**: continue

* Does the bucket have a default encryption configuration? (i.e the
  bucket metadata field `serverSideEncryption.mandatory` is
  `true`)

  * **yes**: encrypt object with the algorithm value of
    `serverSideEncryption.algorithm` and either the
    `serverSideEncryption.configuredMasterKeyId` if it is set, or
    the master key ID `serverSideEncryption.masterKeyId` otherwise

  * **no**: do not encrypt the object

The following steps give the procedure when an object is to be
encrypted:

* if the algorithm value is **AES256**:

  * create server-side bucket encryption parameters if they do not
    exist yet in the `serverSideEncryption` bucket metadata field:

    * set the `algorithm` attribute to `AES256` and the
    `masterKeyId` to a new master key ID for that bucket, as well as
    `cryptoScheme` with value `1`. Do **not** set the
    `mandatory` field, as this configuration shall not apply
    automatically to new objects.

  * encrypt the object with a new data key, itself encrypted with the
    bucket master key

  * store the encrypted data key and master key ID (the same as the
    bucket master key ID) in the object metadata

* if the algorithm value is **aws:kms** and a master key ID is provided:

  * encrypt the object with a new data key, itself encrypted with the
    user-specified master key

  * store the encrypted data key and user-specified master key ID in
    the object metadata. Do not create or alter the bucket server-side
    encryption parameters.

* if the algorithm value is **aws:kms** but no master key ID is
  provided, follow the **AES256** logic above.

## Internal Metadata

The internal metadata format should not change with the introduction
of encryption API support.

### Internal Bucket Metadata

The internal bucket metadata related to encryption is stored under the
`serverSideEncryption` attribute. It may be of three slightly
different formats, described below.

**Note**: The `serverSideEncryption` field used to be created
automatically when the `x-amz-scal-server-side-encryption` was
provided at bucket creation time. It is now created dynamically,
either when a default bucket encryption configuration is applied to
the bucket or when the first object requesting encryption (through the
`x-amz-server-side-encryption` header) is stored in the bucket.

#### Managed KMS Master Key Without a Default Configuration

When a bucket has a managed encryption key associated to it, but no
default encryption applied to new objects automatically, its metadata
`serverSideEncryption` looks like this:

```
serverSideEncryption: {
    cryptoScheme: 1,
    algorithm: "AES256",
    masterKeyId: "48c5b43c9ba57f052deac9c081b2f1a6954dd2886e82f71679bc640a4a0e731b"
}
```

* **cryptoScheme** is used to version the encryption method and
  parameters used internally, like the "salt" value.

* **algorithm** should be "AES256" (the only supported algorithm).

* **masterKeyId** is the bucket's managed master encryption key ID. It
  is normally the identifier of the encryption key stored by the KMS
  server to encrypt/decrypt the individual object encryption keys for
  the bucket, when the user does not provide its own KMS key ID to use
  in the "aws:kms" encryption mode (and when the default bucket
  encryption does not contain an explicit one). It may also be the
  hex-encoded master encryption key itself if there is no KMS server,
  such as in this example (for testing purpose).

**Note**: On AWS, the master key ID is an ARN of the form
  "arn:aws:kms:region:account:key/uuid". S3C does not follow this
  convention, in this example it is a "file" backend key (only used
  for testing), and key IDs look different with a real KMS server.

#### Default Encryption Configuration With "AES256" Algorithm

A bucket may store a default encryption configuration, set via the
PutBucketEncryption API. When this happens, the bucket metadata
`serverSideEncryption` is created or updated with the following fields:

```
serverSideEncryption: {
    cryptoScheme: 1,
    algorithm: "AES256",
    masterKeyId: "48c5b43c9ba57f052deac9c081b2f1a6954dd2886e82f71679bc640a4a0e731b",
    mandatory: true
}
```

Here, we set the `mandatory: true` field to know that the
configuration shall be applied by default on all new objects.

The `serverSideEncryption` metadata attribute is created if it does
not exist yet and a new master key is created via the KMS server, or
if the bucket already has the `serverSideEncryption` attribute, it is
updated by setting the `mandatory: true` field to enforce a default
encryption on new objects, and keeping the other fields as-is, in
particular the `masterKeyId` is not changed (we do not plan to support
other encryption schemes than **AES256** for now so the rest of the
fields can be left alone).

#### Default Encryption Configuration With "aws:kms" Algorithm

In case the default encryption configuration is of type "aws:kms", the
master key ID has been specified explicitly in the "KMSMasterKeyId"
tag during PutBucketEncryption, and it is also stored in this section
under the `configuredMasterKeyId` field, and will take precedence over
the `masterKeyId` when applying the default encryption parameters to
objects:

```
serverSideEncryption: {
    cryptoScheme: 1,
    algorithm: "aws:kms",
    masterKeyId: "48c5b43c9ba57f052deac9c081b2f1a6954dd2886e82f71679bc640a4a0e731b",
    mandatory: true,
    configuredMasterKeyId: "f04e1f176cfc3dd1f104921dc9e1aff82cb15c3c534ee973ff5653e32c7c439d"
}
```

**Note**: this default configured master key ID to use is stored
separately from the bucket managed master key ID in the
`serverSideEncryption` section, because it can be set or removed
dynamically by the bucket encryption configuration API and should not
alter the existing (permanent) bucket encryption managed key that is
used to encrypt objects without an explicit KMS key ID.

### Legacy Bucket Encryption Configuration (create_encrypted_bucket.js)

For buckets that were created encrypted in previous releases using the
`create_encrypted_bucket.js` script, the metadata format is the same
as in the section [Default Encryption Configuration With "AES256"
Algorithm](#default-encryption-configuration-with-aes256-algorithm).

This way, we can treat it in the same way, simplifying the logic and
allowing to return the existing encryption configuration via
GetBucketEncryption.

For reference, the existing metadata for legacy encryption
configuration looks like this:

```
serverSideEncryption: {
    cryptoScheme: 1,
    algorithm: "AES256",
    masterKeyId: "48c5b43c9ba57f052deac9c081b2f1a6954dd2886e82f71679bc640a4a0e731b",
    mandatory: true
}
```

### Internal Object Metadata

Each encrypted object contains in its metadata a set of attributes resembling the following:

```
"x-amz-server-side-encryption": "AES256"|"aws:kms",
"x-amz-server-side-encryption-aws-kms-key-id": "48c5b43c9ba57f052deac9c081b2f1a6954dd2886e82f71679bc640a4a0e731b",
"x-amz-server-side-encryption-customer-algorithm": "",
```

* **x-amz-server-side-encryption** is set to the encryption algorithm,
    "AES256", or "aws:kms" if there is a custom KMS key ID that should
    be returned when fetching object metadata (HEAD object request)

* **x-amz-server-side-encryption-aws-kms-key-id** is the identifier of
    the KMS master key used to encrypt/decrypt the object data encryption
    key, set in all encryption modes.

* **x-amz-server-side-encryption-customer-algorithm**: always set to
    the empty string, relates to customer-provided keys mode (SSE-C)
    which is not supported at the moment.

Each data location object also contains the following attributes:

```
"location": [
    {
        // other location attributes: "key" etc.
        "cryptoScheme": 1,
        "cipheredDataKey": "d88SLogLTw8nzRyO0Tf0qynQPLubT0N9mBDAxEZ5ww0="
    }
]
```

* **cryptoScheme** is used to version the encryption method and
  parameters used internally, like the "salt" value.

* **cipheredDataKey** is the object data encryption key, itself stored
  encrypted with the master key

## Bucket Policies

We currently do not plan to support bucket policies related to
encryption, namely, specifying conditions attached to the
`s3:x-amz-server-side-encryption` attribute in bucket policies.

The main reason to implement those would be to prevent users from
using a different encryption scheme than what is set by default in the
bucket encryption configuration.

When a bucket has a default encryption configuration attached, it is
not possible to send unencrypted objects anymore to the bucket,
because:

* the absence of the `x-amz-server-side-encryption` header in a PUT
  object request automatically applies the default encryption to the
  object

* if present, the `x-amz-server-side-encryption` header must contain a
  valid encryption mode, there is no "unencrypted" mode available.

For this reason, we don't support those bucket policies today.
