# [S3C-2675](https://scality.atlassian.net/browse/S3C-2675): Audit logs requirements document

## Context

Some customers are using tools like [IBM QRadar](https://www.ibm.com/fr-fr/security/security-intelligence/qradar) in order to centralize and monitor who did what on IT systems for security reason. This tool is ingesting events formatted in Leefx format, which is a basic key/value list, separated with a space.

## Requirements

### Storie

As a Storage Administrator, I want audit logs that describe who perform an action on a S3 or IAM ressource, in order connect my Scality products to my IBM QRadar system.

Acceptance Criteria:

* Audit logs must be formatted using LEEF format
* Audit logs must be self-descriptive, that means a Storage Administrator don't need to correlate multiple log lines or log files to know who perform an action on a ressource
* In case of an integration with an External Identity Provider(using web identity or SAML federation), the 'who' must be the External Identity login or username
* Failed action must appear in the Audit logs with the reason of failed (e.g: wrong signature, or privileges...)
* Audit logs should be included into sreport
* Audit logs rotation, purge, activation, desactivation must be configurable
* Audit logs configuration, activation, desactivation must be documented
* Audit logs configuration, activation, desactivation must persist to Scality upgrade

### Who definition

'Who' can be:

* Account root user
* IAM users
* Federated users (using web identity or SAML federation)

### IAM ressources definition

IAM resources can be:
* Account
* Users
* Policies
* Groups
* Identity provided by Identity provider
* Roles

## IAM Actions definition

IAM actions can be:

* AddClientIDToOpenIDConnectProvider
* AddRoleToInstanceProfile
* AddUserToGroup
* AttachGroupPolicy
* AttachRolePolicy
* AttachUserPolicy
* ChangePassword
* CreateAccessKey
* CreateAccountAlias
* CreateGroup
* CreateInstanceProfile
* CreateLoginProfile
* CreateOpenIDConnectProvider
* CreatePolicy
* CreatePolicyVersion
* CreateRole
* CreateSAMLProvider
* CreateServiceLinkedRole
* CreateServiceSpecificCredential
* CreateUser
* CreateVirtualMFADevice
* DeactivateMFADevice
* DeleteAccessKey
* DeleteAccountAlias
* DeleteAccountPasswordPolicy
* DeleteGroup
* DeleteGroupPolicy
* DeleteInstanceProfile
* DeleteLoginProfile
* DeleteOpenIDConnectProvider
* DeletePolicy
* DeletePolicyVersion
* DeleteRole
* DeleteRolePermissionsBoundary
* DeleteRolePolicy
* DeleteSAMLProvider
* DeleteServerCertificate
* DeleteServiceLinkedRole
* DeleteServiceSpecificCredential
* DeleteSigningCertificate
* DeleteSSHPublicKey
* DeleteUser
* DeleteUserPermissionsBoundary
* DeleteUserPolicy
* DeleteVirtualMFADevice
* DetachGroupPolicy
* DetachRolePolicy
* DetachUserPolicy
* EnableMFADevice
* GenerateCredentialReport
* GenerateOrganizationsAccessReport
* GenerateServiceLastAccessedDetails
* GetAccessKeyLastUsed
* GetAccountAuthorizationDetails
* GetAccountPasswordPolicy
* GetAccountSummary
* GetContextKeysForCustomPolicy
* GetContextKeysForPrincipalPolicy
* GetCredentialReport
* GetGroup
* GetGroupPolicy
* GetInstanceProfile
* GetLoginProfile
* GetOpenIDConnectProvider
* GetOrganizationsAccessReport
* GetPolicy
* GetPolicyVersion
* GetRole
* GetRolePolicy
* GetSAMLProvider
* GetServerCertificate
* GetServiceLastAccessedDetails
* GetServiceLastAccessedDetailsWithEntities
* GetServiceLinkedRoleDeletionStatus
* GetSSHPublicKey
* GetUser
* GetUserPolicy
* ListAccessKeys
* ListAccountAliases
* ListAttachedGroupPolicies
* ListAttachedRolePolicies
* ListAttachedUserPolicies
* ListEntitiesForPolicy
* ListGroupPolicies
* ListGroups
* ListGroupsForUser
* ListInstanceProfiles
* ListInstanceProfilesForRole
* ListMFADevices
* ListOpenIDConnectProviders
* ListPolicies
* ListPoliciesGrantingServiceAccess
* ListPolicyVersions
* ListRolePolicies
* ListRoles
* ListRoleTags
* ListSAMLProviders
* ListServerCertificates
* ListServiceSpecificCredentials
* ListSigningCertificates
* ListSSHPublicKeys
* ListUserPolicies
* ListUsers
* ListUserTags
* ListVirtualMFADevices
* PutGroupPolicy
* PutRolePermissionsBoundary
* PutRolePolicy
* PutUserPermissionsBoundary
* PutUserPolicy
* RemoveClientIDFromOpenIDConnectProvider
* RemoveRoleFromInstanceProfile
* RemoveUserFromGroup
* ResetServiceSpecificCredential
* ResyncMFADevice
* SetDefaultPolicyVersion
* SetSecurityTokenServicePreferences
* SimulateCustomPolicy
* SimulatePrincipalPolicy
* TagRole
* TagUser
* UntagRole
* UntagUser
* UpdateAccessKey
* UpdateAccountPasswordPolicy
* UpdateAssumeRolePolicy
* UpdateGroup
* UpdateLoginProfile
* UpdateOpenIDConnectProviderThumbprint
* UpdateRole
* UpdateRoleDescription
* UpdateSAMLProvider
* UpdateServerCertificate
* UpdateServiceSpecificCredential
* UpdateSigningCertificate
* UpdateSSHPublicKey
* UpdateUser
* UploadServerCertificate
* UploadSigningCertificate
* UploadSSHPublicKey

Extracted from [AWS IAM actions][aws_iam_actions].

### S3 ressources definition

S3 ressources can be:

* Buckets
* Objects

### S3 actions definition

S3 actions can be:

* AbortMultipartUpload
* CompleteMultipartUpload
* CopyObject
* CreateBucket
* CreateMultipartUpload
* DeleteBucket
* DeleteBucketAnalyticsConfiguration
* DeleteBucketCors
* DeleteBucketEncryption
* DeleteBucketInventoryConfiguration
* DeleteBucketLifecycle
* DeleteBucketMetricsConfiguration
* DeleteBucketPolicy
* DeleteBucketReplication
* DeleteBucketTagging
* DeleteBucketWebsite
* DeleteObject
* DeleteObjects
* DeleteObjectTagging
* DeletePublicAccessBlock
* GetBucketAccelerateConfiguration
* GetBucketAcl
* GetBucketAnalyticsConfiguration
* GetBucketCors
* GetBucketEncryption
* GetBucketInventoryConfiguration
* GetBucketLifecycle
* GetBucketLifecycleConfiguration
* GetBucketLocation
* GetBucketLogging
* GetBucketMetricsConfiguration
* GetBucketNotification
* GetBucketNotificationConfiguration
* GetBucketPolicy
* GetBucketPolicyStatus
* GetBucketReplication
* GetBucketRequestPayment
* GetBucketTagging
* GetBucketVersioning
* GetBucketWebsite
* GetObject
* GetObjectAcl
* GetObjectLegalHold
* GetObjectLockConfiguration
* GetObjectRetention
* GetObjectTagging
* GetObjectTorrent
* GetPublicAccessBlock
* HeadBucket
* HeadObject
* ListBucketAnalyticsConfigurations
* ListBucketInventoryConfigurations
* ListBucketMetricsConfigurations
* ListBuckets
* ListMultipartUploads
* ListObjects
* ListObjectsV2
* ListObjectVersions
* ListParts
* PutBucketAccelerateConfiguration
* PutBucketAcl
* PutBucketAnalyticsConfiguration
* PutBucketCors
* PutBucketEncryption
* PutBucketInventoryConfiguration
* PutBucketLifecycle
* PutBucketLifecycleConfiguration
* PutBucketLogging
* PutBucketMetricsConfiguration
* PutBucketNotification
* PutBucketNotificationConfiguration
* PutBucketPolicy
* PutBucketReplication
* PutBucketRequestPayment
* PutBucketTagging
* PutBucketVersioning
* PutBucketWebsite
* PutObject
* PutObjectAcl
* PutObjectLegalHold
* PutObjectLockConfiguration
* PutObjectRetention
* PutObjectTagging
* PutPublicAccessBlock
* RestoreObject
* SelectObjectContent
* UploadPart
* UploadPartCopy

Extracted from [AWS S3 actions][aws_s3_actions].

[aws_s3_actions]: https://docs.aws.amazon.com/AmazonS3/latest/API/API_Operations_Amazon_Simple_Storage_Service.html
[aws_iam_actions]: https://docs.aws.amazon.com/IAM/latest/APIReference/API_Operations.html