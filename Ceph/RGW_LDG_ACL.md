## 当前RGW ACL实现
RGWOp \*RGWHandler_REST_Bucket_S3::op_get()\
RGWOp \*RGWHandler_REST_Bucket_S3::op_head()\
RGWOp \*RGWHandler_REST_Obj_S3::get_obj_op(bool get_data)\
RGWOp \*RGWHandler_REST_Obj_S3::op_get()\
RGWOp \*RGWHandler_REST_Obj_S3::op_head()\
这些函数中都可能会new RGWGetACLs_ObjStore_S3

RGWOp \*RGWHandler_REST_Bucket_S3::op_put()\
RGWOp \*RGWHandler_REST_Obj_S3::op_put()\
这些函数中都可能会new RGWPutACLs_ObjStore_S3

当前RGW中有三个ACL的概念：
1. user_acl     -->  std::unique_ptr<RGWAccessControlPolicy>
2. bucket_acl   -->  RGWAccessControlPolicy*
3. object_acl   -->  RGWAccessControlPolicy*

RGW中ACL是以bucket和object的属性的形式存在的

RGWAccessControlPolicy类是对ACL操作的抽象封装，其内包含RGWAccessControlList acl成员变量，这个实际上可以看成是与真正意义的AWS ACL对应的。
外部操作中使用的req_state::user_acl、req_state::bucket_acl、req_state::object_acl实际上都是RGWAccessControlPolicy类的实例。

ACL相关类的继承关系如下：\
RGWOp\
\-&ensp;RGWGetACLs&emsp;&emsp;\\\ rgw_op.h\
\-&ensp;RGWPutACLs&emsp;&emsp;\\\ rgw_op.h\
&emsp;\-&ensp;RGWGetACLs_ObjStore&emsp;&emsp;\\\ rgw_rest.h\
&emsp;\-&ensp;RGWPutACLs_ObjStore&emsp;&emsp;\\\ rgw_rest.h\
&emsp;&emsp;\-&ensp;RGWGetACLs_ObjStore_S3&emsp;&emsp;\\\ rgw_rest_s3.h\
&emsp;&emsp;\-&ensp;RGWPutACLs_ObjStore_S3&emsp;&emsp;\\\ rgw_rest_s3.h

还是从程序处理HTTP请求的入口点process_request函数看起:\
process_request  &emsp;&emsp;&emsp;\\\ rgw_process.cc\
\-&ensp;get_handler()\
&emsp;\-&ensp;preprocess()\
&emsp;\-&ensp;get_manager()\
&emsp;&emsp;\-&ensp;get_resource_mgr()\
&emsp;&emsp;&emsp;\-&ensp;get_resource_mgr_as_default()\
/\*在main函数中，只要配置文件中的enable_api配置了"s3"，基本上都会执行register_default_mgr注册默认的manager，而对于S3来说，为RGWRESTMgr_S3, 因此对于S3来说，此处get_manager会调用get_resource_mgr_as_default，在该函数中则会调用get_resource_mgr_as_default获取default的mgr，对S3来说即为RGWRESTMgr_S3*/\
&emsp;\-&ensp;handler = mgr->get_handler()&emsp;\\\对S3来说实际上就是调用的RGWRESTMgr_S3的get_handler函数
&emsp;&emsp;\-&ensp;会根据url来分别返回RGWHandler_REST_Service_S3(bucket为空)；RGWHandler_REST_Bucket_S3(object为空)；RGWHandler_REST_Obj_S3
  \-&ensp;handler->init()\
