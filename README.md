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

Call the gRPC service using grpcurl and notice the `set-cookie` header
```
grpcurl -v -plaintext -proto whereami.proto $GATEWAY_VIP:80 whereami.Whereami.GetPayload 
```

Output:
```
$ grpcurl -v -plaintext -proto whereami.proto $GATEWAY_VIP:80 whereami.Whereami.GetPayload 

Resolved method descriptor:
// Send host name and get other metadata back
rpc GetPayload ( .whereami.Empty ) returns ( .whereami.WhereamiReply );

Request metadata to send:
(empty)

Response headers received:
content-type: application/grpc
date: Mon, 27 Oct 2025 00:44:04 GMT
grpc-accept-encoding: identity, deflate, gzip
set-cookie: GCILB="e5cf7b28d9d331a4"; Max-Age=3600; Path=/; HttpOnly
via: 1.1 google

Response contents:
{
  "clusterName": "edge-to-mesh-01",
  "metadata": "grpc-frontend",
  "nodeName": "gk3-edge-to-mesh-01-nap-1c08r7is-f94d8802-jkrd",
  "podIp": "10.54.2.154",
  "podName": "whereami-grpc-75b6c67769-mpf5n",
  "podNameEmoji": "🧎🏿‍♀️‍➡",
  "podNamespace": "grpc-cookie",
  "podServiceAccount": "whereami-grpc",
  "projectId": "e2m-private-test-01",
  "timestamp": "2025-10-27T00:44:04",
  "zone": "us-central1-c",
  "gceInstanceId": "4528778889087650357",
  "gceServiceAccount": "e2m-private-test-01.svc.id.goog"
}

Response trailers received:
(empty)
Sent 0 requests and received 1 response
```

Now try the requests again with that header to verify that the same backend pod is used:
```
grpcurl -v -plaintext -proto whereami.proto -H 'Cookie: GCILB="e5cf7b28d9d331a4"' $GATEWAY_VIP:80 whereami.Whereami.GetPayload 
```

It works! 

```
admin@instance-20251026-225348:~$ grpcurl -v -plaintext -proto whereami.proto -H 'Cookie: GCILB="e5cf7b28d9d331a4"' $GATEWAY_VIP:80 whereami.Whereami.GetPayload 

Resolved method descriptor:
// Send host name and get other metadata back
rpc GetPayload ( .whereami.Empty ) returns ( .whereami.WhereamiReply );

Request metadata to send:
cookie: GCILB="e5cf7b28d9d331a4"

Response headers received:
content-type: application/grpc
date: Mon, 27 Oct 2025 01:17:14 GMT
grpc-accept-encoding: identity, deflate, gzip
via: 1.1 google

Response contents:
{
  "clusterName": "edge-to-mesh-01",
  "metadata": "grpc-frontend",
  "nodeName": "gk3-edge-to-mesh-01-nap-1c08r7is-f94d8802-jkrd",
  "podIp": "10.54.2.154",
  "podName": "whereami-grpc-75b6c67769-mpf5n",
  "podNameEmoji": "🧎🏿‍♀️‍➡",
  "podNamespace": "grpc-cookie",
  "podServiceAccount": "whereami-grpc",
  "projectId": "e2m-private-test-01",
  "timestamp": "2025-10-27T01:17:14",
  "zone": "us-central1-c",
  "gceInstanceId": "4528778889087650357",
  "gceServiceAccount": "e2m-private-test-01.svc.id.goog"
}

Response trailers received:
(empty)
Sent 0 requests and received 1 response
admin@instance-20251026-225348:~$ grpcurl -v -plaintext -proto whereami.proto -H 'Cookie: GCILB="e5cf7b28d9d331a4"' $GATEWAY_VIP:80 whereami.Whereami.GetPayload 

Resolved method descriptor:
// Send host name and get other metadata back
rpc GetPayload ( .whereami.Empty ) returns ( .whereami.WhereamiReply );

Request metadata to send:
cookie: GCILB="e5cf7b28d9d331a4"

Response headers received:
content-type: application/grpc
date: Mon, 27 Oct 2025 01:17:15 GMT
grpc-accept-encoding: identity, deflate, gzip
via: 1.1 google

Response contents:
{
  "clusterName": "edge-to-mesh-01",
  "metadata": "grpc-frontend",
  "nodeName": "gk3-edge-to-mesh-01-nap-1c08r7is-f94d8802-jkrd",
  "podIp": "10.54.2.154",
  "podName": "whereami-grpc-75b6c67769-mpf5n",
  "podNameEmoji": "🧎🏿‍♀️‍➡",
  "podNamespace": "grpc-cookie",
  "podServiceAccount": "whereami-grpc",
  "projectId": "e2m-private-test-01",
  "timestamp": "2025-10-27T01:17:15",
  "zone": "us-central1-c",
  "gceInstanceId": "4528778889087650357",
  "gceServiceAccount": "e2m-private-test-01.svc.id.goog"
}

Response trailers received:
(empty)
Sent 0 requests and received 1 response
admin@instance-20251026-225348:~$ grpcurl -v -plaintext -proto whereami.proto -H 'Cookie: GCILB="e5cf7b28d9d331a4"' $GATEWAY_VIP:80 whereami.Whereami.GetPayload 

Resolved method descriptor:
// Send host name and get other metadata back
rpc GetPayload ( .whereami.Empty ) returns ( .whereami.WhereamiReply );

Request metadata to send:
cookie: GCILB="e5cf7b28d9d331a4"

Response headers received:
content-type: application/grpc
date: Mon, 27 Oct 2025 01:17:16 GMT
grpc-accept-encoding: identity, deflate, gzip
via: 1.1 google

Response contents:
{
  "clusterName": "edge-to-mesh-01",
  "metadata": "grpc-frontend",
  "nodeName": "gk3-edge-to-mesh-01-nap-1c08r7is-f94d8802-jkrd",
  "podIp": "10.54.2.154",
  "podName": "whereami-grpc-75b6c67769-mpf5n",
  "podNameEmoji": "🧎🏿‍♀️‍➡",
  "podNamespace": "grpc-cookie",
  "podServiceAccount": "whereami-grpc",
  "projectId": "e2m-private-test-01",
  "timestamp": "2025-10-27T01:17:16",
  "zone": "us-central1-c",
  "gceInstanceId": "4528778889087650357",
  "gceServiceAccount": "e2m-private-test-01.svc.id.goog"
}

Response trailers received:
(empty)
Sent 0 requests and received 1 response
```