1. Ceph中的上下文的概念到底是怎样一种概念？

2. Ceph中的quota是个什么概念？

3. Swift类型账户设置ACL时，为什么要同时设置user acl和bucket ucl，而S3类型的只需设置bucket acl?

4. 为什么一次bucket或者object的操作请求要经过S3V2或者S3V4认证，同时还要通过ACL做权限验证？\
A：因为S3V2或S3V4是对该账户是否为AWS S3的账户进行验证，即这个人到底是不是我家的。而通过ACL做权限验证则主要是针对该账户对所访问的bucket或object是否有相应的操作权限，即这个人到底能不能动家里的这个东西。

5. RGW中请求处理时，会从头到尾使用到一个req_state的变量，这是做什么的？\
A：req_state变量是与每个请求一一对应的，即当接收到一个操作请求时，就会在处理函数process_request中创建一个req_state结构体的变量，这个变量是用来保存该请求的一系列状态信息的。即是该请求涉及到的资源在内存中的一个反映。

6. AWS S3中READ和READ_ACP，以及WRITE和WRITE_ACP有什么区别？