\-&ensp;handler->get_op()&emsp;&emsp;&emsp;\\\实际上会调用RGWHandler_REST::get_op()\
&emsp;\-&ensp;会根据s->op来执行op_get、op_put等操作，即调用RGWHandler_REST_Bucket_S3等继承类自身的op_get等函数\
Note: 这里需要特别说明一下，根据AWS S3，ACL是资源级的访问策略，即只是针对bucket和object的，所以对应的只存在bucket ACL和objec ACLL，所以对于ACL来说，我们实际上只需要关心RGWHandler_REST_Bucket_S3和RGWHandler_REST_Obj_S3\
&emsp;\-&ensp;根据上面已经列出来的，在RGWHandler_REST_Obj_S3和RGWHandler_REST_Bucket_S3的响应的op_*处理函数中会对是否是ACL操作请求进行判断，若是则会创建RGWGetACLs_ObjStore_S3和RGWPutACLs_ObjStore_S3等实例\
&emsp;\-&ensp;op->init()&emsp;&emsp;\\\实际上会调用RGWOp::init()\
&emsp;&emsp;\-&ensp;RGWOp::init()&emsp;&emsp;\\\在该函数中会对Rados \*store，req_state, RGWHandler等赋值\
&emsp;&emsp;&emsp;\-&ensp;其中RGWHandler参数会被赋值为op所对应的handler，在ACL处理过程中，即对应为RGWHandler_REST_Obj_S3和RGWHandler_REST_Bucket_S3。因此实际上op中存在指向其所属的Handler的成员变量。
\-&ensp;verify_requester()\
&emsp;\-&ensp;RGWOp::verify_requester&emsp;&emsp;\\\实际上调用的是RGWOp::verify_requester\
&emsp;&emsp;\-&ensp;handler->authorize()\
&emsp;&emsp;&emsp;\-&ensp;RGWHandler_REST_S3::authorize()\
&emsp;&emsp;&emsp;&emsp;\-&ensp;RGW_Auth_S3::authorize()&emsp;&emsp;\\\此处实际上就是开始进行S3V2或者S3V4认证\
\-&ensp;postauth_init()\
&emsp;\-&ensp;主要进行tenant name、bucket name、object name有效性的检查\
\-&ensp;rgw_process_authenticated()\
&emsp;\-&ensp;handler->init_permissions(op)&emsp;&emsp;\\\对于ACL的处理，实际上调用的是RGWHandler_REST_Bucket_S3或者RGWHandler_REST_Obj_S3的init_permission，最终调用的是RGWHandler_REST::init_permissions\
&emsp;&emsp;\-&ensp;do_init_permission()&emsp;&emsp;\\\rgw_op.cc\
&emsp;&emsp;&emsp;\-&ensp;rgw_build_bucket_policies&emsp;&emsp;\\\rgw_op.cc\
&emsp;&emsp;&emsp;&emsp;\-&ensp;会在此实例化相应的AccessControlPolicy类，对于s3来说，会实例化RGWAccessControlPolicy_S3类，并将该实例赋值给s->bucket_acl；而对于swift类型的用户来说，会实例化RGWAccessControlPolicy_SWIFTAcct赋值给s->user_acl，实例化RGWAccessControlPolicy_SWIFT赋值给s->bucket_acl。\
&emsp;\-&ensp;handler->read_permissions(op)\
&emsp;&emsp;\-&ensp;do_read_permissions\
&emsp;&emsp;&emsp;\-&ensp;rgw_build_object_policies&emsp;&emsp;\\\因为bucket的permission信息已经读取了，所以此处只需要读取obj的permission信息即可，会实例化RGWAccessControlPolicy，并赋值给s->object_acl\
&emsp;\-&ensp;op->init_processing()&emsp;&emsp;\\\ RGWOp::init_processing()\
&emsp;&emsp;\-&ensp;init_quota()\
&emsp;\-&ensp;op->verify_op_mask()&emsp;&emsp;\\\ RGWOp::verify_op_mask()\
&emsp;\-&ensp;op->verify_permission()&emsp;&emsp;\\\ RGWPutACLs::verify_permission()\
&emsp;&emsp;\-&ensp;此处会根据请求操作的是bucket还是object而分别调用verify_bucket_permission和verify_object_permission来对请求者的WRITE_ACP权限进行验证。\
&emsp;\-&ensp;op->verify_params()\
&emsp;\-&ensp;op->pre_exec()\
&emsp;\-&ensp;op->execute()&emsp;&emsp;\\\ACLOp的处理部分就是在此处调用的\
&emsp;\-&ensp;op->complete()&emsp;&emsp;\\\RGWOp::complete()\
&emsp;&emsp;\-&ensp;send_response()&emsp;&emsp;\\\对ACL来说，会调用具体的ACL op子类\

## 对bucket或object操作请求的工作流
process_request
\-&ensp;get_handler()\
&emsp;\-&ensp;preprocess()\
&emsp;\-&ensp;get_manager()\
&emsp;&emsp;\-&ensp;get_resource_mgr()\
&emsp;&emsp;&emsp;\-&ensp;get_resource_mgr_as_default()\
/\*在main函数中，只要配置文件中的enable_api配置了"s3"，基本上都会执行register_default_mgr注册默认的manager，而对于S3来说，为RGWRESTMgr_S3, 因此对于S3来说，此处get_manager会调用get_resource_mgr_as_default，在该函数中则会调用get_resource_mgr_as_default获取default的mgr，对S3来说即为RGWRESTMgr_S3*/\
&emsp;\-&ensp;handler = mgr->get_handler()&emsp;\\\对S3来说实际上就是调用的RGWRESTMgr_S3的get_handler函数
&emsp;&emsp;\-&ensp;会根据url来分别返回RGWHandler_REST_Service_S3(bucket为空)；RGWHandler_REST_Bucket_S3(object为空)；RGWHandler_REST_Obj_S3
  \-&ensp;handler->init()\
