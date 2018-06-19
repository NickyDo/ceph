#CEPH FILESYSTEM

Ceph Filesystem (Ceph FS) là một POSIX-compliant filesystem sử dụng Ceph Storage Cluster để lưu data. Ceph filesystem sử dụng Ceph Storage Cluster giống như Ceph Block Devices, Ceph Object Storage with its S3 and Swift APIs, or native bindings (librados).

![CEPHFS](https://camo.githubusercontent.com/2c0f5bb6588255cba3e73160546e611b854672ad/687474703a2f2f692e696d6775722e636f6d2f745762556f64342e706e67)

Để dùng Ceph FS ta cần có Ceph Metadata Server.

Libcephfs libraries hỗ trợ các client sử dụng mount command. Hỗ trợ CIFS và SMB.