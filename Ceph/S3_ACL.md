# AWS S3 ACL

## 什么是ACL
AWS S3中采用ACL(Access control list)来实现对用户访问权限的控制。ACL是AWS S3中资源层(resource-based)访问策略选项之一，你可以使用ACL来管理其他用户对你所拥有的资源的访问。
这里需要对三点进行说明：
1. S3中资源(resource)包括哪些？
AWS S3中的基本资源包括桶(buckets)和所存储的对象(objects)，除了上述两个基本资源概念外，buckets和objects还分别包含相关联的子资源(subresources)。
> Buckets and objects are primary Amazon S3 resources, and both have associated subresources. For example, bucket subresources include the following:
> *lifecycle* – Stores lifecycle configuration information (see Object Lifecycle Management).
> *website* – Stores website configuration information if you configure your bucket for website hosting (see Hosting a Static Website on Amazon S3.
> *versioning* – Stores versioning configuration (see PUT Bucket versioning).
> *policy and acl (Access Control List)* – Store access permission information for the bucket.
> *cors (Cross-Origin Resource Sharing)* – Supports configuring your bucket to allow cross-origin requests (see Cross-Origin Resource Sharing (CORS)).
> *logging* – Enables you to request Amazon S3 to save bucket access logs.

> Object subresources include the following:
> *acl* – Stores a list of access permissions on the object. This topic discusses how to use this subresource to manage object permissions (see Managing Access with ACLs ).
> *restore* – Supports temporarily restoring an archived object (see POST Object restore). An object in the Glacier storage class is an archived object. To access the object, you must first initiate a restore request, which restores a copy of the archived object. In the request, you specify the number of days that you want the restored copy to exist. For more information about archiving objects, see Object Lifecycle Management.

2. S3中的访问策略包括哪些？
在AWS S3中，可以为一个资源(a resource)指定访问策略，也可以为一个user关联访问策略。S3中的访问策略可以分为：
- Resource-based policies
- User policies

Resource-based policies，即资源层访问策略(ACL所属)，包括Bucket policies和Access control lists(ACLs, 包括bucket ACLs && object ACLs)，你可以为Amazon S3的资源设置这两种策略。
Note: bucket policy允许授予anonymous对资源的访问权限。
User policies，可以使用AWS Identity and Access Management(IAM)来管理user对你的Amazon S3的资源的访问。
Note: user policy不允许授予anonymous权限，因为user policy必须与实际用户进行绑定。

3. AWS S3中包括哪些预定义的群组？
> Authenticated Users group – Represented byhttp://acs.amazonaws.com/groups/global/AuthenticatedUsers.
> This group represents all AWS accounts. Access permission to this group allows any AWS account to access the resource. However, all requests must be signed (authenticated).
> All Users group – Represented by http://acs.amazonaws.com/groups/global/AllUsers.
> Access permission to this group allows anyone to access the resource. The requests can be signed (authenticated) or unsigned (anonymous). Unsigned requests omit the Authentication header in the request.
> Log Delivery group – Represented by http://acs.amazonaws.com/groups/s3/LogDelivery.
>  WRITE permission on a bucket enables this group to write server access logs (see Server Access Logging) to the bucket.
若要为某个预定义群组授予访问权限，需要指定相应的URI。

当你创建一个bucket或者object时，Amazon S3会创建一个默认的ACL，其中会赋予资源的拥有者full control的访问权限。
你也可以使用ACLs来为其他账户授予基本的读写权限。使用ACLs来对权限进行管理时，会有一些基本的限制。包括：
1. 只能为其他AWS账户(account)授予操作权限，无法为自身账户下的用户users授予权限；
2. 无法授予有条件的权限，也无法显式地否认已授予的权限。

S3中，在授予权限时，需要清楚三件事：
1. 我们要把权限授予哪些用户；
2. 哪些Amazon S3的资源需要被授予权限；
3. 需要为那些资源授予什么权限。

Note: 在AWS S3中bucket owner和object owner是两个概念，如果不是object owner，也未被授权访问该object，即使该object存储在你的bucket中，你也无法访问该object。

你可以通过指定某个AWS账户的email地址或者canonical user ID来为该账户授予某些操作权限。即使你提供的是email地址，Amazon S3会找到与该email关联的账户的canonical user ID，然后再添加到ACL中。也就是说，ACL中保存的账户信息始终是canonical user ID。
Note: 需要说明的是，你可以通过ACL为一个AWS账户(account)或者一个预定义的AWS S3群组赋予资源的访问权限，但是却不可以通过ACL为IAM User赋予访问权限。

### 如何获取AWS账户的canonical user ID
两种方法：
- 使用AWS Management console来获取
- 上面已经说过，ACL中保存的即是每个AWS账户的cannonical user ID，所以我们只需要通过读取一个该AWS账户具备访问权限的bucket或者object的ACL，即可从中获取该AWS账户的canonical user ID。

bucket ACLs和object ACLs使用同样的XML模板。

IAM user是什么？

### 什么时候使用一个object ACL
除了object ACL，object拥有者可以通过其他方式来管理object的操作权限。例如：
 - 如果AWS账户同时是object和bucket的拥有者，可以通过写一个bucket policy来对object的操作权限进行管理。
 - 如果AWS账户是object的拥有者，它想为其他账户下的user授予访问权限，可以通过为该user附加一个user policy来为该user设置权限。

那么什么情况下，我们需要使用object ACL来管理对象的权限呢？
> 1. An object ACL is the only way to manage access to objects not owned by the bucket owner – An AWS account that owns the bucket can grant another AWS account permission to upload objects. The bucket owner does not own these objects. The AWS account that created the object must grant permissions using object ACLs.
> 2. Permissions vary by object and you need to manage permissions at the object level – You can write a single policy statement granting an AWS account read permission on millions of objects with a specific key name prefix. For example, grant read permission on objects starting with key name prefix "logs". However, if your access permissions vary by object, granting permissions to individual objects using a bucket policy may not be practical. Also the bucket policies are limited to 20 KB in size.
>    In this case, you may find using object ACLs a suitable alternative. Although, even an object ACL is also limited to a maximum of 100 grants (see Access Control List (ACL) Overview).
> 3. Object ACLs control only object-level permissions – There is a single bucket policy for the entire bucket, but object ACLs are specified per object.
>    An AWS account that owns a bucket can grant another AWS account permission to manage access policy. It allows that account to change anything in the policy. To better manage permissions, you may choose not to give such a broad permission, and instead grant only the READ-ACP and WRITE-ACP permissions on a subset of objects. This limits the account to manage permissions only on specific objects by updating individual object ACLs.

### 什么时候使用一个bucket ACL
对于bucket ACL来说，只在一种使用场景下被推荐使用，即为Amazon S3 Log Delivery group授予写权限以支持存储桶日志功能，将access log objects(存储桶访问日志对象)写入到target bucket中。Bucket ACL也是为LDG授予必要的操作权限的唯一方式。

### 什么时候使用一个bucket policy
> If an AWS account that owns a bucket wants to grant permission to users in its account, it can use either a bucket policy or a user policy. But in the following scenarios, you will need to use a bucket policy.
> You want to manage cross-account permissions for all Amazon S3 permissions – You can use ACLs to grant cross-account permissions to other accounts, but ACLs support only a finite set of permission (What Permissions Can I Grant?), these don't include all Amazon S3 permissions. For example, you cannot grant permissions on bucket subresources (see Managing Access Permissions to Your Amazon S3 Resources) using an ACL.
> Although both bucket and user policies support granting permission for all Amazon S3 operations (see Specifying Permissions in a Policy), the user policies are for managing permissions for users in your account. For cross-account permissions to other AWS accounts or users in another account, you must use a bucket policy.
Note: User policy只能用于管理本账户下的所有user的操作权限。

### 什么时候使用一个user policy
AWS Identity and Access Management(IAM)机制允许你在你的AWS账户下创建多个用户，并且通过user policy来对他们的访问权限进行管理。
一个IAM user若要对资源进行访问，必须具备如下权限：
 - Permission from the parent account – The parent account can grant permissions to its user by attaching a user policy.
 - Permission from the resource owner – The resource owner can grant permission to either the IAM user (using a bucket policy) or the parent account (using a bucket policy, bucket ACL, or object ACL).

很好的一个类比
> This is akin to a child who wants to play with a toy that belongs to someone else. In this case, the child must get permission from a parent to play with the toy and permission from the toy owner.
这可以类比于一个小孩想要玩其他小孩的玩具。在这种情况下，这个小孩必须得到他父母的同意，让他去玩那个玩具；同时也要得到那个玩具的主人的同意，他才能成功玩到那个玩具。


## ACL工作机制
### Amazon如何对一个请求进行认证
当Amazon S3接收到一个请求后，会对该requester相关的所有权限控制策略进行评估，所需进行评估的权限策略包括：
- access policies
- user policies
- resource-based policies(bucket policy, bucket ACL, object ACL)

Note: 若IAM user要对一个资源执行指定的操作，该IAM user需要得到如下账户的授权：
- 该IAM user所属的AWS账户的授权
- 该资源所属的AWS账户的操作授权

> In order to determine whether the requester has permission to perform the specific operation, Amazon S3 does the following, in order, when it receives arequest:
为了判定某个请求者是否具备执行指定操作的权限，Amazon S3会在接收到一个请求后，按顺序执行如下操作：
> 1. Converts all the relevant access policies (user policy, bucket policy, ACLs) at run time into a set of policies for evaluation.
为了方便对权限进行评估，将所有相关的运行时访问策略(包括user policy, bucket policy, ACLs)汇总为一个策略集。
> 2. Evaluates the resulting set of policies in the following steps. In each step, Amazon S3 evaluates a subset of policies in a specific context, based on the context authority.
按照如下步骤对策略的结果集进行评估。在如下的每一步，Amazon S3都会通过每个指定的上下文的权限验证单位在该上下文中对一个策略子集进行评估。
>    a. User context – In the user context, the parent account to which the user belongs is the context authority.
>       Amazon S3 evaluates a subset of policies owned by the parent account. This subset includes the user policy that the parent attaches to the user. If the parent also owns the resource in the request (bucket, object), Amazon S3 also evaluates the corresponding resource policies (bucket policy, bucket ACL, and object ACL) at the same time.
>       A user must have permission from the parent account to perform the operation.
>       This step applies only if the request is made by a user in an AWS account. If the request is made using root credentials of an AWS account, Amazon S3 skips this step.
>    b. Bucket context – In the bucket context, Amazon S3 evaluates policies owned by the AWS account that owns the bucket.
>       If the request is for a bucket operation, the requester must have permission from the bucket owner. If the request is for an object, Amazon S3 evaluates all the policies owned by the bucket owner to check if the bucket owner has not explicitly denied access to the object. If there is an explicit deny set, Amazon S3 does not authorize the request.
>    c. Object context – If the request is for an object, Amazon S3 evaluates the subset of policies owned by the object owner.

### Amazon如何对一个bucket操作请求进行认证
基于上面的请求认证过程，具体到一个bucket操作请求时，当Amazon S3接收到一个bucket操作请求时，Amazon S3会先将所有的相关权限信息(资源层的bucket权限(bucket policy, bucket ACL)，如果是一个user发出的请求，还包括IAM user policy)汇总到一个运行时的策略集中。然后按照下面的步骤顺序进行评估, 评估过程主要针对两个指定的上下文 -- User Context && Bucket Context:
> 1. User context – If the requester is an IAM user, the user must have permission from the parent AWS account to which it belongs. In this step, Amazon S3 evaluates a subset of policies owned by the parent account (also referred to as the context authority). This subset of policies includes the user policy that the parent account attaches to the user. If the parent also owns the resource in the request (in this case, the bucket), Amazon S3 also evaluates the corresponding resource policies (bucket policy and bucket ACL) at the same time. Whenever a request for a bucket operation is made, the server access logs record the canonical user ID of the requester.
> 2. Bucket context – The requester must have permissions from the bucket owner to perform a specific bucket operation. In this step, Amazon S3 evaluates a subset of policies owned by the AWS account that owns the bucket.
>    The bucket owner can grant permission by using a bucket policy or bucket ACL. Note that, if the AWS account that owns the bucket is also the parent account of an IAM user, then it can configure bucket permissions in a user policy.

下面是一些bucket操作请求认证的例子
Example 1: 
> Bucket Operation Requested by Bucket Owner
> In this example, the bucket owner sends a request for a bucket operation using the root credentials of the AWS account.
> Amazon S3 performs the context evaluation as follows:
> 1. Because the request is made by using root credentials of an AWS account, the user context is not evaluated.
> 2. In the bucket context, Amazon S3 reviews the bucket policy to determine if the requester has permission to perform the operation. Amazon S3 authorizes the request.

Example 2:
> Bucket Operation Requested by an AWS Account That Is Not the Bucket Owner
> In this example, a request is made using root credentials of AWS account 1111-1111-1111 for a bucket operation owned by AWS account 2222-2222-2222. No IAM users are involved in this request.
> In this case, Amazon S3 evaluates the context as follows:
> 1. Because the request is made using root credentials of an AWS account, the user context is not evaluated.
> 2. In the bucket context, Amazon S3 examines the bucket policy. If the bucket owner (AWS account 2222-2222-2222) has not authorized AWS account 1111-1111-1111 to perform the requested operation, Amazon S3 denies the request. Otherwise, Amazon S3 grants the request and performs the operation.

Example 3:
> Bucket Operation Requested by an IAM User Whose Parent AWS Account Is Also the Bucket Owner
> In the example, the request is sent by Jill, an IAM user in AWS account 1111-1111-1111, which also owns the bucket.
> Amazon S3 performs the following context evaluation:
> 1. Because the request is from an IAM user, in the user context, Amazon S3 evaluates all policies that belong to the parent AWS account to determine if Jill has permission to perform the operation.
     In this example, parent AWS account 1111-1111-1111, to which the user belongs, is also the bucket owner. As a result, in addition to the user policy, Amazon S3 also evaluates the bucket policy and bucket ACL in the same context, because they belong to the same account.
> 2. Because Amazon S3 evaluated the bucket policy and bucket ACL as part of the user context, it does not evaluate the bucket context.

Example 4:
> Bucket Operation Requested by an IAM User Whose Parent AWS Account Is Not the Bucket Owner
> In this example, the request is sent by Jill, an IAM user whose parent AWS account is 1111-1111-1111, but the bucket is owned by another AWS account, 2222-2222-2222.
> Jill will need permissions from both the parent AWS account and the bucket owner. Amazon S3 evaluates the context as follows:
> 1. Because the request is from an IAM user, Amazon S3 evaluates the user context by reviewing the policies authored by the account to verify that Jill has the necessary permissions. If Jill has permission, then Amazon S3 moves on to evaluate the bucket context; if not, it denies the request.
> 2. In the bucket context, Amazon S3 verifies that bucket owner 2222-2222-2222 has granted Jill (or her parent AWS account) permission to perform the requested operation. If she has that permission, Amazon S3 grants the request and performs the operation; otherwise, Amazon S3 denies the request.

### Amazon如何对一个object操作请求进行认证
基于上面的请求认证过程, 具体到对object操作请求进行认证，当Amazon S3接收到一个object操作请求时，Amazon S3会先将所有相关的权限信息(包括资源层的权限信息(object access control list(ACL), bucket policy, bucket ACL)和IAM user策略)汇总为一个运行时的策略集。然后按照下述步骤逐步进行评估, 评估过程主要包括三个指定的上下文 -- user context, bucket context, object context。
> 1. User context – If the requester is an IAM user, the user must have permission from the parent AWS account to which it belongs. In this step, Amazon S3 evaluates a subset of policies owned by the parent account (also referred as the context authority). This subset of policies includes the user policy that the parent attaches to the user. If the parent also owns the resource in the request (bucket, object), Amazon S3 evaluates the corresponding resource policies (bucket policy, bucket ACL, and object ACL) at the same time.
Note: 若该user所属的AWS账户是所请求资源的拥有者，该账户可以选择使用user policy或者resource policy来为该IAM User授予该资源的访问权限，两种方式可选其一。
> 2. Bucket context – In this context, Amazon S3 evaluates policies owned by the AWS account that owns the bucket.
>    If the AWS account that owns the object in the request is not same as the bucket owner, in the bucket context Amazon S3 checks the policies if the bucket owner has explicitly denied access to the object. If there is an explicit deny set on the object, Amazon S3 does not authorize the request.
> 3. Object context – The requester must have permissions from the object owner to perform a specific object operation. In this step, Amazon S3 evaluates the object ACL.
Note: If bucket and object owners are the same, access to the object can be granted in the bucket policy, which is evaluated at the bucket context. If the owners are different, the object owners must use an object ACL to grant permissions. If the AWS account that owns the object is also the parent account to which the IAM user belongs, it can configure object permissions in a user policy, which is evaluated at the user context. 

下面是对object操作请求进行认证的例子：
Example 1:
> Object Operation Request
> In this example, IAM user Jill, whose parent AWS account is 1111-1111-1111, sends an object operation request (for example, Get object) for an object owned by AWS account 3333-3333-3333 in a bucket owned by AWS account 2222-2222-2222.
即IAM user所属的账户，object所属的账户，以及bucket所属的账户均不同
> Jill will need permission from the parent AWS account, the bucket owner, and the object owner. Amazon S3 evaluates the context as follows:
> 1. Because the request is from an IAM user, Amazon S3 evaluates the user context to verify that the parent AWS account 1111-1111-1111 has given Jill permission to perform the requested operation. If she has that permission, Amazon S3 evaluates the bucket context. Otherwise, Amazon S3 denies the request.
> 2. In the bucket context, the bucket owner, AWS account 2222-2222-2222, is the context authority. Amazon S3 evaluates the bucket policy to determine if the bucket owner has explicitly denied Jill access to the object.
> 3. In the object context, the context authority is AWS account 3333-3333-3333, the object owner. Amazon S3 evaluates the object ACL to determine if Jill has permission to access the object. If she does, Amazon S3 authorizes the request.

## 如何设置ACL
基于当前的使用场景，我们只关注通过REST API来管理ACLs。因为对于ACL来说，包括bucket ACL和object ACL，因此下面将针对bucket ACL和object ACL分别进行描述。
### 如何向ACL中添加授权信息
使用HTTP PUT操作请求来进行ACL信息添加。
#### 如何向bucket ACL中添加授权信息
你可以在请求体中指定相应的ACL，也可以在请求头中指定相应的访问权限，**但是不可以同时在请求头和请求体中指定访问权限**。
- 通过请求体来设置bucket ACL的权限信息
> PUT /?acl HTTP/1.1
> Host: BucketName.s3.amazonaws.com
> Date: date
> Authorization: authorization string (see Authenticating Requests (AWS Signature Version4))
>
> <AccessControlPolicy>
>   <Owner>
>     <ID>ID</ID>
>     <DisplayName>EmailAddress</DisplayName>
>   </Owner>
>   <AccessControlList>
>     <Grant>
>       <Grantee xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:type="CanonicalUser">
>         <ID>ID</ID>
>         <DisplayName>EmailAddress</DisplayName>
>       </Grantee>
>       <Permission>Permission</Permission>
>     </Grant>
>     ...
>   </AccessControlList>
> </AccessControlPolicy> 

Amazon S3支持一个预定义的ACLs集合，即canned ACLs。

- 通过请求头来设置bucket ACL的权限信息
当然你也可以通过请求头来发送请求，设置访问权限。例如：
> x-amz-grant-write: uri="http://acs.amazonaws.com/groups/s3/LogDelivery", emailAddress="xyz@amazon.com", emailAddress="abc@amazon.com"

Example:
Sample Request: Access permissions specified in the body

The following request grants access permission to the existing examplebucket bucket. The request specifies the ACL in the body. In addition to granting full control to the bucket owner, the XML specifies the following grants.

 - Grant AllUsers group READ permission on the bucket.
 - Grant the LogDelivery group WRITE permission on the bucket.
 - Grant an AWS account, identified by email address, WRITE_ACP permission.
 - Grant an AWS account, identified by canonical user ID, READ_ACP permission.

> PUT ?acl HTTP/1.1
> Host: examplebucket.s3.amazonaws.com
> Content-Length: 1660
> x-amz-date: Thu, 12 Apr 2012 20:04:21 GMT
> Authorization: authorization string
> 
> <AccessControlPolicy xmlns="http://s3.amazonaws.com/doc/2006-03-01/">
>   <Owner>
>     <ID>852b113e7a2f25102679df27bb0ae12b3f85be6BucketOwnerCanonicalUserID</ID>
>     <DisplayName>OwnerDisplayName</DisplayName>
>   </Owner>
>   <AccessControlList>
>     <Grant>
>       <Grantee xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:type="CanonicalUser">
>         <ID>852b113e7a2f25102679df27bb0ae12b3f85be6BucketOwnerCanonicalUserID</ID>
>         <DisplayName>OwnerDisplayName</DisplayName>
>       </Grantee>
>       <Permission>FULL_CONTROL</Permission>
>     </Grant>
>     <Grant>
>       <Grantee xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:type="Group">
>         <URI xmlns="">http://acs.amazonaws.com/groups/global/AllUsers</URI>
>       </Grantee>
>       <Permission xmlns="">READ</Permission>
>     </Grant>
>     <Grant>
>       <Grantee xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:type="Group">
>         <URI xmlns="">http://acs.amazonaws.com/groups/s3/LogDelivery</URI>
>       </Grantee>
>       <Permission xmlns="">WRITE</Permission>
>     </Grant>
>     <Grant>
>       <Grantee xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:type="AmazonCustomerByEmail">
>         <EmailAddress xmlns="">xyz@amazon.com</EmailAddress>
>       </Grantee>
>       <Permission xmlns="">WRITE_ACP</Permission>
>     </Grant>
>     <Grant>
>       <Grantee xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:type="CanonicalUser">
>         <ID xmlns="">f30716ab7115dcb44a5ef76e9d74b8e20567f63TestAccountCanonicalUserID</ID>
>       </Grantee>
>       <Permission xmlns="">READ_ACP</Permission>
>     </Grant>
>   </AccessControlList>
> </AccessControlPolicy>

sample response
> HTTP/1.1 200 OK
> x-amz-id-2: NxqO3PNiMHXXGwjgv15LLgUoAmPVmG0xtZw2sxePXLhpIvcyouXDrcQUaWWXcOK0
> x-amz-request-id: C651BC9B4E1BD401
> Date: Thu, 12 Apr 2012 20:04:28 GMT
> Content-Length: 0
> Server: AmazonS3

Sample Request: Access permissions specified using headers

The following request uses ACL-specific request headers to grant the following permissions:
Write permission to the Amazon S3 LogDelivery group and an AWS account identified by the email xyz@amazon.com.
Read permission to the Amazon S3 AllUsers group

> PUT ?acl HTTP/1.1
> Host: examplebucket.s3.amazonaws.com
> x-amz-date: Sun, 29 Apr 2012 22:00:57 GMT
> x-amz-grant-write: uri="http://acs.amazonaws.com/groups/s3/LogDelivery", emailAddress="xyz@amazon.com"
> x-amz-grant-read: uri="http://acs.amazonaws.com/groups/global/AllUsers"
> Accept: */*
> Authorization: authorization string

sample response
> HTTP/1.1 200 OK
> x-amz-id-2: 0w9iImt23VF9s6QofOTDzelF7mrryz7d04Mw23FQCi4O205Zw28Zn+d340/RytoQ
> x-amz-request-id: A6A8F01A38EC7138
> Date: Sun, 29 Apr 2012 22:01:10 GMT
> Content-Length: 0
> Server: AmazonS3

#### 如何向object ACL中添加授权信息
从整个流程和方法上，与bucket ACL基本相同，不过object ACL需要注意的是，object有一个version的概念，而object的ACL是在version级被设置的。默认情况下，PUT请求设置的是当前版本object的ACL，若要设置其他版本的ACL，需要使用versionID子资源。

- 通过请求体来设置object ACL的权限信息
> PUT /ObjectName?acl HTTP/1.1
> Host: BucketName.s3.amazonaws.com
> Date: date
> Authorization: authorization string (see Authenticating Requests (AWS Signature Version4))
> 
> <AccessControlPolicy>
>   <Owner>
>     <ID>ID</ID>
>     <DisplayName>EmailAddress</DisplayName>
>   </Owner>
>   <AccessControlList>
>     <Grant>
>       <Grantee xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:type="CanonicalUser">
>         <ID>ID</ID>
>         <DisplayName>EmailAddress</DisplayName>
>       </Grantee>
>       <Permission>Permission</Permission>
>     </Grant>
>     ...
>   </AccessControlList>
> </AccessControlPolicy> 

- 通过请求头来设置object ACL的权限信息
> x-amz-grant-read: emailAddress="xyz@amazon.com", emailAddress="abc@amazon.com"
此处需要针对指定versionid的情况进行特殊说明
> x-amz-version-id
需要用到上面这个参数来对versionid进行指定。

Example:
sample request body:
> PUT /my-image.jpg?acl HTTP/1.1
> Host: bucket.s3.amazonaws.com
> Date: Wed, 28 Oct 2009 22:32:00 GMT
> Authorization: authorization string
> Content-Length: 124
> 
> <AccessControlPolicy>
>   <Owner>
>     <ID>75aa57f09aa0c8caeab4f8c24e99d10f8e7faeebf76c078efc7c6caea54ba06a</ID>
>     <DisplayName>CustomersName@amazon.com</DisplayName>
>   </Owner>
>   <AccessControlList>
>     <Grant>
>       <Grantee xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:type="CanonicalUser">
>         <ID>75aa57f09aa0c8caeab4f8c24e99d10f8e7faeeExampleCanonicalUserID</ID>
>         <DisplayName>CustomerName@amazon.com</DisplayName>
>       </Grantee>
>       <Permission>FULL_CONTROL</Permission>
>     </Grant>
>   </AccessControlList>
> </AccessControlPolicy>

sample response:
> HTTP/1.1 200 OK
> x-amz-id-2: eftixk72aD6Ap51T9AS1ed4OpIszj7UDNEHGran
> x-amz-request-id: 318BC8BC148832E5
> x-amz-version-id: 3/L4kqtJlcpXrof3vjVBH40Nr8X8gdRQBpUMLUo
> Date: Wed, 28 Oct 2009 22:32:00 GMT
> Last-Modified: Sun, 1 Jan 2006 12:00:00 GMT
> Content-Length: 0
> Connection: close
> Server: AmazonS3

Sample Request: Setting the ACL of a specified object version
> PUT /my-image.jpg?acl&versionId=3HL4kqtJlcpXroDTDmJ+rmSpXd3dIbrHY+MTRCxf3vjVBH40Nrjfkd HTTP/1.1
> Host: bucket.s3.amazonaws.com
> Date: Wed, 28 Oct 2009 22:32:00 GMT
> Authorization: authorization string
> Content-Length: 124
>  
> <AccessControlPolicy>
>   <Owner>
>     <ID>75aa57f09aa0c8caeab4f8c24e99d10f8e7faeebf76c078efc7c6caea54ba06a</ID>
>     <DisplayName>mtd@amazon.com</DisplayName>
>   </Owner>
>   <AccessControlList>
>     <Grant>
>       <Grantee xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:type="CanonicalUser">
>         <ID>75aa57f09aa0c8caeab4f8c24e99d10f8e7faeebf76c078efc7c6caea54ba06a</ID>
>         <DisplayName>mtd@amazon.com</DisplayName>
>       </Grantee>
>       <Permission>FULL_CONTROL</Permission>
>     </Grant>
>   </AccessControlList>
> </AccessControlPolicy>

sample response
> HTTP/1.1 200 OK
> x-amz-id-2: eftixk72aD6Ap51u8yU9AS1ed4OpIszj7UDNEHGran
> x-amz-request-id: 318BC8BC148832E5
> x-amz-version-id: 3/L4kqtJlcpXro3vjVBH40Nr8X8gdRQBpUMLUo
> Date: Wed, 28 Oct 2009 22:32:00 GMT
> Last-Modified: Sun, 1 Jan 2006 12:00:00 GMT
> Content-Length: 0
> Connection: close
> Server: AmazonS3

Sample Request: Access permissions specified using headers
> PUT ExampleObject.txt?acl HTTP/1.1
> Host: examplebucket.s3.amazonaws.com
> x-amz-acl: public-read
> Accept: */*
> Authorization: authorization string
> Host: s3.amazonaws.com
> Connection: Keep-Alive

sample response
> HTTP/1.1 200 OK
> x-amz-id-2: w5YegkbG6ZDsje4WK56RWPxNQHIQ0CjrjyRVFZhEJI9E3kbabXnBO9w5G7Dmxsgk
> x-amz-request-id: C13B2827BD8455B1
> Date: Sun, 29 Apr 2012 23:24:12 GMT
> Content-Length: 0
> Server: AmazonS3

### 如何获取当前的ACL信息
使用GET操作来获取当前ACL信息
#### 如何获取当前bucket ACL中的授权信息
要通过GET获取bucket的ACL信息，你必须对该bucket有READ_ACP读权限。
请求语法：
> GET /?acl HTTP/1.1
> Host: BucketName.s3.amazonaws.com
> Date: date
> Authorization: authorization string (see Authenticating Requests (AWS Signature Version4))

Example:
Sample Request
> GET ?acl HTTP/1.1
> Host: bucket.s3.amazonaws.com
> Date: Wed, 28 Oct 2009 22:32:00 GMT
> Authorization: authorization string

Sample Response
> HTTP/1.1 200 OK
> x-amz-id-2: eftixk72aD6Ap51TnqcoF8eFidJG9Z/2mkiDFu8yU9AS1ed4OpIszj7UDNEHGran
> x-amz-request-id: 318BC8BC148832E5
> Date: Wed, 28 Oct 2009 22:32:00 GMT
> Last-Modified: Sun, 1 Jan 2006 12:00:00 GMT
> Content-Length: 124
> Content-Type: text/plain
> Connection: close
> Server: AmazonS3
> 
> <AccessControlPolicy>
>   <Owner>
>     <ID>75aa57f09aa0c8caeab4f8c24e99d10f8e7faeebf76c078efc7c6caea54ba06a</ID>
>     <DisplayName>CustomersName@amazon.com</DisplayName>
>   </Owner>
>   <AccessControlList>
>     <Grant>
>       <Grantee xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
>             xsi:type="CanonicalUser">
>         <ID>75aa57f09aa0c8caeab4f8c24e99d10f8e7faeebf76c078efc7c6caea54ba06a</ID>
>         <DisplayName>CustomersName@amazon.com</DisplayName>
>       </Grantee>
>       <Permission>FULL_CONTROL</Permission>
>     </Grant>
>   </AccessControlList>
> </AccessControlPolicy> 


#### 如何获取当前object ACL中的授权信息
操作桶bucket ACL，需要特别注意的是上面提到过的versionid的区别。
请求语法
> GET /ObjectName?acl HTTP/1.1
> Host: BucketName.s3.amazonaws.com
> Date: date
> Authorization: authorization string (see Authenticating Requests (AWS Signature Version4))
> Range:bytes=byte_range

Example:
Sample Request
> GET /my-image.jpg?acl HTTP/1.1
> Host: bucket.s3.amazonaws.com
> Date: Wed, 28 Oct 2009 22:32:00 GMT
> Authorization: authorization string

Sample Response
> HTTP/1.1 200 OK
> x-amz-id-2: eftixk72aD6Ap51TnqcoF8eFidJG9Z/2mkiDFu8yU9AS1ed4OpIszj7UDNEHGran
> x-amz-request-id: 318BC8BC148832E5
> x-amz-version-id: 4HL4kqtJlcpXroDTDmJ+rmSpXd3dIbrHY+MTRCxf3vjVBH40Nrjfkd
> Date: Wed, 28 Oct 2009 22:32:00 GMT
> Last-Modified: Sun, 1 Jan 2006 12:00:00 GMT
> Content-Length: 124
> Content-Type: text/plain
> Connection: close
> Server: AmazonS3
>  
> <AccessControlPolicy>
>   <Owner>
>     <ID>75aa57f09aa0c8caeab4f8c24e99d10f8e7faeebf76c078efc7c6caea54ba06a</ID>
>     <DisplayName>mtd@amazon.com</DisplayName>
>   </Owner>
>   <AccessControlList>
>     <Grant>
>       <Grantee xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:type="CanonicalUser">
>         <ID>75aa57f09aa0c8caeab4f8c24e99d10f8e7faeebf76c078efc7c6caea54ba06a</ID>
>         <DisplayName>mtd@amazon.com</DisplayName>
>       </Grantee>
>       <Permission>FULL_CONTROL</Permission>
>     </Grant>
>   </AccessControlList>
> </AccessControlPolicy>

Sample Request Getting the ACL of the Specific Version of an Object
> GET /my-image.jpg?versionId=3/L4kqtJlcpXroDVBH40Nr8X8gdRQBpUMLUo&acl HTTP/1.1
> Host: bucket.s3.amazonaws.com
> Date: Wed, 28 Oct 2009 22:32:00 GMT
> Authorization: authorization string

Sample Response Showing the ACL of the Specific Version
> HTTP/1.1 200 OK
> x-amz-id-2: eftixk72aD6Ap51TnqcoF8eFidJG9Z/2mkiDFu8yU9AS1ed4OpIszj7UDNEHGran
> x-amz-request-id: 318BC8BC148832E5
> Date: Wed, 28 Oct 2009 22:32:00 GMT
> Last-Modified: Sun, 1 Jan 2006 12:00:00 GMT
> x-amz-version-id: 3/L4kqtJlcpXroDTDmJ+rmSpXd3dIbrHY+MTRCxf3vjVBH40Nr8X8gdRQBpUMLUo
> Content-Length: 124
> Content-Type: text/plain
> Connection: close
> Server: AmazonS3
>  
> <AccessControlPolicy>
>   <Owner>
>     <ID>75aa57f09aa0c8caeab4f8c24e99d10f8e7faeebf76c078efc7c6caea54ba06a</ID>
>     <DisplayName>mdtd@amazon.com</DisplayName>
>   </Owner>
>   <AccessControlList>
>     <Grant>
>       <Grantee xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:type="CanonicalUser">
>         <ID>75aa57f09aa0c8caeab4f8c24e99d10f8e7faeebf76c078efc7c6caea54ba06a</ID>
>         <DisplayName>mdtd@amazon.com</DisplayName>
>       </Grantee>
>       <Permission>FULL_CONTROL</Permission>
>     </Grant>
>   </AccessControlList>
> </AccessControlPolicy>

### 如何从ACL中删除授权信息
通过PUT操作重写ACL中已有的授权信息，将允许的操作设置为空，即无任何操作权限被设置，即可达到删除授权信息的目的。

## Ceph RGW中当前ACL的支持情况