\-&ensp;handler->get_op()&emsp;&emsp;&emsp;\\\实际上会调用RGWHandler_REST::get_op()\
&emsp;\-&ensp;会根据s->op来执行op_get、op_put等操作，即调用RGWHandler_REST_Bucket_S3等继承类自身的op_get等函数，对于上传Object来说，实际上就是返回RGWPutObj_ObjStore_S3\
&emsp;\-&ensp;op->init()\
\-&ensp;verify_requester()\
&emsp;\-&ensp;RGWOp::verify_requester&emsp;&emsp;\\\实际上调用的是RGWOp::verify_requester\
&emsp;&emsp;\-&ensp;handler->authorize()\
&emsp;&emsp;&emsp;\-&ensp;RGWHandler_REST_S3::authorize()\
&emsp;&emsp;&emsp;&emsp;\-&ensp;RGW_Auth_S3::authorize()&emsp;&emsp;\\\此处实际上就是开始进行S3V2或者S3V4认证\
\-&ensp;postauth_init()\
&emsp;\-&ensp;主要进行tenant name、bucket name、object name有效性的检查\
\-&ensp;rgw_process_authenticated()\
&emsp;\-&ensp;handler->init_permissions(op)&emsp;&emsp;\\\对于ACL的处理，实际上调用的是RGWHandler_REST_Bucket_S3或者RGWHandler_REST_Obj_S3的init_permission，最终调用的是RGWHandler_REST::init_permissions\
&emsp;&emsp;\-&ensp;do_init_permission()&emsp;&emsp;\\\rgw_op.cc\
&emsp;&emsp;&emsp;\-&ensp;rgw_build_bucket_policies\
&emsp;\-&ensp;handler->read_permissions(op)\
&emsp;&emsp;\-&ensp;do_read_permissions\
&emsp;&emsp;&emsp;\-&ensp;rgw_build_object_policies\
&emsp;\-&ensp;op->init_processing()&emsp;&emsp;\\\ RGWOp::init_processing()\
&emsp;\-&ensp;op->verify_op_mask()&emsp;&emsp;\\\ RGWOp::verify_op_mask()\
&emsp;\-&ensp;op->verify_permission()&emsp;&emsp;\\\ RGWPutObj::verify_permission() 对于该请求的权限验证就是在此处完成的，即CL验证是在此处完成的\
&emsp;\-&ensp;op->verify_params()\
&emsp;\-&ensp;op->pre_exec()\
&emsp;\-&ensp;op->execute()&emsp;&emsp;\\\实际上执行的是RGWPutObj::execute()\
&emsp;\-&ensp;op->complete()&emsp;&emsp;\\\RGWOp::complete()\
&emsp;&emsp;\-&ensp;send_response()&emsp;&emsp;\\\对ACL来说，会调用具体的ACL op子类


## 引入LDG ACL后的处理流程
本部分着重从用户角度和程序实现角度出发进行分析
1. 在启用BL的过程中，需要使能source bucket的BL功能，使能的过程中需要指定target bucket，按照AWS S3的处理，会对LDG进行判断，若target bucket当前未赋予LDG写权限，则使能source bucket的BL功能的处理失败，会返回400 error code，错误内容为：You must give the log-delivery group WRITE and READ_ACP permissions to the target bucket。即在为source bucket指定target bucket前，会对LDG当前对于target bucket的操作权限进行检测。

RGW: 对应到我们的RGW，需要在用户使能source bucket的BL功能时，对bl_deliver用户对target bucket的操作权限进行检测。

2. 上面提到的，要先为LDG赋予target bucket的写权限

RGW：对应到RGW，需要能够为bl_deliver赋予target bucket的写权限。
即在RGWPutACLs_ObjStore_S3::execute()中，需要能够处理bl_deliver，能够为bl_deliver赋予target bucket的写权限。

3. 当后台通过LDG向targetbucket中写access log时，会通过bucket ACL对LDG的权限进行验证。

RGW：当前的实现即支持，只需要关注对bl_deliver进行权限验证的处理过程与其他用户是否有区别即可。
当前在RGWPutObj::verify_permission中，若user是bl_deliver，则直接返回0。其实按照逻辑来说，若是能够成功使能source bucket，则LDG一定具备target bucket的写权限，所以此处保留现有处理，直接返回0也是可以的。

![BL使能过程](https://raw.githubusercontent.com/ZVampirEM77/ChosenOne/master/Image/RGW_LDG_ACL_1.png)

## 为实现RGW LDG ACL需要添加处理的部分
RGW中预定义的Group:\
// rgw_acl.h
>enum ACLGroupTypeEnum {\
/* numbers are encoded should not change */\
    ACL_GROUP_NONE                = 0,\
    ACL_GROUP_ALL_USERS           = 1,\
    ACL_GROUP_AUTHENTICATED_USERS = 2,\
};

// rgw_acl_s3.h
>#define RGW_URI_ALL_USERS       "http://acs.amazonaws.com/groups/global/AllUsers"\
>#define RGW_URI_AUTH_USERS      "http://acs.amazonaws.com/groups/global/AuthenticatedUsers"

因为LDG对于实现BL功能是必需的，所以此处需要添加LDG的定义

## 当前存在的问题
1. RGW中存在Key ACL？对应到AWS S3中的什么？

2. recurse是什么模式？

3. bl_deliver是否需要对log obj有可读权限呢？

4. 因为当前在PutObj和GetObj操作时，若是bl_deliver，则直接返回0，不会通过ACL对权限加以验证，所以bldeliver可以向任何bucket中写入obj，后面这部分肯定要结合上面第3个问题进行修改的。
