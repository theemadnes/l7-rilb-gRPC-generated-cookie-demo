# l7-rilb-gRPC-generated-cookie-demo
Testing out using generated cookie for session affinity using an L7 RILB for a gRPC service

### usage

deploy gRPC app and namespace
```
kubectl apply -f whereami-grpc
```

deploy Gateway API resources
```
kubectl apply -f gateway-resources
```

find the VIP of the LB
```
export GATEWAY_VIP=$(kubectl get gateway internal-http -n grpc-cookie -o jsonpath='{.status.addresses[0].value}')
echo $GATEWAY_VIP # now you have the VIP of the internal LB
```

set up VM in same VPC to test:
- install grpcurl
- copy over proto file 

example:
```
# install binary
curl -sSL "https://github.com/fullstorydev/grpcurl/releases/download/v1.8.7/grpcurl_1.8.7_linux_x86_64.tar.gz" | sudo tar -xz -C /usr/local/bin

cat <<EOF > whereami.proto
syntax = "proto3";

package whereami;

//import "google/protobuf/struct.proto";

// The whereami service definition.
service Whereami {
  // Send host name and get other metadata back
  rpc GetPayload (Empty) returns (WhereamiReply) {}
}

// expecting an empty request from client
message Empty {

}

// The response message containing the metadata
// Removed any of the 'header' content since we're using gRPC
message WhereamiReply {
    WhereamiReply backend_result = 1;
    string cluster_name = 2;
    string metadata = 3;
    string node_name = 4;
    string pod_ip = 5;
    string pod_name = 6;
    string pod_name_emoji = 7;
    string pod_namespace = 8;
    string pod_service_account = 9;
    string project_id = 10;
    string timestamp = 11;
    string zone = 12;
    string gce_instance_id = 13;
    string gce_service_account = 14;
}
EOF

# replace with your value from earlier
export GATEWAY_VIP="10.128.0.33"
```

Call the gRPC service using grpcurl
```
grpcurl -plaintext -proto whereami.proto $GATEWAY_VIP:80 whereami.Whereami.GetPayload
```